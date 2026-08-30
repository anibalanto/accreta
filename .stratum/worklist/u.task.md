---
title: El endpoint task no resuelve a ningún archivo
status: done
created_at: 2026-08-29T23:50:00Z
updated_at: 2026-08-30T01:36:34Z
parent: 5
---

# El endpoint task no resuelve a ningún archivo

Tres fuentes decían tres cosas distintas sobre dónde vive el ítem al que apunta `task <id>`, y ninguna coincidía con el disco.

| Fuente | Decía |
|---|---|
| `reference.md`, `bilink.md`, `worklist/concepts/bilink-tasks.md` | `<project-root>/.stratum/worklist/<id>.task.md` |
| `task.rs::resolve_task_path` | `<project-root>/.stratum/worklist/<id>.task` — sin `.md` |
| El disco | `.stratum/worklist/1/n/o.task.md` — anidado bajo épica y user story |

Lo del `.md` era un typo. Lo otro no: **el path de un ítem dependía de una ascendencia que el endpoint no lleva**, así que ninguna ruta fija lo alcanzaba. Y no había ni un solo endpoint `task` en uso en las seis capas — por eso nadie lo había notado.

## Cómo se resolvió: el worklist se aplanó

En vez de agregarle una búsqueda o un índice a bilinker, se sacó la causa. Los ítems dejan de vivir en carpetas por épica y user story: **son archivos sueltos en la raíz de `worklist/`, y la jerarquía la declara el campo `parent`**.

```
1.epic.md
n.user-story.md      parent: 1
o.task.md            parent: n
_sprints/1.sprint.md
```

Dos propiedades que la jerarquía en carpetas no podía dar:

- **La dirección es componible.** `<id>.<tipo>.md` en un solo directorio. El tipo no viaja en el endpoint y no hace falta: los ids son únicos, así que se busca por el prefijo `<id>.`.
- **La dirección no cambia nunca.** Recolgar un ítem de otra user story edita un campo; el archivo no se mueve. Antes le cambiaba el path a un archivo que puede tener bilinks apuntándole, y todos pasaban a MOVED por una decisión de planificación.

Y los hijos **se calculan** —los ítems cuyo `parent` es `n`—, que es la misma regla que el backlog ya seguía por el mismo motivo: una lista escrita obligaría a editar dos lugares y los dos podrían divergir.

## Qué se tocó

**Specs de worklist:** `concepts/item.md` (campo `parent`, § "Jerarquía" reescrita, invariantes 3 y 4 reemplazadas), `concepts/hierarchy.md` (la relación deja de expresarse con carpetas; la regla del `_` se simplifica porque ya ningún directorio puede ser un ítem), `concepts/bilink-tasks.md` (resolución por prefijo) y `architecture.md`, que además estaba vieja en tres puntos más: extensiones sin `.md`, `source_bilink` en el frontmatter —que `item.md` contradice— y el archivo `<uuid>.tasks`, que `bilink-tasks.md` inv. 3 dice explícitamente que no existe.

**Specs de bilinker:** `concepts/reference.md` § "Endpoint task" y la fila de la tabla de tipos en `concepts/bilink.md`.

**Código:** `task.rs::resolve_task_path` busca por prefijo en un solo directorio y devuelve `Ok(None)` si no hay ítem; dos archivos con el mismo id son un error del worklist y se reportan en vez de elegir uno. `check_task` mapea "no está" a TODO o BROKEN según haya hash aceptado, con el mismo criterio que un endpoint layer. Y `accept.rs` tenía **una segunda copia del bug** —`format!(".stratum/worklist/{id}.task")` para preguntarle el commit a git—, que ahora sale del archivo encontrado.

**Migración:** los 34 ítems se movieron a la raíz con su `parent`, y se reescribieron los links relativos. De paso se arreglaron cinco que ya estaban rotos: los ítems a tres niveles de profundidad usaban `../../../subsystems/...` donde hacían falta cuatro `../`. Con los archivos planos la profundidad es siempre la misma, así que el error deja de ser posible.

**Skill:** las dos copias, `.claude/skills/worklist/` e `ia/skills/worklist/`.

## Cuándo estuvo hecha

`bilinker check` limpio en las seis capas. Y un `task <id>` resuelve por primera vez:

```
$ bilinker check .          # bilink con link.1: task u
6e1cc357  (PENDING, PENDING)
$ bilinker check .          # bilink con link.1: task zz9
10ec5b2f  (PENDING, TODO)
```

Tests: `a_task_endpoint_resolves_by_id_whatever_the_item_type` y `an_unknown_task_id_is_todo`, en `crates/bilinker-cli/tests/integration.rs`. Los dos fallan contra el código viejo.

## Lo que se llevó puesto de paso

`concepts/reference.md` § "Discriminación de tipos" no listaba `capture <uuid>`: presentaba como *la* forma estructural lo que el código trata como formato anterior al split, pendiente de `migrate`. Era el segundo bloqueo de estos cinco endpoints y se corrigió acá, con las cinco condiciones en el orden real de `link.rs::from_str` y las dos filas legacy marcadas como tales. La reescritura completa que ADR-0003 pide sigue siendo la task `7`.
