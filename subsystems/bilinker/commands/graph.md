# Especificación: comando `bilinker graph`

## Propósito

Recorre el grafo de bilinks a partir de un archivo o fragmento y muestra todos
los nodos conectados, cruzando capas. Responde a la pregunta: *¿con qué está
linkedeado esto, y a través de qué caminos?*

Es una herramienta de navegación y exploración — no modifica nada.

## Firma

```
bilinker graph <selector> [--depth <n>] [--state <estado>] [--format <tree|flat|dot>]
```

| Argumento / Flag | Descripción |
|---|---|
| `selector` | Archivo, posición `archivo:línea:col`, o UUID de bilink. |
| `--depth <n>` | Profundidad máxima de traversal. Por defecto: sin límite. |
| `--state <estado>` | Muestra solo bilinks que tengan al menos un endpoint en ese estado. |
| `--format` | Formato de salida: `tree` (por defecto), `flat`, `dot` (Graphviz). |

## Algoritmo de traversal

El recorrido es un BFS sobre el grafo de bilinks. Cada bilink es un nodo del
grafo; sus endpoints son las aristas hacia los artefactos o hacia los nodos
adyacentes en otras capas.

```
graph(selector):
  1. Resolver selector → (archivo, query?)
  2. bilinks = index.lookup(archivo)   // todos los bilinks que referencian ese archivo
  3. Para cada (uuid, n) en bilinks:
       bl = cargar uuid.bilink
       other = endpoint opuesto a n
       emitir: uuid, state.n ↔ state.other, descripción de other
       si other es Layer y depth no alcanzado y uuid no visitado:
         adjacent = resolver_layer_link(other, uuid)
         encolar adjacent para traversal
  4. Deduplicar por UUID — si el mismo UUID ya fue visitado, no se vuelve a recorrer.
```

Los endpoints estructurales son hojas: se muestran pero no se atraviesan.
Los endpoints layer son aristas hacia otras capas: se atraviesan recursivamente.

El selector `<uuid>` entra directamente como nodo de partida sin lookup por archivo.

## Salida — formato `tree`

```
$ bilinker graph src/Persona.java:45:1

src/Persona.java :: (class_declaration name: Persona)
│
├── 7f3d8e9a  [OK ↔ CHAIN_DIRTY]
│   │  link.0  src/Persona.java :: vote   (este nodo)
│   │  link.1  → .stratum/tech-decisions
│   │
│   └── 7f3d8e9a  [CHAIN_DIRTY ↔ OK]      (.stratum/tech-decisions)
│       │  link.0  → ../..
│       │  link.1  → .stratum/impl
│       │
│       └── 7f3d8e9a  [OK ↔ OK]            (.stratum/tech-decisions/.stratum/impl)
│              link.0  → ../tech-decisions
│              link.1  specs/voting.yaml :: impl
│
└── a3f9c821  [OK ↔ OK]
       link.0  src/Persona.java :: invariant-check
       link.1  specs/persona.md :: vote-invariants
```

Cada línea de bilink muestra:
- UUID corto (8 chars)
- Estado del endpoint de entrada ↔ estado del endpoint de salida
- La layer donde vive este nodo (entre paréntesis si no es la raíz)

## Salida — formato `flat`

```
$ bilinker graph src/Persona.java:45:1 --format flat

7f3d8e9a  OK ↔ CHAIN_DIRTY  src/Persona.java::vote  →  .stratum/tech-decisions
7f3d8e9a  CHAIN_DIRTY ↔ OK  ../..  →  .stratum/impl            [tech-decisions]
7f3d8e9a  OK ↔ OK           ../tech-decisions  →  specs/voting.yaml::impl  [impl]
a3f9c821  OK ↔ OK           src/Persona.java::invariant-check  →  specs/persona.md::vote-invariants
```

## Salida — formato `dot`

Emite un grafo Graphviz que puede renderizarse con `dot -Tsvg`:

```
$ bilinker graph src/Persona.java:45:1 --format dot | dot -Tsvg > graph.svg
```

Cada nodo es un artefacto (archivo o fragmento); cada arista es un bilink con
su estado como label.

## Traversal entre repos

Si un endpoint layer apunta a un repo distinto del actual, `graph` intenta
resolver el path relativo desde el directorio de trabajo. Si el repo adyacente
no está presente localmente, emite el nodo con estado `UNREACHABLE` y detiene
el traversal en esa rama:

```
└── 7f3d8e9a  [OK ↔ UNREACHABLE]
       link.1  → ../other-repo  (repo no disponible localmente)
```

## Ciclos

Si el traversal encuentra un UUID ya visitado, lo muestra con `[ya visitado]`
y no continúa:

```
└── 7f3d8e9a  [ya visitado]
```

## Invariantes

1. `graph` nunca modifica ningún archivo.
2. Usa el índice (`.bilink/.index`) si está disponible; cae a scan O(N) si no.
3. Un bilink con dos endpoints estructurales (link directo) aparece como hoja en
   ambos extremos — no genera traversal adicional.
4. `--depth 1` muestra solo los bilinks directamente conectados al selector,
   sin cruzar capas.

## Código de salida

| Código | Condición |
|---|---|
| 0 | Traversal completado (incluso si hay nodos UNREACHABLE). |
| 1 | Selector no resuelve a ningún archivo o bilink conocido. |
| 2 | Error de lectura. |
