# Especificación: comando `bilinker abstracts`

## Propósito

Qué abstracciones hay para consumir, **con su código**.

Es el paso previo a `chain new --from-repo`: para colgarse de algo hay que poder ver de qué. Sin esto, la lista de lo que un proveedor publica viaja por chat, y elegir es elegir entre uuids.

## Firma

```
bilinker abstracts [<alias>] [-n <líneas>]
```

| Argumento | Descripción |
|---|---|
| `<alias>` | El proveedor a mirar. Sin argumento, lo que publica **esta** capa. |
| `-n` | Cuántas líneas del fragmento mostrar. Default 3; `0` muestra todas. |

## Dos preguntas, un comando

Con alias, la del consumidor: *¿qué publica `hsi`, y de qué me conviene colgarme?*

```
$ bilinker abstracts hsi
hsi · 3 abstracción(es)

  ca4dbbd9  src/main/java/ar/hsi/Permisos.java
            public boolean puede(String usuario, String operacion) {
                return consultar(usuario, operacion);
            }

  d290a546  src/main/java/ar/hsi/Turnos.java   ← ya lo consumís
            public Turno reservar(String paciente, LocalDate fecha) {

  5ed1d14e  src/main/java/ar/hsi/Padron.java
            public int contar(String institucion) {

Para colgarse de una: bilinker chain new --from-repo 'hsi:<uuid>' --tip <tu fragmento>
```

Sin alias, la del proveedor: *¿qué estoy publicando?* — que es una pregunta que conviene poder contestar antes de que alguien la haga.

> **Y esa mitad es [`chain list --link abstract`](chain.md#y-abstracts-sin-alias-es-esto-con---link-abstract).** Listar los bilinks de la capa propia con una punta `abstract` no es de este comando: es el listado general con un filtro puesto, y tenerlo dos veces con dos formatos es lo que hace que se separen. Lo que sí es de acá es la fila con alias, que mira **el repo de otro** — y eso `chain list` no lo puede hacer.

**La marca `← ya lo consumís`** sólo aparece mirando a un proveedor, y es lo que evita colgarse dos veces de lo mismo. Del lado propio no tiene sentido: nadie consume lo suyo.

## Muestra el código, no una lista de ids

Una lista de uuids no alcanza para elegir. Lo que decide de cuál colgarse es **qué dice el fragmento**, y por eso el catálogo lo trae.

Con `-n 0` se muestra entero. El default de 3 líneas es para poder recorrer diez abstracciones de un vistazo; leer una en detalle es [`get`](get.md).

## No trae nada, y no amplía el sparse

Ésta es la propiedad que lo hace barato, y no es obvia: **el clon recorta el árbol de trabajo, no el object store.**

El [`fetch`](../concepts/frontier.md#el-clon-check-nunca-hace-red) trae un commit entero con sus blobs, y el sparse-checkout decide sólo qué sale a disco. Así que el fragmento de una abstracción que no consumís se lee igual —con `git show`— aunque su archivo no esté en el árbol.

De ahí que mirar el catálogo:

- **no hace red**: opera sobre lo que el último `fetch` ya trajo;
- **no saca ningún archivo al árbol**;
- **no ensucia el conjunto sparse**, que sigue siendo derivado de lo que se consume y no de lo que se miró.

Mirar y consumir quedan separados, que es lo correcto: mirar diez para elegir una no debería dejar nueve archivos ajenos en tu working tree.

## Sin clon no hay catálogo

```
$ bilinker abstracts hsi
error: el repo 'hsi' no está clonado. Traerlo primero: `bilinker fetch hsi`.
```

No lo clona solo, por la misma razón que [`check`](check.md) no lo hace: traer un repo ajeno es un acto explícito.

## Qué cuenta como abstracción

Un bilink con **una punta `abstract`**. Los demás bilinks del proveedor son suyos —vínculos internos entre sus capas— y no están abiertos a que nadie se cuelgue: el catálogo no los lista.

Si el capture de una no resuelve contra la versión traída, se dice y no se inventa:

```
  5ed1d14e  src/main/java/ar/hsi/Padron.java
            (el fragmento no se pudo resolver)
```

Es información útil: una abstracción publicada cuyo fragmento no se localiza es algo que el proveedor tiene que mirar, y colgarse de ella hoy sería colgarse de algo roto.

## Códigos de salida

| Código | Condición |
|---|---|
| 0 | Listado, o no hay ninguna. |
| 1 | El repo del proveedor no está clonado; o el alias no está declarado; o su formato no se entiende. |

## Propiedades garantizadas

- **No hace red.**
- **No escribe nada**: ni en el árbol, ni en la cache, ni en el sparse.
- Sólo lista bilinks con una punta `abstract`.
- Un fragmento que no resuelve se reporta como tal.
