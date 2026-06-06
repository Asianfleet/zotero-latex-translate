# Preferences, Source Resolver, Task Registry Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 建立 LaTeXTransPlus 配置读取、fixed args 解析、来源识别、多选映射和运行中任务去重的纯逻辑基础。

**Architecture:** 将纯逻辑放入 `src/latextrans/`，Zotero item 访问通过小型接口传入，避免单元测试依赖真实 Zotero 数据库。偏好模块继续复用现有 `src/utils/prefs.ts` 包装，配置校验返回结构化错误而不是直接弹窗。

**Tech Stack:** TypeScript、Mocha/Chai、Zotero preference wrapper。

---

## Task 1: 偏好键与默认值

**Files:**
- Modify: `addon/prefs.js`
- Modify: `typings/prefs.d.ts`
- Create: `src/latextrans/argv.ts`
- Create: `src/latextrans/preferences.ts`
- Test: `test/latextrans/preferences.test.ts`

- [ ] **Step 1: 写偏好类型和默认值测试**

创建 `test/latextrans/preferences.test.ts`：

```ts
import { assert } from "chai";
import {
  buildLatextransConfig,
  validateLatextransConfig,
} from "../../src/latextrans/preferences";

describe("latextrans preferences", function () {
  it("builds typed config from raw preference values", function () {
    const config = buildLatextransConfig({
      projectDir: "D:\\Workspace\\tools\\LaTeXTransPlus",
      executable: "python.exe",
      fixedArgs: "main.py",
      configPath: "config/default.toml",
      outputDir: "outputs",
      sourceLanguage: "en",
      targetLanguage: "ch",
      overwriteExistingAttachment: false,
    });

    assert.deepEqual(config, {
      projectDir: "D:\\Workspace\\tools\\LaTeXTransPlus",
      executable: "python.exe",
      fixedArgs: ["main.py"],
      configPath: "config/default.toml",
      outputDir: "outputs",
      sourceLanguage: "en",
      targetLanguage: "ch",
      overwriteExistingAttachment: false,
    });
  });

  it("reports empty required fields before running CLI", function () {
    const result = validateLatextransConfig({
      projectDir: "",
      executable: "",
      fixedArgs: [],
      configPath: "config/default.toml",
      outputDir: "outputs",
      sourceLanguage: "",
      targetLanguage: "ch",
      overwriteExistingAttachment: false,
    });

    assert.isFalse(result.ok);
    assert.deepEqual(result.errors.map((error) => error.code), [
      "projectDir.empty",
      "executable.empty",
      "sourceLanguage.empty",
    ]);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```powershell
npm run test
```

Expected: FAIL，错误包含 `Cannot find module '../../src/latextrans/preferences'`。

- [ ] **Step 3: 更新偏好默认值**

将 `addon/prefs.js` 改为：

```js
pref("projectDir", "");
pref("executable", "");
pref("fixedArgs", "");
pref("configPath", "config/default.toml");
pref("outputDir", "outputs");
pref("sourceLanguage", "en");
pref("targetLanguage", "ch");
pref("overwriteExistingAttachment", false);
```

- [ ] **Step 4: 更新偏好类型声明**

将 `typings/prefs.d.ts` 的 `PluginPrefsMap` 改为：

```ts
declare namespace _ZoteroTypes {
  interface Prefs {
    PluginPrefsMap: {
      projectDir: string;
      executable: string;
      fixedArgs: string;
      configPath: string;
      outputDir: string;
      sourceLanguage: string;
      targetLanguage: string;
      overwriteExistingAttachment: boolean;
    };
  }
}
```

- [ ] **Step 5: 先创建 `argv.ts` 中的 fixed args parser**

创建 `src/latextrans/argv.ts`，保证 `preferences.ts` 在本 Task 内可编译：

```ts
/**
 * 按 shell-like 规则解析用户填写的 fixed args，但命令执行时仍使用 argv 数组。
 */
export function parseFixedArgs(input: string): string[] {
  const args: string[] = [];
  let current = "";
  let quote: '"' | "'" | undefined;

  for (let index = 0; index < input.length; index += 1) {
    const char = input[index];
    if (quote) {
      if (char === quote) {
        quote = undefined;
      } else {
        current += char;
      }
      continue;
    }

    if (char === '"' || char === "'") {
      quote = char;
      continue;
    }

    if (/\s/.test(char)) {
      if (current) {
        args.push(current);
        current = "";
      }
      continue;
    }

    current += char;
  }

  if (current) {
    args.push(current);
  }

  return args;
}
```

- [ ] **Step 6: 实现 `preferences.ts`**

创建 `src/latextrans/preferences.ts`：

```ts
import { getPref } from "../utils/prefs";
import { parseFixedArgs } from "./argv";

export interface LatextransConfig {
  projectDir: string;
  executable: string;
  fixedArgs: string[];
  configPath: string;
  outputDir: string;
  sourceLanguage: string;
  targetLanguage: string;
  overwriteExistingAttachment: boolean;
}

export interface RawLatextransPrefs {
  projectDir: string;
  executable: string;
  fixedArgs: string;
  configPath: string;
  outputDir: string;
  sourceLanguage: string;
  targetLanguage: string;
  overwriteExistingAttachment: boolean;
}

export interface ConfigValidationError {
  code:
    | "projectDir.empty"
    | "executable.empty"
    | "configPath.empty"
    | "outputDir.empty"
    | "sourceLanguage.empty"
    | "targetLanguage.empty";
  message: string;
}

export type ConfigValidationResult =
  | { ok: true; errors: [] }
  | { ok: false; errors: ConfigValidationError[] };

/**
 * 从 Zotero preferences 读取 LaTeXTransPlus 配置。
 */
export function readLatextransConfig(): LatextransConfig {
  return buildLatextransConfig({
    projectDir: getPref("projectDir") || "",
    executable: getPref("executable") || "",
    fixedArgs: getPref("fixedArgs") || "",
    configPath: getPref("configPath") || "config/default.toml",
    outputDir: getPref("outputDir") || "outputs",
    sourceLanguage: getPref("sourceLanguage") || "en",
    targetLanguage: getPref("targetLanguage") || "ch",
    overwriteExistingAttachment: Boolean(getPref("overwriteExistingAttachment")),
  });
}

/**
 * 把原始偏好值标准化为运行时配置。
 */
export function buildLatextransConfig(
  prefs: RawLatextransPrefs,
): LatextransConfig {
  return {
    projectDir: prefs.projectDir.trim(),
    executable: prefs.executable.trim(),
    fixedArgs: parseFixedArgs(prefs.fixedArgs),
    configPath: prefs.configPath.trim(),
    outputDir: prefs.outputDir.trim(),
    sourceLanguage: prefs.sourceLanguage.trim(),
    targetLanguage: prefs.targetLanguage.trim(),
    overwriteExistingAttachment: prefs.overwriteExistingAttachment,
  };
}

/**
 * 校验不会触碰文件系统的必填配置，文件存在性在命令编排阶段检查。
 */
export function validateLatextransConfig(
  config: LatextransConfig,
): ConfigValidationResult {
  const errors: ConfigValidationError[] = [];
  const required: Array<[keyof LatextransConfig, ConfigValidationError["code"], string]> = [
    ["projectDir", "projectDir.empty", "LaTeXTransPlus 项目目录不能为空"],
    ["executable", "executable.empty", "可执行文件路径不能为空"],
    ["configPath", "configPath.empty", "config.toml 路径不能为空"],
    ["outputDir", "outputDir.empty", "output 目录不能为空"],
    ["sourceLanguage", "sourceLanguage.empty", "source_language 不能为空"],
    ["targetLanguage", "targetLanguage.empty", "target_language 不能为空"],
  ];

  for (const [key, code, message] of required) {
    if (typeof config[key] === "string" && !config[key]) {
      errors.push({ code, message });
    }
  }

  return errors.length === 0 ? { ok: true, errors: [] } : { ok: false, errors };
}
```

- [ ] **Step 7: 运行测试**

Run:

```powershell
npm run test
npm run build
```

Expected: `preferences.test.ts` 通过；`npm run build` 通过 TypeScript 检查。

- [ ] **Step 8: Commit**

```powershell
git add addon/prefs.js typings/prefs.d.ts src/latextrans/argv.ts src/latextrans/preferences.ts test/latextrans/preferences.test.ts
git commit -m "feat(config): add latextrans preferences"
```

## Task 2: fixed args 解析与 argv 组装

**Files:**
- Modify: `src/latextrans/argv.ts`
- Test: `test/latextrans/argv.test.ts`

- [ ] **Step 1: 写 fixed args 与 argv 测试**

创建 `test/latextrans/argv.test.ts`：

```ts
import { assert } from "chai";
import { buildLatextransArgv, parseFixedArgs } from "../../src/latextrans/argv";

describe("latextrans argv", function () {
  it("splits fixed args with quotes", function () {
    assert.deepEqual(parseFixedArgs('run -n latextrans python "main script.py"'), [
      "run",
      "-n",
      "latextrans",
      "python",
      "main script.py",
    ]);
  });

  it("builds argv for arxiv batch and appends json events", function () {
    const argv = buildLatextransArgv({
      executable: "python.exe",
      fixedArgs: ["main.py"],
      configPath: "config/default.toml",
      outputDir: "outputs",
      sourceLanguage: "en",
      targetLanguage: "ch",
      source: {
        kind: "arxiv",
        values: ["2508.18791", "2508.18792v2"],
      },
    });

    assert.deepEqual(argv, [
      "python.exe",
      "main.py",
      "--config",
      "config/default.toml",
      "--output",
      "outputs",
      "--source_language",
      "en",
      "--target_language",
      "ch",
      "--json-events",
      "stdout",
      "--arxiv",
      "2508.18791,2508.18792v2",
    ]);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```powershell
npm run test
```

Expected: FAIL，错误包含 `buildLatextransArgv is not a function`。

- [ ] **Step 3: 实现 `argv.ts`**

创建 `src/latextrans/argv.ts`：

```ts
export type LatextransSourceArg =
  | { kind: "arxiv"; values: string[] }
  | { kind: "project-url"; value: string }
  | { kind: "project"; value: string };

export interface BuildArgvInput {
  executable: string;
  fixedArgs: string[];
  configPath: string;
  outputDir: string;
  sourceLanguage: string;
  targetLanguage: string;
  source: LatextransSourceArg;
}

/**
 * 按 shell-like 规则解析用户填写的 fixed args，但命令执行时仍使用 argv 数组。
 */
export function parseFixedArgs(input: string): string[] {
  const args: string[] = [];
  let current = "";
  let quote: '"' | "'" | undefined;

  for (let index = 0; index < input.length; index += 1) {
    const char = input[index];
    if (quote) {
      if (char === quote) {
        quote = undefined;
      } else {
        current += char;
      }
      continue;
    }

    if (char === '"' || char === "'") {
      quote = char;
      continue;
    }

    if (/\s/.test(char)) {
      if (current) {
        args.push(current);
        current = "";
      }
      continue;
    }

    current += char;
  }

  if (current) {
    args.push(current);
  }

  return args;
}

/**
 * 组装 LaTeXTransPlus CLI argv，调用方负责设置 cwd。
 */
export function buildLatextransArgv(input: BuildArgvInput): string[] {
  const sourceArg =
    input.source.kind === "arxiv"
      ? ["--arxiv", input.source.values.join(",")]
      : input.source.kind === "project-url"
        ? ["--project-url", input.source.value]
        : ["--project", input.source.value];

  return [
    input.executable,
    ...input.fixedArgs,
    "--config",
    input.configPath,
    "--output",
    input.outputDir,
    "--source_language",
    input.sourceLanguage,
    "--target_language",
    input.targetLanguage,
    "--json-events",
    "stdout",
    ...sourceArg,
  ];
}
```

- [ ] **Step 4: 运行测试**

Run:

```powershell
npm run test
npm run build
```

Expected: argv 测试通过；TypeScript 检查通过。

- [ ] **Step 5: Commit**

```powershell
git add src/latextrans/argv.ts test/latextrans/argv.test.ts
git commit -m "feat(runner): build latextrans argv"
```

## Task 3: 来源解析与多选 arXiv 映射

**Files:**
- Create: `src/latextrans/sourceResolver.ts`
- Test: `test/latextrans/sourceResolver.test.ts`

- [ ] **Step 1: 写来源解析测试**

创建 `test/latextrans/sourceResolver.test.ts`：

```ts
import { assert } from "chai";
import {
  buildArxivBatchMapping,
  classifyUserInput,
  resolveArxivFromItemSnapshot,
} from "../../src/latextrans/sourceResolver";

describe("sourceResolver", function () {
  it("extracts modern arxiv IDs from URL, DOI, extra and abstract", function () {
    assert.equal(
      resolveArxivFromItemSnapshot({
        fields: { url: "https://arxiv.org/abs/2508.18791v2" },
        attachments: [],
      }),
      "2508.18791v2",
    );
    assert.equal(
      resolveArxivFromItemSnapshot({
        fields: { DOI: "10.48550/arXiv.2508.18791" },
        attachments: [],
      }),
      "2508.18791",
    );
    assert.equal(
      resolveArxivFromItemSnapshot({
        fields: { extra: "arXiv: 2508.18792" },
        attachments: [],
      }),
      "2508.18792",
    );
    assert.equal(
      resolveArxivFromItemSnapshot({
        fields: { abstractNote: "Preprint available as arXiv:2508.18793v3." },
        attachments: [],
      }),
      "2508.18793v3",
    );
  });

  it("classifies user input into CLI source kinds", function () {
    assert.deepEqual(classifyUserInput("https://arxiv.org/abs/2508.18791"), {
      ok: true,
      source: { kind: "arxiv", value: "2508.18791" },
    });
    assert.deepEqual(classifyUserInput("https://example.org/source.zip"), {
      ok: true,
      source: { kind: "project-url", value: "https://example.org/source.zip" },
    });
    assert.deepEqual(classifyUserInput("D:\\papers\\source.zip"), {
      ok: true,
      source: { kind: "project", value: "D:\\papers\\source.zip" },
    });
  });

  it("deduplicates arxiv IDs and keeps item-level mapping", function () {
    const mapping = buildArxivBatchMapping([
      { itemID: 1, arxivID: "2508.18791" },
      { itemID: 2, arxivID: "2508.18791" },
      { itemID: 3, arxivID: "2508.18792v2" },
    ]);

    assert.deepEqual(mapping.argvValues, ["2508.18791", "2508.18792v2"]);
    assert.deepEqual(Array.from(mapping.projectToItemIDs.entries()), [
      ["2508.18791", [1, 2]],
      ["2508.18792v2", [3]],
    ]);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```powershell
npm run test
```

Expected: FAIL，错误包含 `Cannot find module '../../src/latextrans/sourceResolver'`。

- [ ] **Step 3: 实现 `sourceResolver.ts`**

创建 `src/latextrans/sourceResolver.ts`：

```ts
export interface ItemSourceSnapshot {
  fields: Partial<Record<"url" | "DOI" | "extra" | "abstractNote" | "archiveLocation", string>>;
  attachments: Array<{
    title?: string;
    url?: string;
    path?: string;
  }>;
}

export type ResolvedInputSource =
  | { kind: "arxiv"; value: string }
  | { kind: "project-url"; value: string }
  | { kind: "project"; value: string };

export type InputClassificationResult =
  | { ok: true; source: ResolvedInputSource }
  | { ok: false; reason: "empty" | "unsupported" };

export interface ArxivBatchMapping {
  argvValues: string[];
  projectToItemIDs: Map<string, number[]>;
}

const MODERN_ARXIV_RE = /\b(?:arxiv:|arxiv\.|abs\/|pdf\/)?(\d{4}\.\d{4,5}(?:v\d+)?)\b/i;
const OLD_ARXIV_RE = /\b([a-z-]+(?:\.[A-Z]{2})?\/\d{7}(?:v\d+)?)\b/i;

/**
 * 从条目快照中提取第一版承诺支持的新式 arXiv ID。
 */
export function resolveArxivFromItemSnapshot(
  snapshot: ItemSourceSnapshot,
): string | undefined {
  const candidates = [
    snapshot.fields.url,
    snapshot.fields.DOI,
    snapshot.fields.extra,
    snapshot.fields.abstractNote,
    snapshot.fields.archiveLocation,
    ...snapshot.attachments.flatMap((attachment) => [
      attachment.url,
      attachment.title,
      attachment.path,
    ]),
  ].filter((value): value is string => Boolean(value));

  for (const candidate of candidates) {
    const modern = candidate.match(MODERN_ARXIV_RE);
    if (modern?.[1]) {
      return modern[1];
    }
  }

  return undefined;
}

/**
 * 旧式 arXiv ID 只用于单选用户输入，不作为第一版批量映射承诺。
 */
export function resolveAnyArxivFromText(input: string): string | undefined {
  return input.match(MODERN_ARXIV_RE)?.[1] || input.match(OLD_ARXIV_RE)?.[1];
}

/**
 * 将用户输入分类为 LaTeXTransPlus 支持的来源类型。
 */
export function classifyUserInput(input: string): InputClassificationResult {
  const value = input.trim();
  if (!value) {
    return { ok: false, reason: "empty" };
  }

  const arxiv = resolveAnyArxivFromText(value);
  if (arxiv) {
    return { ok: true, source: { kind: "arxiv", value: arxiv } };
  }

  if (/^https?:\/\//i.test(value)) {
    return { ok: true, source: { kind: "project-url", value } };
  }

  if (/^[a-zA-Z]:[\\/]/.test(value) || value.startsWith("/") || value.startsWith(".")) {
    return { ok: true, source: { kind: "project", value } };
  }

  return { ok: false, reason: "unsupported" };
}

/**
 * 多选批量只传唯一 arXiv ID，同时保留 project 到 Zotero itemID 的一对多映射。
 */
export function buildArxivBatchMapping(
  inputs: Array<{ itemID: number; arxivID: string }>,
): ArxivBatchMapping {
  const projectToItemIDs = new Map<string, number[]>();
  for (const input of inputs) {
    const itemIDs = projectToItemIDs.get(input.arxivID) || [];
    itemIDs.push(input.itemID);
    projectToItemIDs.set(input.arxivID, itemIDs);
  }

  return {
    argvValues: Array.from(projectToItemIDs.keys()),
    projectToItemIDs,
  };
}
```

- [ ] **Step 4: 运行测试**

Run:

```powershell
npm run test
npm run build
```

Expected: source resolver 测试通过；TypeScript 检查通过。

- [ ] **Step 5: Commit**

```powershell
git add src/latextrans/sourceResolver.ts test/latextrans/sourceResolver.test.ts
git commit -m "feat(source): resolve latextrans input sources"
```

## Task 4: 任务去重 registry

**Files:**
- Create: `src/latextrans/taskRegistry.ts`
- Modify: `src/addon.ts`
- Test: `test/latextrans/taskRegistry.test.ts`

- [ ] **Step 1: 写 registry 测试**

创建 `test/latextrans/taskRegistry.test.ts`：

```ts
import { assert } from "chai";
import { TaskRegistry } from "../../src/latextrans/taskRegistry";

describe("TaskRegistry", function () {
  it("marks duplicate running items as skipped candidates", function () {
    const registry = new TaskRegistry();
    assert.deepEqual(registry.tryAcquireMany([1, 2]), {
      acquired: [1, 2],
      alreadyRunning: [],
    });

    const second = registry.tryAcquireMany([2, 3]);
    assert.deepEqual(second, {
      acquired: [3],
      alreadyRunning: [2],
    });
  });

  it("releases itemIDs after command finishes", function () {
    const registry = new TaskRegistry();
    registry.tryAcquireMany([1, 2]);
    registry.releaseMany([1, 2]);

    assert.deepEqual(registry.tryAcquireMany([1]), {
      acquired: [1],
      alreadyRunning: [],
    });
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```powershell
npm run test
```

Expected: FAIL，错误包含 `Cannot find module '../../src/latextrans/taskRegistry'`。

- [ ] **Step 3: 实现 `taskRegistry.ts`**

创建 `src/latextrans/taskRegistry.ts`：

```ts
export interface AcquireManyResult {
  acquired: number[];
  alreadyRunning: number[];
}

/**
 * 维护插件进程内正在运行的 Zotero itemID。
 */
export class TaskRegistry {
  private readonly runningItemIDs = new Set<number>();

  tryAcquireMany(itemIDs: number[]): AcquireManyResult {
    const uniqueIDs = Array.from(new Set(itemIDs));
    const alreadyRunning = uniqueIDs.filter((itemID) =>
      this.runningItemIDs.has(itemID),
    );
    const acquired = uniqueIDs.filter(
      (itemID) => !this.runningItemIDs.has(itemID),
    );

    for (const itemID of acquired) {
      this.runningItemIDs.add(itemID);
    }

    return { acquired, alreadyRunning };
  }

  releaseMany(itemIDs: number[]): void {
    for (const itemID of itemIDs) {
      this.runningItemIDs.delete(itemID);
    }
  }

  clear(): void {
    this.runningItemIDs.clear();
  }

  has(itemID: number): boolean {
    return this.runningItemIDs.has(itemID);
  }
}
```

- [ ] **Step 4: 在 addon data 中挂载 registry**

修改 `src/addon.ts`：

```ts
import { TaskRegistry } from "./latextrans/taskRegistry";
```

在 `data` 类型中加入：

```ts
taskRegistry: TaskRegistry;
```

在 constructor 的 `this.data` 中加入：

```ts
taskRegistry: new TaskRegistry(),
```

- [ ] **Step 5: 运行测试**

Run:

```powershell
npm run test
npm run build
```

Expected: registry 测试通过；`addon.data.taskRegistry` 类型可通过编译。

- [ ] **Step 6: Commit**

```powershell
git add src/latextrans/taskRegistry.ts src/addon.ts test/latextrans/taskRegistry.test.ts
git commit -m "feat(tasks): track running translation items"
```

## 自审清单

- 每个偏好键在 `addon/prefs.js`、`typings/prefs.d.ts`、`preferences.ts` 中名称一致。
- `parseFixedArgs` 只解析参数，不负责命令执行。
- 多选映射 key 使用传给 `--arxiv` 的同一个 ID 字符串。
- `TaskRegistry.tryAcquireMany()` 只获取当前未运行的 itemID，并把已运行 itemID 返回给编排层计入 skipped。
- 本文档不触碰 Zotero 附件导入和外部进程执行逻辑。
