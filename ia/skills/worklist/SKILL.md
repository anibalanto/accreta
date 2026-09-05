---
name: worklist
description: "Cómo leer y mover el trabajo de este proyecto — épicas, user stories, tasks y sprints en `.worklist/`. Cargar esta skill cuando haya que saber qué sigue, qué hay en el sprint actual, qué está en el backlog, dónde está una tarea, al crear o mover un ítem, o **antes de escribir código**: lleva los chequeos que hay que pasar primero. También al planificar o repartir trabajo."
---

Los ítems viven en `<project-root>/.worklist/`, que es el repo propio del worklist — no una capa de stratum, así que un clon de accreta no lo trae y `stratum pull` tampoco. Spec completa en [`subsystems/worklist/`](../../../subsystems/worklist/concepts/item.md) — acá va lo operativo.

**Están todos en `insecure/all`, y ahí no se trabaja.** El panorama es de donde se lee y contra lo que se consulta; para tocar algo se corta una vista segura. Es la primera regla y está abajo.

**El proyecto lo nombra el directorio y no la rama**, porque el worklist es de accreta y la herramienta no; `insecure/all` es el panorama y se llama igual en todos, porque lo nombra la spec de sincronización. Así lo resuelve `bilinker` un endpoint `issue`, y **siempre contra el panorama, nunca contra una ventana**: si resolviera contra la rama abierta, el mismo `issue <id>` resolvería o no según qué sprint tengas cortado.

## Hay dos repos, y los dos son bare

Es lo que más se confunde, y con razón: están los dos en la misma máquina y ninguno tiene un working tree propio.

| | Dónde | Qué es |
|---|---|---|
| **el servidor** | `~/.local/share/accreta/worklist-sync/worklist.git` | recibe los push. **Es el único con hooks** |
| **tu clon** | `<project-root>/.worklist/` | de donde cuelgan los diecisiete worktrees en los que trabajás. En él, el servidor se llama `srv` |

> **Lo que los distingue no es dónde están: es que uno tiene hooks y el otro no.** Que el servidor sea local es una casualidad de hoy — mañana es un GitLab y nada cambia.

Y el sentido es **de una sola dirección**:

```
tu vista  ──push──▶  el servidor  ──resuelve, y commitea encima──▶  se queda ahí
          ◀──fetch───
```

**El servidor nunca te empuja.** Lo que escribe —los `rename <slug> -> <clave>`, los `normalize:`, el `key` del sprint— vive en *sus* ramas hasta que vos lo traés. Por eso el chequeo 3 de abajo existe: tu worktree puede estar atrasado y verse limpio.

## Se trabaja en una vista segura. El panorama es para leer

> **`insecure/all` no se toca.** Toda modificación de un ítem se hace parado en una vista segura — `secure/…`.

No es una convención: una rama insegura **no se puede verificar** —crece sin techo— así que no acepta escrituras, y no hay forma de empujarle. Lo que se escriba ahí no cruza al proveedor y no pasa por ningún chequeo. **Trabajar en el panorama es escribir en el único lugar del que nada sale.**

El panorama es lo que se **consulta**: qué sigue, dónde está un ítem, qué lleva un sprint, y contra qué resuelve un endpoint `issue`. Para eso es, y para eso lo tiene todo.

Para trabajar se corta la vista, y se edita ahí:

```bash
worklist window open <sprint-id>          # produce secure/sprint/<id>
```

### Y hay un hueco, que conviene saber antes de chocarlo

**Un ítem que no está en ningún sprint no tiene vista segura donde cortarse.** El backlog es inseguro por definición, y la vista para trabajarlo todavía no existe: falta una vista derivada de una consulta al proveedor —`to-work`— y falta que el backlog tenga camino al proveedor. Las dos están decididas y sin implementar.

| | |
|---|---|
| el ítem pertenece a un sprint | se crea y se edita **adentro de su ventana** |
| no pertenece a ninguno | hoy no hay dónde, y hacerlo en el panorama es una excepción a la vista |

Decirlo así y no dar una regla que a veces no se puede cumplir: la excepción existe, y lo que importa es que **se note al hacerla** en vez de que sea el camino por defecto.

### Antes de escribir una línea de código

> **Cuatro cosas, y las cuatro se contestan sin ningún comando del worklist.** Las dos primeras son banderas rojas: no se avanza, se arregla.

**1 · La vista es segura.** Si no, lo que escribas no se puede empujar, no se verifica y no cruza al proveedor. No es una advertencia sobre después: es que el trabajo no tiene dónde ir. Ver § "Cómo saber si la vista es segura".

**2 · El ítem no lleva `@`.** Un `@<slug>` es un id que el servidor todavía no reemplazó, así que **no hay con qué prefijar el commit** — y `AGENTS.md` § Commits pide que arranque con el id del ítem. La salida es sincronizar, no inventar un id.

**3 · La vista está al día.** El servidor **commitea encima** de lo que empujaste —los `rename`, los `normalize:`, el `key` del sprint— y eso no está en tu worktree hasta que lo traigas:

```bash
git fetch srv && git merge --ff-only srv/$(git rev-parse --abbrev-ref HEAD)
```

`--ff-only` y no un merge común: el caso sano es siempre un fast-forward, y **que no lo sea quiere decir que la rama divergió** — alguien más la empujó, o se re-cortó. Un merge automático ahí escondería justo lo que hay que mirar.

**Y no un `reset --hard`**, aunque en el caso sano aterrice en el mismo commit: `reset --hard` **descarta sin preguntar**, y `--ff-only` **se niega**. La negativa es el dato. Un `reset --hard` es lo que se usa cuando ya decidiste tirar lo local — es un `--force`, no un `pull`.

**Y refrescar es una vista por vez**, porque cada una es una rama checkouteada en su propio worktree y git mueve de a una. Traer las dieciséis es un `fetch` y dieciséis merges: eso es lo que `worklist sync` existe para volver un comando.

Y no alcanza con que `git status` diga *"limpio"*: una vista atrasada se ve limpia. Medido: las 16 ventanas estuvieron un commit atrás durante horas y ninguna se veía pendiente.

**4 · El ítem está `in-progress`, no `open`.** `open` quiere decir *"nadie lo tomó"*, y arrancar sin moverlo deja el trabajo invisible para todo lo demás — el board, el sprint, y cualquiera que pregunte qué se está haciendo. Se cambia **en la vista**, con el resto del trabajo, así que viaja al proveedor por el mismo camino.

### El orden que evita los cuatro

```
crear el ítem  →  sincronizar  →  cortar o refrescar la vista  →  pasarlo a in-progress  →  trabajar
```

Es el que el método ya pedía —*primero hay una tarea*— con lo que faltaba: **la tarea no está lista cuando se escribe, está lista cuando tiene id, está en tu vista y dice que la estás haciendo.**

**Los tres comandos que van a hacer esto solos están decididos y no existen todavía**: `worklist is-secure`, `worklist status` y `worklist sync`. Mientras tanto, los chequeos son los de arriba.

### Cómo saber si la vista es segura, hoy

**Es el nombre de la rama, parado en la vista:**

```bash
git rev-parse --abbrev-ref HEAD
```

| Devuelve | |
|---|---|
| `secure/…` | **segura** — se puede empujar, y el push se verifica |
| `insecure/…` | **insegura** — no acepta escrituras |
| cualquier otra cosa | **no estás en una vista del worklist** |

La tercera fila es la que más pasa y la que menos se espera: parado en `accreta` la rama es `main`, y en la capa impl también. Que el comando no diga *"insegura"* ahí es correcto — no es una vista, es otro repo.

**Y hoy esto no es una aproximación: es exacto.** El `pre-receive` decide la clase con `starts_with("refs/heads/secure/")` y nada más, así que preguntar por el nombre calcula **lo mismo** que el servidor va a calcular.

> Deja de ser exacto el día que la clase se **derive** en vez de creerle al nombre — que es lo mismo que trae `worklist is-secure`. Cuando ese comando exista, este chequeo se reemplaza por él y esta sección se borra.

De paso: en `.worklist/` el path espeja el nombre de la rama —`.worklist/secure/sprint/7` lleva `secure/sprint/7`— así que el `pwd` dice lo mismo. **La rama es la autoridad**, porque es lo que el hook lee.

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
relation.<tipo>: [<id>, …] # opcional — `relation.depends` es el que se usa
---
```

Un `.sprint.md` lleva además `items: [<id>, …]` en vez de `parent`: los ids que referencia directamente, y ni uno más — la regla del ancestro dice que un ítem entra con su subárbol entero, así que `items` nunca nombra una task cuya user story ya está en la lista.

Nada más. El tipo lo dice la extensión, la pertenencia la dice `parent`, y la asociación con bilinks **se declara desde el bilink, no desde el ítem**.

**Los hijos se calculan**: los hijos de `n` son los ítems cuyo `parent` es `n`. No hay lista que mantener, igual que el backlog.

## Dos formas de agrupar, y no se mezclan

- **Épica → US → task** es *descomposición*: el campo `parent`.
- **Sprint → ítems** es *planificación*: el campo `items` del `.sprint.md`.

Las dos son campos y las dos se editan en un solo lugar. La diferencia es qué preguntan: `parent` dice de qué es parte un ítem, el sprint dice cuándo se hace.

**El cuerpo del sprint sigue siendo prosa**, y es donde va todo lo que no es membresía: por qué esos ítems son un sprint, en qué orden, qué quedó afuera, cómo cerró. `items` reemplaza al link que declaraba pertenencia, no al texto que explica por qué.

**La regla del ancestro:** *lo que entra a un sprint es **un subárbol entero**.* Una user story entra con **todas** sus tasks, o no entra — sus tasks no se enumeran, van con ella. Y una task se nombra sola sólo cuando no cuelga de ninguna user story. La cadena de ancestros se lee siguiendo `parent` hasta que se acaba.

No alcanza con decir *"entra sólo si ninguno de sus ancestros entra"*: eso deja pasar una task suelta cuando su user story no entra a **ningún** sprint, y por esa puerta la user story queda partida igual, con la única diferencia de que nadie la nombró. La unidad es el subárbol, no la ausencia de conflicto.

De ahí sale que **una US no puede atravesar sprints**: si no cabe en una iteración, está mal dimensionada — y eso es un problema de descomposición, no de planificación.

## Cómo se nombra un ítem

> **`<id> <título>`.** Nunca el id solo.

En markdown el link envuelve las dos cosas:

```markdown
[`3z` Error de cobertura: el vecindario de una firma resuelve a la firma misma, y el tipo que importa no se pregunta](3z.task.md)   ← adentro del worklist, donde el renombre sí llega
```

El id solo obliga a abrir el archivo para saber de qué se habla, y **el que lo abre es el que menos contexto tiene**: quien escribió la referencia ya sabía cuál era. Vale en los ítems y en los sprints.

### Y en una spec no se cita un ítem

> **Ninguna spec nombra un ítem.** Ni con un link, ni con un `` `5y` `` en la prosa.

El id de un ítem **cambia por diseño**: `@arreglar-el-hook` pasa a `ACC-347` el día que cruza. El renombre reescribe su propio repo y ninguno más, así que una cita desde una spec queda apuntando a un archivo que ya no existe — y nada lo detecta, porque es markdown, no un bilink.

**Una spec que cita un ítem apuesta a que su id no cambie.**

Y el proyecto ya tiene dónde va cada cosa:

| | |
|---|---|
| **la spec** | lo que es cierto |
| **el ADR**, en `docs/adr/` de la capa impl | la decisión, y por qué se tomó |
| **el ítem** | el trabajo que la ejecuta |

Una spec que necesita justificar algo cita **el ADR**. Si la justificación sólo existe adentro de un ítem, lo que falta es el ADR — la cita es el síntoma.

**Un endpoint `issue <id>` es otra cosa** y sí referencia un ítem: es una referencia verificada y repuntable, que `bilinker` mantiene. La regla es sobre la prosa, no sobre el mecanismo que existe para esto.

**El título va verbatim, no una glosa.** Una glosa envejece cuando el título cambia y nada lo detecta; un título copiado envejece igual, pero se ve al lado del que cambió. Y donde el título no describa al ítem, lo que hay que arreglar es el título.

Las referencias ya escritas **se corrigen al tocarlas**, no de una barrida.

## Cómo contestar "qué sigue"

1. Buscar el sprint con `status: in-progress`. Si no hay, el próximo `open` por número.
2. Sus links son el compromiso de la iteración. Bajar a la US y de ahí a sus tasks.
3. Cada task dice **qué specs toca**, no qué archivos de código: el código sale de los bilinks que se rompan.

**El backlog no es un archivo.** Se calcula, no se mantiene: tenerlo escrito obligaría a editar dos lugares al mover algo. Y el cálculo va sobre el subárbol, que es lo que un sprint referencia — **un ítem está en el backlog si el tope de su rama no lo nombra ningún sprint**. Las tasks de una user story planificada no se cuentan aparte, y una user story que ningún sprint nombra está en el backlog con todas sus tasks, sin importar cuántas alguien haya querido adelantar.

## Al crear o mover

Los ítems **se escriben a mano hoy**: `worklist new` está especificado pero no implementado, y además delega la asignación de ids a un servidor que no existe. Al crear uno, tomar el siguiente id base-36 libre del contador que corresponda, y escribirlo en la raíz de la vista con su `parent`.

**Y en la vista donde se va a trabajar** — ver § "Se trabaja en una vista segura". Si el ítem pertenece a un sprint, en su ventana; si no pertenece a ninguno, hoy no hay dónde y se hace en el panorama sabiendo que es la excepción.

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
