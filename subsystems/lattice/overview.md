# Lattice

Lattice es el grafo de conocimiento interconectado del ecosistema Accreta. Define un modelo único de nodos y aristas sobre el que se apoyan las herramientas de análisis, y agrega en una sola estructura conexiones de naturaleza muy distinta: referencias verificadas entre capas, llamadas derivadas del código en vivo, y links escritos dentro de documentos.

El nombre viene de la metáfora raíz del ecosistema: la acreción produce redes cristalinas. Lattice es la estructura que emerge de todo lo que Accreta va acumulando.

## Problema que resuelve

Las conexiones de un proyecto ya existen, pero viven dispersas y cada herramienta las recorre por su cuenta. Bilinker sabe de bilinks; el LSP sabe de llamadas; los documentos tienen links markdown que nadie mira. Hoy `impact` recalcula el alcance de un cambio escaneando los bilinks y recorriendo cadenas él mismo — conocimiento del formato de bilinker filtrado a otro subsistema, que habrá que arreglar dos veces el día que ese formato cambie.

El problema más profundo es que esas conexiones **no valen lo mismo**. Un bilink es una referencia que un humano escribió y aceptó, verificada por hash. Una arista de llamada del LSP es una inferencia de una herramienta externa que falla con dispatch dinámico, macros y callbacks. Un link markdown es una afirmación sin ninguna verificación. Una herramienta que las aplane todas a "links" pierde exactamente la propiedad que hace valioso a un bilink.

Lattice existe para que esas conexiones convivan en un grafo sin perder su procedencia.

## Nodos y aristas

Un **nodo** es un fragmento direccionable: un rango de bytes en un archivo de una capa, un archivo completo, un ítem de worklist o un recurso externo.

Una **arista** conecta dos nodos y declara siempre de dónde viene y qué garantiza. Las tres garantías son `accepted` (declarada y aceptada por un humano, verificable), `derived` (calculada por una herramienta, heurística) y `asserted` (escrita, sin verificación).

Lattice también resuelve la **contención** entre nodos — qué fragmento contiene a qué otro — como relación estructural de primera clase, porque es la operación que permite preguntar "¿hay un bilink que cubra esta función?".

Ver [concepts/node.md](concepts/node.md) y [concepts/edge.md](concepts/edge.md).

## Proveedores

Lattice no produce aristas. No parsea código, no habla LSP y no lee markdown. Define un contrato y cada fuente registra el suyo:

```
bilinker            → aristas bilink, impact e issue   (accepted)
proveedor LSP       → aristas de llamada               (derived)
proveedor markdown  → links entre documentos           (asserted)
```

Cada proveedor entrega sus aristas con los nodos ya resueltos a forma canónica. La resolución de una cadena de bilinks a través de capas la hace bilinker, porque es conocimiento de su formato; componerla con aristas ajenas lo hace lattice.

Un proveedor ausente reduce el grafo, no lo invalida — y lattice lo reporta explícitamente en vez de devolver un resultado silenciosamente incompleto.

Ver [concepts/provider.md](concepts/provider.md).

## Consulta

Un solo comando de consulta, [`lattice graph`](commands/graph.md), recorre el grafo desde un selector con las aristas que se le habiliten. Lo que en bilinker eran dos comandos —`graph` para recorrer cadenas e `impact` para subir por el call graph— son el mismo traversal con distinto filtro de `kind`.

[`lattice daemon`](commands/daemon.md) gestiona los language servers que alimentan al proveedor `lsp`.

## Posición en el ecosistema

```
stratum      ← estructura y navegación entre capas
bilinker     ← referencias verificables entre fragmentos
lattice      ← el grafo que forman todas las conexiones del proyecto
impact       ← analiza el alcance de un cambio sobre ese grafo
worklist     ← registra el trabajo pendiente
accreta      ← gobierna la resolución del cambio
```

Lattice es consumidor de bilinker y proveedor de impact. No reemplaza a ninguno de los dos: bilinker sigue siendo el único que verifica y repara referencias; impact sigue siendo el único que evalúa coherencia semántica y abre hilos.

## Principios de diseño

- **La procedencia nunca se aplana**: toda arista declara su origen y su garantía.
  Una consulta puede filtrar por garantía, pero nunca recibe aristas sin ella.
- **Lattice no persiste el grafo**: las aristas `accepted` las persiste su dueño;
  las `derived` se calculan en el momento. El ecosistema ya intentó cachear un
  call graph en archivos (`subgraph.N`, sciplinks) y lo revirtió — un índice
  derivado bajo control de versiones se desincroniza y miente.
- **No produce aristas**: agregar un tipo de conector es escribir un proveedor,
  no modificar el modelo. El grafo crece por acreción.
- **Degradación explícita**: la ausencia de un proveedor se reporta, nunca se
  silencia.
- **Solo lectura**: lattice no escribe en ninguna de sus fuentes.
