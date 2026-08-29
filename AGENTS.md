# Accreta

Specs de un ecosistema de herramientas: **bilinker** (referencias verificadas entre fragmentos), **stratum** (capas), **lattice** (grafo), **impact**, **worklist**. Cada subsistema tiene sus specs acá y su implementación Rust en `subsystems/<nombre>/.stratum/impl/`, que es un repo git independiente y gitignoreado por su padre.

## Dónde está el trabajo

En `.stratum/worklist/` — épicas, user stories, tasks y sprints. Para saber qué sigue: el sprint con `status: in-progress` en `_sprints/` —o, si no hay ninguno, el próximo `open` por número—, y de ahí a los ítems que referencia.

Guía operativa del formato: `.claude/skills/worklist/SKILL.md`. Spec completa: `subsystems/worklist/`.

Las decisiones que ese trabajo ejecuta viven en `docs/adr/` de la capa impl del subsistema correspondiente.

## Cómo se trabaja acá

1. Se toca **la spec**, nunca el código primero.
2. `bilinker check .` reporta los endpoints que quedaron no-OK.
3. Cada no-OK es un puntero al fragmento de código que implementaba esa spec. Se sigue con `bilinker get`.
4. Se cambia el código y se acepta.

**El inventario de trabajo de un cambio *es* la lista de no-OK.** Buscar el código a mano produce una lista que envejece el mismo día que se escribe; los bilinks están vivos.

Guía operativa: `.claude/skills/bilinker/SKILL.md` — **hoy describe el formato anterior**, ver el aviso adentro.

## Paths

Los paths se escriben con tokens Stratum —`*` raíz, `<` subir, `>name` bajar— y se resuelven con `$(stratum '...')`. No hardcodear rutas absolutas. Guía: `.claude/skills/stratum-paths/SKILL.md`.

## Commits

Mensaje de una línea, sin trailer de co-autoría.
