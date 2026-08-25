# Especificación: comando `lattice graph`

## Propósito

Recorre el grafo agregado desde un selector y emite los nodos y aristas alcanzados. Es el único comando de consulta: absorbe lo que en bilinker eran `graph` (recorrer el grafo de bilinks) e `impact` (subir por el call graph hasta encontrar bilinks), que son el mismo traversal con distinto conjunto de aristas habilitadas.

Solo lectura. No escribe ningún archivo.

## Firma

```
lattice graph <selector>
  [--up | --down | --both]
  [--via <kinds>]
  [--guarantee <nivel>]
  [--state <filtro>]
  [--depth <n>]
  [--format <tree|flat|json|dot|html>]
  [--recursive]
```

| Flag | Descripción |
|---|---|
| `--up` | Sigue aristas dirigidas en sentido inverso (callers). Default en presencia de `--state`. |
| `--down` | Sigue aristas dirigidas en sentido directo (callees). |
| `--both` | Ambos sentidos. Default cuando no se especifica ninguno. |
| `--via <kinds>` | Lista de `kind` habilitados: `bilink,governs,task,call,doclink,external`. Default: todos los disponibles. |
| `--guarantee <nivel>` | Garantía mínima: `accepted`, `derived`, `asserted`. Default: `asserted` (todas). |
| `--state <filtro>` | Solo nodos alcanzados por aristas con ese estado. `non-ok` selecciona todo lo distinto de OK. |
| `--depth <n>` | Profundidad máxima. Default: sin límite. |
| `--recursive` | Recolecta también desde las capas descendientes. Se delega en el proveedor: dónde vive una capa es conocimiento de Stratum y del formato bilink, no del grafo agregado. |

Las aristas no dirigidas (`bilink`, `governs`, `task`) se recorren en ambos sentidos siempre; `--up` / `--down` solo afectan a las dirigidas.

## Selectores

| Selector | Comportamiento |
|---|---|
| `<archivo>` | Nodos que referencian ese archivo. |
| `<archivo>:<línea>:<col>` | Nodos que cubren esa posición, del más específico al más general. |
| `<uuid>.<N>` | El nodo del endpoint N de ese vínculo — un fragmento. |
| `<uuid>` | **El vínculo**, no un nodo: la arista con ese `ref`, más el nodo del archivo `.bilink` que la identifica. |
| `.` | Todos los nodos de la capa actual (con `--recursive`, de todas las capas). |

### `<uuid>` vs `<uuid>.<N>`

Los dos selectores parten de lugares distintos y responden preguntas distintas:

```bash
# ¿Con qué está conectado el fragmento del endpoint 1?
lattice graph fac79bf8.1

# ¿Qué documentos gobiernan este vínculo?
lattice graph fac79bf8 --via governs
```

La distinción importa porque una arista `governs` no apunta a un fragmento: apunta al vínculo entre dos fragmentos. Su nodo destino es el archivo `.bilink`, que comparte `ref` con la arista `bilink` de esa cadena — ver [integration/bilinker.md](../integration/bilinker.md) § "El endpoint bilink y las aristas `governs`".

Con `<uuid>` el traversal arranca desde ese nodo *y* desde los dos tips, de modo que una consulta sin `--via` devuelve el vínculo completo con lo que lo gobierna.

## Equivalencias

Los dos comandos que lattice reemplaza son casos particulares:

```bash
# antes: bilinker graph commands/check.md
lattice graph commands/check.md --via bilink

# antes: bilinker impact
lattice graph . --state non-ok --up --via bilink,call

# antes: bilinker impact src/check.rs:210:5
lattice graph src/check.rs:210:5 --up --via bilink,call
```

Que `impact` fuese un comando aparte era consecuencia de que el traversal por callers viviera en otro lado que el traversal por cadenas. Con un modelo único de arista, es un filtro.

## Algoritmo

```
1. Consultar available() de cada proveedor registrado.
2. Resolver el selector → nodos de partida.
3. Enumerar aristas de los proveedores que enumeran (bilink, doc) en el scope.
4. Construir el índice de contención sobre los nodos obtenidos.
5. BFS desde los nodos de partida:
     - expandir por aristas ya conocidas
     - para proveedores con edges_from (lsp): resolver el anclaje del nodo
       y pedir sus aristas incidentes
     - en cada nodo alcanzado, consultar cubriendo() para conectar con nodos
       de otra garantía
     - cortar por visited-set y por --depth
6. Deduplicar y emitir, con el estado de cada proveedor.
```

El paso 5 es donde se cruza de una garantía a otra: al llegar a una función vía una arista `call`, `cubriendo()` responde si hay un endpoint `accepted` que la contiene. Ese es el salto de "esto podría estar afectado" a "esto está roto".

### Corte del traversal

Una rama que alcanza un nodo cubierto por una arista `accepted` **se detiene**, salvo que sea el nodo de partida. Sin ese corte, el traversal seguiría subiendo más allá del límite del subgrafo que alguien documentó, que es justamente el borde que interesa.

Es el comportamiento de `impact.rs:210-214`, generalizado a cualquier arista `accepted`.

### Ciclos

Visited-set sobre la forma canónica del nodo. Un nodo ya visitado se muestra marcado y no se expande.

## Salida — `tree`

```
$ lattice graph . --state non-ok --up

proveedores: bilink OK · lsp OK · doc no registrado

◆ fac79bf8  [ALTERED]
  .stratum/impl::crates/bilinker/src/check.rs#5100~7300   check_structural
  ↕ bilink (accepted, commit ca76a590)
  .::commands/check.md#1240~2180                          § "Algoritmo de detección"

◆ 9662a432  [OK]
  .stratum/impl::crates/bilinker/src/check.rs#1200~2400   check_file
  ↑ call (derived)  ← check_structural  src/check.rs:210
  ↕ bilink (accepted, commit ca76a590)
  .::commands/check.md#3100~3900                          § "Escritura de cache"
```

Cada arista muestra su `kind` y su garantía. Un consumidor puede distinguir de un vistazo qué parte del resultado es verificable y qué parte es inferencia de un language server.

## Salida — `json`

```json
{
  "providers": [
    {"name": "bilink", "status": "available"},
    {"name": "lsp",    "status": "unavailable", "reason": "daemon no responde"}
  ],
  "nodes": [
    {"id": ".::commands/check.md#1240~2180", "layer": ".", "path": "commands/check.md"}
  ],
  "edges": [
    {
      "from": ".stratum/impl::crates/bilinker/src/check.rs#5100~7300",
      "to":   ".::commands/check.md#1240~2180",
      "kind": "bilink", "guarantee": "accepted", "provider": "bilinker",
      "directed": false, "ref": "fac79bf8-...",
      "state": ["OK", "ALTERED"], "commit": ["ca76a590", "ca76a590"]
    }
  ]
}
```

`providers` va primero y siempre está presente, incluso cuando todos respondieron. Un consumidor no debería tener que inferir la completitud del grafo a partir de su contenido.

## Salida — `dot`

```bash
lattice graph . --format dot | dot -Tsvg > graph.svg
```

Nodos agrupados en `subgraph cluster_N` por capa. **La garantía se codifica en el trazo**: continuo para `accepted`, punteado para `derived`, punteado fino para `asserted`. Un grafo que mezcla lo verificado con lo inferido sin marcarlo induce a confiar en la inferencia.

## Salida — `html`

```bash
lattice graph . --format html > graph.html
xdg-open graph.html
```

Archivo HTML autocontenido (sin servidor) con:

- **Grafo interactivo** con Cytoscape.js: zoom, pan, clusters por capa en columnas por profundidad stratum (spec izquierda → impl derecha), file-groups que agrupan fragmentos del mismo archivo.
- **Panel de detalle** al hacer click en un **nodo**: el contenido del fragmento. `.md` renderizado; código con syntax highlighting y números de línea; link `file://` para abrirlo en el sistema.
- **Panel de detalle** al hacer click en una **arista**: los dos fragmentos vinculados, con el `ref` y el estado en el separador.
- Fragmentos distintos del mismo archivo como nodos separados.

Cada arista lleva su `kind` y su garantía: en bilinker todas eran del mismo tipo, acá no, y una llamada inferida no puede verse igual que una referencia verificada.

Requiere conexión a internet para las CDN de Cytoscape.

## Degradación

Un proveedor `Unavailable` reduce el grafo sin abortar la consulta:

```
warn: proveedor lsp no disponible (daemon no responde) — 0 aristas `call` en este grafo
```

Con `--via call` como único `kind` y el proveedor caído, el resultado es vacío y el código de salida es 3.

## Código de salida

| Código | Condición |
|---|---|
| 0 | Traversal completado con todos los proveedores registrados disponibles. |
| 1 | Selector no resuelve a ningún nodo. |
| 2 | Error de lectura o de configuración. |
| 3 | Traversal completado en modo degradado — algún proveedor no disponible. |

El 3 permite que un pipeline de CI distinga "no hay impacto" de "no pude ver el impacto".
