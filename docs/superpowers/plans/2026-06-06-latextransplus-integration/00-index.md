# LaTeXTransPlus 集成 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 Zotero 普通条目增加“生成翻译版 PDF”入口，调用外部 LaTeXTransPlus CLI，并把生成 PDF 保存回条目子附件。

**Architecture:** 插件只作为 Zotero 集成层，核心流程拆成配置读取、来源解析、外部 runner、JSON Lines 事件解析、结果 PDF 定位、附件导入和命令编排。纯逻辑模块先行实现并用单元测试锁定行为，Zotero API 依赖集中在 UI、附件和命令入口模块。

**Tech Stack:** TypeScript、zotero-plugin-scaffold、zotero-plugin-toolkit、Zotero 7 plugin APIs、Mocha/Chai、LaTeXTransPlus CLI JSON Lines events。

---

## 执行顺序

按下列文档顺序实现。每个聚焦计划完成后运行该文档的验证命令并提交一次 commit。

1. `01-preferences-source-task-registry.md`
2. `02-runner-events-results.md`
3. `03-attachment-command-ui.md`
4. `04-integration-verification.md`

## 文件结构总览

- Create: `src/latextrans/preferences.ts`，读取和校验 LaTeXTransPlus 配置。
- Create: `src/latextrans/argv.ts`，解析 fixed args 并组装 CLI argv。
- Create: `src/latextrans/sourceResolver.ts`，从 Zotero item 或用户输入解析 `arxiv`、`project-url`、`project`。
- Create: `src/latextrans/taskRegistry.ts`，以内存 Set 维护运行中 itemID。
- Create: `src/latextrans/events.ts`，解析 LaTeXTransPlus JSON Lines。
- Create: `src/latextrans/runner.ts`，调用外部进程并收集 stdout/stderr/events。
- Create: `src/latextrans/resultResolver.ts`，按事件、精确路径、fallback 扫描定位 PDF。
- Create: `src/latextrans/attachmentService.ts`，把 PDF 导入为 stored child attachment。
- Create: `src/latextrans/translateCommand.ts`，编排选择、跳过、运行、附件保存和汇总。
- Create: `src/modules/latextransMenu.ts`，注册右键菜单入口。
- Modify: `src/hooks.ts`，移除模板示例启动逻辑并注册真实入口。
- Modify: `src/utils/locale.ts`，让 `getString()` 能读取新增 mainWindow/preferences 文案。
- Modify: `src/addon.ts`，在插件 data 中挂载 `taskRegistry`。
- Modify: `src/modules/preferenceScript.ts`，替换示例偏好 UI 绑定。
- Modify: `addon/content/preferences.xhtml`，替换为 LaTeXTransPlus 配置表单。
- Modify: `addon/prefs.js`，增加真实偏好默认值。
- Modify: `typings/prefs.d.ts`，声明插件偏好键。
- Modify: `typings/i10n.d.ts`，声明新增 Fluent message IDs。
- Modify: `addon/locale/zh-CN/*.ftl` 与 `addon/locale/en-US/*.ftl`，补齐菜单、偏好和反馈文案。
- Create/Modify: `test/latextrans/*.test.ts`，覆盖纯逻辑和 mock 集成流程。

## 全局约束

- 命令必须用 argv 数组组装，不拼接 shell 字符串。
- 每次运行必须追加 `--json-events stdout`。
- 多选场景只批量处理可自动识别 arXiv ID 的普通条目。
- 单选未识别 arXiv 时才弹出来源输入框。
- 所有已登记 itemID 必须在 `finally` 中释放。
- 批量 PDF 定位禁止只扫描全局 output 目录并选择最新 PDF。
- 附件第一版只创建 stored child attachment。

## 推荐提交切分

1. `feat(config): add latextrans preferences and source resolver`
2. `feat(runner): parse latextrans events and resolve PDFs`
3. `feat(command): add Zotero menu attachment import workflow`
4. `test(integration): cover latextrans command orchestration`

## 最终验证

在全部计划完成后运行：

```powershell
npm run build
npm run test
npm run lint:check
```

期望结果：

- `npm run build` 通过 `zotero-plugin build` 与 `tsc --noEmit`。
- `npm run test` 所有 Mocha/Chai 测试通过。
- `npm run lint:check` 没有 Prettier 或 ESLint 错误。
