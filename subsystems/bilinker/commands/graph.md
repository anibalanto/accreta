# Especificación: comando `bilinker graph`

## Propósito

Recorre el grafo de bilinks a partir de un archivo, fragmento o UUID y muestra todos los nodos conectados, cruzando capas. Responde a la pregunta: *¿con qué está linkedeado esto, y a través de qué caminos?*

Es una herramienta de navegación y exploración — no modifica nada.

## Firma

```
bilinker graph <selector>
  [--depth <n>]
  [--format <tree|flat|dot|html>]
  [--recursive]
  [--bilink-detail]
  [--url-scheme <line|file|none>]
  [--show-query] [--show-range] [--show-data]
```

| Argumento / Flag | Descripción |
|---|---|
| `selector` | Archivo, posición `archivo:línea:col`, UUID de bilink, `.` (todos en capa actual) o `*` (igual que `.`). |
| `--depth <n>` | Profundidad máxima de traversal. Por defecto: sin límite. |
| `--format` | Formato de salida: `tree` (por defecto), `flat`, `dot` (Graphviz), `html` (visor interactivo). |
| `--recursive` | Con selector `.`/`*`: recolectar bilinks de **todas las capas** bajo la raíz del proyecto. |
| `--bilink-detail` | En formato `dot`: mostrar nodos intermedios de bilink (diamantes). Por defecto solo se muestran aristas directas entre archivos. |
| `--url-scheme` | URLs en nodos: `line` (default, `file://path#Lnúmero`), `file` (`file://path`), `none`. |
| `--show-query` | Incluir query AST en labels de nodos (formatos `dot`). |
| `--show-range` | Incluir rango de bytes en labels de nodos (formatos `dot`). |
| `--show-data` | Incluir primera y última línea del fragmento en labels (formatos `dot`). |

## Selectores

| Selector | Comportamiento |
|----------|----------------|
| `archivo.md` | Todos los bilinks que referencian ese archivo en la capa actual |
| `archivo.md:42:5` | Bilinks cuyo `range.N` cubre esa posición |
| `<uuid>` (8+ hex chars) | Bilink concreto por UUID o prefijo |
| `.` o `*` | Todos los bilinks en la capa actual (con `--recursive`: en todas las capas) |

## Algoritmo de traversal

BFS sobre el grafo de bilinks. Cada fragmento de archivo es un nodo; los endpoints layer son aristas hacia otras capas.

```
graph(selector):
  1. Resolver selector → lista de bilinks iniciales
  2. Para cada bilink:
       emitir fragmento(s) estructural(es)
       para cada endpoint Layer no visitado:
         adjacent = stratum::resolve(layer_path)
         si adjacent/.bilink/uuid.bilink existe: encolar
  3. Deduplicar por (UUID, layer_root, start_line) — fragmentos distintos
     del mismo archivo son nodos separados.
```

Usa `.bilink/.index` si está disponible (O(1)); si no, scan O(N).

## Formato `tree` (default)

```
$ bilinker graph commands/pull.md

commands/pull.md
│
├── c0feab23  [OK ↔ OK]
│   │  link.0  commands/pull.md
│   │  link.1  >impl
│   │
│   └── c0feab23  [OK ↔ OK]  (.stratum/impl)
│       │  link.0  <
│       │  link.1  crates/estrato-cli/src/main.rs :: (enum_item ...) @target
│       │
└── b95021d2  [OK ↔ OK]
    └── b95021d2  [OK ↔ OK]  (.stratum/impl)
        │  link.1  crates/estrato-cli/src/main.rs :: (function_item ...) @target
        │
```

## Formato `flat`

Una línea por nodo — útil para scripting:

```
$ bilinker graph commands/pull.md --format flat

c0feab23  OK ↔ OK  commands/pull.md  →  >impl  [.]
c0feab23  OK ↔ OK  <  →  crates/main.rs :: (enum_item ...)  [.stratum/impl]
```

## Formato `dot` — Graphviz

```bash
# Grafo simplificado (aristas directas archivo↔archivo, default)
bilinker graph "." --format dot | dot -Tsvg > graph.svg

# Con nodos bilink intermedios visibles
bilinker graph "." --format dot --bilink-detail | dot -Tsvg > graph.svg

# Con detalle de fragmentos
bilinker graph "." --format dot --show-query --show-range --show-data | dot -Tsvg > graph.svg

# Sistema completo
bilinker graph "." --recursive --format dot | dot -Tsvg > system.svg
```

Nodos agrupados en `subgraph cluster_N` por capa, ordenados en columnas según profundidad stratum (izquierda = spec, derecha = impl). Aristas bidireccionales etiquetadas con UUID y estado.

## Formato `html` — visor interactivo

```bash
bilinker graph "." --recursive --format html > graph.html
xdg-open graph.html
```

Genera un archivo HTML autocontenido (sin servidor) con:
- **Grafo interactivo** con Cytoscape.js: zoom, pan, clusters por capa, columnas por profundidad
- **Panel de detalle** al hacer click en un nodo:
  - Archivos `.md`: renderizado Markdown (títulos, tablas, código)
  - Código fuente: syntax highlighting con highlight.js, números de línea, scroll horizontal
  - Link `file://` para abrir el archivo en el programa por defecto del sistema
- Fragmentos distintos del mismo archivo como nodos separados (`file.rs#L42`)

## Traversal entre repos

Si el `.bilink/<uuid>.bilink` en la capa adyacente no existe localmente (repo no clonado), el traversal se detiene silenciosamente. El nodo actual se muestra con su endpoint layer.

## Ciclos

Si el traversal encuentra un (UUID, layer) ya visitado, lo muestra con `[ya visitado]` y no continúa.

## Invariantes

1. `graph` nunca modifica ningún archivo.
2. Fragmentos distintos del mismo archivo generan nodos separados (identificados por línea de inicio).
3. `--depth 1` muestra solo los bilinks directamente conectados al selector.
4. El formato `html` es autocontenido — solo requiere conexión a internet para cargar las CDN.

## Código de salida

| Código | Condición |
|---|---|
| 0 | Traversal completado. |
| 1 | Selector no resuelve a ningún archivo o bilink conocido. |
| 2 | Error de lectura. |
