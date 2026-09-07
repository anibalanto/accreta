# Accreta

Specs de un ecosistema de herramientas: **bilinker** (referencias verificadas entre fragmentos), **stratum** (capas), **lattice** (grafo), **lspd** (multiplexor de language servers), **impact**, **worklist**. Cada subsistema tiene sus specs acá y su implementación Rust en `subsystems/<nombre>/.stratum/impl/`, que es un repo git independiente y gitignoreado por su padre.

**Las capas se traen con `stratum pull`**, que lee la declaración de cada una en `.stratum/.<nombre>.toml`. La de `impact` **no está declarada porque su repo no está publicado**: existe sólo local, con su ADR adentro.

## Antes de nada: las tres guías

**Cargá las tres antes de leer, buscar o tocar cualquier cosa.** No hay condición que evaluar.

| | |
|---|---|
| `ia/skills/worklist/SKILL.md` | el formato del trabajo: ítems, sprints, cómo se nombra y cómo se mueve |
| `ia/skills/bilinker/SKILL.md` | las referencias verificadas — prerequisito del método |
| `ia/skills/stratum-paths/SKILL.md` | cómo se compone cualquier path |

**Las tres las usa todo el trabajo de este repo**, y el método de abajo lo hace explícito: el paso 0 es worklist, los pasos 2 y 4 son bilinker, y cualquier comando con un path es stratum.

Antes decían *cuándo* aplicaba cada una — *"si la tarea toca bilinks"*, *"si vas a componer un path"*. **Ese juicio sólo se puede hacer después de haber empezado a leer**, y para entonces ya leíste sin la guía que dice cómo.

`.claude/skills` es un symlink relativo a `ia/skills`: hay una sola copia de cada guía, y no puede divergir.

## Dónde está el trabajo

En `.worklist/insecure/all/` — épicas, user stories, tasks y sprints. Para saber qué sigue: el sprint con `status: in-progress` en `_sprints/` —o, si no hay ninguno, el próximo `open` por número—, y de ahí a los ítems que referencia.

**No es una capa de stratum**: es el aparato de seguimiento del proyecto, más pariente de `.bilink/` que de `subsystems/`. Vive en `.worklist/`, que es un **contenedor de worktrees** de su repo propio — un clon de accreta no lo trae, y `stratum pull` tampoco: es un `git clone` aparte.

**Ese repo es local**: un bare en `~/.local/share/accreta/worklist-sync/worklist.git`, con los hooks que sincronizan con Jira. No hay copia en GitHub y no la va a haber; el día que haya acuerdo con la empresa, el servidor pasa a un GitLab suyo — **misma forma, otro host**.

`insecure/all` es el panorama, con todos los ítems; una ventana —`secure/sprint/<id>`— lleva sólo los de ese sprint. El path dice la capacidad: a una rama insegura no se le puede empujar. Ver [`subsystems/worklist/concepts/sync.md`](subsystems/worklist/concepts/sync.md).

Spec completa: `subsystems/worklist/`.

Las decisiones que ese trabajo ejecuta viven en `docs/adr/` de la capa impl del subsistema correspondiente.

## Cómo se trabaja acá

0. **Primero hay una tarea.** Ninguna modificación al repo empieza sin un ítem en `.worklist/insecure/all/`.
1. Se toca **la spec**, nunca el código primero.
2. `bilinker check .` reporta los endpoints que quedaron no-OK.
3. Cada no-OK es un puntero al fragmento de código que implementaba esa spec. Se sigue con `bilinker get`.
4. Se cambia el código y se acepta.

**El paso 0 no es burocracia: es lo que hace que el paso 2 signifique algo.** El inventario de no-OK contesta *"qué código hay que tocar"*, y no contesta *"para qué"* — eso lo dice el ítem, y sin él un cambio queda sin criterio de cierre y sin nada contra lo cual auditarlo después.

Aplica a specs, a código, a ADRs y a propuestas. **La única excepción es crear o mover un ítem del worklist**, porque exigirle tarea a eso sería recursivo.

Cuando una conversación de diseño produce algo que vale escribir, **lo primero que se escribe es la tarea.**

**El inventario de trabajo de un cambio *es* la lista de no-OK.** Buscar el código a mano produce una lista que envejece el mismo día que se escribe; los bilinks están vivos.

## El código se escribe en dos idiomas, y el corte no es arbitrario

> **Los identificadores van en inglés. Los comentarios y los doc-comments se quedan en castellano.**

Identificador es todo lo que el compilador lee como nombre: tipos, funciones, métodos, campos, variantes, variables, módulos, features de `Cargo.toml`.

**Un archivo de código de este proyecto tiene dos capas de texto, y sólo una es código.** El identificador es la parte que **sale del repo** —aparece en un `Result` que otro crate destructura, en el mensaje de un panic, en un stack trace—, así que se escribe en el idioma en el que se lee código. El comentario nunca sale de acá: es el mismo registro que una spec o un ADR, escrito donde se lee, y traducirlo sería reescribir el razonamiento del proyecto en un idioma en el que nadie lo pensó.

**Aplica a lo nuevo.** Un identificador en castellano se traduce **cuando se lo toca por otra razón**, nunca de una barrida: un renombre de una pasada deja en `MOVED` todos los bilinks de la capa, y lo hace por un cambio que no arregla nada roto. Convive castellano e inglés por un tiempo largo, y eso es aceptado.

**Y cuando el término es de la spec, gana la spec.** Si la spec dice *"el panorama"* o *"la ventana"*, el código los nombra así: un tipo en inglés ahí obliga a traducir en la cabeza en cada lectura. La regla fija el idioma por defecto, no una prohibición.

## Paths

Los paths se escriben con tokens Stratum —`*` raíz, `<` subir, `>name` bajar— y se resuelven con `$(stratum '...')`. No hardcodear rutas absolutas.

## Commits

Mensaje de una línea, sin trailer de co-autoría.

**Arranca con el id del ítem que se ejecuta** — `1e: separar absorber de decidir`. Es lo que vuelve el log navegable por tarea, y la vuelta que cierra el paso 0: si toda modificación empieza con un ítem, el commit dice cuál.

**Un commit ejecuta un solo ítem.** Si un cambio ejecuta dos, son dos commits.

**Y el prefijo envejece a propósito.** Un ítem se renombra cuando cruza al proveedor —`@arreglar-el-hook` pasa a `ACC-347`— y la historia no se reescribe, así que `git log --grep '^@arreglar-el-hook:'` encuentra commits que el árbol ya no nombra. **Se acepta, y siempre para adelante**: no hay tabla de equivalencias que mantener, porque el log del remoto del worklist ya la tiene —los `rename <viejo> -> <nuevo>` están todos ahí— y una segunda fuente sólo podría diferir de la primera. Lo que se pierde es el salto automático de un commit a su ítem, no entender qué se hizo.

**Sin prefijo** sólo cuando el commit no ejecuta un ítem sino que **crea o anota varios** — ahí el ítem se nombra en la prosa, porque forzar un prefijo obligaría a elegir a uno como dueño de la creación de todos.
