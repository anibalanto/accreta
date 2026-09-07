---
name: worklist
description: "Cómo leer y mover el trabajo de este proyecto — épicas, user stories, tasks y sprints en `.worklist/`. Cargar esta skill cuando haya que saber qué sigue, qué hay en el sprint actual, qué está en el backlog, dónde está una tarea, al crear o mover un ítem, o **antes de escribir código**: lleva los chequeos que hay que pasar primero. También al planificar o repartir trabajo."
---

Los ítems viven en `<project-root>/.worklist/`, que es el repo propio del worklist — no una capa de stratum, así que un clon de accreta no lo trae y `stratum pull` tampoco. Spec completa en [`subsystems/worklist/`](../../../subsystems/worklist/concepts/item.md) — acá va lo operativo.

**Lo que tenés en `.worklist/` son las ventanas.** El panorama —`insecure/all`, donde están todos los ítems— **no baja al clon**: vive en el bare del servidor y se lee de ahí. No es que no se trabaje en él: es que no está.

**El proyecto lo nombra el directorio y no la rama**, porque el worklist es de accreta y la herramienta no; `insecure/all` es el panorama y se llama igual en todos, porque lo nombra la spec de sincronización.

## Hay dos repos, y los dos son bare

Es lo que más se confunde, y con razón: están los dos en la misma máquina y ninguno tiene un working tree propio.

| | Dónde | Qué es |
|---|---|---|
| **el servidor** | `~/.local/share/accreta/worklist-sync/worklist.git` | recibe los push. **Es el único con hooks** |
| **tu clon** | `<project-root>/.worklist/` | de donde cuelgan los dieciséis worktrees en los que trabajás — uno por ventana, **ninguno del panorama**. En él, el servidor se llama `srv` |

> **Lo que los distingue no es dónde están: es que uno tiene hooks y el otro no.** Que el servidor sea local es una casualidad de hoy — mañana es un GitLab y nada cambia.

Y el sentido es **de una sola dirección**:

```
tu vista  ──push──▶  el servidor  ──resuelve, y commitea encima──▶  se queda ahí
          ◀──fetch───
```

**El servidor nunca te empuja.** Lo que escribe —los `rename <slug> -> <clave>`, los `normalize:`, el `key` del sprint— vive en *sus* ramas hasta que vos lo traés. Por eso el chequeo 3 de abajo existe: tu worktree puede estar atrasado y verse limpio.

## Se trabaja en una vista segura. El panorama es para leer

> **`insecure/all` no se toca, y ya no se puede.** Toda modificación de un ítem se hace parado en una vista segura — `secure/…`.

No es una convención: una rama insegura **no se puede verificar** —crece sin techo— así que no acepta escrituras, y no hay forma de empujarle. Y desde que el panorama no baja al clon, tampoco hay dónde equivocarse: **el único árbol que tenés a mano es una ventana.**

El panorama es lo que se **consulta**, y se consulta en el servidor:

```bash
SRV=~/.local/share/accreta/worklist-sync/worklist.git
git -C $SRV show insecure/all:ACC-3.task.md          # un ítem
git -C $SRV ls-tree --name-only insecure/all         # el inventario
```

Para trabajar se corta la vista, **y el que corta es el servidor** —el panorama del que sale está de ese lado—; el clon la trae con `fetch` y un worktree:

```bash
cd $SRV && worklist-server window open <sprint-id>   # recorta, parado en el bare

cd $(stratum '*')/.worklist
git fetch srv && git worktree add secure/sprint/<id> secure/sprint/<id>
```

**El nombre es provisorio.** El comando que corta tiene tres candidatos —`window open`, `view add`, `window new`— y no son sinónimos: uno de ellos ya significa *ampliar un recorte que existe*, no cortar uno nuevo. Está sin decidir, así que lo de arriba es lo que hay hoy, no la forma final.

### Y hay un hueco, que conviene saber antes de chocarlo

**Un ítem que no está en ningún sprint no tiene vista segura donde cortarse.** El backlog es inseguro por definición, y la vista para trabajarlo todavía no existe: falta una vista derivada de una consulta al proveedor —`to-work`— y falta que el backlog tenga camino al proveedor. Las dos están decididas y sin implementar.

| | |
|---|---|
| el ítem pertenece a un sprint | se crea y se edita **adentro de su ventana** |
| no pertenece a ninguno | nace igual en una ventana y sube al panorama con ella; lo que falta es la vista para **trabajarlo**, no un lugar donde escribirlo |

**Y la excepción de antes ya no existe.** Decía que un ítem de backlog se escribía en el panorama sabiendo que era una excepción a la vista; el panorama no está de este lado, así que no hay dónde hacerla. Lo que queda es lo de siempre: un ítem nace en una vista y viaja con ella.

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

**Esta receta es para una ventana**, que es lo único que tenés checkouteado. El panorama no se refresca porque no está: es del servidor, y ahí no hay nada que traer.

**4 · El ítem está `in-progress`, no `open`.** `open` quiere decir *"nadie lo tomó"*, y arrancar sin moverlo deja el trabajo invisible para todo lo demás — el board, el sprint, y cualquiera que pregunte qué se está haciendo. Se cambia **en la vista**, con el resto del trabajo, así que viaja al proveedor por el mismo camino.

### El orden que evita los cuatro

```
crear el ítem  →  sincronizar  →  cortar o refrescar la vista  →  pasarlo a in-progress  →  trabajar
```

Es el que el método ya pedía —*primero hay una tarea*— con lo que faltaba: **la tarea no está lista cuando se escribe, está lista cuando tiene id, está en tu vista y dice que la estás haciendo.**

**Los tres comandos que van a hacer esto solos están decididos y no existen todavía**: `worklist is-secure`, `worklist status` y `worklist sync`. Mientras tanto, los chequeos son los de arriba.

### Y el panorama ya no se sincroniza a mano

Acá había una receta de dos direcciones —el servidor tirando del clon para subir, un `--ff-only` o un rebase para bajar— y **se borró entera**: el clon no tiene panorama, así que no hay dos copias que reconciliar.

> **Lo que se sacó no fue la receta: fue la segunda copia.** La receta existía porque había una, y era justamente la que un comando automático leía cuando se quedaba vieja.

Un comando del servidor que necesita el panorama lo lee de sus propias refs, que son la autoridad. Ver [`subsystems/worklist/concepts/sync.md`](../../../subsystems/worklist/concepts/sync.md) § "El panorama vive en un solo lado, y la ventana en los dos".

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

## En una spec no se cita un ítem

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

Las referencias ya escritas **se corrigen al tocarlas**, no de una barrida.

## Cómo contestar "qué sigue": dos pasos, y el orden es obligatorio

> **En el panorama se ve qué hay. En la vista segura se constata que sea lo último.**

No es prolijidad: son las dos promesas, y **son excluyentes**. El panorama eligió estar completo, y por eso mismo *"no puede prometer que estén actualizados"*. Preguntarle si algo está al día es preguntarle lo único que declaró no poder contestar.

**Paso 1 — el panorama, para el inventario.** Está en el servidor, así que se lee con `git -C $SRV show` o `ls-tree`:

1. Buscar el sprint con `status: in-progress`. Si no hay, el próximo `open` por número.
2. Sus `items` son el compromiso de la iteración. Bajar a la US y de ahí a sus tasks.
3. Cada task dice **qué specs toca**, no qué archivos de código: el código sale de los bilinks que se rompan.

**Paso 2 — la vista segura del sprint, para constatar.** Cortada o refrescada, es la única que puede verificarse entera contra el proveedor. El `status` que vale es el de ahí.

**Y no se puede invertir.** La vista no sabe lo que no tiene: preguntarle *"qué sigue"* devuelve lo de su sprint y **calla el resto sin decir que calló**.

### Cuando la vista no está cortada

Pasa seguido, y hoy pasa con todo lo que no sea de los sprints ya subidos. Entonces el segundo paso no se puede dar, y la respuesta es:

> **"Esto es lo que hay, y no está constatado."**

Decirlo **es** el paso. La alternativa es presentar como actual algo que nadie verificó, que es el mismo error de forma que el resto de las reglas de acá evitan.

**El backlog no es un archivo.** Se calcula, no se mantiene: tenerlo escrito obligaría a editar dos lugares al mover algo. **Y se calcula donde está el todo**, que es el servidor: sobre una ventana la misma cuenta da mal, no da menos. Y el cálculo va sobre el subárbol, que es lo que un sprint referencia — **un ítem está en el backlog si el tope de su rama no lo nombra ningún sprint**. Las tasks de una user story planificada no se cuentan aparte, y una user story que ningún sprint nombra está en el backlog con todas sus tasks, sin importar cuántas alguien haya querido adelantar.

## Al crear o mover

Los ítems **se escriben a mano hoy**: `worklist new` está especificado pero no implementado, y además delega la asignación de ids a un servidor que no existe. Al crear uno, tomar el siguiente id base-36 libre del contador que corresponda, y escribirlo en la raíz de la vista con su `parent`.

**Y en la vista donde se va a trabajar** — ver § "Se trabaja en una vista segura". Si el ítem pertenece a un sprint, en su ventana; si no pertenece a ninguno, se escribe igual en una ventana y sube con ella, porque el panorama ya no es un lugar donde se pueda escribir.

Mover un ítem es editar **un solo campo o un solo link**, nunca un archivo:

- de sprint: el link sale de un `.sprint.md` y entra en otro.
- de padre: cambia `parent`. El archivo no se mueve, así que su path no cambia y ningún bilink que lo apunte se entera.
