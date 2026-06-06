# Attachment, Command, UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将纯逻辑接入 Zotero：注册右键菜单、获取当前选择、运行 LaTeXTransPlus、导入 stored child attachment，并显示 item 级汇总。

**Architecture:** `translateCommand.ts` 作为编排层，通过依赖注入调用 runner、PDF resolver、附件服务、输入服务和进度服务。真实 Zotero UI 与 API 放在 adapter 中，mock 集成测试只验证编排行为和状态汇总。

**Tech Stack:** TypeScript、zotero-plugin-toolkit Menu/ProgressWindow、Zotero Items/Attachments APIs、Mocha/Chai。

---

## Task 1: 附件导入服务

**Files:**
- Create: `src/latextrans/attachmentService.ts`
- Test: `test/latextrans/attachmentService.test.ts`

- [ ] **Step 1: 写附件服务测试**

创建 `test/latextrans/attachmentService.test.ts`：

```ts
import { assert } from "chai";
import { saveTranslatedPdfAttachment } from "../../src/latextrans/attachmentService";

describe("attachmentService", function () {
  it("imports translated pdf as stored child attachment", async function () {
    const calls: unknown[] = [];
    const result = await saveTranslatedPdfAttachment({
      itemID: 10,
      pdfPath: "D:\\latextrans\\outputs\\2508.18791\\ch_2508.18791.pdf",
      targetLanguage: "ch",
      overwriteExistingAttachment: false,
      api: {
        async findExistingTranslatedAttachments() {
          return [];
        },
        async trashAttachment() {
          throw new Error("should not trash");
        },
        async importStoredAttachment(input) {
          calls.push(input);
          return 100;
        },
        async addTag(attachmentID, tag) {
          calls.push({ attachmentID, tag });
        },
      },
    });

    assert.deepEqual(result, { ok: true, attachmentID: 100 });
    assert.deepEqual(calls, [
      {
        parentItemID: 10,
        file: "D:\\latextrans\\outputs\\2508.18791\\ch_2508.18791.pdf",
        title: "翻译版 PDF (ch)",
      },
      { attachmentID: 100, tag: "latextransplus-translated" },
    ]);
  });

  it("trashes existing translated attachments only when overwrite is enabled", async function () {
    const trashed: number[] = [];
    await saveTranslatedPdfAttachment({
      itemID: 10,
      pdfPath: "D:\\new.pdf",
      targetLanguage: "ch",
      overwriteExistingAttachment: true,
      api: {
        async findExistingTranslatedAttachments() {
          return [90, 91];
        },
        async trashAttachment(id) {
          trashed.push(id);
        },
        async importStoredAttachment() {
          return 100;
        },
        async addTag() {},
      },
    });

    assert.deepEqual(trashed, [90, 91]);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```powershell
npm run test
```

Expected: FAIL，错误包含 `Cannot find module '../../src/latextrans/attachmentService'`。

- [ ] **Step 3: 实现服务和 Zotero adapter**

创建 `src/latextrans/attachmentService.ts`：

```ts
export const TRANSLATED_ATTACHMENT_TAG = "latextransplus-translated";

export interface AttachmentApi {
  findExistingTranslatedAttachments(itemID: number, title: string): Promise<number[]>;
  trashAttachment(attachmentID: number): Promise<void>;
  importStoredAttachment(input: {
    parentItemID: number;
    file: string;
    title: string;
  }): Promise<number>;
  addTag(attachmentID: number, tag: string): Promise<void>;
}

export interface SaveAttachmentInput {
  itemID: number;
  pdfPath: string;
  targetLanguage: string;
  overwriteExistingAttachment: boolean;
  api: AttachmentApi;
}

export type SaveAttachmentResult =
  | { ok: true; attachmentID: number }
  | { ok: false; error: string; pdfPath: string };

/**
 * 将翻译 PDF 保存为 Zotero stored child attachment。
 */
export async function saveTranslatedPdfAttachment(
  input: SaveAttachmentInput,
): Promise<SaveAttachmentResult> {
  const title = `翻译版 PDF (${input.targetLanguage})`;
  try {
    if (input.overwriteExistingAttachment) {
      const existing = await input.api.findExistingTranslatedAttachments(
        input.itemID,
        title,
      );
      for (const attachmentID of existing) {
        await input.api.trashAttachment(attachmentID);
      }
    }

    const attachmentID = await input.api.importStoredAttachment({
      parentItemID: input.itemID,
      file: input.pdfPath,
      title,
    });
    await input.api.addTag(attachmentID, TRANSLATED_ATTACHMENT_TAG);
    return { ok: true, attachmentID };
  } catch (error) {
    return {
      ok: false,
      error: error instanceof Error ? error.message : String(error),
      pdfPath: input.pdfPath,
    };
  }
}

export class ZoteroAttachmentApi implements AttachmentApi {
  async findExistingTranslatedAttachments(
    itemID: number,
    title: string,
  ): Promise<number[]> {
    const item = await Zotero.Items.getAsync(itemID);
    const childIDs = item.getAttachments();
    const matches: number[] = [];
    for (const childID of childIDs) {
      const child = await Zotero.Items.getAsync(childID);
      if (
        child.isAttachment() &&
        child.getField("title") === title &&
        child.hasTag(TRANSLATED_ATTACHMENT_TAG)
      ) {
        matches.push(childID);
      }
    }
    return matches;
  }

  async trashAttachment(attachmentID: number): Promise<void> {
    await Zotero.Items.trashTx(attachmentID);
  }

  async importStoredAttachment(input: {
    parentItemID: number;
    file: string;
    title: string;
  }): Promise<number> {
    const attachment = await Zotero.Attachments.importFromFile({
      file: input.file,
      parentItemID: input.parentItemID,
      title: input.title,
    });
    return attachment.id;
  }

  async addTag(attachmentID: number, tag: string): Promise<void> {
    const attachment = await Zotero.Items.getAsync(attachmentID);
    attachment.addTag(tag);
    await attachment.saveTx();
  }
}
```

- [ ] **Step 4: 运行测试与构建**

Run:

```powershell
npm run test
npm run build
```

Expected: 附件服务测试通过；Zotero attachment adapter 编译通过。

- [ ] **Step 5: Commit**

```powershell
git add src/latextrans/attachmentService.ts test/latextrans/attachmentService.test.ts
git commit -m "feat(attachments): import translated PDFs"
```

## Task 2: translateCommand 编排模型

**Files:**
- Create: `src/latextrans/translateCommand.ts`
- Test: `test/latextrans/translateCommand.test.ts`

- [ ] **Step 1: 写 mock 集成测试**

创建 `test/latextrans/translateCommand.test.ts`：

```ts
import { assert } from "chai";
import { executeTranslateCommand } from "../../src/latextrans/translateCommand";
import { TaskRegistry } from "../../src/latextrans/taskRegistry";

describe("translateCommand", function () {
  it("merges multi-select arxiv items into one CLI run", async function () {
    const registry = new TaskRegistry();
    const runnerCalls: unknown[] = [];

    const result = await executeTranslateCommand({
      selectedItems: [
        {
          itemID: 1,
          isRegularItem: true,
          snapshot: { fields: { url: "https://arxiv.org/abs/2508.18791" }, attachments: [] },
        },
        {
          itemID: 2,
          isRegularItem: true,
          snapshot: { fields: { extra: "arXiv: 2508.18792" }, attachments: [] },
        },
      ],
      config: {
        projectDir: "D:\\latextrans",
        executable: "python.exe",
        fixedArgs: ["main.py"],
        configPath: "config/default.toml",
        outputDir: "outputs",
        sourceLanguage: "en",
        targetLanguage: "ch",
        overwriteExistingAttachment: false,
      },
      registry,
      services: {
        async promptForSource() {
          throw new Error("multi-select must not prompt");
        },
        async runLatextrans(input) {
          runnerCalls.push(input.argv);
          return {
            exitCode: 0,
            stderrLines: [],
            unparsedStdoutLines: [],
            events: [
              {
                schema_version: 1,
                type: "project_complete",
                timestamp: "2026-06-06T00:00:00Z",
                project_name: "2508.18791",
                pdf_path: "outputs\\2508.18791\\ch_2508.18791.pdf",
              },
              {
                schema_version: 1,
                type: "project_complete",
                timestamp: "2026-06-06T00:00:01Z",
                project_name: "2508.18792",
                pdf_path: "outputs\\2508.18792\\ch_2508.18792.pdf",
              },
            ],
          };
        },
        async resolvePdf(input) {
          return {
            ok: true,
            strategy: "event-pdf-path",
            path: `D:\\latextrans\\${input.event?.pdf_path}`,
          };
        },
        async saveAttachment(input) {
          return { ok: true, attachmentID: input.itemID + 100 };
        },
        progress: {
          update() {},
          finish() {},
        },
      },
      commandStartedAt: 300,
    });

    assert.deepEqual(runnerCalls, [
      [
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
        "2508.18791,2508.18792",
      ],
    ]);
    assert.equal(result.succeeded, 2);
    assert.equal(result.failed, 0);
    assert.equal(result.skipped, 0);
    assert.isFalse(registry.has(1));
    assert.isFalse(registry.has(2));
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```powershell
npm run test
```

Expected: FAIL，错误包含 `Cannot find module '../../src/latextrans/translateCommand'`。

- [ ] **Step 3: 实现编排类型与主函数**

创建 `src/latextrans/translateCommand.ts`，先实现测试覆盖的多选路径：

```ts
import { buildLatextransArgv, LatextransSourceArg } from "./argv";
import { saveTranslatedPdfAttachment, SaveAttachmentInput, SaveAttachmentResult } from "./attachmentService";
import { collectProjectResults, LatextransEvent } from "./events";
import { LatextransConfig } from "./preferences";
import { ResolveProjectPdfInput, PdfResolutionResult } from "./resultResolver";
import { RunLatextransInput, RunLatextransResult } from "./runner";
import {
  buildArxivBatchMapping,
  classifyUserInput,
  ItemSourceSnapshot,
  resolveArxivFromItemSnapshot,
} from "./sourceResolver";
import { TaskRegistry } from "./taskRegistry";

export interface SelectedTranslateItem {
  itemID: number;
  isRegularItem: boolean;
  snapshot: ItemSourceSnapshot;
}

export interface TranslateProgress {
  update(message: string): void;
  finish(summary: TranslateSummary): void;
}

export interface TranslateServices {
  promptForSource(item: SelectedTranslateItem): Promise<string | undefined>;
  runLatextrans(input: RunLatextransInput): Promise<RunLatextransResult>;
  resolvePdf(input: ResolveProjectPdfInput): Promise<PdfResolutionResult>;
  saveAttachment(input: SaveAttachmentInput): Promise<SaveAttachmentResult>;
  progress: TranslateProgress;
}

export interface ExecuteTranslateCommandInput {
  selectedItems: SelectedTranslateItem[];
  config: LatextransConfig;
  registry: TaskRegistry;
  services: TranslateServices;
  commandStartedAt: number;
}

export interface TranslateSummary {
  succeeded: number;
  failed: number;
  skipped: number;
  failures: string[];
}

/**
 * 编排一次右键翻译命令，按 Zotero item 级别统计结果。
 */
export async function executeTranslateCommand(
  input: ExecuteTranslateCommandInput,
): Promise<TranslateSummary> {
  const summary: TranslateSummary = {
    succeeded: 0,
    failed: 0,
    skipped: 0,
    failures: [],
  };

  const regularItems = input.selectedItems.filter((item) => item.isRegularItem);
  summary.skipped += input.selectedItems.length - regularItems.length;
  if (regularItems.length === 0) {
    input.services.progress.finish(summary);
    return summary;
  }

  const sources = await resolveCommandSources(regularItems, input.services);
  summary.skipped += sources.skipped;
  if (sources.runnable.length === 0) {
    input.services.progress.finish(summary);
    return summary;
  }

  const acquire = input.registry.tryAcquireMany(
    sources.runnable.map((item) => item.itemID),
  );
  summary.skipped += acquire.alreadyRunning.length;
  const acquiredItemIDs = new Set(acquire.acquired);
  const runnable = sources.runnable.filter((item) =>
    acquiredItemIDs.has(item.itemID),
  );
  if (runnable.length === 0) {
    input.services.progress.finish(summary);
    return summary;
  }

  try {
    const sourceArg = toSourceArg(runnable);

    const runResult = await input.services.runLatextrans({
      cwd: input.config.projectDir,
      argv: buildLatextransArgv({ ...input.config, source: sourceArg }),
      executor: {
        async run() {
          throw new Error("executor is supplied by service implementation");
        },
      },
    });

    const projectResults = collectProjectResults(runResult.events);
    const projectToItemIDs = buildProjectItemMapping(runnable, projectResults);
    for (const [projectName, itemIDs] of projectToItemIDs) {
      const event = projectResults.get(projectName);
      if (!event || event.type === "project_error") {
        summary.failed += itemIDs.length;
        summary.failures.push(event?.error || `${projectName} 执行失败`);
        continue;
      }

      const pdf = await input.services.resolvePdf({
        projectDir: input.config.projectDir,
        outputDir: input.config.outputDir,
        targetLanguage: input.config.targetLanguage,
        commandStartedAt: input.commandStartedAt,
        projectName,
        event,
        fs: {
          async exists() {
            return false;
          },
          async listPdfFiles() {
            return [];
          },
          join: (...parts) => parts.join("\\"),
          resolve: (_base, path) => path,
        },
      });
      if (!pdf.ok) {
        summary.failed += itemIDs.length;
        summary.failures.push(`${projectName} 未找到生成 PDF`);
        continue;
      }

      for (const itemID of itemIDs) {
        const saved = await input.services.saveAttachment({
          itemID,
          pdfPath: pdf.path,
          targetLanguage: input.config.targetLanguage,
          overwriteExistingAttachment: input.config.overwriteExistingAttachment,
          api: {
            async findExistingTranslatedAttachments() {
              return [];
            },
            async trashAttachment() {},
            async importStoredAttachment() {
              return 0;
            },
            async addTag() {},
          },
        });
        if (saved.ok) {
          summary.succeeded += 1;
        } else {
          summary.failed += 1;
          summary.failures.push(saved.error);
        }
      }
    }
  } finally {
    input.registry.releaseMany(acquire.acquired);
  }

  input.services.progress.finish(summary);
  return summary;
}

async function resolveCommandSources(
  items: SelectedTranslateItem[],
  services: TranslateServices,
) {
  const runnable: Array<SelectedTranslateItem & { source: { kind: "arxiv"; value: string } | { kind: "project-url"; value: string } | { kind: "project"; value: string } }> = [];
  let skipped = 0;
  const isSingle = items.length === 1;

  for (const item of items) {
    const arxiv = resolveArxivFromItemSnapshot(item.snapshot);
    if (arxiv) {
      runnable.push({ ...item, source: { kind: "arxiv", value: arxiv } });
      continue;
    }
    if (!isSingle) {
      skipped += 1;
      continue;
    }
    const userInput = await services.promptForSource(item);
    if (!userInput) {
      skipped += 1;
      continue;
    }
    const classified = classifyUserInput(userInput);
    if (classified.ok) {
      runnable.push({ ...item, source: classified.source });
    } else {
      skipped += 1;
    }
  }

  return { runnable, skipped };
}

function toSourceArg(
  items: Array<SelectedTranslateItem & { source: { kind: "arxiv"; value: string } | { kind: "project-url"; value: string } | { kind: "project"; value: string } }>,
): LatextransSourceArg {
  if (items.length > 1) {
    const mapping = buildArxivBatchMapping(
      items.map((item) => ({ itemID: item.itemID, arxivID: item.source.value })),
    );
    return { kind: "arxiv", values: mapping.argvValues };
  }
  const source = items[0].source;
  return source.kind === "arxiv"
    ? { kind: "arxiv", values: [source.value] }
    : { kind: source.kind, value: source.value };
}

function buildProjectItemMapping(
  items: Array<SelectedTranslateItem & { source: { kind: "arxiv"; value: string } | { kind: "project-url"; value: string } | { kind: "project"; value: string } }>,
  projectResults: Map<string, unknown>,
): Map<string, number[]> {
  if (items.every((item) => item.source.kind === "arxiv")) {
    return buildArxivBatchMapping(
      items.map((item) => ({ itemID: item.itemID, arxivID: item.source.value })),
    ).projectToItemIDs;
  }

  const onlyItem = items[0];
  const [onlyProjectName] = Array.from(projectResults.keys());
  return new Map([[onlyProjectName || onlyItem.source.value, [onlyItem.itemID]]]);
}
```

- [ ] **Step 4: 收紧服务接口，去掉测试替身中的假 executor 与 fake fs**

将 `TranslateServices.runLatextrans` 改为接收 `{ cwd: string; argv: string[]; onEvent?: (event: LatextransEvent) => void }`，将 `TranslateServices.resolvePdf` 改为接收不含 `fs` 的业务参数。真实 adapter 在 Task 4 中绑定 `ZoteroSubprocessExecutor` 和 `ZoteroResultFileSystem`，测试中的 mock 不需要关心 runtime adapter。

最终接口应为：

```ts
export interface TranslateServices {
  promptForSource(item: SelectedTranslateItem): Promise<string | undefined>;
  runLatextrans(input: {
    cwd: string;
    argv: string[];
    onEvent?: (event: LatextransEvent) => void;
  }): Promise<RunLatextransResult>;
  resolvePdf(input: Omit<ResolveProjectPdfInput, "fs">): Promise<PdfResolutionResult>;
  saveAttachment(input: Omit<SaveAttachmentInput, "api">): Promise<SaveAttachmentResult>;
  progress: TranslateProgress;
}
```

在 `executeTranslateCommand()` 调用 runner 时传入事件进度回调：

```ts
const runResult = await input.services.runLatextrans({
  cwd: input.config.projectDir,
  argv: buildLatextransArgv({ ...input.config, source: sourceArg }),
  onEvent(event) {
    if (event.type === "project_start" && event.project_name) {
      input.services.progress.update(
        `[${event.index || "?"}/${event.total || "?"}] 运行 LaTeXTransPlus: ${event.project_name}`,
      );
    } else if (event.type === "project_complete" && event.project_name) {
      input.services.progress.update(`查找生成 PDF: ${event.project_name}`);
    } else if (event.type === "project_error" && event.project_name) {
      input.services.progress.update(`执行失败: ${event.project_name}`);
    }
  },
});
```

- [ ] **Step 5: 运行测试与构建**

Run:

```powershell
npm run test
npm run build
```

Expected: mock 集成测试通过；`finally` 后 registry 不含已运行 itemID。

- [ ] **Step 6: Commit**

```powershell
git add src/latextrans/translateCommand.ts test/latextrans/translateCommand.test.ts
git commit -m "feat(command): orchestrate translation workflow"
```

## Task 3: Zotero item 快照、菜单和进度 adapter

**Files:**
- Create: `src/modules/latextransMenu.ts`
- Modify: `src/hooks.ts`
- Modify: `src/utils/locale.ts`
- Modify: `addon/locale/zh-CN/mainWindow.ftl`
- Modify: `addon/locale/en-US/mainWindow.ftl`
- Modify: `typings/i10n.d.ts`

- [ ] **Step 1: 实现菜单模块**

创建 `src/modules/latextransMenu.ts`：

```ts
import { getString } from "../utils/locale";
import { ZoteroAttachmentApi, saveTranslatedPdfAttachment } from "../latextrans/attachmentService";
import { readLatextransConfig } from "../latextrans/preferences";
import { ZoteroResultFileSystem, resolveProjectPdf } from "../latextrans/resultResolver";
import { runLatextrans, ZoteroSubprocessExecutor } from "../latextrans/runner";
import {
  executeTranslateCommand,
  SelectedTranslateItem,
  TranslateSummary,
} from "../latextrans/translateCommand";

export function registerLatextransItemMenu(): void {
  ztoolkit.Menu.register("item", {
    tag: "menuitem",
    id: "zotero-itemmenu-zoterolatextranslate-generate-translated-pdf",
    label: getString("menu-generate-translated-pdf"),
    commandListener: () => runFromCurrentSelection(),
    icon: `chrome://${addon.data.config.addonRef}/content/icons/favicon@0.5x.png`,
  });
}

async function runFromCurrentSelection(): Promise<void> {
  const selected = ztoolkit.getGlobal("ZoteroPane").getSelectedItems();
  const selectedItems = await Promise.all(selected.map(toSelectedTranslateItem));
  const config = readLatextransConfig();
  const progress = createProgress();
  await executeTranslateCommand({
    selectedItems,
    config,
    registry: addon.data.taskRegistry,
    commandStartedAt: Date.now(),
    services: {
      async promptForSource() {
        return promptForSource();
      },
      async runLatextrans(input) {
        return runLatextrans({
          cwd: input.cwd,
          argv: input.argv,
          executor: new ZoteroSubprocessExecutor(),
          onEvent: input.onEvent,
        });
      },
      async resolvePdf(input) {
        return resolveProjectPdf({
          ...input,
          fs: new ZoteroResultFileSystem(),
        });
      },
      async saveAttachment(input) {
        return saveTranslatedPdfAttachment({
          ...input,
          api: new ZoteroAttachmentApi(),
        });
      },
      progress,
    },
  });
}

async function toSelectedTranslateItem(
  item: Zotero.Item,
): Promise<SelectedTranslateItem> {
  const attachmentIDs = item.isRegularItem() ? item.getAttachments() : [];
  const attachments = await Promise.all(
    attachmentIDs.map(async (id) => {
      const attachment = await Zotero.Items.getAsync(id);
      return {
        title: attachment.getField("title") || undefined,
        url: attachment.getField("url") || undefined,
        path: attachment.getFilePath() || undefined,
      };
    }),
  );

  return {
    itemID: item.id,
    isRegularItem: item.isRegularItem() && !(item as any).isFeedItem,
    snapshot: {
      fields: {
        url: item.getField("url") || undefined,
        DOI: item.getField("DOI") || undefined,
        extra: item.getField("extra") || undefined,
        abstractNote: item.getField("abstractNote") || undefined,
        archiveLocation: item.getField("archiveLocation") || undefined,
      },
      attachments,
    },
  };
}

function promptForSource(): string | undefined {
  const input = { value: "" };
  const ok = Services.prompt.prompt(
    Zotero.getMainWindow(),
    getString("prompt-source-title"),
    getString("prompt-source-message"),
    input,
    null,
    {},
  );
  return ok ? input.value : undefined;
}

function createProgress() {
  const win = new ztoolkit.ProgressWindow(addon.data.config.addonName, {
    closeOnClick: true,
    closeTime: -1,
  })
    .createLine({ text: getString("progress-preparing"), progress: 0 })
    .show();
  return {
    update(message: string) {
      win.changeLine({ text: message });
    },
    finish(summary: TranslateSummary) {
      win.changeLine({
        text: getString("progress-finished", {
          args: {
            succeeded: summary.succeeded,
            failed: summary.failed,
            skipped: summary.skipped,
          },
        }),
        progress: 100,
      });
      win.startCloseTimer(8000);
    },
  };
}
```

- [ ] **Step 2: 在 hooks 中注册真实菜单并移除模板 UI 示例调用**

修改 `src/hooks.ts` imports：

```ts
import { registerPrefsScripts } from "./modules/preferenceScript";
import { registerLatextransItemMenu } from "./modules/latextransMenu";
import { initLocale } from "./utils/locale";
import { createZToolkit } from "./utils/ztoolkit";
```

在 `onStartup()` 中保留：

```ts
initLocale();
await Promise.all(Zotero.getMainWindows().map((win) => onMainWindowLoad(win)));
addon.data.initialized = true;
```

在 `onMainWindowLoad()` 中保留：

```ts
addon.data.ztoolkit = createZToolkit();
win.MozXULElement.insertFTLIfNeeded(
  `${addon.data.config.addonRef}-mainWindow.ftl`,
);
registerLatextransItemMenu();
```

在 `onShutdown()` 中释放运行中任务记录：

```ts
addon.data.taskRegistry.clear();
ztoolkit.unregisterAll();
addon.data.dialog?.window?.close();
addon.data.alive = false;
// @ts-expect-error - Plugin instance is not typed
delete Zotero[addon.data.config.addonInstance];
```

删除 `BasicExampleFactory`、`KeyExampleFactory`、`UIExampleFactory`、`PromptExampleFactory`、`HelperExampleFactory` 的启动调用。

- [ ] **Step 3: 更新 locale**

在 `addon/locale/zh-CN/mainWindow.ftl` 增加：

```ftl
menu-generate-translated-pdf = 生成翻译版 PDF
prompt-source-title = 输入 TeX 来源
prompt-source-message = 请输入 arXiv ID、arXiv URL、远程源码压缩包 URL、本地源码目录或本地源码压缩包路径。
progress-preparing = 正在准备 LaTeXTransPlus 输入
progress-finished = 完成：成功 { $succeeded }，失败 { $failed }，跳过 { $skipped }
```

在 `addon/locale/en-US/mainWindow.ftl` 增加：

```ftl
menu-generate-translated-pdf = Generate translated PDF
prompt-source-title = Enter TeX source
prompt-source-message = Enter an arXiv ID, arXiv URL, remote source archive URL, local source directory, or local source archive path.
progress-preparing = Preparing LaTeXTransPlus input
progress-finished = Done: { $succeeded } succeeded, { $failed } failed, { $skipped } skipped
```

- [ ] **Step 4: 扩展 locale 加载文件**

修改 `src/utils/locale.ts` 的 `initLocale()`：

```ts
function initLocale() {
  const l10n = new (
    typeof Localization === "undefined"
      ? ztoolkit.getGlobal("Localization")
      : Localization
  )(
    [
      `${config.addonRef}-addon.ftl`,
      `${config.addonRef}-mainWindow.ftl`,
      `${config.addonRef}-preferences.ftl`,
    ],
    true,
  );
  addon.data.locale = {
    current: l10n,
  };
}
```

- [ ] **Step 5: 更新 FluentMessageId 类型**

在 `typings/i10n.d.ts` 的 `FluentMessageId` union 中加入：

```ts
  | 'menu-generate-translated-pdf'
  | 'prompt-source-title'
  | 'prompt-source-message'
  | 'progress-preparing'
  | 'progress-finished'
```

- [ ] **Step 6: 运行构建**

Run:

```powershell
npm run build
```

Expected: `hooks.ts` 不再引用模板示例 factory；菜单模块编译通过。

- [ ] **Step 7: Commit**

```powershell
git add src/modules/latextransMenu.ts src/hooks.ts src/utils/locale.ts addon/locale/zh-CN/mainWindow.ftl addon/locale/en-US/mainWindow.ftl typings/i10n.d.ts
git commit -m "feat(ui): register translated PDF menu"
```

## 自审清单

- 单选缺少 arXiv 时调用 `promptForSource()`；多选缺少 arXiv 时不弹窗。
- 多选只有一个外部 runner 调用。
- 附件保存失败只影响对应 item，不影响同 project 其他 item。
- `finally` 一定释放 `TaskRegistry` 已获取 itemID。
- `onShutdown()` 调用 `addon.data.taskRegistry.clear()`。
- 右键菜单只负责启动命令，不承载业务逻辑。
