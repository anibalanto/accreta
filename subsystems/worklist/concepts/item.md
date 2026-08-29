# Ítem

Un ítem es la unidad de trabajo en worklist. Representa algo que hay que hacer como consecuencia de un cambio en alguna capa del sistema. El fragmento que lo origina puede estar en cualquier capa o repo — arriba, abajo, o en la misma.

## Tipos

| Tipo | Sufijo | Descripción |
|------|--------|-------------|
| **Epic** | `.epic.md` | Objetivo de alto nivel. Agrupa user stories o tasks relacionados. |
| **User Story** | `.user-story.md` | Funcionalidad desde la perspectiva del usuario. Agrupa tasks. |
| **Task** | `.task.md` | Unidad de trabajo concreta y ejecutable. Sin hijos. |
| **Sprint** | `.sprint.md` | Iteración. Agrupa por **referencia**, no por contención. Vive en `sprints/`. |

## Identificación

**Los sprints llevan índice propio.** Su id no sale del contador de épicas, user stories y tasks: `sprints/1.sprint.md` es el sprint 1. Son otro eje —tiempo, no descomposición— y su número es parte de cómo se los nombra, así que compartir contador daría `u.sprint.md` titulado "Sprint 1", dos nombres para lo mismo.

El costo es que `1` deja de identificar una sola cosa. Se desambigua con el sufijo, que ya está en el nombre del archivo: `worklist show 1` es el ítem, `worklist show 1.sprint` es el sprint.

El nombre del archivo es `<id>.<tipo>.md`. El **ID base-36 corto** lo asigna el servidor al crear el ítem; el tipo va en el medio; la extensión es siempre `.md`:

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
---

Descripción opcional en Markdown.
```

La asociación con bilinks se declara desde el bilink (endpoint `todo <id>`), no desde el ítem. Ver [asociación tarea ↔ bilink](bilink-tasks.md).

## Estados y transiciones

| Estado | Significado | Cómo se llega | Cómo se sale |
|--------|-------------|---------------|--------------|
| `open` | Pendiente, no iniciado | `worklist new` | `worklist start`, `worklist done`, `worklist remove` |
| `in-progress` | En curso | `worklist start` | `worklist done`, `worklist remove` |
| `done` | Completado | `worklist done` | — |
| `removed` | Ya no aplica | `worklist remove` | — |

## Jerarquía

La relación padre-hijo se expresa con carpetas:

```
1.epic.md
1/
  2.user-story.md
  2/
    3.task.md
4.task.md
```

## Invariantes

1. El nombre del archivo es un ID base-36 válido asignado por el servidor.
2. La extensión es `epic`, `user-story`, o `task`.
3. Un `task` no tiene carpeta homónima.
4. Si existe una carpeta homónima, el archivo padre debe existir.
5. El frontmatter siempre contiene `title`, `status`, y `created_at`.
6. El frontmatter no contiene `source_bilink`. La asociación con bilinks se declara desde el bilink.
