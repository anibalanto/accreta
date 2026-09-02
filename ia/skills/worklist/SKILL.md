---
name: worklist
description: "Cómo leer y mover el trabajo de este proyecto — épicas, user stories, tasks y sprints en `.stratum/worklist-accreta/`. Cargar esta skill cuando haya que saber qué sigue, qué hay en el sprint actual, qué está en el backlog, dónde está una tarea, o al crear o mover un ítem. También al planificar o repartir trabajo."
---

El trabajo vive en `<project-root>/.stratum/worklist-accreta/`, que es **una capa con repo propio** — un clon de accreta no la trae. Spec completa en [`subsystems/worklist/`](../../../subsystems/worklist/concepts/item.md) — acá va lo operativo.

El nombre lleva el proyecto porque el worklist es de accreta y la herramienta no. La convención es el prefijo: la capa de worklist de un proyecto es su `.stratum/worklist*`, y así la busca `bilinker` para resolver un endpoint `issue`.

## Qué hay

Todos los ítems son **archivos sueltos en la raíz**. No hay carpetas por ítem.

```
1.epic.md                 épica 1
n.user-story.md           parent: 1
o.task.md                 parent: n
q.task.md                 sin parent — suelta
_sprints/                 el otro eje; el `_` garantiza que nunca sea un id
  1.sprint.md … 7.sprint.md
```

| Tipo | Sufijo | Puede tener de padre |
|---|---|---|
| Epic | `.epic.md` | nada |
| User Story | `.user-story.md` | nada, epic |
| Task | `.task.md` | nada, epic, user story |
| Sprint | `.sprint.md` | nada — vive en `_sprints/` |

Ids **base-36** (`1…9, a…z, 10…`), de orden de creación, no de prioridad. Los sprints llevan **contador aparte**: `_sprints/1.sprint.md` es el sprint 1. Por eso `1` solo es ambiguo — desambiguar con el sufijo: `show 1` es el ítem, `show 1.sprint` es el sprint.

Frontmatter, cuatro campos obligatorios y uno opcional:

```yaml
---
title: <string>
status: open | in-progress | done
created_at: <iso8601-utc>
updated_at: <iso8601-utc>
parent: <id>              # opcional — ausente en un ítem de raíz
---
```

Nada más. El tipo lo dice la extensión, la pertenencia la dice `parent`, y la asociación con bilinks **se declara desde el bilink, no desde el ítem**.

**Los hijos se calculan**: los hijos de `n` son los ítems cuyo `parent` es `n`. No hay lista que mantener, igual que el backlog.

## Dos formas de agrupar, y no se mezclan

- **Épica → US → task** es *descomposición*: el campo `parent`.
- **Sprint → ítems** es *planificación*: links en el cuerpo del sprint.

Las dos son referencias y las dos se editan en un solo lugar. La diferencia es qué preguntan: `parent` dice de qué es parte un ítem, el sprint dice cuándo se hace.

**La regla del ancestro:** un ítem entra a un sprint sólo si ninguno de sus ancestros entra. Si va la US, sus tasks van con ella y no se enumeran. Una task se nombra sola cuando no tiene US arriba. La cadena de ancestros se lee siguiendo `parent` hasta que se acaba.

De ahí sale que **una US no puede atravesar sprints**: si no cabe en una iteración, está mal dimensionada — y eso es un problema de descomposición, no de planificación.

## Cómo se nombra un ítem

> **`<id> <título>`.** Nunca el id solo.

En markdown el link envuelve las dos cosas:

```markdown
[`3z` Error de cobertura: el vecindario de una firma resuelve a la firma misma, y el tipo que importa no se pregunta](3z.task.md)
```

El id solo obliga a abrir el archivo para saber de qué se habla, y **el que lo abre es el que menos contexto tiene**: quien escribió la referencia ya sabía cuál era. Vale en los ítems, en los sprints y en las specs que nombran un ítem — el problema es el mismo en los tres.

**El título va verbatim, no una glosa.** Una glosa envejece cuando el título cambia y nada lo detecta; un título copiado envejece igual, pero se ve al lado del que cambió. Y donde el título no describa al ítem, lo que hay que arreglar es el título.

Las referencias ya escritas **se corrigen al tocarlas**, no de una barrida.

## Cómo contestar "qué sigue"

1. Buscar el sprint con `status: in-progress`. Si no hay, el próximo `open` por número.
2. Sus links son el compromiso de la iteración. Bajar a la US y de ahí a sus tasks.
3. Cada task dice **qué specs toca**, no qué archivos de código: el código sale de los bilinks que se rompan.

**El backlog no es un archivo.** Un ítem que ningún sprint referencia está en el backlog por definición; se calcula, no se mantiene. Tenerlo escrito obligaría a editar dos lugares al mover algo.

## Al crear o mover

Los ítems **se escriben a mano hoy**: `worklist new` está especificado pero no implementado, y además delega la asignación de ids a un servidor que no existe. Al crear uno, tomar el siguiente id base-36 libre del contador que corresponda, y escribirlo en la raíz de `worklist/` con su `parent`.

Mover un ítem es editar **un solo campo o un solo link**, nunca un archivo:

- de sprint: el link sale de un `.sprint.md` y entra en otro.
- de padre: cambia `parent`. El archivo no se mueve, así que su path no cambia y ningún bilink que lo apunte se entera.

### El título

> **Infinitivo cuando ya sabés qué hacer. Diagnóstico cuando lo que tenés es el síntoma.**

No son dos gustos: un título en infinitivo **obliga a nombrar la solución**. Sobre un bug recién visto eso es inventarla antes de decidirla, y el título queda casado con una hipótesis. Sobre trabajo ya decidido es al revés — un título diagnóstico hace parecer que la tarea *causa* el problema en vez de resolverlo.

**Infinitivo:** *"Especificar e implementar `bilinker history`"* · *"Proteger `refs/bilink/*`"* · *"Reescribir `concepts/migration.md`"*.

**Diagnóstico**, con la forma `<Categoría>: <síntoma observable> <cuándo pasa>` — el síntoma en términos de lo que se ve, no de la causa sospechada:

> **Error de reporte:** un endpoint no resuelto no informa qué query ni qué anchor buscaba

El vocabulario de categorías **se deja crecer**, no se inventa por adelantado.

En los dos casos el **cuerpo** abre con el diagnóstico. Lo que cambia es el título.
