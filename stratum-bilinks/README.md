# stratum-bilinks

Grafos HTML interactivos de los bilinks del sistema. Generados con `bilinker graph --format html`. Abrí en cualquier browser.

## Archivos

| Archivo | Descripción | Regenerar |
|---------|-------------|-----------|
| `system-graph.html` | Grafo completo del sistema (todos los subsistemas) | `bilinker graph "." --recursive --format html > stratum-bilinks/system-graph.html` |
| `stratum-graph.html` | Bilinks del subsistema `stratum` (spec ↔ impl) | `cd subsystems/stratum && bilinker graph "." --format html > ../../stratum-bilinks/stratum-graph.html` |
| `bilinker-graph.html` | Bilinks del subsistema `bilinker` (spec ↔ impl) | `cd subsystems/bilinker && bilinker graph "." --format html > ../../stratum-bilinks/bilinker-graph.html` |

## Uso

```bash
xdg-open stratum-bilinks/system-graph.html
```

- Click en un nodo → panel con contenido del fragmento vinculado
- Archivos `.md` → renderizado Markdown
- Código fuente → syntax highlighting, números de línea, scroll horizontal
- Aristas muestran `UUID\nstate_spec↔state_impl`
- Link "Open file" → abre el archivo en el programa por defecto del sistema

## Regenerar todo

```bash
ROOT=$(pwd)
bilinker graph "." --recursive --format html > stratum-bilinks/system-graph.html
cd subsystems/stratum  && bilinker graph "." --format html > "$ROOT/stratum-bilinks/stratum-graph.html"  && cd "$ROOT"
cd subsystems/bilinker && bilinker graph "." --format html > "$ROOT/stratum-bilinks/bilinker-graph.html" && cd "$ROOT"
```
