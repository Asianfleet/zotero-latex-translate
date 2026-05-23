# Zotero 条目生成 LaTeXTransPlus 翻译 PDF 设计

## 背景

本项目当前基于 `zotero-plugin-template`，目标是开发成一个 Zotero 插件：用户选中普通条目后，插件自动识别论文的 arXiv 编号或链接，并调用本机 LaTeXTransPlus 生成翻译版 PDF，再把生成的 PDF 保存回该 Zotero 条目。

LaTeXTransPlus 位于用户配置的项目目录中，插件不内置翻译逻辑。当前 LaTeXTransPlus 已支持以下输入：

- `--arxiv`：arXiv ID 或 arXiv URL。
- `--project`：本地 TeX 项目目录或本地源码压缩包。
- `--project-url`：远程 TeX 源码压缩包 URL。
- `--source_language` 与 `--target_language`：覆盖配置文件中的源语言和目标语言。
- `--output`：覆盖输出目录。

## 目标

- 为 Zotero 普通条目提供手动触发入口。
- 支持用户一次选择多个普通条目，并按队列串行生成翻译 PDF。
- 自动从条目元数据中提取 arXiv ID 或 arXiv URL。
- 未检测到 arXiv 时，主动要求用户输入 TeX 源码下载链接或本地路径。
- 使用用户配置的外部 runner 调用 LaTeXTransPlus，不绑定 conda、venv 或系统 Python。
- 在 LaTeXTransPlus 执行成功后，将翻译版 PDF 作为当前条目的子附件保存到 Zotero。
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

缺点：

- 进度只能先做阶段级反馈。
- 如果没有机器可读输出，PDF 定位需要使用目录规则和 fallback 扫描。

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

第一版选择方案 A。后续可以在 LaTeXTransPlus 中增加 `--json-result <path>`，让 CLI 仍保持边界清晰，同时解决 PDF 路径和状态读取问题。

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

未检测到 arXiv 时，弹出输入框要求用户输入 TeX 源码来源。用户可输入 arXiv ID、arXiv URL、远程源码压缩包 URL、本地项目目录或本地源码压缩包。

### `latextransRunner`

负责组装并执行外部命令。

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
  sourceFlag,
  sourceValue
]
```

`latextransRunner` 不负责识别 Zotero 条目，也不负责保存附件。

### `resultResolver`

负责在 LaTeXTransPlus 结束后定位生成的 PDF。

第一版采用两级策略：

1. 根据输出规则查找：

   ```text
   <output>/<target_language>_<projectName>/<target_language>_<projectName>.pdf
   ```

2. 如果精确路径未命中，在本次命令开始时间之后扫描 `<output>` 下新增或更新的 PDF，选择最新的 PDF。

后续推荐让 LaTeXTransPlus 支持 `--json-result <path>`，插件优先读取 JSON 中的 `pdf_path`。

### `attachmentService`

负责将 PDF 保存到 Zotero 条目。

行为：

- 将 PDF 作为当前普通条目的子附件导入或链接。
- 附件标题建议为 `翻译版 PDF (<target_language>)`。
- 可选添加 tag：`latextransplus-translated`。
- 如果已有同名翻译附件，默认保留并创建新附件；只有用户启用“覆盖已有翻译附件”时才替换。

### `translateCommand`

右键菜单和主编排模块。

职责：

- 获取 Zotero 当前选中条目。
- 过滤非普通条目。
- 串行处理多个选中条目。
- 调用 `sourceResolver`、`latextransRunner`、`resultResolver` 和 `attachmentService`。
- 管理 ProgressWindow 和错误提示。

## 多选处理

第一版支持多选，但不并发执行。原因是 LaTeXTransPlus 会调用 LLM、下载源码并运行 LaTeX 编译，并发执行容易造成资源抢占、日志混杂和输出定位困难。

多选规则：

- 从 Zotero 当前选择中筛出普通条目。
- attachment、note、feed item 等非普通条目直接记为 skipped。
- 对普通条目建立 FIFO 队列，逐条处理。
- 每个条目独立执行 arXiv 检测、必要时用户输入、命令运行、PDF 定位和附件保存。
- 单个条目失败不阻断后续条目。
- 全部完成后显示批量汇总：成功数、失败数、跳过数。

用户输入规则：

- 如果某个条目自动检测到 arXiv，不弹出输入框。
- 如果某个条目未检测到 arXiv，则针对该条目弹出输入框，标题或提示中包含条目标题，避免用户混淆。
- 用户取消输入时，只跳过当前条目，不取消整个队列。
- 可以在输入框中提供“取消全部”按钮作为后续增强；第一版只需要“取消当前条目”。

附件命名规则：

- 每个条目独立保存子附件。
- 默认附件标题为 `翻译版 PDF (<target_language>)`。
- 如果一个条目已存在同名翻译附件，按用户配置决定保留旧附件并新增，或覆盖旧附件。

## 执行流程

1. 用户在 Zotero 中选中一个或多个普通条目。
2. 用户右键点击 `生成翻译版 PDF`。
3. 插件读取当前选中项。
4. 插件将普通条目加入串行队列，将非普通条目标记为 skipped。
5. 插件对队列中的每个条目逐一处理。
6. 插件从当前条目字段中提取 arXiv。
7. 如果识别到 arXiv，构造 `--arxiv` 输入。
8. 如果未识别到 arXiv，弹窗要求用户为当前条目输入 TeX 源码链接或本地路径。
9. 插件分类当前条目的输入：
   - arXiv ID 或 arXiv URL -> `--arxiv`
   - `http/https` URL -> `--project-url`
   - 本地路径 -> `--project`
10. 插件校验 LaTeXTransPlus 配置。
11. 插件启动外部命令。
12. 命令执行期间显示阶段状态。
13. 命令退出后，插件定位生成的 PDF。
14. 插件将 PDF 保存为当前条目的子附件。
15. 插件记录当前条目的成功、失败或跳过状态，并继续队列。
16. 队列结束后显示批量汇总。

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
- 未检测到 arXiv，且用户取消输入。
- 用户输入不是 arXiv、URL 或本地路径。
- 本地路径不存在。

处理方式：跳过当前条目，批量场景继续处理后续条目。

### 执行错误

- 外部进程启动失败。
- LaTeXTransPlus 返回非 0 退出码。
- LaTeXTransPlus 执行完成但找不到 PDF。

处理方式：记录当前条目失败，显示简短错误，并附带 stdout、stderr 或 LaTeXTransPlus `latextrans.log` 的路径。批量场景继续处理后续条目。

### 附件错误

- PDF 文件不存在或不可读。
- Zotero 附件导入失败。
- 保存条目失败。

处理方式：记录当前条目失败，显示 PDF 路径，提示用户可手动导入，并保留 LaTeXTransPlus 输出。批量场景继续处理后续条目。

## 进度与用户反馈

第一版使用阶段级进度：

- 准备输入。
- 运行 LaTeXTransPlus。
- 查找生成 PDF。
- 保存到 Zotero。
- 完成或失败。

多选场景下，ProgressWindow 需要包含当前队列位置，例如 `[2/5] 运行 LaTeXTransPlus`，并在全部完成后显示成功、失败和跳过数量。

因为 LaTeXTransPlus 当前 CLI 没有稳定的结构化进度事件，插件不解析普通 stdout 作为精确进度。后续如果 LaTeXTransPlus 提供 JSON lines 事件流，再扩展细粒度进度。

## 测试设计

### 单元测试

覆盖以下纯逻辑：

- arXiv ID 提取。
- arXiv URL 提取。
- DOI、extra、abstractNote 中的 arXiv 提取。
- 用户输入分类。
- fixed args 解析。
- argv 组装。
- 输出 PDF 精确路径定位。
- 输出 PDF fallback 扫描。
- 配置校验。

### 集成测试

使用 mock runner 覆盖：

- 外部命令成功。
- 外部命令失败。
- 命令成功但 PDF 不存在。
- PDF 附件保存失败。
- 多条目串行处理，一条失败不阻断后续条目。
- 多选中混有普通条目、附件和笔记时，只处理普通条目并汇总 skipped。
- 多选中某条缺少 arXiv 且用户取消输入时，只跳过该条。

### 手动验证

在真实 Zotero 环境中验证：

- 含 arXiv URL 的条目可以自动生成翻译 PDF。
- 不含 arXiv 的条目会主动要求输入来源。
- 输入远程源码压缩包 URL 可以走 `--project-url`。
- 输入本地源码目录或压缩包可以走 `--project`。
- 生成 PDF 成为当前条目的子附件。

## 后续扩展

- LaTeXTransPlus 增加 `--json-result <path>`，输出精确 `pdf_path`、状态、错误和日志路径。
- 插件支持取消运行中的外部进程。
- 插件支持任务队列和历史记录。
- 插件支持自动跳过已存在翻译 PDF 的条目。
- 插件支持从 DOI 或仓储页面自动发现 TeX 源码链接。

## 决策总结

第一版采用 CLI 集成方案。插件不绑定 conda 或任何特定 Python 环境，而是让用户配置 executable 与 fixed args。插件只负责 Zotero 入口、来源解析、命令执行、PDF 定位和附件保存；LaTeXTransPlus 继续负责所有翻译与编译工作。
