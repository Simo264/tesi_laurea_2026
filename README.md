# README

La tesi è stata scritta in Latex e TexStudio: esegui -> apri main.tex -> compila.

La presentazione PPTX è stata prodotta usando Marp (Markdown Presentation Ecosystem) partendo dal file markdown `presentazione.md`.

Installa marp-cli in locale:

```bash
npm install --save-dev @marp-team/marp-cli
```

Preview dinamica con Brave:

```bash
npx @marp-team/marp-cli@latest -w -p presentazione.md --theme-set presentazione.css --browser-path /usr/bin/brave 
```

Esporta in PDF:

```bash
npx @marp-team/marp-cli@latest presentazione.md --theme-set presentazione.css --pdf --browser-path /usr/bin/brave -o presentazione.pdf --allow-local-files
```

Esporta in PPTX:

```bash
npx @marp-team/marp-cli@latest presentazione.md --theme-set presentazione.css --output presentazione.pptx --browser-path /usr/bin/brave --allow-local-files
```