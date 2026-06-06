# Runner, Events, Result Resolver Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现 LaTeXTransPlus 外部命令执行、JSON Lines 事件解析和每个 project 的 PDF 定位策略。

**Architecture:** `runner.ts` 只依赖一个小型 `ProcessExecutor` 接口，真实 Zotero 进程调用封装在 adapter 中，测试使用 mock executor。`resultResolver.ts` 通过 `ResultFileSystem` 接口读取文件状态，先覆盖纯策略，再接入 Zotero runtime 的文件 API。

**Tech Stack:** TypeScript、Mocha/Chai、Zotero `Subprocess` 或等价进程 API、LaTeXTransPlus JSON Lines events。

---

## Task 1: JSON Lines 事件解析

**Files:**
- Create: `src/latextrans/events.ts`
- Test: `test/latextrans/events.test.ts`

- [ ] **Step 1: 写事件解析测试**

创建 `test/latextrans/events.test.ts`：

```ts
import { assert } from "chai";
import {
  parseLatextransJsonLine,
  collectProjectResults,
} from "../../src/latextrans/events";

describe("latextrans events", function () {
  it("parses supported schema version 1 events", function () {
    const parsed = parseLatextransJsonLine(
      '{"schema_version":1,"type":"project_complete","timestamp":"2026-06-06T00:00:00Z","project_name":"2508.18791","pdf_path":"outputs/2508.18791/ch_2508.18791.pdf","log_path":"outputs/2508.18791/latextrans.log"}',
    );

    assert.deepEqual(parsed, {
      ok: true,
      event: {
        schema_version: 1,
        type: "project_complete",
        timestamp: "2026-06-06T00:00:00Z",
        project_name: "2508.18791",
        pdf_path: "outputs/2508.18791/ch_2508.18791.pdf",
        log_path: "outputs/2508.18791/latextrans.log",
      },
    });
  });

  it("keeps non-json lines as diagnostics", function () {
    assert.deepEqual(parseLatextransJsonLine("plain stdout"), {
      ok: false,
      reason: "invalid-json",
      rawLine: "plain stdout",
    });
  });

  it("collects latest project complete and error events by project name", function () {
    const results = collectProjectResults([
      {
        schema_version: 1,
        type: "project_complete",
        timestamp: "2026-06-06T00:00:00Z",
        project_name: "2508.18791",
        pdf_path: "a.pdf",
      },
      {
        schema_version: 1,
        type: "project_error",
        timestamp: "2026-06-06T00:00:01Z",
        project_name: "2508.18792",
        error: "compile failed",
        log_path: "latextrans.log",
      },
    ]);

    assert.equal(results.get("2508.18791")?.type, "project_complete");
    assert.equal(results.get("2508.18792")?.type, "project_error");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```powershell
npm run test
```

Expected: FAIL，错误包含 `Cannot find module '../../src/latextrans/events'`。

- [ ] **Step 3: 实现事件类型与解析器**

创建 `src/latextrans/events.ts`：

```ts
export type LatextransEventType =
  | "run_start"
  | "project_start"
  | "project_complete"
  | "project_error"
  | "run_complete"
  | string;

export interface LatextransEvent {
  schema_version: number;
  type: LatextransEventType;
  timestamp: string;
  total?: number;
  index?: number;
  project_name?: string;
  project_dir?: string;
  output_dir?: string;
  pdf_path?: string;
  log_path?: string;
  errors_report_path?: string;
  validation_summary?: unknown;
  error?: string;
  ok?: boolean;
  status?: string;
}

export type EventParseResult =
  | { ok: true; event: LatextransEvent }
  | { ok: false; reason: "empty" | "invalid-json" | "missing-required" | "unsupported-schema"; rawLine: string };

/**
 * 将 stdout 的单行 JSON Lines 转为插件内部事件。
 */
export function parseLatextransJsonLine(line: string): EventParseResult {
  const rawLine = line;
  const trimmed = line.trim();
  if (!trimmed) {
    return { ok: false, reason: "empty", rawLine };
  }

  let value: unknown;
  try {
    value = JSON.parse(trimmed);
  } catch {
    return { ok: false, reason: "invalid-json", rawLine };
  }

  if (!isEventShape(value)) {
    return { ok: false, reason: "missing-required", rawLine };
  }

  if (value.schema_version !== 1) {
    return { ok: false, reason: "unsupported-schema", rawLine };
  }

  return { ok: true, event: value };
}

/**
 * 按 project_name 收集最终 project 结果。
 */
export function collectProjectResults(
  events: LatextransEvent[],
): Map<string, LatextransEvent> {
  const results = new Map<string, LatextransEvent>();
  for (const event of events) {
    if (
      (event.type === "project_complete" || event.type === "project_error") &&
      event.project_name
    ) {
      results.set(event.project_name, event);
    }
  }
  return results;
}

function isEventShape(value: unknown): value is LatextransEvent {
  if (!value || typeof value !== "object") {
    return false;
  }
  const candidate = value as Partial<LatextransEvent>;
  return (
    typeof candidate.schema_version === "number" &&
    typeof candidate.type === "string" &&
    typeof candidate.timestamp === "string"
  );
}
```

- [ ] **Step 4: 运行测试**

Run:

```powershell
npm run test
npm run build
```

Expected: 事件解析测试通过；未知事件类型不会抛错。

- [ ] **Step 5: Commit**

```powershell
git add src/latextrans/events.ts test/latextrans/events.test.ts
git commit -m "feat(events): parse latextrans json lines"
```

## Task 2: runner 与进程 executor

**Files:**
- Create: `src/latextrans/runner.ts`
- Test: `test/latextrans/runner.test.ts`

- [ ] **Step 1: 写 runner 测试**

创建 `test/latextrans/runner.test.ts`：

```ts
import { assert } from "chai";
import { runLatextrans } from "../../src/latextrans/runner";

describe("latextrans runner", function () {
  it("calls executor with cwd, command and arguments", async function () {
    const result = await runLatextrans({
      cwd: "D:\\Workspace\\tools\\LaTeXTransPlus",
      argv: ["python.exe", "main.py", "--json-events", "stdout"],
      executor: {
        async run(request) {
          assert.deepEqual(request, {
            cwd: "D:\\Workspace\\tools\\LaTeXTransPlus",
            command: "python.exe",
            args: ["main.py", "--json-events", "stdout"],
          });
          request.onStdout(
            '{"schema_version":1,"type":"run_complete","timestamp":"2026-06-06T00:00:00Z","ok":true}\n',
          );
          return { exitCode: 0 };
        },
      },
    });

    assert.equal(result.exitCode, 0);
    assert.lengthOf(result.events, 1);
    assert.deepEqual(result.stderrLines, []);
    assert.deepEqual(result.unparsedStdoutLines, []);
  });

  it("preserves stderr and unparsed stdout diagnostics", async function () {
    const result = await runLatextrans({
      cwd: "C:\\latextrans",
      argv: ["latextrans.exe", "--json-events", "stdout"],
      executor: {
        async run(request) {
          request.onStdout("plain log\n");
          request.onStderr("compile failed\n");
          return { exitCode: 2 };
        },
      },
    });

    assert.equal(result.exitCode, 2);
    assert.deepEqual(result.unparsedStdoutLines, ["plain log"]);
    assert.deepEqual(result.stderrLines, ["compile failed"]);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```powershell
npm run test
```

Expected: FAIL，错误包含 `Cannot find module '../../src/latextrans/runner'`。

- [ ] **Step 3: 实现 `runner.ts` 的接口与纯 runner**

创建 `src/latextrans/runner.ts`：

```ts
import { LatextransEvent, parseLatextransJsonLine } from "./events";

export interface ProcessRequest {
  cwd: string;
  command: string;
  args: string[];
  onStdout: (chunk: string) => void;
  onStderr: (chunk: string) => void;
}

export interface ProcessExecutor {
  run(request: ProcessRequest): Promise<{ exitCode: number }>;
}

export interface RunLatextransInput {
  cwd: string;
  argv: string[];
  executor: ProcessExecutor;
  onEvent?: (event: LatextransEvent) => void;
}

export interface RunLatextransResult {
  exitCode: number;
  events: LatextransEvent[];
  stderrLines: string[];
  unparsedStdoutLines: string[];
}

/**
 * 执行 LaTeXTransPlus 并按 JSON Lines 解析 stdout。
 */
export async function runLatextrans(
  input: RunLatextransInput,
): Promise<RunLatextransResult> {
  const [command, ...args] = input.argv;
  const events: LatextransEvent[] = [];
  const stderrLines: string[] = [];
  const unparsedStdoutLines: string[] = [];
  const stdoutBuffer = createLineBuffer((line) => {
    const parsed = parseLatextransJsonLine(line);
    if (parsed.ok) {
      events.push(parsed.event);
      input.onEvent?.(parsed.event);
    } else if (parsed.reason !== "empty") {
      unparsedStdoutLines.push(parsed.rawLine);
    }
  });
  const stderrBuffer = createLineBuffer((line) => {
    if (line.trim()) {
      stderrLines.push(line);
    }
  });

  const processResult = await input.executor.run({
    cwd: input.cwd,
    command,
    args,
    onStdout: stdoutBuffer.push,
    onStderr: stderrBuffer.push,
  });

  stdoutBuffer.flush();
  stderrBuffer.flush();

  return {
    exitCode: processResult.exitCode,
    events,
    stderrLines,
    unparsedStdoutLines,
  };
}

function createLineBuffer(onLine: (line: string) => void) {
  let pending = "";
  return {
    push(chunk: string) {
      pending += chunk;
      const lines = pending.split(/\r?\n/);
      pending = lines.pop() || "";
      for (const line of lines) {
        onLine(line);
      }
    },
    flush() {
      if (pending) {
        onLine(pending);
        pending = "";
      }
    },
  };
}
```

- [ ] **Step 4: 查证并加入 Zotero runtime executor**

先在本机依赖或 Zotero 源码中查证 `Subprocess.call` 当前签名：

```powershell
rg -n "Subprocess.call|declare.*Subprocess|interface.*Subprocess" node_modules typings src
```

Expected: 找到可用签名，确认参数名为 `command`、`arguments`、`workdir`、`stdout`、`stderr`，返回值含 `exitCode`。如果本机依赖没有类型定义，保留下方局部 runtime adapter，并通过 `npm run build` 验证类型处理。

在 `runner.ts` 末尾加入：

```ts
/**
 * Zotero runtime 下的进程 executor。
 */
export class ZoteroSubprocessExecutor implements ProcessExecutor {
  async run(request: ProcessRequest): Promise<{ exitCode: number }> {
    const Subprocess = ztoolkit.getGlobal("Subprocess");
    const result = await Subprocess.call({
      command: request.command,
      arguments: request.args,
      workdir: request.cwd,
      stdout: (data: string) => request.onStdout(data),
      stderr: (data: string) => request.onStderr(data),
    });
    return { exitCode: result.exitCode };
  }
}
```

- [ ] **Step 5: 运行测试与构建**

Run:

```powershell
npm run test
npm run build
```

Expected: runner 测试通过；如果 `Subprocess` 类型缺失，给 `ztoolkit.getGlobal("Subprocess")` 附近加局部类型声明，不扩大为 `any` 全文件禁用。

- [ ] **Step 6: Commit**

```powershell
git add src/latextrans/runner.ts test/latextrans/runner.test.ts
git commit -m "feat(runner): execute latextrans process"
```

## Task 3: PDF 结果定位

**Files:**
- Create: `src/latextrans/resultResolver.ts`
- Test: `test/latextrans/resultResolver.test.ts`

- [ ] **Step 1: 写 PDF 定位测试**

创建 `test/latextrans/resultResolver.test.ts`：

```ts
import { assert } from "chai";
import { resolveProjectPdf } from "../../src/latextrans/resultResolver";

describe("resultResolver", function () {
  const fs = {
    async exists(path: string) {
      return path.endsWith("ch_2508.18791.pdf") || path.endsWith("fallback.pdf");
    },
    async listPdfFiles(path: string) {
      assert.equal(path, "D:\\latextrans\\outputs\\2508.18791");
      return [
        {
          path: "D:\\latextrans\\outputs\\2508.18791\\old.pdf",
          modifiedTime: 100,
        },
        {
          path: "D:\\latextrans\\outputs\\2508.18791\\fallback.pdf",
          modifiedTime: 300,
        },
      ];
    },
    join(...parts: string[]) {
      return parts.join("\\").replace(/\\+/g, "\\");
    },
    resolve(base: string, path: string) {
      return /^[a-zA-Z]:[\\/]/.test(path) ? path : `${base}\\${path}`;
    },
  };

  it("prefers project_complete pdf_path", async function () {
    const resolved = await resolveProjectPdf({
      projectDir: "D:\\latextrans",
      outputDir: "outputs",
      targetLanguage: "ch",
      commandStartedAt: 200,
      projectName: "2508.18791",
      event: {
        schema_version: 1,
        type: "project_complete",
        timestamp: "2026-06-06T00:00:00Z",
        project_name: "2508.18791",
        pdf_path: "outputs\\2508.18791\\ch_2508.18791.pdf",
      },
      fs,
    });

    assert.deepEqual(resolved, {
      ok: true,
      strategy: "event-pdf-path",
      path: "D:\\latextrans\\outputs\\2508.18791\\ch_2508.18791.pdf",
    });
  });

  it("falls back to newest project pdf after command start", async function () {
    const resolved = await resolveProjectPdf({
      projectDir: "D:\\latextrans",
      outputDir: "outputs",
      targetLanguage: "ch",
      commandStartedAt: 200,
      projectName: "2508.18791",
      event: {
        schema_version: 1,
        type: "project_complete",
        timestamp: "2026-06-06T00:00:00Z",
        project_name: "2508.18791",
        output_dir: "outputs\\2508.18791",
      },
      fs,
    });

    assert.deepEqual(resolved, {
      ok: true,
      strategy: "fallback-scan",
      path: "D:\\latextrans\\outputs\\2508.18791\\fallback.pdf",
    });
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```powershell
npm run test
```

Expected: FAIL，错误包含 `Cannot find module '../../src/latextrans/resultResolver'`。

- [ ] **Step 3: 实现 `resultResolver.ts`**

创建 `src/latextrans/resultResolver.ts`：

```ts
import { LatextransEvent } from "./events";

export interface PdfFileInfo {
  path: string;
  modifiedTime: number;
}

export interface ResultFileSystem {
  exists(path: string): Promise<boolean>;
  listPdfFiles(path: string): Promise<PdfFileInfo[]>;
  join(...parts: string[]): string;
  resolve(base: string, path: string): string;
}

export type PdfResolutionResult =
  | { ok: true; strategy: "event-pdf-path" | "exact-project-path" | "fallback-scan"; path: string }
  | { ok: false; attemptedPaths: string[] };

export interface ResolveProjectPdfInput {
  projectDir: string;
  outputDir: string;
  targetLanguage: string;
  commandStartedAt: number;
  projectName: string;
  event?: LatextransEvent;
  fs: ResultFileSystem;
}

/**
 * 为单个 LaTeXTransPlus project 定位生成 PDF。
 */
export async function resolveProjectPdf(
  input: ResolveProjectPdfInput,
): Promise<PdfResolutionResult> {
  const attemptedPaths: string[] = [];

  if (input.event?.pdf_path) {
    const eventPath = input.fs.resolve(input.projectDir, input.event.pdf_path);
    attemptedPaths.push(eventPath);
    if (await input.fs.exists(eventPath)) {
      return { ok: true, strategy: "event-pdf-path", path: eventPath };
    }
  }

  const projectOutputDir = input.fs.resolve(
    input.projectDir,
    input.event?.output_dir || input.fs.join(input.outputDir, input.projectName),
  );
  const exactPath = input.fs.join(
    projectOutputDir,
    `${input.targetLanguage}_${input.projectName}.pdf`,
  );
  attemptedPaths.push(exactPath);
  if (await input.fs.exists(exactPath)) {
    return { ok: true, strategy: "exact-project-path", path: exactPath };
  }

  const candidates = (await input.fs.listPdfFiles(projectOutputDir))
    .filter((file) => file.modifiedTime >= input.commandStartedAt)
    .sort((a, b) => b.modifiedTime - a.modifiedTime);
  if (candidates[0]) {
    return { ok: true, strategy: "fallback-scan", path: candidates[0].path };
  }

  return { ok: false, attemptedPaths };
}
```

- [ ] **Step 4: 增加 Zotero 文件系统 adapter**

在 `resultResolver.ts` 末尾加入：

```ts
export class ZoteroResultFileSystem implements ResultFileSystem {
  async exists(path: string): Promise<boolean> {
    return IOUtils.exists(path);
  }

  async listPdfFiles(path: string): Promise<PdfFileInfo[]> {
    const entries = await IOUtils.getChildren(path);
    const pdfs = entries.filter((entry) => entry.toLowerCase().endsWith(".pdf"));
    return Promise.all(
      pdfs.map(async (pdf) => {
        const stat = await IOUtils.stat(pdf);
        return { path: pdf, modifiedTime: stat.lastModified };
      }),
    );
  }

  join(...parts: string[]): string {
    return PathUtils.join(...parts);
  }

  resolve(base: string, path: string): string {
    return PathUtils.isAbsolute(path) ? path : PathUtils.join(base, path);
  }
}
```

- [ ] **Step 5: 运行测试与构建**

Run:

```powershell
npm run test
npm run build
```

Expected: result resolver 测试通过；TypeScript 对 `IOUtils` 和 `PathUtils` 可见。

- [ ] **Step 6: Commit**

```powershell
git add src/latextrans/resultResolver.ts test/latextrans/resultResolver.test.ts
git commit -m "feat(results): resolve latextrans project PDFs"
```

## 自审清单

- stdout 非 JSON 行保留到 `unparsedStdoutLines`，不作为结构化事件。
- `schema_version !== 1` 时降级诊断，不抛异常。
- `runner.ts` 不读取 Zotero item，也不导入附件。
- PDF fallback 只扫描 project output dir，不扫描全局 output dir。
- `commandStartedAt` 使用与 `modifiedTime` 相同单位；集成时统一使用 `Date.now()`。
- 事件缺少 `output_dir` 时使用用户配置的 `outputDir`，不能硬编码 `outputs`。
