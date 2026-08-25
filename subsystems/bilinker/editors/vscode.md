# Editor VS Code: extensión bilinker

Integra bilinker en VS Code con tres capacidades: hover sobre fragmentos binkados, code lens por línea y comandos de grafo interactivo.

## Activación

La extensión activa en `onStartupFinished`. Busca `bilinker-lsp` con `findBinary`: primero `which`, luego `~/.cargo/bin/<name>` como fallback (necesario en Flatpak, donde PATH es reducido). Si no lo encuentra, muestra un error y no registra los comandos.

## LSP: hover

Cuando el cursor está sobre un fragmento con bilinks, muestra en el tooltip el contenido del extremo opuesto formateado en Markdown: código con syntax highlighting, sección markdown con títulos y tablas.

## LSP: code lens

Cada línea con bilinks muestra una lente `⬡ N bilink(s)`. Al hacer click abre un WebviewPanel lateral con los IDs de los bilinks del fragmento.

## Comandos de grafo

El visor vive en [lattice](../../lattice/commands/graph.md): bilinker recorre cadenas, lattice compone el grafo de todos los proveedores y lo renderiza. La extensión sigue usando `bilinker-lsp` para hover y code lens, que sí son de bilinker.

Requiere el ejecutable `lattice` en PATH, además de `bilinker-lsp`.

### `bilinker.openGraph` — Grafo del archivo actual

Corre `lattice graph <ruta-relativa> --format html` desde la raíz del workspace. Muestra el resultado en un WebviewPanel con `enableScripts: true`.

### `bilinker.openSystemGraph` — Grafo del sistema completo

Corre `lattice graph . --recursive --format html` desde la raíz del workspace.

### Código de salida 3

`lattice` devuelve 3 cuando algún proveedor no respondió — típicamente el daemon LSP apagado. **El grafo es válido igual**, así que la extensión lo muestra y avisa con un warning en vez de tratarlo como error. Confundir "grafo incompleto" con "falló" dejaría al usuario sin visor cada vez que el daemon no esté corriendo, que es el caso normal.

## Resolución del binario

`findBinary(name)`: intenta `which <name>` vía `execSync`; si lanza (PATH reducido), verifica `~/.cargo/bin/<name>` con `fs.existsSync`.

## Empaquetado

El `.vsix` se genera con `esbuild --bundle --external:vscode`, que incluye `vscode-languageclient` en el bundle. `node_modules/` permanece excluido del `.vsix` vía `.vscodeignore`.
