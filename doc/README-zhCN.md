# Zotero LaTeX Translate

Zotero LaTeX Translate 是一个 Zotero 插件，用于从 Zotero 条目中的 arXiv 信息或 LaTeX 源码项目生成翻译版 PDF。

当前项目基于 Zotero 7 插件脚手架开发，计划工作流如下：

- 从选中的 Zotero 普通条目中识别 arXiv ID 或 arXiv URL。
- 未识别到 arXiv 信息时，提示用户输入 TeX 源码链接或本地路径。
- 调用外部 LaTeX 翻译工具，例如 LaTeXTransPlus。
- 将生成的翻译版 PDF 保存回 Zotero 条目。

## 开发

安装依赖：

```shell
npm install
```

启动开发服务：

```shell
npm start
```

构建插件：

```shell
npm run build
```

## 插件元信息

- package name：`zotero-latex-translate`
- 插件名称：`Zotero LaTeX Translate`
- 插件 ID：`zotero-latex-translate@local`
- namespace：`zoterolatextranslate`
- Zotero 实例名：`ZoteroLatexTranslate`
- 首选项前缀：`extensions.zotero.zoterolatextranslate`

## 许可证

AGPL-3.0-or-later
