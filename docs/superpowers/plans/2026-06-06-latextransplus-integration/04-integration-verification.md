# Integration Verification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成偏好 UI、本地化、错误反馈、mock 集成覆盖和真实 Zotero 手动验证，确保第一版符合 spec 的用户流程。

**Architecture:** 偏好 UI 只读写 Zotero preference keys；命令编排错误在 `translateCommand` 内归为 skipped/failed/succeeded 三类，UI 只展示汇总和关键诊断。最终验证包含自动测试、构建、lint 和真实 Zotero 环境检查。

**Tech Stack:** XHTML preferences、Fluent `.ftl`、TypeScript、Mocha/Chai、zotero-plugin-scaffold。

---

## Task 1: 偏好 UI 替换模板示例

**Files:**
- Modify: `addon/content/preferences.xhtml`
- Modify: `src/modules/preferenceScript.ts`
- Modify: `addon/locale/zh-CN/preferences.ftl`
- Modify: `addon/locale/en-US/preferences.ftl`
- Modify: `typings/i10n.d.ts`

- [ ] **Step 1: 替换 preferences XHTML**

将 `addon/content/preferences.xhtml` 改为：

```xml
<linkset>
  <html:link rel="localization" href="__addonRef__-preferences.ftl" />
</linkset>
<vbox
  onload="Zotero.__addonInstance__.hooks.onPrefsEvent('load', { window })"
>
  <label><html:h2 data-l10n-id="pref-title"></html:h2></label>

  <grid>
    <columns>
      <column />
      <column flex="1" />
    </columns>
    <rows>
      <row align="center">
        <html:label for="zotero-prefpane-__addonRef__-projectDir" data-l10n-id="pref-project-dir"></html:label>
        <html:input id="zotero-prefpane-__addonRef__-projectDir" preference="projectDir" type="text"></html:input>
      </row>
      <row align="center">
        <html:label for="zotero-prefpane-__addonRef__-executable" data-l10n-id="pref-executable"></html:label>
        <html:input id="zotero-prefpane-__addonRef__-executable" preference="executable" type="text"></html:input>
      </row>
      <row align="center">
        <html:label for="zotero-prefpane-__addonRef__-fixedArgs" data-l10n-id="pref-fixed-args"></html:label>
        <html:input id="zotero-prefpane-__addonRef__-fixedArgs" preference="fixedArgs" type="text"></html:input>
      </row>
      <row align="center">
        <html:label for="zotero-prefpane-__addonRef__-configPath" data-l10n-id="pref-config-path"></html:label>
        <html:input id="zotero-prefpane-__addonRef__-configPath" preference="configPath" type="text"></html:input>
      </row>
      <row align="center">
        <html:label for="zotero-prefpane-__addonRef__-outputDir" data-l10n-id="pref-output-dir"></html:label>
        <html:input id="zotero-prefpane-__addonRef__-outputDir" preference="outputDir" type="text"></html:input>
      </row>
      <row align="center">
        <html:label for="zotero-prefpane-__addonRef__-sourceLanguage" data-l10n-id="pref-source-language"></html:label>
        <html:input id="zotero-prefpane-__addonRef__-sourceLanguage" preference="sourceLanguage" type="text"></html:input>
      </row>
      <row align="center">
        <html:label for="zotero-prefpane-__addonRef__-targetLanguage" data-l10n-id="pref-target-language"></html:label>
        <html:input id="zotero-prefpane-__addonRef__-targetLanguage" preference="targetLanguage" type="text"></html:input>
      </row>
    </rows>
  </grid>

  <checkbox
    id="zotero-prefpane-__addonRef__-overwriteExistingAttachment"
    preference="overwriteExistingAttachment"
    data-l10n-id="pref-overwrite-existing-attachment"
  />

  <html:p data-l10n-id="pref-help"></html:p>
</vbox>
```

- [ ] **Step 2: 简化 preferenceScript**

将 `src/modules/preferenceScript.ts` 改为：

```ts
import { readLatextransConfig, validateLatextransConfig } from "../latextrans/preferences";
import { getString } from "../utils/locale";

/**
 * 偏好窗口加载后绑定配置校验提示。
 */
export async function registerPrefsScripts(_window: Window) {
  addon.data.prefs = {
    window: _window,
    columns: [],
    rows: [],
  };
  bindPrefEvents(_window);
}

function bindPrefEvents(win: Window) {
  const selectors = [
    "projectDir",
    "executable",
    "fixedArgs",
    "configPath",
    "outputDir",
    "sourceLanguage",
    "targetLanguage",
    "overwriteExistingAttachment",
  ];
  for (const key of selectors) {
    win.document
      ?.querySelector(`#zotero-prefpane-${addon.data.config.addonRef}-${key}`)
      ?.addEventListener("change", () => showConfigValidation(win));
  }
}

function showConfigValidation(win: Window) {
  const validation = validateLatextransConfig(readLatextransConfig());
  if (validation.ok) {
    return;
  }
  win.console.warn(
    getString("prefs-validation-error"),
    validation.errors.map((error) => error.message).join("; "),
  );
}
```

- [ ] **Step 3: 更新中文偏好 locale**

将 `addon/locale/zh-CN/preferences.ftl` 改为：

```ftl
pref-title = Zotero LaTeX Translate
pref-project-dir = LaTeXTransPlus 项目目录
pref-executable = 可执行文件路径
pref-fixed-args = 固定参数
pref-config-path = config.toml 路径
pref-output-dir = output 目录
pref-source-language = source_language
pref-target-language = target_language
pref-overwrite-existing-attachment =
    .label = 覆盖已有翻译附件
pref-help = 命令会以 argv 数组执行：可执行文件、固定参数、插件动态参数和论文来源参数依次追加。
prefs-validation-error = LaTeXTransPlus 配置不完整
```

- [ ] **Step 4: 更新英文偏好 locale**

将 `addon/locale/en-US/preferences.ftl` 改为：

```ftl
pref-title = Zotero LaTeX Translate
pref-project-dir = LaTeXTransPlus project directory
pref-executable = Executable path
pref-fixed-args = Fixed arguments
pref-config-path = config.toml path
pref-output-dir = Output directory
pref-source-language = source_language
pref-target-language = target_language
pref-overwrite-existing-attachment =
    .label = Overwrite existing translated attachments
pref-help = The command runs as argv: executable, fixed arguments, plugin dynamic arguments, and source arguments.
prefs-validation-error = LaTeXTransPlus configuration is incomplete
```

- [ ] **Step 5: 更新 FluentMessageId 类型**

在 `typings/i10n.d.ts` 的 `FluentMessageId` union 中加入：

```ts
  | 'pref-project-dir'
  | 'pref-executable'
  | 'pref-fixed-args'
  | 'pref-config-path'
  | 'pref-output-dir'
  | 'pref-source-language'
  | 'pref-target-language'
  | 'pref-overwrite-existing-attachment'
  | 'prefs-validation-error'
```

- [ ] **Step 6: 运行构建**

Run:

```powershell
npm run build
```

Expected: preferences XHTML 资源可打包；`preferenceScript.ts` 编译通过。

- [ ] **Step 7: Commit**

```powershell
git add addon/content/preferences.xhtml src/modules/preferenceScript.ts addon/locale/zh-CN/preferences.ftl addon/locale/en-US/preferences.ftl typings/i10n.d.ts
git commit -m "feat(prefs): add latextrans configuration UI"
```

## Task 2: 补齐编排错误场景测试

**Files:**
- Modify: `test/latextrans/translateCommand.test.ts`
- Modify: `src/latextrans/translateCommand.ts`

- [ ] **Step 1: 增加跳过和失败测试**

在 `test/latextrans/translateCommand.test.ts` 追加：

```ts
it("skips non-regular items and multi-select items without arxiv", async function () {
  const result = await executeTranslateCommand({
    selectedItems: [
      { itemID: 1, isRegularItem: false, snapshot: { fields: {}, attachments: [] } },
      { itemID: 2, isRegularItem: true, snapshot: { fields: {}, attachments: [] } },
      { itemID: 3, isRegularItem: true, snapshot: { fields: { url: "https://arxiv.org/abs/2508.18791" }, attachments: [] } },
    ],
    config: baseConfig(),
    registry: new TaskRegistry(),
    services: successfulServices(),
    commandStartedAt: 300,
  });

  assert.equal(result.succeeded, 1);
  assert.equal(result.failed, 0);
  assert.equal(result.skipped, 2);
});

it("marks unfinished project as failed when run_complete is false", async function () {
  const services = successfulServices();
  services.runLatextrans = async () => ({
    exitCode: 1,
    stderrLines: ["failed"],
    unparsedStdoutLines: [],
    events: [
      {
        schema_version: 1,
        type: "run_complete",
        timestamp: "2026-06-06T00:00:00Z",
        ok: false,
        status: "failed",
      },
    ],
  });

  const result = await executeTranslateCommand({
    selectedItems: [
      { itemID: 3, isRegularItem: true, snapshot: { fields: { url: "https://arxiv.org/abs/2508.18791" }, attachments: [] } },
    ],
    config: baseConfig(),
    registry: new TaskRegistry(),
    services,
    commandStartedAt: 300,
  });

  assert.equal(result.succeeded, 0);
  assert.equal(result.failed, 1);
  assert.include(result.failures[0], "2508.18791");
});

function baseConfig() {
  return {
    projectDir: "D:\\latextrans",
    executable: "python.exe",
    fixedArgs: ["main.py"],
    configPath: "config/default.toml",
    outputDir: "outputs",
    sourceLanguage: "en",
    targetLanguage: "ch",
    overwriteExistingAttachment: false,
  };
}

function successfulServices() {
  return {
    async promptForSource() {
      return undefined;
    },
    async runLatextrans() {
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
        ],
      };
    },
    async resolvePdf() {
      return {
        ok: true,
        strategy: "event-pdf-path",
        path: "D:\\latextrans\\outputs\\2508.18791\\ch_2508.18791.pdf",
      };
    },
    async saveAttachment() {
      return { ok: true, attachmentID: 100 };
    },
    async validateSource() {
      return { ok: true };
    },
    progress: {
      update() {},
      finish() {},
    },
  };
}
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```powershell
npm run test
```

Expected: 新增错误场景测试失败，指出 `run_complete` 缺少 project 结果时未计入 failed。

- [ ] **Step 3: 更新 `translateCommand.ts` 错误归类**

在 CLI 运行后、遍历 `projectToItemIDs` 时确保：

```ts
if (!event) {
  summary.failed += itemIDs.length;
  summary.failures.push(`${projectName} 没有返回 project 结果事件`);
  continue;
}

if (event.type === "project_error") {
  summary.failed += itemIDs.length;
  summary.failures.push(event.error || `${projectName} 执行失败`);
  continue;
}
```

在 runner 调用异常外层增加：

```ts
} catch (error) {
  summary.failed += acquire.acquired.length;
  summary.failures.push(error instanceof Error ? error.message : String(error));
} finally {
  input.registry.releaseMany(acquire.acquired);
}
```

- [ ] **Step 4: 运行测试**

Run:

```powershell
npm run test
npm run build
```

Expected: 跳过、执行失败、registry 释放测试都通过。

- [ ] **Step 5: Commit**

```powershell
git add src/latextrans/translateCommand.ts test/latextrans/translateCommand.test.ts
git commit -m "test(command): cover skipped and failed translations"
```

## Task 3: 配置文件系统校验

**Files:**
- Modify: `src/latextrans/preferences.ts`
- Modify: `src/modules/latextransMenu.ts`
- Test: `test/latextrans/preferences.test.ts`

- [ ] **Step 1: 写文件系统校验测试**

在 `test/latextrans/preferences.test.ts` 追加：

```ts
import { validateLatextransFileConfig } from "../../src/latextrans/preferences";

it("reports missing project dir, config file and output creation failure", async function () {
  const result = await validateLatextransFileConfig(
    {
      projectDir: "D:\\missing",
      executable: "python.exe",
      fixedArgs: [],
      configPath: "config/default.toml",
      outputDir: "outputs",
      sourceLanguage: "en",
      targetLanguage: "ch",
      overwriteExistingAttachment: false,
    },
    {
      async exists(path) {
        return path === "C:\\Python311\\python.exe";
      },
      async ensureDir() {
        return false;
      },
      async resolveExecutable(name) {
        return name === "python.exe" ? "C:\\Python311\\python.exe" : undefined;
      },
      resolve(base, path) {
        return `${base}\\${path}`;
      },
    },
  );

  assert.deepEqual(result.errors.map((error) => error.code), [
    "projectDir.missing",
    "configPath.missing",
    "outputDir.uncreatable",
  ]);
});
```

- [ ] **Step 2: 实现文件系统校验接口**

在 `src/latextrans/preferences.ts` 增加：

```ts
export interface ConfigFileSystem {
  exists(path: string): Promise<boolean>;
  ensureDir(path: string): Promise<boolean>;
  resolveExecutable(executable: string): Promise<string | undefined>;
  resolve(base: string, path: string): string;
}

export interface FileConfigValidationError {
  code:
    | "projectDir.missing"
    | "executable.missing"
    | "configPath.missing"
    | "outputDir.uncreatable";
  message: string;
}

export async function validateLatextransFileConfig(
  config: LatextransConfig,
  fs: ConfigFileSystem,
): Promise<{ ok: boolean; errors: FileConfigValidationError[] }> {
  const errors: FileConfigValidationError[] = [];
  if (!(await fs.exists(config.projectDir))) {
    errors.push({ code: "projectDir.missing", message: "LaTeXTransPlus 项目目录不存在" });
  }
  const executablePath = await fs.resolveExecutable(config.executable);
  if (!executablePath) {
    errors.push({ code: "executable.missing", message: "可执行文件不存在或无法解析" });
  }
  const configPath = fs.resolve(config.projectDir, config.configPath);
  if (!(await fs.exists(configPath))) {
    errors.push({ code: "configPath.missing", message: "config.toml 不存在" });
  }
  const outputPath = fs.resolve(config.projectDir, config.outputDir);
  if (!(await fs.ensureDir(outputPath))) {
    errors.push({ code: "outputDir.uncreatable", message: "output 目录不存在且无法创建" });
  }
  return { ok: errors.length === 0, errors };
}
```

- [ ] **Step 3: 在菜单入口执行前阻断配置错误**

在 `src/modules/latextransMenu.ts` 中读取 config 后加入：

```ts
const fieldValidation = validateLatextransConfig(config);
if (!fieldValidation.ok) {
  showConfigErrors(fieldValidation.errors.map((error) => error.message));
  return;
}
const fileValidation = await validateLatextransFileConfig(config, new ZoteroConfigFileSystem());
if (!fileValidation.ok) {
  showConfigErrors(fileValidation.errors.map((error) => error.message));
  return;
}
```

同文件增加：

```ts
class ZoteroConfigFileSystem {
  async exists(path: string): Promise<boolean> {
    return IOUtils.exists(path);
  }

  async ensureDir(path: string): Promise<boolean> {
    try {
      await IOUtils.makeDirectory(path, { createAncestors: true });
      return true;
    } catch {
      return false;
    }
  }

  async resolveExecutable(executable: string): Promise<string | undefined> {
    if (PathUtils.isAbsolute(executable) && (await IOUtils.exists(executable))) {
      return executable;
    }
    const envPath = Services.env.get("PATH") || "";
    const separator = Zotero.isWin ? ";" : ":";
    const extensions = Zotero.isWin ? ["", ".exe", ".cmd", ".bat"] : [""];
    for (const dir of envPath.split(separator).filter(Boolean)) {
      for (const ext of extensions) {
        const candidate = PathUtils.join(dir, executable.endsWith(ext) ? executable : `${executable}${ext}`);
        if (await IOUtils.exists(candidate)) {
          return candidate;
        }
      }
    }
    return undefined;
  }

  resolve(base: string, path: string): string {
    return PathUtils.isAbsolute(path) ? path : PathUtils.join(base, path);
  }
}

function showConfigErrors(messages: string[]): void {
  Services.prompt.alert(
    Zotero.getMainWindow(),
    getString("prefs-validation-error"),
    messages.join("\n"),
  );
}
```

- [ ] **Step 4: 校验用户输入的本地 project 路径**

在 `src/latextrans/sourceResolver.ts` 增加：

```ts
export interface SourceFileSystem {
  exists(path: string): Promise<boolean>;
}

export async function validateResolvedInputSource(
  source: ResolvedInputSource,
  fs: SourceFileSystem,
): Promise<{ ok: true } | { ok: false; reason: "local-path-missing" }> {
  if (source.kind !== "project") {
    return { ok: true };
  }
  return (await fs.exists(source.value))
    ? { ok: true }
    : { ok: false, reason: "local-path-missing" };
}
```

在 `src/latextrans/translateCommand.ts` 的单选用户输入分支中，`classifyUserInput()` 成功后调用服务校验：

```ts
const classified = classifyUserInput(userInput);
if (classified.ok) {
  const validation = await services.validateSource(classified.source);
  if (validation.ok) {
    runnable.push({ ...item, source: classified.source });
  } else {
    skipped += 1;
  }
} else {
  skipped += 1;
}
```

将 `TranslateServices` 增加：

```ts
validateSource(
  source: ResolvedInputSource,
): Promise<{ ok: true } | { ok: false; reason: "local-path-missing" }>;
```

在 `src/modules/latextransMenu.ts` 的真实 service 中实现：

```ts
async validateSource(source) {
  return validateResolvedInputSource(source, {
    async exists(path) {
      return IOUtils.exists(path);
    },
  });
}
```

在 `test/latextrans/translateCommand.test.ts` 增加单选本地路径不存在测试：

```ts
it("skips single selected local project input when the path does not exist", async function () {
  const services = successfulServices();
  services.promptForSource = async () => "D:\\missing\\source.zip";
  services.validateSource = async () => ({ ok: false, reason: "local-path-missing" });

  const result = await executeTranslateCommand({
    selectedItems: [
      { itemID: 3, isRegularItem: true, snapshot: { fields: {}, attachments: [] } },
    ],
    config: baseConfig(),
    registry: new TaskRegistry(),
    services,
    commandStartedAt: 300,
  });

  assert.equal(result.succeeded, 0);
  assert.equal(result.failed, 0);
  assert.equal(result.skipped, 1);
});
```

- [ ] **Step 5: 运行测试与构建**

Run:

```powershell
npm run test
npm run build
```

Expected: 配置文件系统校验测试通过；配置错误会在 runner 前阻断。

- [ ] **Step 6: Commit**

```powershell
git add src/latextrans/preferences.ts src/latextrans/sourceResolver.ts src/latextrans/translateCommand.ts src/modules/latextransMenu.ts test/latextrans/preferences.test.ts test/latextrans/translateCommand.test.ts
git commit -m "feat(config): validate latextrans runtime paths"
```

## Task 4: 最终自动验证

**Files:**
- All modified implementation and test files

- [ ] **Step 1: 运行完整测试**

Run:

```powershell
npm run test
```

Expected: 所有测试通过，至少包含：

- `preferences.test.ts`
- `argv.test.ts`
- `sourceResolver.test.ts`
- `taskRegistry.test.ts`
- `events.test.ts`
- `runner.test.ts`
- `resultResolver.test.ts`
- `attachmentService.test.ts`
- `translateCommand.test.ts`
- `startup.test.ts`

- [ ] **Step 2: 运行构建**

Run:

```powershell
npm run build
```

Expected: `zotero-plugin build` 成功，`tsc --noEmit` 无错误。

- [ ] **Step 3: 运行 lint**

Run:

```powershell
npm run lint:check
```

Expected: Prettier 和 ESLint 均通过。

- [ ] **Step 4: 检查 diff 只包含任务相关变更**

Run:

```powershell
git status --short
git diff --stat
```

Expected: diff 只包含 `src/latextrans/`、菜单和偏好相关文件、locale、测试与计划文档。

## Task 5: 真实 Zotero 手动验证

**Files:**
- No source changes expected

- [ ] **Step 1: 启动开发版插件**

Run:

```powershell
npm run start
```

Expected: Zotero 加载插件，`startup.test.ts` 仍能证明插件实例存在。

- [ ] **Step 2: 配置 LaTeXTransPlus**

在 Zotero preferences 中填写：

```text
LaTeXTransPlus 项目目录: D:\Workspace\tools\LaTeXTransPlus
可执行文件路径: D:\envs\latextrans\Scripts\python.exe
固定参数: main.py
config.toml 路径: config/default.toml
output 目录: outputs
source_language: en
target_language: ch
覆盖已有翻译附件: 关闭
```

Expected: 关闭并重新打开 preferences 后，值仍保留。

- [ ] **Step 3: 验证单选 arXiv 条目**

准备一个普通条目，`url` 为：

```text
https://arxiv.org/abs/2508.18791
```

右键点击 `生成翻译版 PDF`。

Expected:

- ProgressWindow 显示准备、运行和完成状态。
- LaTeXTransPlus 命令包含 `--json-events stdout` 和 `--arxiv 2508.18791`。
- 生成 PDF 被导入为当前条目的 stored child attachment。
- 附件标题为 `翻译版 PDF (ch)`，tag 包含 `latextransplus-translated`。

- [ ] **Step 4: 验证单选非 arXiv 输入**

准备一个无 arXiv 的普通条目，右键点击 `生成翻译版 PDF`，在弹窗输入：

```text
https://example.org/source.zip
```

Expected:

- 插件使用 `--project-url https://example.org/source.zip`。
- 用户取消弹窗时，该条目计入 skipped，不启动 LaTeXTransPlus。

- [ ] **Step 5: 验证多选批量**

选择三个普通条目：

```text
item A url: https://arxiv.org/abs/2508.18791
item B extra: arXiv: 2508.18792v2
item C no arXiv
```

右键点击 `生成翻译版 PDF`。

Expected:

- 只启动一次 LaTeXTransPlus。
- 命令包含 `--arxiv 2508.18791,2508.18792v2`。
- item C 不弹窗并计入 skipped。
- item A 和 item B 分别获得自己的 child attachment。

- [ ] **Step 6: 验证重复任务去重**

在一个 arXiv 条目正在运行时，再次对同一条目点击 `生成翻译版 PDF`。

Expected:

- 第二次触发不启动新 runner。
- 该条目计入 skipped。
- 第一次任务结束后再次触发可以正常运行。

## 最终交付

- [ ] **Step 1: 汇总验证结果**

记录以下命令的最终状态：

```powershell
npm run test
npm run build
npm run lint:check
git status --short
```

- [ ] **Step 2: 提供最终 commit message 建议**

建议：

```text
feat(latextrans): integrate external translated PDF workflow
```

## 自审清单

- 偏好 UI 不再包含模板示例表格。
- 中文和英文 locale key 与 `getString()` key 一致。
- 配置错误在启动 runner 前阻断。
- `run_complete ok=false` 且缺少 project 结果事件时，已登记 item 计入 failed。
- 手动验证覆盖单选 arXiv、单选用户输入、多选 arXiv、重复任务、附件保存。
