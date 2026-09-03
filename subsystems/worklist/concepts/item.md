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

El nombre del archivo es `<id>.<tipo>.md`, y vive en la raíz de `worklist/`. El **ID base-36 corto** lo asigna el servidor al crear el ítem; el tipo va en el medio; la extensión es siempre `.md`:

```
1.epic.md
2.user-story.md
3.task.md
a.task.md
1f.task.md
2z.epic.md
```

El ID es base-36 con el alfabeto `[0-9a-z]`, empezando en `1`. El servidor mantiene el contador e incrementa al procesar cada creación:

```
1, 2, 3, …, 9, a, b, …, z, 10, 11, …, 1a, 1b, …, 1z, 20, …
```

**El sprint agrupa distinto que los demás.** Una épica agrupa por descomposición y lo expresa con carpetas: sus user stories viven adentro. Un sprint agrupa por tiempo, y lo expresa con **links en su cuerpo**: los ítems que se lleva siguen viviendo donde estaban. Un mismo sprint puede tomar ítems de épicas distintas, y una user story no deja de pertenecer a su épica por entrar en uno.

**La extensión es `.md` a propósito.** El contenido de un ítem es markdown —frontmatter más descripción— y los ítems se editan a mano, a diferencia de los archivos de bilinker. Un `.epic` a secas no lo abre ningún editor con resaltado, ningún visor lo previsualiza, y las herramientas que indexan una carpeta por extensión —Obsidian entre ellas— no lo ven. El tipo sigue estando en el nombre, que es lo que permite leerlo de un `ls` sin abrir el archivo.

Se referencia directamente por ID:

```bash
worklist show 3
worklist show 1f
```

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

**El invariante rige sobre lo publicado.** Un ítem que el proveedor todavía no creó no tiene clave, y poder nombrarlo por su nombre provisorio es la única forma de que algo pueda ordenar las creaciones. Así que una copia local puede llevar una referencia sin clave, y **lo publicado no**.

**Los ciclos se rechazan.** Una dependencia circular no tiene orden de trabajo ni orden de creación.

## Invariantes

1. El nombre del archivo es `<id>.<tipo>.md`, con un ID base-36 válido asignado por el servidor.
2. El tipo es `epic`, `user-story`, `task` o `sprint`.
3. Todo ítem vive en la raíz de `worklist/`; los sprints, en `_sprints/`. No hay carpetas por ítem.
4. `parent` lleva el id de un ítem que existe, o está ausente. Ningún ítem es su propio ancestro.
5. Un `task` no tiene hijos: ningún ítem lo declara como `parent`.
6. El frontmatter siempre contiene `title`, `status`, `created_at` y `updated_at`.
7. El frontmatter no contiene `source_bilink`. La asociación con bilinks se declara desde el bilink.
8. El frontmatter no contiene `relation.children` ni ningún otro campo que reescriba lo que `parent` ya declara.
9. Todo valor de un `relation.<tipo>` publicado es la clave, con la forma del proveedor, de un ítem que existe. El grafo que forman no tiene ciclos.
