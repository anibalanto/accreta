---
title: El endpoint del worklist se llama `issue`, no `task`
status: done
created_at: 2026-08-30T01:59:11Z
updated_at: 2026-08-30T01:59:11Z
parent: 5
---

# El endpoint del worklist se llama `issue`, no `task`

El prefijo `task <id>` nombraba mal lo que apunta, y la task `u` lo empeoró al hacerlo explícito: el endpoint resuelve **cualquier** ítem del worklist, así que `task n` puede apuntar a `n.user-story.md`.

## Por qué chirriaba

La palabra significa dos cosas en dos niveles del ecosistema:

| Nivel | "task" es |
|---|---|
| Accreta ([`concepts/workitem.md`](../../concepts/workitem.md)) | cualquier unidad de trabajo — épica, story o task son WorkItems de tipo `Task` |
| worklist ([`item.md`](../../subsystems/worklist/concepts/item.md)) | el tipo hoja, el que no tiene hijos |

El endpoint usaba el primer sentido y quien lo lee está parado en el segundo. Que `item.md` hubiera escrito `todo <id>` en algún momento muestra que el roce ya se había sentido.

## Por qué `issue` y no las otras

[ADR-0005](../../subsystems/bilinker/.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md) fijó la regla al nombrar el endpoint repo: **el nombre sale de qué es la cosa, no de cómo se la trae.** De ahí sale la convención que tienen los cuatro prefijos —`capture` nombra un capture y no a bilinker, `repo` nombra un repo y no a git— y de ahí salen los descartes:

| | Por qué no |
|---|---|
| `worklist <id>` | nombra al proveedor, que es el eje que la regla descarta |
| `work-item <id>` | `concepts/workitem.md` lo tiene tomado para el **supertipo** de todo Accreta: prometería una Spec o un Vote y entregaría un ítem de worklist |
| `item <id>` | no carga la noción de trabajo |
| `todo <id>` | falso en cuanto el ítem pasa a `done`, y el bilink sobrevive como registro histórico |

`issue` nombra la cosa, es término corriente para una unidad de trabajo con id, y no lo reclamaba nadie: cero apariciones en las seis capas antes de este cambio.

## Alcance

**bilinker:** `concepts/reference.md` § "Endpoint issue" —que ahora explica el porqué del nombre—, `bilink.md` (tabla de tipos, prefijos, invariante 7), `capture.md`, `consistency.md`, `proposals/bilink-endpoint.md`. En el código, `LinkEndpoint::Task` → `Issue` y `task.rs` → `issue.rs`, con `resolve_task_path` → `resolve_issue_path`.

**lattice:** el renombre se propaga porque lattice consume el vocabulario. El `kind` de la arista pasa a `issue` y la forma canónica del nodo, de `task:<id>` a `issue:<id>` — `concepts/{node,edge,provider}.md`, `commands/graph.md`, `integration/bilinker.md`, `overview.md`, más `model.rs` y `provider.rs`.

**worklist:** `concepts/item.md` y `concepts/bilink-tasks.md`.

## Sin migración, y por poco

No hay ni un solo endpoint de este tipo en las seis capas, así que `task <id>` nunca llegó a escribirse en disco y el renombre no deja nada que convertir. Después de [ADR-0006](../../subsystems/bilinker/.stratum/impl/docs/adr/0006-formato-como-crate-versionado.md) el mismo cambio sería bump de versión de formato más migración: se hizo ahora porque era gratis ahora.

El parser tampoco acepta el prefijo viejo. Un `task 3a` cae a path Stratum como cualquier valor sin prefijo conocido, y hay un test que lo fija.

## Tests

`parse_issue_endpoint`, `parse_issue_endpoint_longer_id` y `the_old_task_prefix_is_not_an_issue` en `link.rs`; `an_issue_endpoint_resolves_by_id_whatever_the_item_type` y `an_unknown_issue_id_is_todo` en `integration.rs`, renombrados desde los de la task `u`.
