---
name: worklist
description: "Cómo leer y mover el trabajo de este proyecto — épicas, user stories, tasks y sprints en `.stratum/worklist/`. Cargar esta skill cuando haya que saber qué sigue, qué hay en el sprint actual, qué está en el backlog, dónde está una tarea, o al crear o mover un ítem. También al planificar o repartir trabajo."
---

El trabajo vive en `<project-root>/.stratum/worklist/`. Spec completa en [`subsystems/worklist/`](../../../subsystems/worklist/concepts/item.md) — acá va lo operativo.

## Qué hay

```
1.epic.md                 épica
1/                        contenido de la épica 1 — contención
  n.user-story.md
  n/
    8.task.md
sprints/                  el otro eje, fuera del árbol de descomposición
  1.sprint.md … 7.sprint.md
```

| Tipo | Sufijo | Vive en |
|---|---|---|
| Epic | `.epic.md` | raíz |
| User Story | `.user-story.md` | raíz o carpeta de épica |
| Task | `.task.md` | raíz, carpeta de épica o de user story |
| Sprint | `.sprint.md` | `sprints/` |

Ids **base-36** (`1…9, a…z, 10…`), de orden de creación, no de prioridad. Los sprints llevan **contador aparte**: `sprints/1.sprint.md` es el sprint 1. Por eso `1` solo es ambiguo — desambiguar con el sufijo: `show 1` es el ítem, `show 1.sprint` es el sprint.

Frontmatter, exactamente cuatro campos:

```yaml
---
title: <string>
status: open | in-progress | done
created_at: <iso8601-utc>
updated_at: <iso8601-utc>
---
```

Nada más. El tipo lo dice la extensión, la pertenencia la dice la carpeta, y la asociación con bilinks **se declara desde el bilink, no desde el ítem**.

## Dos formas de agrupar, y no se mezclan

- **Épica → US → task** es *contención*: carpetas.
- **Sprint → ítems** es *referencia*: links en el cuerpo del sprint. Los ítems siguen viviendo donde estaban.

**La regla del ancestro:** un ítem entra a un sprint sólo si ninguno de sus ancestros entra. Si va la US, sus tasks van con ella y no se enumeran. Una task se nombra sola cuando no tiene US arriba.

De ahí sale que **una US no puede atravesar sprints**: si no cabe en una iteración, está mal dimensionada — y eso es un problema de descomposición, no de planificación.

## Cómo contestar "qué sigue"

1. Buscar el sprint con `status: in-progress`. Si no hay, el próximo `open` por número.
2. Sus links son el compromiso de la iteración. Bajar a la US y de ahí a sus tasks.
3. Cada task dice **qué specs toca**, no qué archivos de código: el código sale de los bilinks que se rompan.

**El backlog no es un archivo.** Un ítem que ningún sprint referencia está en el backlog por definición; se calcula, no se mantiene. Tenerlo escrito obligaría a editar dos lugares al mover algo.

## Al crear o mover

Los ítems **se escriben a mano hoy**: `worklist new` está especificado pero no implementado, y además delega la asignación de ids a un servidor que no existe. Al crear uno, tomar el siguiente id base-36 libre del contador que corresponda.

Mover un ítem entre sprints es editar **un solo lugar**: el link sale de un `.sprint.md` y entra en otro. No hay campo `sprint:` en el ítem que mantener en sincronía.
