# Ejemplos de grafos bilink

Archivos HTML generados con `bilinker graph --format html`. Abrí en cualquier browser.

## Archivos

| Archivo | Descripción | Regenerar |
|---------|-------------|-----------|
| `stratum-graph.html` | Grafo de bilinks del subsistema `stratum` (spec ↔ impl) | `cd subsystems/stratum && bilinker graph "." --format html > ../.examples/stratum-graph.html` |
| `system-graph.html` | Grafo completo del sistema (todos los subsistemas) | `bilinker graph "." --recursive --format html > .examples/system-graph.html` |

## Uso

```bash
xdg-open .examples/system-graph.html
```

- Click en un nodo → panel con contenido del fragmento vinculado
- Archivos `.md` → renderizado Markdown
- Código fuente → syntax highlighting con números de línea
- Link "Open file" → abre el archivo en el programa por defecto del sistema

## Otras variantes

```bash
# Con detalle de queries AST en Graphviz
bilinker graph "." --recursive --format dot --show-query | dot -Tsvg > .examples/system-graph.svg

# Solo un subsistema
cd subsystems/bilinker
bilinker graph "." --format html > ../.examples/bilinker-graph.html
```
