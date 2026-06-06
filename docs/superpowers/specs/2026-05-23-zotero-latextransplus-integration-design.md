# Zotero 条目生成 LaTeXTransPlus 翻译 PDF 设计

## 背景

本项目当前基于 `zotero-plugin-template`，目标是开发成一个 Zotero 插件：用户选中普通条目后，插件自动识别论文的 arXiv 编号或链接，并调用本机 LaTeXTransPlus 生成翻译版 PDF，再把生成的 PDF 保存回该 Zotero 条目。

LaTeXTransPlus 位于用户配置的项目目录中，插件不内置翻译逻辑。当前 LaTeXTransPlus 已支持以下输入：

- `--arxiv`：arXiv ID 或 arXiv URL。
- `--project`：本地 TeX 项目目录或本地源码压缩包。
- `--project-url`：远程 TeX 源码压缩包 URL。
- `--source_language` 与 `--target_language`：覆盖配置文件中的源语言和目标语言。
- `--output`：覆盖输出目录。
- `--json-events stdout`：将 JSON Lines 事件输出到 stdout。
- `--json-events-file <path>`：将 JSON Lines 事件写入 UTF-8 JSONL 文件。

同时，LaTeXTransPlus CLI 已提供 JSON Lines 事件流。插件调用 CLI 时应追加 `--json-events stdout`，并把 stdout 中每一行 JSON 对象作为结构化事件处理，用于项目级状态展示、错误摘要和生成 PDF 路径定位。启用该模式后，LaTeXTransPlus 会把普通 workflow 日志从 stdout 移走；插件仍应保留 stderr 和事件中的 `log_path` 供诊断。

## 目标

- 为 Zotero 普通条目提供手动触发入口。
- 支持用户一次选择多个普通条目；多选时只处理能自动识别出 arXiv ID 的条目，并合并为一次 LaTeXTransPlus `--arxiv` 批量调用。
- 自动从条目元数据中提取 arXiv ID 或 arXiv URL。
- 单选且未检测到 arXiv 时，主动要求用户输入 TeX 源码下载链接或本地路径；多选时不弹出输入框，缺少 arXiv 的条目直接跳过。
- 使用用户配置的外部 runner 调用 LaTeXTransPlus，不绑定 conda、venv 或系统 Python。
- 在 LaTeXTransPlus 执行成功后，将翻译版 PDF 作为当前条目的子附件保存到 Zotero。
- 对正在翻译的条目做任务去重，再次触发时提示而不是新建重复任务。
- 对配置错误、输入错误、执行错误和附件保存错误给出清晰反馈。

## 非目标

- 不在插件内实现 TeX 源码下载、解压、翻译、校验或 PDF 编译。
- 第一版不做自动监听新增条目或自动批量后台翻译。
- 第一版不实现 LaTeXTransPlus 常驻服务。
- 第一版不解析 DOI 页面、HTML 下载页或论文仓储 record 页面。
- 第一版不把 LaTeXTransPlus 打包进 Zotero 插件。

## 推荐方案

采用“Zotero 插件作为集成层，LaTeXTransPlus 作为外部翻译引擎”的设计。

插件负责：

- Zotero 菜单入口。
- 条目类型检查。
- arXiv 来源识别。
- 用户输入来源收集。
- 外部命令组装与执行。
- 结果 PDF 定位。
- PDF 附件保存。

LaTeXTransPlus 负责：

- arXiv 源码获取。
- 远程源码包下载。
- 本地源码项目处理。
- LaTeX 翻译、校验、重构和编译。
- 输出日志、报告和 PDF。

这个边界可以让插件保持较薄，避免把 Python 环境、LLM 调用、TeX 编译和下载逻辑扩散到 Zotero 插件内。

## 备选方案与取舍

### 方案 A：插件直接调用 LaTeXTransPlus CLI

这是第一版推荐方案。插件通过用户配置的 executable 和 fixed args 调用外部命令，再追加 LaTeXTransPlus 参数。

优点：

- 实现路径短。
- 与 LaTeXTransPlus 当前能力匹配。
- 不绑定具体 Python 环境。
- 插件与翻译引擎边界清楚。
- 可以通过 JSON Lines 事件流获得项目级状态、错误摘要、日志路径和生成 PDF 路径。

缺点：

- 需要维护 CLI JSON Lines 事件 schema 的兼容约定。
- 事件流缺失或字段不完整时，仍需要目录规则和 fallback 扫描。

### 方案 B：插件调用 LaTeXTransPlus Python API

插件直接进入 Python 模块接口获取结构化结果。

优点：

- 可以精确拿到 `pdf_path` 和项目状态。
- 进度事件更容易细化。

缺点：

- Zotero 插件需要处理 Python 解释器、依赖和模块路径。
- 跨平台与 Windows 路径问题更复杂。
- 对 LaTeXTransPlus 内部 API 稳定性要求更高。

### 方案 C：LaTeXTransPlus 提供本地常驻服务

插件通过 HTTP 或 IPC 请求本地服务执行翻译。

优点：

- 用户体验最好，适合队列、取消、实时进度和状态查询。

缺点：

- 第一版过重。
- 需要处理服务启动、端口、安全、生命周期和异常恢复。

第一版选择方案 A，并把 LaTeXTransPlus CLI 的 JSON Lines 事件流作为首选状态与结果来源。这样 CLI 边界仍然清晰，插件也能获得机器可读的项目状态、错误和 PDF 路径。

## 用户配置

插件偏好设置需要提供以下配置：

- `LaTeXTransPlus 项目目录`：外部命令的工作目录。
- `可执行文件路径`：例如 `python.exe`、`latextrans.exe`、`uv.exe`、`conda.exe`，也可以是绝对路径。
- `固定参数`：追加在 executable 后、插件动态参数前，例如 `main.py` 或 `run -n latextrans python main.py`。
- `config.toml 路径`：可选，默认 `config/default.toml`。
- `output 目录`：可选，默认 `outputs`。
- `source_language`：可选，默认 `en`。
- `target_language`：可选，默认 `ch`。
- `覆盖已有翻译附件`：可选，默认不覆盖，避免误删用户已有附件。

命令必须按 argv 数组组装，而不是把完整字符串拼接后交给 shell。这样可以减少 Windows 空格路径和引号转义问题。

示例配置：

```text
项目目录: D:\Workspace\tools\LaTeXTransPlus
可执行文件路径: D:\envs\latextrans\Scripts\python.exe
固定参数: main.py
config.toml 路径: config/default.toml
output 目录: outputs
source_language: en
target_language: ch
```

## 模块设计

### `preferences`

负责读取、校验和保存插件配置。

职责：

- 读取 LaTeXTransPlus 项目目录。
- 读取 executable 与 fixed args。
- 读取 config、output、source language、target language。
- 提供配置校验结果。

### `sourceResolver`

负责把 Zotero 条目或用户输入解析成 LaTeXTransPlus 支持的来源类型。

来源类型：

- `arxiv`：对应 `--arxiv`。
- `project-url`：对应 `--project-url`。
- `project`：对应 `--project`。

自动检测字段：

- `url`
- `DOI`
- `extra`
- `abstractNote`
- `archiveLocation`
- 子附件的 URL、title 或 path 作为辅助信号

单选且未检测到 arXiv 时，弹出输入框要求用户输入 TeX 源码来源。用户可输入 arXiv ID、arXiv URL、远程源码压缩包 URL、本地项目目录或本地源码压缩包。多选时，`sourceResolver` 只返回自动检测到的 arXiv 来源，未检测到 arXiv 的条目由 `translateCommand` 记为 skipped。

### `latextransRunner`

负责组装并执行外部命令，并按行读取 LaTeXTransPlus stdout 中的 JSON Lines 事件流。

命令结构：

```text
cwd = LaTeXTransPlus 项目目录
argv = [
  executable,
  ...fixedArgs,
  "--config", configPath,
  "--output", outputDir,
  "--source_language", sourceLanguage,
  "--target_language", targetLanguage,
  "--json-events", "stdout",
  sourceFlag,
  sourceValueOrValues
]
```

当多选批量处理 arXiv 条目时，`sourceFlag` 为 `--arxiv`，`sourceValueOrValues` 使用逗号分隔的 arXiv ID 列表，例如 `2501.00001,2501.00002`。插件不在多选场景生成 `--project` 或 `--project-url` 批量命令。

`latextransRunner` 不负责识别 Zotero 条目，也不负责保存附件。它只暴露外部进程生命周期、原始 stdout/stderr 片段和已解析的 CLI 事件。

### `latextransEventParser`

负责把 LaTeXTransPlus CLI 输出的 JSON Lines 转换成插件内部事件。

解析规则：

- stdout 按行切分，每行独立解析为 JSON 对象。
- 空行忽略。
- 启用 `--json-events stdout` 后，stdout 预期只包含 JSON Lines；无法解析为 JSON 的行不作为结构化事件处理，但保留到原始 stdout 诊断日志并提示事件流异常。
- 未识别的事件类型不报错，只记录调试日志，避免 LaTeXTransPlus 后续增加事件时破坏插件。
- 每条事件应包含 `schema_version`、`type` 和 `timestamp`。第一版只支持 `schema_version = 1`，未来版本不兼容时降级为阶段级提示和原始日志诊断。

第一版事件类型：

- `run_start`：携带 `total`、`config_path`、`output_dir`、`source_language`、`target_language`。
- `project_start`：标记单个 LaTeXTransPlus project 开始处理。
- `project_complete`：携带 `project_name`、`pdf_path`、`log_path` 等单个 project 最终产物信息。
- `project_error`：携带 `project_name`、可展示的错误摘要和可选日志路径。
- `run_complete`：携带本次 CLI 批处理的总成功、失败状态。

项目事件的稳定字段：

- `index` 与 `total`：当前 project 在本次 CLI 运行中的位置。
- `project_name` 与 `project_dir`：LaTeXTransPlus project 标识和源码目录。
- `output_dir` 与 `log_path`：该 project 的输出目录和日志路径。
- `pdf_path`、`errors_report_path`、`validation_summary`、`error`：仅在 `project_complete` 或 `project_error` 中稳定读取。
- `status`、`project_terms_path`、`project_terms_decisions_path`：仅在 LaTeXTransPlus workflow 结果提供时透传，插件不应假定必然存在。

### `resultResolver`

负责在 LaTeXTransPlus 结束后定位生成的 PDF。

第一版采用三级策略：

1. 优先使用 JSON Lines `project_complete` 事件中的 `pdf_path`。如果该路径是相对路径，按 LaTeXTransPlus 项目目录解析为绝对路径。
2. 如果事件流没有提供有效 `pdf_path`，优先根据同一 project 事件中的 `output_dir` 和 `project_name` 查找：

   ```text
   <project_output_dir>/<target_language>_<projectName>.pdf
   ```

3. 如果精确路径未命中，在该 project 的 `output_dir` 下扫描本次命令开始时间之后新增或更新的 PDF，选择最新的 PDF。

如果 JSON Lines 事件提供的 `pdf_path` 不存在或不可读，记录该路径并继续 fallback。批量场景禁止只扫描全局 `<output>` 并选择最新 PDF，因为这会把其他 project 的 PDF 误挂到当前 Zotero 条目。

### `attachmentService`

负责将 PDF 保存到 Zotero 条目。

行为：

- 将 LaTeXTransPlus 生成的 PDF 导入为当前普通条目的 stored child attachment，即复制到 Zotero storage。
- 第一版不创建 linked attachment，避免 LaTeXTransPlus 输出目录被清理、移动或覆盖后导致 Zotero 附件失效。
- 附件标题建议为 `翻译版 PDF (<target_language>)`。
- 可选添加 tag：`latextransplus-translated`。
- 如果已有同名翻译附件，默认保留并创建新附件；只有用户启用“覆盖已有翻译附件”时才替换。

### `translateCommand`

右键菜单和主编排模块。

职责：

- 获取 Zotero 当前选中条目。
- 过滤非普通条目。
- 多选时批量处理自动识别出的 arXiv 条目，并跳过缺少 arXiv 的条目。
- 调用 `sourceResolver`、`latextransRunner`、`resultResolver` 和 `attachmentService`。
- 管理 ProgressWindow 和错误提示。

### `taskRegistry`

负责维护插件进程内正在运行的翻译任务。

职责：

- 以 Zotero itemID 作为 key 记录运行中的条目。
- 在任务入队前判断当前条目是否已经在运行。
- 任务结束、失败或跳过时释放对应 itemID。
- 插件关闭或卸载时清空 registry。

第一版只需要内存级 registry，不需要持久化到 Zotero 数据库。Zotero 重启后，插件无法可靠恢复外部 LaTeXTransPlus 进程状态，因此不承诺跨重启去重。

## 多选处理

第一版支持多选，但多选只处理能自动识别出 arXiv ID 的普通条目。插件把这些 arXiv ID 合并为一次 LaTeXTransPlus `--arxiv` 批量调用，由 LaTeXTransPlus 在单个外部进程内逐项目处理。这样可以利用 LaTeXTransPlus 的批处理能力，同时避免插件侧并发启动多个翻译进程。

多选规则：

- 从 Zotero 当前选择中筛出普通条目。
- attachment、note、feed item 等非普通条目直接记为 skipped。
- 已经存在运行中任务的普通条目不再入队，直接提示“该条目正在翻译”，并记为 skipped。
- 对剩余普通条目逐个执行 arXiv 检测。
- 多选时不弹出用户输入框；未检测到 arXiv 的普通条目直接记为 skipped。
- 对检测到 arXiv 的条目建立 `latexProjectName -> itemID[]` 映射。
- `latexProjectName` 必须与 LaTeXTransPlus JSON Lines 事件中的 `project_name` 完全一致。第一版对 arXiv 来源，插件从 arXiv URL 中提取 ID 后保留版本号，并使用该 ID 作为传给 `--arxiv` 的值和映射 key。
- 第一版只承诺支持新式 arXiv ID，例如 `2508.18791`、`2508.18791v2`。旧式分类 ID，例如 `hep-th/9901001`，可检测但不作为第一版自动批量映射的承诺能力，除非后续 LaTeXTransPlus 事件增加稳定的原始 `source_id` 字段。
- 同一批中重复的 arXiv ID 只传给 LaTeXTransPlus 一次；生成成功后，结果 PDF 分别保存到所有对应 Zotero 条目。
- 如果没有任何可处理的 arXiv 条目，不启动 LaTeXTransPlus，只显示 skipped 汇总。
- LaTeXTransPlus 批处理中单个 project 失败时，只标记映射到该 `latexProjectName` 的条目失败；其他 project 的成功结果仍继续保存附件。
- 全部完成后显示批量汇总：成功数、失败数、跳过数。

汇总计数默认按 Zotero item 级统计，而不是按 LaTeXTransPlus project 级统计。一个 arXiv project 映射到多个 Zotero 条目时，每个条目的附件保存结果独立计入成功或失败。CLI project 级成功但某个条目附件保存失败时，该条目标记为附件失败，不影响同一 project 映射下其他条目的保存结果。

用户输入规则：

- 单选时，如果条目自动检测到 arXiv，不弹出输入框。
- 单选时，如果条目未检测到 arXiv，则弹出输入框要求用户输入 TeX 源码来源。
- 多选时不要求用户补充 TeX 源码来源；无 arXiv 的条目一律 skipped。

附件命名规则：

- 每个条目独立保存子附件。
- 同一批中多个条目映射到同一个 arXiv ID 时，共用同一个生成 PDF，但分别保存为各自条目的子附件。
- 默认附件标题为 `翻译版 PDF (<target_language>)`。
- 如果一个条目已存在同名翻译附件，按用户配置决定保留旧附件并新增，或覆盖旧附件。

## 执行流程

1. 用户在 Zotero 中选中一个或多个普通条目。
2. 用户右键点击 `生成翻译版 PDF`。
3. 插件读取当前选中项。
4. 插件将非普通条目标记为 skipped。
5. 插件检查每个普通条目是否已经在 `taskRegistry` 中运行。
6. 已在运行的条目提示“该条目正在翻译”，不新建任务，并标记为 skipped。
7. 插件从未运行的普通条目字段中提取 arXiv。
8. 如果是单选且未识别到 arXiv，弹窗要求用户输入 TeX 源码链接或本地路径。
9. 如果是多选且未识别到 arXiv，将该条目标记为 skipped，不弹窗。
10. 插件分类可处理输入：
   - arXiv ID 或 arXiv URL -> `--arxiv`
   - `http/https` URL -> `--project-url`
   - 本地路径 -> `--project`
11. 插件校验 LaTeXTransPlus 配置和可处理输入。
12. 单选时，插件登记当前 itemID 到 `taskRegistry`，并按单个来源启动外部命令。
13. 多选时，插件只登记检测到 arXiv 的 itemID 到 `taskRegistry`，建立 `latexProjectName -> itemID[]` 映射，并构造单个 `--arxiv` 批量输入。
14. 插件启动外部命令。登记到 `taskRegistry` 之后，所有成功、失败、取消、配置异常和附件异常路径都必须在统一 `finally` 中释放对应 itemID。
15. 命令执行期间按行解析 JSON Lines 事件，并根据 `run_start`、`project_start`、`project_complete`、`project_error` 更新 ProgressWindow。
16. 命令退出后，插件优先根据每个 project 的 `project_complete` 或 `project_error` 事件记录结果。
17. 对成功 project，插件使用事件中的 `pdf_path` 定位 PDF，必要时再使用目录规则和 fallback 扫描。
18. 插件将 PDF 保存为映射到该 project 的 Zotero 条目的子附件。
19. 插件记录每个条目的成功、失败或跳过状态，并从 `taskRegistry` 释放对应 itemID。
20. 本次运行结束后显示批量汇总。

## 错误处理

### 配置错误

- LaTeXTransPlus 项目目录不存在。
- executable 不存在且无法从 PATH 解析。
- config 文件不存在。
- output 目录不存在且无法创建。
- source language 或 target language 为空。

处理方式：在执行前阻断，并提示用户打开偏好设置修正配置。

### 输入错误

- 当前选择不是普通 Zotero 条目。
- 当前条目已经有运行中的翻译任务。
- 单选时未检测到 arXiv，且用户取消输入。
- 多选时普通条目未检测到 arXiv。
- 用户输入不是 arXiv、URL 或本地路径。
- 本地路径不存在。

处理方式：跳过当前条目，批量场景继续处理后续条目。

### 执行错误

- 外部进程启动失败。
- LaTeXTransPlus 返回非 0 退出码，或 JSON Lines `project_error` 事件表示某个 project 执行失败。
- LaTeXTransPlus 执行完成后，事件中的 `pdf_path`、输出目录规则和 fallback 扫描都找不到 PDF。

处理方式：记录当前条目失败，优先显示 `project_error` 事件中的简短错误；如果没有结构化错误，则显示退出码和通用错误。反馈中附带 JSON Lines 事件中的 `log_path`、stderr 或 LaTeXTransPlus `latextrans.log` 的路径。批量场景继续处理后续条目。如果只收到 `run_complete ok=false` 而没有对应 project 事件，则把仍未完成的已登记条目标记为执行失败。

### 附件错误

- PDF 文件不存在或不可读。
- Zotero 附件导入失败。
- 保存条目失败。

处理方式：记录当前条目失败，显示 PDF 路径，提示用户可手动导入，并保留 LaTeXTransPlus 输出。批量场景继续处理后续条目。

## 进度与用户反馈

第一版优先使用 JSON Lines 事件流更新项目级进度：

- 准备输入。
- 运行 LaTeXTransPlus，并根据 `project_start` 显示当前 project 位置和名称。
- 查找生成 PDF。
- 保存到 Zotero。
- 完成或失败。

多选场景下，ProgressWindow 需要根据 JSON Lines 事件显示当前 project 位置，例如 `[2/5] 运行 LaTeXTransPlus: 2508.18791`，并在全部完成后显示成功、失败和跳过数量。

LaTeXTransPlus 第一版事件流只承诺项目级事件，不承诺 parse、translate、validate、compile 等内部阶段或百分比。插件不解析非 JSON stdout 作为精确进度。事件流缺失、字段缺失或事件类型未知时，ProgressWindow 降级为阶段级进度，并保留原始 stdout/stderr 供诊断。

## 测试设计

### 单元测试

覆盖以下纯逻辑：

- arXiv ID 提取。
- arXiv URL 提取。
- DOI、extra、abstractNote 中的 arXiv 提取。
- 用户输入分类。
- 多选 arXiv ID 去重与 `latexProjectName -> itemID[]` 映射。
- fixed args 解析。
- argv 组装。
- JSON Lines 事件解析。
- `schema_version`、`type`、`timestamp` 基础字段校验。
- 未知事件和非 JSON 行容错。
- `project_complete` 事件中的 `pdf_path` 解析。
- `project_start` / `project_complete` / `project_error` 的 `project_name` 到 Zotero item 映射。
- 输出 PDF 精确路径定位。
- 输出 PDF fallback 扫描。
- 配置校验。

### 集成测试

使用 mock runner 覆盖：

- 外部命令成功。
- 外部命令追加 `--json-events stdout`。
- 外部命令输出 JSON Lines `run_start`、`project_start`、`project_complete`、`run_complete` 事件。
- 外部命令失败。
- 外部命令输出 JSON Lines `project_error` 事件。
- JSON Lines 事件缺失 `pdf_path` 时，在对应 project 的 `output_dir` 下 fallback 定位 PDF。
- `run_complete ok=false` 但缺少 project 结果事件时，未完成条目会被标记为失败。
- 命令成功但 PDF 不存在。
- PDF 附件保存失败。
- 多选 arXiv 条目合并为一次 `--arxiv` 批量调用。
- 多选中混有普通条目、附件和笔记时，只处理有 arXiv 的普通条目并汇总 skipped。
- 多选中某条普通条目缺少 arXiv 时，不弹窗并记为 skipped。
- 批量调用中单个 project 失败时，只标记映射到该 `latexProjectName` 的条目失败，不影响其他成功条目的附件保存。
- 多选中重复 arXiv ID 只调用一次 LaTeXTransPlus，并把同一个 PDF 分别保存到多个对应条目。
- 同一条目已有运行中任务时，再次触发不会新建任务，并计入 skipped。
- 运行中任务结束、失败或跳过后，对应 itemID 会从 registry 释放。

### 手动验证

在真实 Zotero 环境中验证：

- 含 arXiv URL 的条目可以自动生成翻译 PDF。
- 单选不含 arXiv 的条目会主动要求输入来源。
- 单选输入远程源码压缩包 URL 可以走 `--project-url`。
- 单选输入本地源码目录或压缩包可以走 `--project`。
- 多选不含 arXiv 的条目会直接 skipped，不弹出输入框。
- 多选中多个 arXiv 条目会通过一次 `--arxiv` 批量调用处理。
- LaTeXTransPlus 运行时，ProgressWindow 会根据 JSON Lines 项目事件更新状态。
- 生成 PDF 成为当前条目的子附件。

## 后续扩展

- 扩展 LaTeXTransPlus JSON Lines 事件 schema，提供更细的阶段、token 用量、下载状态和编译状态。
- 插件支持取消运行中的外部进程。
- 插件支持任务队列和历史记录。
- 插件支持自动跳过已存在翻译 PDF 的条目。
- 插件支持从 DOI 或仓储页面自动发现 TeX 源码链接。

## 决策总结

第一版采用 CLI 集成方案。插件不绑定 conda 或任何特定 Python 环境，而是让用户配置 executable 与 fixed args。插件只负责 Zotero 入口、来源解析、命令执行、JSON Lines 事件消费、PDF 定位和附件保存；LaTeXTransPlus 继续负责所有翻译与编译工作。
