# Especificación: comando `bilinker graph`

## Propósito

Recorre el grafo de bilinks a partir de un archivo o fragmento y muestra todos los nodos conectados, cruzando capas. Responde a la pregunta: *¿con qué está linkedeado esto, y a través de qué caminos?*

Es una herramienta de navegación y exploración — no modifica nada.

## Firma

```
bilinker graph <selector> [--depth <n>] [--format <tree|flat|dot>]
```

| Argumento / Flag | Descripción |
|---|---|
| `selector` | Archivo, posición `archivo:línea:col`, o UUID de bilink. |
| `--depth <n>` | Profundidad máxima de traversal. Por defecto: sin límite. |
| `--format` | Formato de salida: `tree` (por defecto), `flat`, `dot` (Graphviz). |

## Algoritmo de traversal

El recorrido es un BFS sobre el grafo de bilinks. Cada bilink es un nodo del grafo; sus endpoints son las aristas hacia los artefactos o hacia los nodos adyacentes en otras capas.

```
graph(selector):
  1. Resolver selector → archivo (o UUID directo)
  2. bilinks = index.lookup(archivo)   // O(1) con .index, O(N) sin él
  3. Para cada bilink encontrado:
       bl = cargar uuid.bilink
       emitir: uuid, state.0 ↔ state.1, link.0, link.1
       para cada endpoint Layer no visitado y dentro del límite de profundidad:
         adjacent_path = stratum::resolve(layer_endpoint)
         si adjacent_path/.bilink/uuid.bilink existe: encolar
  4. Deduplicar por (UUID, layer_root) — si el mismo UUID en la misma capa
     ya fue visitado, no se vuelve a recorrer.
```

Los endpoints estructurales son hojas: se muestran pero no se atraviesan. Los endpoints layer son aristas hacia otras capas: se atraviesan recursivamente.

El selector `<uuid>` entra directamente como nodo de partida sin lookup por archivo.

Usa `.bilink/.index` si está disponible y actualizado (O(1)); si no, escanea los `.bilink` de la layer actual (O(N)).

## Salida — formato `tree`

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
│       │  link.1  crates/estrato-cli/src/main.rs :: (enum_item name: (type_identifier) @n0 (#eq? @n0 "Commands")) @target
│       │
│
└── b95021d2  [OK ↔ OK]
    │  link.0  commands/pull.md
    │  link.1  >impl
    │
    └── b95021d2  [OK ↔ OK]  (.stratum/impl)
        │  link.0  <
        │  link.1  crates/estrato-cli/src/main.rs :: (function_item name: (identifier) @n0 (#eq? @n0 "cmd_pull")) @target
        │
```

Cada nodo muestra:
- UUID corto (8 chars)
- Estado de ambos endpoints: `[state.0 ↔ state.1]`
- La layer donde vive el nodo (entre paréntesis si no es la raíz)
- `link.0` y `link.1` completos

## Salida — formato `flat`

Una línea por nodo — útil para scripting:

```
$ bilinker graph commands/pull.md --format flat

c0feab23  OK ↔ OK  commands/pull.md  →  >impl  [.]
c0feab23  OK ↔ OK  <  →  crates/estrato-cli/src/main.rs :: (enum_item ...)  [.stratum/impl]
b95021d2  OK ↔ OK  commands/pull.md  →  >impl  [.]
b95021d2  OK ↔ OK  <  →  crates/estrato-cli/src/main.rs :: (function_item ...)  [.stratum/impl]
```

## Salida — formato `dot`

Emite un grafo Graphviz que puede renderizarse con `dot -Tsvg`:

```
$ bilinker graph commands/pull.md --format dot | dot -Tsvg > graph.svg
```

Cada nodo archivo es un rectángulo; cada nodo bilink es un diamante con UUID y estados; las aristas indican el endpoint (`.0` o `.1`).

## Traversal entre repos

Si el `.bilink/<uuid>.bilink` en la capa adyacente no existe localmente (repo no clonado), el traversal se detiene silenciosamente en esa rama. El nodo actual se muestra igualmente con su endpoint layer visible.

## Ciclos

Si el traversal encuentra un (UUID, layer) ya visitado, lo muestra con `[ya visitado]` y no continúa:

```
└── 7f3d8e9a  [ya visitado]
```

## Invariantes

1. `graph` nunca modifica ningún archivo.
2. Un bilink con dos endpoints estructurales (link directo) aparece como hoja — no genera traversal adicional.
3. `--depth 1` muestra solo los bilinks directamente conectados al selector, sin cruzar capas.

## Código de salida

| Código | Condición |
|---|---|
| 0 | Traversal completado. |
| 1 | Selector no resuelve a ningún archivo o bilink conocido. |
| 2 | Error de lectura. |
