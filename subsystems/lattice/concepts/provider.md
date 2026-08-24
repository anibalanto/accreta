# Concepto: proveedor

Un proveedor es una fuente de aristas. Lattice no produce ninguna: descubre proveedores, les pregunta por sus aristas en un scope, y compone el resultado.

## Contrato

```
kinds()                 → [(kind, guarantee)]     qué emite y con qué garantía
available()             → Availability            si puede responder ahora
edges(scope)            → [Edge]                  sus aristas en ese scope
```

Opcionalmente, un proveedor que sepa expandir el grafo bajo demanda implementa:

```
edges_from(node)        → [Edge]                  aristas incidentes a un nodo
```

Sin `edges_from`, lattice enumera todo el scope y filtra. Con `edges_from`, el traversal expande de a un nodo — necesario para el proveedor LSP, donde enumerar todas las llamadas del proyecto es inviable pero preguntar por los callers de una función es barato.

### `kinds()`

La garantía de un `kind` es fija y la declara el proveedor, no cada arista. Un proveedor puede emitir varios `kind`:

```
bilinker  → [(bilink, accepted), (impact, accepted), (task, accepted)]
lsp       → [(call, derived)]
doc       → [(doclink, asserted), (external, asserted)]
```

### `available()`

```
Available                    puede responder
Degraded { reason }          responde parcialmente
Unavailable { reason }       no puede responder
```

Se consulta **antes** de componer el grafo, no al primer error. Un proveedor que falla a mitad de un traversal deja un grafo incompleto que ya se reportó como completo.

`HtmlGraph.daemon_ok` (`html_graph.rs:105`) es la versión actual de esto: un `Option<bool>` que se resuelve en el primer uso. La diferencia es que el estado se consulta al inicio y viaja en el resultado.

## Los tres proveedores

### `bilink`

Lee los `.bilink` de la capa y de las capas alcanzables. Emite una arista por cadena, entre los dos tips estructurales — nunca entre nodos `.bilink` intermedios.

Se alimenta de `bilinker graph --format json`, que ya entrega los nodos en forma canónica con la topología de cadena resuelta.

Lleva dos campos que ningún otro proveedor tiene y que impact necesita:

| Campo | Uso |
|---|---|
| `state` | La tupla `(state.0, state.1)` de los tips. Filtrar por no-OK. |
| `commit` | El `commit.N` de cada tip. Baseline de `git log <commit>..HEAD`. |

Sin `commit` en la arista, impact tendría que volver a abrir los `.bilink` para calcular su diff — que es exactamente la duplicación que este subsistema elimina.

### `lsp`

Consulta el daemon (`callees`, `callers`) sobre el socket Unix. Implementa `edges_from`, no `edges`: el call graph no se enumera, se expande.

Requiere resolver el anclaje del nodo antes de preguntar — ver [node.md](node.md) § "Anclaje".

Es el único proveedor cuya ausencia es esperable en operación normal: el daemon puede no estar corriendo, el language server puede no estar instalado, o el lenguaje puede no tener soporte. Los tres casos son `Unavailable` con razón distinta, y las tres razones importan al consumidor.

### `doc`

Extrae links de documentos markdown. Emite `doclink` para destinos dentro del proyecto y `external` para URIs.

Un `doclink` cuyo destino no existe se emite igual, marcado como roto. Un link muerto en un documento es información, no un error de lattice.

## Composición

```
1. Consultar available() de cada proveedor registrado.
2. Pedir edges(scope) a los disponibles que enumeran.
3. Construir el índice de contención con los nodos obtenidos.
4. Expandir con edges_from() los proveedores que lo implementan, según el traversal.
5. Deduplicar.
6. Emitir el grafo junto con el estado de cada proveedor.
```

El paso 3 va antes del 4 a propósito: el índice de contención tiene que existir para que una arista `derived` recién descubierta pueda conectarse con un nodo `accepted` ya conocido.

## Deduplicación

Por `(from, to, kind)`. Se conserva la garantía más fuerte (`accepted` > `derived` > `asserted`) y se registran todos los proveedores de origen.

## Registro

Los proveedores se descubren por configuración, no por hardcodeo. Un proveedor ausente de la configuración no participa y no se reporta como faltante — a diferencia de uno registrado que está `Unavailable`, que sí se reporta.

## Invariantes

1. Lattice no produce aristas propias. La contención no es una arista.
2. Todo proveedor declara sus `kinds()` con garantía fija antes de emitir.
3. `available()` se consulta antes de componer, no al primer fallo.
4. Los nodos llegan en forma canónica y ya resueltos entre capas.
5. Todo resultado lleva el estado de cada proveedor registrado.
6. Un proveedor nunca escribe en sus fuentes al responder una query.
