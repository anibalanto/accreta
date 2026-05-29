# Ítem

Un ítem es la unidad de trabajo en worklist. Representa algo que hay que hacer
como consecuencia de un cambio en alguna capa del sistema. El fragmento que lo
origina puede estar en cualquier capa o repo — arriba, abajo, o en la misma.

## Tipos

| Tipo | Extensión | Descripción |
|------|-----------|-------------|
| **Epic** | `.epic` | Objetivo de alto nivel. Agrupa user stories o tasks relacionados. |
| **User Story** | `.user-story` | Funcionalidad desde la perspectiva del usuario. Agrupa tasks. |
| **Task** | `.task` | Unidad de trabajo concreta y ejecutable. Sin hijos. |

## Identificación

El nombre del archivo es un **ID base-36 corto**, asignado por el servidor al
crear el ítem. La extensión define el tipo:

```
1.epic
2.user-story
3.task
a.task
1f.task
2z.epic
```

El ID es base-36 con el alfabeto `[0-9a-z]`, empezando en `1`. El servidor
mantiene el contador e incrementa al procesar cada creación:

```
1, 2, 3, …, 9, a, b, …, z, 10, 11, …, 1a, 1b, …, 1z, 20, …
```

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
source_bilink: <uuid>
---

Descripción opcional en Markdown.
```

`source_bilink` es el UUID del bilink creado por `worklist new`. Puede apuntar
a un fragmento en cualquier capa o repo del ecosistema. Es opcional si el ítem
fue creado sin selector.

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
1.epic
1/
  2.user-story
  2/
    3.task
4.task
```

## Invariantes

1. El nombre del archivo es un ID base-36 válido asignado por el servidor.
2. La extensión es `epic`, `user-story`, o `task`.
3. Un `task` no tiene carpeta homónima.
4. Si existe una carpeta homónima, el archivo padre debe existir.
5. El frontmatter siempre contiene `title`, `status`, y `created_at`.
