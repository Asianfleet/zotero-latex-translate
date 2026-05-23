# Zotero LaTeX Translate

Zotero LaTeX Translate is a Zotero plugin for generating translated PDFs from arXiv and LaTeX source projects attached to Zotero items.

The plugin is being developed from a Zotero 7 plugin scaffold. Its planned workflow is:

- Detect an arXiv ID or arXiv URL from the selected Zotero item.
- Ask for a TeX source URL or local source path when arXiv metadata is unavailable.
- Call an external LaTeX translation runner, such as LaTeXTransPlus.
- Save the generated translated PDF back to the Zotero item.

## Development

Install dependencies:

```shell
npm install
```

Start the development server:

```shell
npm start
```

Build the plugin:

```shell
npm run build
```

## Plugin Metadata

- Package name: `zotero-latex-translate`
- Add-on name: `Zotero LaTeX Translate`
- Add-on ID: `zotero-latex-translate@local`
- Namespace: `zoterolatextranslate`
- Zotero instance: `ZoteroLatexTranslate`
- Preferences prefix: `extensions.zotero.zoterolatextranslate`

## License

AGPL-3.0-or-later
