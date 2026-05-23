# Zotero LaTeX Translate

Zotero LaTeX Translate est un plugin Zotero destiné à générer des PDF traduits à partir d'informations arXiv ou de projets source LaTeX associés aux éléments Zotero.

Le projet est développé à partir d'un échafaudage de plugin Zotero 7. Le flux prévu est le suivant :

- Détecter un identifiant arXiv ou une URL arXiv depuis l'élément Zotero sélectionné.
- Demander une URL de source TeX ou un chemin local quand aucune information arXiv n'est disponible.
- Appeler un outil externe de traduction LaTeX, par exemple LaTeXTransPlus.
- Enregistrer le PDF traduit généré dans l'élément Zotero.

## Développement

Installer les dépendances :

```shell
npm install
```

Démarrer le serveur de développement :

```shell
npm start
```

Construire le plugin :

```shell
npm run build
```

## Métadonnées du plugin

- Nom du package : `zotero-latex-translate`
- Nom du module : `Zotero LaTeX Translate`
- ID du module : `zotero-latex-translate@local`
- Espace de noms : `zoterolatextranslate`
- Instance Zotero : `ZoteroLatexTranslate`
- Préfixe des préférences : `extensions.zotero.zoterolatextranslate`

## Licence

AGPL-3.0-or-later
