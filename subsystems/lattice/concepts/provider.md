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
bilinker  → [(bilink, accepted), (task, accepted)]        # governs: ver edge.md
lsp       → [(call, derived)]
doc       → [(doclink, asserted), (external, asserted)]
```

### `available()`

```
Available                    puede responder, y lo que devuelve es completo
Degraded { reason }          responde, pero lo que devuelve está incompleto
Unavailable { reason }       no puede responder
```

Se consulta **antes** de componer el grafo, no al primer error. Un proveedor que falla a mitad de un traversal deja un grafo incompleto que ya se reportó como completo.

`Degraded` no es un matiz cosmético: separa *"puedo pedirle aristas"* de *"lo que devuelve alcanza para afirmar que no hay más"*. El caso que lo motiva es el daemon recién arrancado — responde al ping enseguida, pero el language server detrás sigue indexando, así que `callers` devuelve vacío. Reportarlo como `Available` haría pasar **"todavía no sé"** por **"no hay llamadas"**, que es la confusión más cara que puede cometer este subsistema.

Un proveedor `Degraded` cuenta como grafo incompleto a los efectos del código de salida.

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

Es el único proveedor cuya ausencia es esperable en operación normal. Si el daemon no responde, lo arranca (ver [commands/daemon.md](../commands/daemon.md) § "Auto-start") y queda `Degraded` mientras el language server indexa. Que el ejecutable no esté instalado o que el lenguaje no tenga soporte sí son `Unavailable`, con razones distintas que le importan al consumidor.

### `doc`

Extrae links de documentos markdown. Emite `doclink` para destinos dentro del proyecto y `external` para URIs.

Un `doclink` cuyo destino no existe se emite igual, con `broken`. Un link muerto en un documento es información, no un error de lattice.

El nodo de origen y el de destino son **archivos completos**, no fragmentos — un link markdown apunta a un documento. Que un nodo de archivo completo contenga a los fragmentos de ese archivo (ver [node.md](node.md) § "Contención") es lo que permite que un doclink alcance los bilinks declarados sobre sus partes.

#### Qué no se emite

Dos construcciones tienen forma de link y no lo son:

| Construcción | Por qué no |
|---|---|
| Links dentro de bloques de código | Un ejemplo no es una referencia. |
| Imágenes `![alt](x.png)` | Un embed no es una referencia a otro documento. Modelarlo pediría un `kind` propio, y hoy no hay consumidor que lo justifique. |

Un documento con estos cuatro casos —los cuatro realistas en este ecosistema— produce **una** arista, no cinco:

````markdown
Una referencia real: [node](node.md).          ← se emite

```markdown
Ver [capture.md](capture.md) para el formato.   ← documentar la sintaxis
```

```markdown
Evaluá el cambio contra [la decisión]({{adr}}). ← plantilla de skill
```

```markdown
Ver el hilo en [impact](../threads/3a.md).      ← ejemplo del formato .task
```
````

El caso de la **plantilla** es el que decide la regla. `{{adr}}` no existe ni puede existir, así que aparecería como `broken` **de forma permanente** — y encontrar links muertos es el uso principal de este proveedor. Un falso positivo que nunca se puede resolver enseña a ignorar el resultado, que es la peor falla posible para una herramienta de verificación.

El de **documentar la sintaxis** es el más probable: es lo que pasa cuando una spec muestra un ejemplo del formato que define. Ahí el destino sí existe, así que no sale roto — el grafo simplemente afirma una relación que nadie escribió.

#### Es una regla preventiva

Al escribirla, el proyecto tenía **cero** links dentro de bloques de código y **cero** imágenes markdown. La regla no corrige nada observado.

Lo que la justifica es dónde viven los bloques que ya existen: de los tres bloques etiquetados `markdown` en el ecosistema, dos son ejemplos de formato de archivo (`worklist/concepts/item.md`, `worklist/architecture.md`) y el tercero es una plantilla de prompt (`impact/concepts/skills.md`). Las tres son exactamente las construcciones que más probablemente terminen conteniendo un link.

Si la regla estorba, sacarla es de dos líneas y los tests distinguen los dos comportamientos.

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
