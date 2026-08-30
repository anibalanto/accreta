---
title: El endpoint task no resuelve a ningún archivo
status: open
created_at: 2026-08-29T23:52:50Z
updated_at: 2026-08-30T00:20:49Z
---

# El endpoint task no resuelve a ningún archivo

Tres fuentes dicen tres cosas distintas sobre dónde vive el ítem al que apunta `task <id>`, y ninguna de las tres coincide con el disco.

| Fuente | Dice |
|---|---|
| [`concepts/reference.md`](../../../subsystems/bilinker/concepts/reference.md) § "Endpoint task" | `<project-root>/.stratum/worklist/<id>.task.md` |
| [`concepts/bilink.md`](../../../subsystems/bilinker/concepts/bilink.md) tabla de tipos | `<project-root>/.stratum/worklist/<id>.task.md` |
| [`worklist/concepts/bilink-tasks.md`](../../../subsystems/worklist/concepts/bilink-tasks.md) inv. 1 | `<project-root>/.stratum/worklist/<id>.task.md` |
| `task.rs::resolve_task_path` | `<project-root>/.stratum/worklist/<id>.task` — sin `.md` |
| El disco | `.stratum/worklist/1/n/o.task.md` — anidado bajo épica y user story |

El desacuerdo del `.md` es un typo. El otro no: los ítems se contienen en carpetas por épica y user story, así que **ningún path fijo los alcanza**. Resolver `task <id>` exige buscar por id bajo `.stratum/worklist/`, no componer una ruta.

Decidir eso es el trabajo. El typo se arregla solo después.

## Por qué aparece ahora

La cobertura de la task `o` creó el bilink que faltaba entre § "Endpoint task" y `task.rs`. Ese módulo tenía **cero** bilinks: es exactamente el agujero que `o` existía para cerrar, y el primero que cerró produjo un hallazgo.

## Qué destraba

Cinco endpoints no-OK que no se pueden aceptar sin mentir:

| Bilink | Fragmento |
|---|---|
| `36f0c759` | `reference.md` § "Endpoint task" ↔ `task.rs::resolve_task_path` |
| `1e318d3a`, `98cf8bce`, `990486fd` | `concepts/bilink.md` ↔ `bilink.rs` / `link.rs` |
| `d4530c07` | `concepts/reference.md` ↔ `link.rs` |

## Cuándo está hecha

`bilinker check .` en la raíz y en la capa impl de bilinker no reporta ninguno de esos cinco, y el path que resuelve `task o` encuentra `.stratum/worklist/1/n/o.task.md`.

## Prioridad

No es urgente. Los cinco endpoints no-OK que deja son **conocidos y esperados** hasta que esta task se haga: no son ruido a limpiar ni motivo para aceptar nada. Cualquiera que corra `bilinker check .` y vea esos cinco está viendo esta task, que es exactamente lo que un inventario vivo debería hacer.
