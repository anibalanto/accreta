# Editor VS Code: extensión bilinker

Integra bilinker en VS Code con tres capacidades: hover sobre fragmentos binkados, code lens por línea y comandos de grafo interactivo.

## Activación

La extensión activa en `onStartupFinished`. Busca `bilinker-lsp` con `findBinary`: primero `which`, luego `~/.cargo/bin/<name>` como fallback (necesario en Flatpak, donde PATH es reducido). Si no lo encuentra, muestra un error y no registra los comandos.

## LSP: hover

Cuando el cursor está sobre un fragmento con bilinks, muestra en el tooltip el contenido del extremo opuesto formateado en Markdown: código con syntax highlighting, sección markdown con títulos y tablas.

## LSP: code lens

Cada línea con bilinks muestra una lente `⬡ N bilink(s)`. Al hacer click abre un WebviewPanel lateral con los IDs de los bilinks del fragmento.

## Comandos de grafo

### `bilinker.openGraph` — Grafo del archivo actual

Corre `bilinker graph <ruta-relativa> --format html` desde la raíz del workspace. El selector es la ruta del archivo activo relativa al workspace root. Muestra el resultado en un WebviewPanel con `enableScripts: true`.

### `bilinker.openSystemGraph` — Grafo del sistema completo

Corre `bilinker graph . --recursive --format html` desde la raíz del workspace. Muestra el resultado en un WebviewPanel.

## Resolución del binario

`findBinary(name)`: intenta `which <name>` vía `execSync`; si lanza (PATH reducido), verifica `~/.cargo/bin/<name>` con `fs.existsSync`.

## Empaquetado

El `.vsix` se genera con `esbuild --bundle --external:vscode`, que incluye `vscode-languageclient` en el bundle. `node_modules/` permanece excluido del `.vsix` vía `.vscodeignore`.
