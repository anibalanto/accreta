# Accreta

Specs de un ecosistema de herramientas: **bilinker** (referencias verificadas entre fragmentos), **stratum** (capas), **lattice** (grafo), **lspd** (multiplexor de language servers), **impact**, **worklist**. Cada subsistema tiene sus specs acá y su implementación Rust en `subsystems/<nombre>/.stratum/impl/`, que es un repo git independiente y gitignoreado por su padre.

**Las capas se traen con `stratum pull`**, que lee la declaración de cada una en `.stratum/.<nombre>.toml`. La de `impact` **no está declarada porque su repo no está publicado**: existe sólo local, con su ADR adentro.

## Dónde está el trabajo

En `.stratum/worklist-accreta/` — épicas, user stories, tasks y sprints. Para saber qué sigue: el sprint con `status: in-progress` en `_sprints/` —o, si no hay ninguno, el próximo `open` por número—, y de ahí a los ítems que referencia.

**Es una capa con repo propio**, `git@github.com:anibalanto/worklist-accreta.git`, igual que las capas impl de cada subsistema. Un clon de accreta no la trae: `stratum pull` sí. El nombre lleva el proyecto porque el worklist es de accreta y la herramienta no — la spec del formato está en `subsystems/worklist/` y no nombra a nadie.

Guía operativa del formato: `.claude/skills/worklist/SKILL.md`. Spec completa: `subsystems/worklist/`.

Las decisiones que ese trabajo ejecuta viven en `docs/adr/` de la capa impl del subsistema correspondiente.

## Cómo se trabaja acá

0. **Primero hay una tarea.** Ninguna modificación al repo empieza sin un ítem en `.stratum/worklist-accreta/`.
1. Se toca **la spec**, nunca el código primero.
2. `bilinker check .` reporta los endpoints que quedaron no-OK.
3. Cada no-OK es un puntero al fragmento de código que implementaba esa spec. Se sigue con `bilinker get`.
4. Se cambia el código y se acepta.

**El paso 0 no es burocracia: es lo que hace que el paso 2 signifique algo.** El inventario de no-OK contesta *"qué código hay que tocar"*, y no contesta *"para qué"* — eso lo dice el ítem, y sin él un cambio queda sin criterio de cierre y sin nada contra lo cual auditarlo después.

Aplica a specs, a código, a ADRs y a propuestas. **La única excepción es crear o mover un ítem del worklist**, porque exigirle tarea a eso sería recursivo.

Conversar, leer, explorar y proponer no modifican nada y no necesitan ítem. Pero cuando una conversación de diseño produce algo que vale escribir, **lo primero que se escribe es la tarea.**

**El inventario de trabajo de un cambio *es* la lista de no-OK.** Buscar el código a mano produce una lista que envejece el mismo día que se escribe; los bilinks están vivos.

Guía operativa: `ia/skills/bilinker/SKILL.md`. `.claude/skills` es un symlink relativo a `ia/skills`: hay una sola copia de cada skill, y no puede divergir.

## Paths

Los paths se escriben con tokens Stratum —`*` raíz, `<` subir, `>name` bajar— y se resuelven con `$(stratum '...')`. No hardcodear rutas absolutas. Guía: `.claude/skills/stratum-paths/SKILL.md`.

## Commits

Mensaje de una línea, sin trailer de co-autoría.

**Arranca con el id del ítem que se ejecuta** — `1e: separar absorber de decidir`. Es lo que vuelve el log navegable por tarea, y la vuelta que cierra el paso 0: si toda modificación empieza con un ítem, el commit dice cuál.

**Un commit ejecuta un solo ítem.** Si un cambio ejecuta dos, son dos commits.

**Sin prefijo** sólo cuando el commit no ejecuta un ítem sino que **crea o anota varios** — ahí el ítem se nombra en la prosa, porque forzar un prefijo obligaría a elegir a uno como dueño de la creación de todos.
