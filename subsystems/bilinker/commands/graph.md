# Especificación: comando `bilinker graph`

## Propósito

Recorre el grafo de bilinks a partir de un archivo, fragmento o UUID y muestra todos los nodos conectados, cruzando capas. Responde a la pregunta: *¿con qué está linkedeado esto, y a través de qué caminos?*

Es una herramienta de navegación y exploración — no modifica nada.

## Firma

```
bilinker graph <selector>
  [--depth <n>]
  [--format <tree|flat|json>]
  [--recursive]
```

| Argumento / Flag | Descripción |
|---|---|
| `selector` | Archivo, posición `archivo:línea:col`, UUID de bilink, `.` (todos en capa actual) o `*` (igual que `.`). |
| `--depth <n>` | Profundidad máxima de traversal. Por defecto: sin límite. |
| `--format` | Formato de salida: `tree` (por defecto), `flat`, `json` (contrato de proveedor lattice). |
| `--recursive` | Con selector `.`/`*`: recolectar bilinks de **todas las capas** bajo la raíz del proyecto. |

## Selectores

| Selector | Comportamiento |
|----------|----------------|
| `archivo.md` | Todos los bilinks que referencian ese archivo en la capa actual |
| `archivo.md:42:5` | Bilinks cuyo capture tiene un `range` que cubre esa posición |
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

Usa `.bilink/index/index` si está disponible (O(1)); si no, scan O(N).

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

## Formato `json` — contrato de proveedor

Emite las aristas de bilinker en el modelo de [lattice](../../lattice/concepts/edge.md), con los nodos ya resueltos a forma canónica. Es la forma en que bilinker actúa como proveedor: la resolución de una cadena a través de capas la hace bilinker, porque la topología es conocimiento de su formato; componerla con aristas de otros proveedores es tarea de lattice.

```json
[
  {
    "from":      ".::commands/pull.md#312~358",
    "to":        ".stratum/impl::crates/estrato-cli/src/main.rs#245~389",
    "kind":      "bilink",
    "guarantee": "accepted",
    "provider":  "bilinker",
    "directed":  false,
    "ref":       "c0feab23-1b2c-4d5e-8f6a-7b8c9d0e1f2a",
    "state":     ["OK", "OK"]
  }
]
```

Una cadena de N nodos emite **una** arista entre sus dos tips estructurales, no N-1 aristas entre nodos `.bilink`. Los mids son mecanismo interno de bilinker, no conexiones del proyecto.

`state` lleva la tupla `(state.0, state.1)` de los tips. Los `kind` emitidos son `bilink`, `governs` y `task`, todos con garantía `accepted`.

> Los formatos `dot` y `html` se movieron a [`lattice graph`](../../lattice/commands/graph.md). Recorrer una cadena es conocimiento del formato bilink; renderizar el grafo nunca lo fue, y en lattice el visor puede mostrar además las aristas de los otros proveedores.

## Traversal entre repos

Si el `.bilink/<uuid>.bilink` en la capa adyacente no existe localmente (repo no clonado), el traversal se detiene silenciosamente. El nodo actual se muestra con su endpoint layer.

## Ciclos

Si el traversal encuentra un (UUID, layer) ya visitado, lo muestra con `[ya visitado]` y no continúa.

## Invariantes

1. `graph` nunca modifica ningún archivo.
2. Fragmentos distintos del mismo archivo generan nodos separados (identificados por línea de inicio).
3. `--depth 1` muestra solo los bilinks directamente conectados al selector.

## Código de salida

| Código | Condición |
|---|---|
| 0 | Traversal completado. |
| 1 | Selector no resuelve a ningún archivo o bilink conocido. |
| 2 | Error de lectura. |
