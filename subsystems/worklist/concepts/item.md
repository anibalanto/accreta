# Ítem

Un ítem es la unidad de trabajo en worklist. Representa algo que hay que hacer como consecuencia de un cambio en alguna capa del sistema. El fragmento que lo origina puede estar en cualquier capa o repo — arriba, abajo, o en la misma.

## Tipos

| Tipo | Sufijo | Descripción |
|------|--------|-------------|
| **Epic** | `.epic.md` | Objetivo de alto nivel. Agrupa user stories o tasks relacionados. |
| **User Story** | `.user-story.md` | Funcionalidad desde la perspectiva del usuario. Agrupa tasks. |
| **Task** | `.task.md` | Unidad de trabajo concreta y ejecutable. Sin hijos. |
| **Sprint** | `.sprint.md` | Iteración. Agrupa por **referencia**, no por contención. Vive en `_sprints/`. |

## Identificación

**Los sprints llevan índice propio.** Su id no sale del contador de épicas, user stories y tasks: `_sprints/1.sprint.md` es el sprint 1. Son otro eje —tiempo, no descomposición— y su número es parte de cómo se los nombra, así que compartir contador daría `u.sprint.md` titulado "Sprint 1", dos nombres para lo mismo.

El costo es que `1` deja de identificar una sola cosa. Se desambigua con el sufijo, que ya está en el nombre del archivo: `worklist show 1` es el ítem, `worklist show 1.sprint` es el sprint.

El nombre del archivo es `<id>.<tipo>.md`, y vive en la raíz de `worklist/`. El tipo va en el medio y la extensión es siempre `.md`.

### Un ítem tiene un solo id a la vez

> **Nace con `@<slug>`, y al sincronizar el servidor le da el suyo.** El `@<slug>` deja de existir: no hay dos ids vivos, ni un campo que guarde el otro.

```
@arreglar-el-hook.task.md   escrito y todavía sin sincronizar
8y.task.md                  sincronizado, sin proveedor configurado
ACC-347.task.md             sincronizado, con Jira configurado
```

**Un worklist se puede usar al 100% sin ningún proveedor**, y el día que se adopta uno los ids migran — ver [`adopt`](../commands/adopt.md). La alternativa era que el id local fuera la identidad para siempre y la clave un campo; se descartó porque le da al ítem **dos nombres a la vez**, y entonces todo lo que lo nombra tiene que elegir cuál usa.

### Y lo que va después del `@` es un slug descriptivo, no un contador

> **Un id no se puede asignar a mano.** Tomar *"el siguiente base-36 libre"* es leer un contador del filesystem, y dos personas trabajando a la vez toman **el mismo**.

Y es lo peor que puede colisionar: colisiona exactamente cuando dos personas trabajan al mismo tiempo —el caso normal— y sobre ítems que no tienen nada que ver entre sí.

| | Colisiona | Cuesta |
|---|---|---|
| contador leído del filesystem | cuando dos trabajan a la vez — **siempre** | nada de escribir |
| aleatorio o uuid | nunca | no se puede escribir ni referenciar mientras redactás |
| **slug descriptivo** | sólo si dos nombran lo mismo igual | hay que nombrarlo |

```
@user-story-de-la-creacion-y-eliminacion.user-story.md
@tarea-de-creacion.task.md          parent: @user-story-de-la-creacion-y-eliminacion
@tarea-de-eliminacion.task.md
```

**Y el slug no necesita ser único entre personas.** El renombre es lo primero que hace el servidor y la propagación es lo último, así que **el `@<slug>` nunca llega al panorama**: es local a la vista donde se escribió.

```
A empuja  @arreglar-el-hook  →  el servidor le da 8y   →  sube 8y
B empuja  @arreglar-el-hook  →  el servidor le da 8z   →  sube 8z
```

Al servidor los pedidos le llegan **en orden**, y cada uno se lleva el siguiente id libre. Son dos ítems distintos con dos ids distintos, y ninguno de los dos vio nunca el nombre del otro. Alcanza con que el slug sea único dentro de lo que sube junto, y eso lo garantiza el filesystem: dos archivos no comparten nombre.

**Lo que sí puede chocar es el título**, porque un pedido se reconoce del otro lado buscando por `summary`. Eso no es de esta decisión —está igual con contadores— y es la identidad implícita por título, que se arregla aparte.

### El contador base-36 sigue existiendo, y es del servidor

Sin proveedor configurado, el id que reemplaza al `@<slug>` sale de un contador base-36 con el alfabeto `[0-9a-z]`, empezando en `1`:

```
1, 2, 3, …, 9, a, b, …, z, 10, 11, …, 1a, 1b, …, 1z, 20, …
```

**Lo que se descartó no fue el contador sino asignarlo a mano**, y son dos cosas distintas: al servidor los pedidos le llegan de a uno, así que su contador **no puede colisionar por construcción** — por la misma razón por la que dos `@arreglar-el-hook` empujados a la vez terminan con dos ids distintos.

| La instalación tiene | El `@<slug>` pasa a ser |
|---|---|
| sólo worklist | un **id base-36**, del contador del servidor |
| un proveedor | la **clave del proveedor** |

**Y no depende del clima sino de la configuración.** Un proveedor caído no da un id base-36 de consuelo: rechaza, como cualquier escritura que no puede probar lo que promete.

**Nada registra el nombre viejo** — ni el frontmatter, ni el nombre del archivo, ni un campo. Lo único que queda es el historial del remoto, donde el hook escribió `rename @arreglar-el-hook -> ACC-347`. Ver [`sync.md`](sync.md) § "El renombre y la reescritura son un solo commit".

**El sprint agrupa distinto que los demás.** Una épica agrupa por descomposición y lo expresa con carpetas: sus user stories viven adentro. Un sprint agrupa por tiempo, y lo expresa con **links en su cuerpo**: los ítems que se lleva siguen viviendo donde estaban. Un mismo sprint puede tomar ítems de épicas distintas, y una user story no deja de pertenecer a su épica por entrar en uno.

**La extensión es `.md` a propósito.** El contenido de un ítem es markdown —frontmatter más descripción— y los ítems se editan a mano, a diferencia de los archivos de bilinker. Un `.epic` a secas no lo abre ningún editor con resaltado, ningún visor lo previsualiza, y las herramientas que indexan una carpeta por extensión —Obsidian entre ellas— no lo ven. El tipo sigue estando en el nombre, que es lo que permite leerlo de un `ls` sin abrir el archivo.

Se referencia directamente por ID, lleve marca o no:

```bash
worklist show 3
worklist show @arreglar-el-hook
```

### El alfabeto de un id

> **Un id es `[A-Za-z0-9_-]+`. Si todavía no cruzó al proveedor, lleva `@` adelante: `@[A-Za-z0-9_-]+`.**

El alfabeto es más ancho que el del contador —`[0-9a-z]`— y no es lo mismo: el contador dice qué se **genera**, el alfabeto dice qué se puede **escribir**. Una clave de proveedor trae mayúsculas y un guión, y un id que alguien escribe a mano puede traer los dos.

**No es una restricción nueva: es la que ya estaba y nadie declaraba.** La implementación la asume en siete lugares —el borde que evita que renombrar `4h` toque a `4h1`, el `parent:` del frontmatter, y los extractores de `relation.*` y de `items`— y ninguna spec la decía. Un id con un carácter afuera de ese conjunto no falla al crearse: falla más tarde, cuando el renombre no lo encuentra o lo parte por la mitad. Escribirlo acá **saca** una restricción de la sombra en vez de agregar una.

Dos caracteres quedan afuera por lo que rompen, y no por gusto:

| | |
|---|---|
| **`.`** | es el separador de tipo. `worklist show 1` es el ítem y `worklist show 1.sprint` es el sprint; con un punto adentro del id, `@a.sprint` es el ítem `@a.sprint` o el sprint `@a`, y nada puede decidir cuál |
| **`/`** | ya estaba prohibido sin decirlo: el parseo del nombre descarta cualquier stem que lo lleve, y de eso depende que `_sprints/17.sprint.md` no se lea como un ítem de la raíz |

### La marca `@`: lo local se declara, no se infiere

> **Todo id local que un proveedor pueda reemplazar lleva `@` adelante. Lo que no lo lleva, es del proveedor.**

```
@arreglar-el-hook.task.md   →   ACC-347.task.md
parent: @arreglar-el-hook   →   parent: ACC-347
```

**La marca va del lado local porque es el único lado que controlamos.** No se le puede exigir una forma a lo que devuelve un proveedor —Jira da `ACC-347`, GitHub da `1234`, el que venga dará lo suyo—, y no hace falta: la pregunta a contestar es siempre *"¿esto ya cruzó?"*, y para eso alcanza con marcar lo que generamos nosotros.

**Y es positiva en vez de inferida.** *"Es local porque no matchea el patrón de Jira"* obliga a conocer el formato de clave de todos los proveedores para no equivocarse con ninguno; *"es local porque empieza con `@`"* no obliga a conocer ninguno. Esa es la diferencia entre una definición y una conjetura, y es lo que le saca el proveedor de adentro a la función que decide qué es un pedido — ver [`sync.md`](sync.md) § "Qué es un pedido".

#### La marca es transitoria, y significa una sola cosa

`@` quiere decir **"esto existe sólo de este lado"**, y nada más. Lo lleva un ítem entre que se escribe y que sincroniza; lo lleva una vista dinámica mientras exista. Los dos lo pierden en el mismo momento: **cuando la cosa pasa a existir del otro lado.**

**No es una marca de persona.** `@julio` se lee como alguien en cualquier otra herramienta, y acá no lo es: el día que haya una dimensión de asignación, se escribe de otra forma. Un formato donde un caracter significa dos cosas obliga a mirar el contexto para leer un nombre.

**Y es legal donde tiene que serlo.** Un id nombra archivos y también ramas —una vista se llama como lo que recorta—, y git acepta `@` en un refname: `refs/heads/@mia` y `refs/heads/vista/@mia` se crean, se checkoutean y se resuelven. Lo único que git reserva es `@` solo —es `HEAD`— y la secuencia `@{` —es la sintaxis de reflog—, y **las dos caen fuera del alfabeto por construcción**: `@[A-Za-z0-9_-]+` exige al menos un caracter después de la marca y no admite `{`. No hace falta ninguna regla extra.

#### Un sprint nunca lleva `@`

No porque se lo exceptúe: porque la marca dice *"esto todavía puede ser reemplazado"* y el nombre de un sprint no lo es, ni al cruzar ni después. El proveedor le da una clave, pero esa clave no es un nombre —en su interfaz no se muestra, no se puede buscar y nadie la escribe—, así que vive en el frontmatter como una **coordenada** y no compite con nada.

> **Un ítem tiene un id. Un sprint tiene un nombre y una coordenada.**

Queda **fuera del alcance** de la regla, que es distinto de estar exento de ella.

## Formato del archivo

```markdown
---
title: <string>
status: open | in-progress | done
created_at: <iso8601-utc>
updated_at: <iso8601-utc>
parent: <id>                      # opcional — ausente en un ítem de raíz
relation.depends: [<id>, …]       # opcional — ver § Relaciones
---

Descripción opcional en Markdown.
```

`parent` y los `relation.<tipo>` son los campos opcionales. Los otros cuatro están siempre.

La asociación con bilinks se declara desde el bilink (endpoint `issue <id>`), no desde el ítem. Ver [asociación ítem ↔ bilink](bilink-tasks.md).

## Estados y transiciones

| Estado | Significado | Cómo se llega | Cómo se sale |
|--------|-------------|---------------|--------------|
| `open` | Pendiente, no iniciado | `worklist new` | `worklist start`, `worklist done`, `worklist remove` |
| `in-progress` | En curso | `worklist start` | `worklist done`, `worklist remove` |
| `done` | Completado | `worklist done` | — |
| `removed` | Ya no aplica | `worklist remove` | — |

## Jerarquía

La relación padre-hijo se declara con el campo `parent`, que lleva el id del padre. Un ítem sin `parent` está en la raíz del árbol.

```
1.epic.md
2.user-story.md      parent: 1
3.task.md            parent: 2
5.task.md            parent: 1      ← task directa bajo la épica
4.task.md                           ← task sin padre
```

**Los hijos se calculan.** Los hijos de `2` son los ítems cuyo `parent` es `2`; no hay lista que mantener. Es la misma regla que el backlog y por el mismo motivo: una lista escrita obligaría a editar dos lugares para recolgar un ítem, y los dos podrían divergir.

**Y no hay carpetas por ítem.** Todos viven en la raíz de `worklist/`. Eso da dos cosas que la jerarquía en carpetas no puede dar:

- **La dirección es componible.** `<id>.<tipo>.md` alcanza para llegar a cualquier ítem, sin recorrer el árbol ni mantener un índice. Con carpetas hay que buscar, porque el path de un ítem depende de una ascendencia que todavía no se conoce.
- **La dirección no cambia nunca.** Recolgar un ítem es editar un campo; el archivo no se mueve. Con carpetas, recolgarlo le cambia el path a un archivo que puede tener bilinks apuntándole, y todos pasan a MOVED por una decisión de planificación.

## Relaciones

Una relación entre dos ítems que **no** es de descomposición se declara con `relation.<tipo>`, y lleva una lista de ids:

```yaml
relation.depends: [4h, 4j]
```

Hoy existe **`relation.depends` y nada más**. Los demás tipos entran cuando haya un caso, no por adelantado.

**Es un namespace y no un campo por tipo** porque es la forma que ya tiene el proveedor: sus tipos de vínculo son un vocabulario abierto que quien administra el proyecto puede extender. Un campo plano por tipo obligaría a tocar este formato cada vez que aparece uno nuevo; el namespace no.

### Se escribe de un lado, y por eso no hay `relation.children`

Una relación la declara **un** extremo. Para padre/hijo ese extremo es el hijo, con `parent`, y § "Jerarquía" ya cierra el otro: *los hijos se calculan*. Un `relation.children` sería el mismo hecho escrito dos veces, y cuando las dos listas discrepen no falla nada — sólo miente.

### Toda referencia es una clave del proveedor

> **Un valor de `relation.<tipo>` es el id de un ítem que existe, con la forma que el proveedor le da.**

Qué forma tiene esa clave es **configuración del proyecto**, no del formato: un proyecto la define y el resto se verifica contra ella.

**El invariante rige sobre lo publicado.** Un ítem que el proveedor todavía no creó no tiene clave, y poder nombrarlo por su id local —el que lleva `@`— es la única forma de que algo pueda ordenar las creaciones. Así que una copia local puede llevar una referencia marcada, y **lo publicado no**: una referencia con `@` en una rama publicada es un ítem que no cruzó.

**Los ciclos se rechazan.** Una dependencia circular no tiene orden de trabajo ni orden de creación.

## Invariantes

1. El nombre del archivo es `<id>.<tipo>.md`, con un id del alfabeto `[A-Za-z0-9_-]+` —sin `.` y sin `/`— y con `@` adelante mientras no haya sincronizado.
2. El tipo es `epic`, `user-story`, `task` o `sprint`.
3. Todo ítem vive en la raíz de `worklist/`; los sprints, en `_sprints/`. No hay carpetas por ítem.
4. `parent` lleva el id de un ítem que existe, o está ausente. Ningún ítem es su propio ancestro.
5. Un `task` no tiene hijos: ningún ítem lo declara como `parent`.
6. El frontmatter siempre contiene `title`, `status`, `created_at` y `updated_at`.
7. El frontmatter no contiene `source_bilink`. La asociación con bilinks se declara desde el bilink.
8. El frontmatter no contiene `relation.children` ni ningún otro campo que reescriba lo que `parent` ya declara.
9. Todo valor de un `relation.<tipo>` publicado es la clave, con la forma del proveedor, de un ítem que existe. El grafo que forman no tiene ciclos.
10. Que un id sea local o del proveedor se lee de la marca `@` y nunca del formato de la clave. Nada en el worklist conoce la forma de clave de ningún proveedor para decidirlo.
11. Un ítem tiene **un** id: el `@<slug>` lo escribe quien crea el ítem, el que lo reemplaza lo asigna el servidor al sincronizar, y ninguno de los dos sobrevive al otro. Ningún campo del frontmatter guarda un id.
