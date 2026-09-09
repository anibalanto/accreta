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
| **el servidor** | lo dice `git remote get-url srv` | recibe los push. **Es el único con hooks** |
| **tu clon** | `<project-root>/.worklist/` | de donde cuelga un worktree por ventana cortada, y **ninguno del panorama**. En él, el servidor se llama `srv` |

> **Lo que los distingue no es dónde están: es que uno tiene hooks y el otro no.** Que el servidor sea local es una casualidad de hoy — mañana es un GitLab y nada cambia.

Y por eso la ruta del servidor no se escribe a mano, se pregunta — es lo que hacen los binarios, que la sacan del remoto y no de una constante:

```bash
SRV=$(git -C $(stratum '*')/.worklist remote get-url srv)
```

Cuántas ventanas hay también es una pregunta, no un dato de esta página: `git -C $(stratum '*')/.worklist worktree list`. Hay una por ventana cortada y traída, que son menos que los sprints.

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
git -C $SRV show insecure/all:ACC-3.task.md          # un ítem
git -C $SRV ls-tree --name-only insecure/all         # el inventario
git -C $SRV show insecure/all:.metadata/product.yaml # la composición
```

### Y hay un hueco, que conviene saber antes de chocarlo

**Un ítem que no está en ningún sprint no tiene vista segura donde cortarse.** El backlog es inseguro por definición, y la vista para trabajarlo todavía no existe: falta una vista derivada de una consulta al proveedor —`to-work`— y falta que el backlog tenga camino al proveedor. Las dos están decididas y sin implementar.

| | |
|---|---|
| el ítem pertenece a un sprint | se crea y se edita **adentro de su ventana** |
| no pertenece a ninguno | nace igual en una ventana y sube al panorama con ella; lo que falta es la vista para **trabajarlo**, no un lugar donde escribirlo |

**Y la excepción de antes ya no existe.** Decía que un ítem de backlog se escribía en el panorama sabiendo que era una excepción a la vista; el panorama no está de este lado, así que no hay dónde hacerla. Lo que queda es lo de siempre: un ítem nace en una vista y viaja con ella.

## Abrir una ventana

> Hay dos casos, y confundirlos es el error caro: si la ventana ya está cortada y traída, la abrís con `worklist pull` y nada más. Los dos comandos de abajo son para la que todavía no existe.

Recortar es del servidor —lee el panorama, que vive de un solo lado— y el clon la trae con git. Son dos pasos, y el segundo es el que le da un worktree:

```bash
cd $SRV
worklist-server window open <sprint-id>               # recorta, parado en el bare

cd $(stratum '*')/.worklist
git fetch srv
git worktree add secure/sprint/<id> secure/sprint/<id>
```

El comando se llama `window open`, tiene [spec](../../../subsystems/worklist/commands/window-open.md) y está implementado. Acá decía que el nombre era provisorio y que había tres candidatos sin decidir; está decidido, y la duda sobraba: hacía dudar del único comando que ya no la tiene.

Antes de escribir la rama se puede preguntar qué entraría:

```bash
worklist-server window open <sprint-id> --dry-run
```

| Flag | Para qué |
|---|---|
| `--dry-run` | lista los archivos que entrarían y no escribe la rama |
| `--from <rama>` | de dónde cortar. Por defecto el panorama |
| `--force` | tira a sabiendas lo local que no se pudo replantar. No es el camino normal |

### Y para una ventana que ya existe, el comando es `pull`

`worklist pull` recorta él mismo: su paso 1 invoca a `window open` en el bare, y después absorbe lo del board, trae y replanta lo tuyo encima. Correr el comando del servidor a mano antes de un `pull` es hacer dos veces lo mismo.

```bash
worklist pull            # esta ventana
worklist pull --all      # todas las del clon; sólo bajada
worklist pull --dry-run  # qué traería y qué replantaría
```

Y es idempotente por construcción, no por suerte: recortar dos veces sobre un panorama quieto produciría dos commits distintos —el sha lleva la hora adentro— así que `window open` compara el árbol y no mueve la rama si el corte es el mismo. Ante la duda, corrélo.

### Si la ventana falta, se corta — no se declara inconstatable

Es la salida para el caso de abajo, *"cuando la vista no está cortada"*: cortar una ventana que falta son las dos líneas de arriba, y decir *"no está constatado"* cuando el comando existe convierte en limitación permanente algo que se arregla en una invocación.

Sigue habiendo un caso sin comando, y es otro: un ítem que no está en *ningún* sprint, porque no hay de dónde cortarle una ventana.

## Leer una ventana

Los ítems son archivos sueltos en la raíz, así que leer la ventana es `ls` y `cat`. No hay `worklist show` ni `worklist list`: están especificados y el binario no los tiene — los subcomandos que hay son `status`, `pull`, `push`, `state change` y `remove`.

```bash
ls *.md                                    # los ítems de esta ventana
grep -m1 '^status:' ACC-343.task.md        # su estado
```

Y lo importante es lo que *no* está adentro:

| | Dónde está | Por qué no baja |
|---|---|---|
| los ítems del sprint, con sus ancestros | en la ventana | es lo que la ventana promete verificar |
| el vocabulario de estados, si el proyecto lo declara | en la ventana | la ventana tiene que cerrar adentro |
| la composición — qué ítems lleva el sprint, el orden del backlog, el `key` | sólo en el panorama | una copia de la composición del lado del cliente es una fuente de verdad que sólo puede quedarse vieja |
| cualquier ítem de otro sprint | sólo en el panorama | parado adentro, un ítem ajeno no puede confundirse con uno tuyo porque no está |

### La composición no se lee del `.sprint.md`, aunque el archivo esté ahí

> El `.sprint.md` viaja a la ventana y ya no es de donde sale el `items`. La autoridad es `.metadata/product.yaml`, que está en el panorama y no baja.

Es el error más fácil de cometer, porque la copia vieja es lo único que hay a mano: la ventana trae `_sprints/<id>.sprint.md` y no trae `.metadata/`. Medido el 2026-09-09 sobre el sprint 21, el `.sprint.md` nombraba 31 ítems y la composición 24: nueve estaban sólo en el archivo y dos sólo en el YAML.

Y la divergencia no es ruido, tiene causa: siete de esos nueve son los que el board sacó del sprint, y el archivo siguió nombrándolos porque nadie lo lee para decidir nada.

```bash
git -C $SRV show insecure/all:.metadata/product.yaml   # la que vale
```

El corte es el que ya está escrito en el método: `window_files` lee la composición con `product::leer`, y el `.sprint.md` viaja sólo porque las pasadas de sprint todavía lo leen de la ventana. Se va con ellas.

### Y del board sí baja algo

`absorb` es del servidor —tiene la credencial y la rama— y escribe en la ventana lo que el proveedor dice. No se invoca a mano: `pull` lo corre en su paso 0 y después lo trae como cualquier otra cosa que el servidor haya escrito.

Así que *"el servidor nunca te empuja"* sigue siendo cierto del transporte, y no significa que del board no baje nada: baja, y baja por `pull`. Lo que todavía no baja es el *cuerpo* de un ítem, y está escrito por qué y con qué número.

## Antes de escribir una línea de código

> **Corré `worklist status` antes de tocar nada.** Si dice que la vista está atrás, ponela al día primero. Si no se puede, **decilo antes de trabajar**, no después.

No es una recomendación: es verificable, y por eso `status` sale con 1 cuando algo necesita atención.

```bash
worklist status --exit-code && …
```

Contesta cuatro preguntas por separado, porque no cuestan lo mismo — las tres baratas corren siempre, la del proveedor se pide con `--verify`:

```
  vista        secure/sprint/21       ventana
  servidor     al dia
  sin empujar  5 commit(s)            → worklist push
  local        limpio
  proveedor    sin verificar          → worklist status --verify
```

**`sin verificar` no es `coincide`.** Ocupa un renglón en vez de callarse, porque dar por bueno lo que nadie miró es el mismo error que el resto de estas reglas evita.

Y lo que sigue es qué significa cada renglón, y qué hacer con él.

> **Cinco cosas.** Las dos primeras son banderas rojas: no se avanza, se arregla.

**1 · La vista es segura.** Si no, lo que escribas no se puede empujar, no se verifica y no cruza al proveedor. No es una advertencia sobre después: es que el trabajo no tiene dónde ir. Ver § "Cómo saber si la vista es segura".

**2 · El ítem no lleva `@`.** Un `@<slug>` es un id que el servidor todavía no reemplazó, así que **no hay con qué prefijar el commit** — y `AGENTS.md` § Commits pide que arranque con el id del ítem. La salida es sincronizar, no inventar un id.

**3 · La vista está al día**, y eso es el renglón `servidor`. El servidor **commitea encima** de lo que empujaste —los `rename`, los `normalize:`, el `key` del sprint— y eso no está en tu worktree hasta que lo traigas:

```bash
worklist pull            # esta vista
worklist pull --all      # todas las del clon
```

Y con eso alcanza: `pull` recorta, absorbe, trae y replanta. Acá decía *"recortá antes de bajar, que es el paso que más se olvida"*, y eso describía las tripas del comando como si fuera un paso de quien lo corre — ver § [Abrir una ventana](#abrir-una-ventana). Lo que sí es cierto y sigue valiendo es el motivo: sin recortar se baja el corte de la última vez que alguien recortó, y la ventana queda al día contra una foto vieja del panorama, que se ve idéntica a estarlo de verdad.

Acá había una receta a mano —`git fetch` y un `merge --ff-only`, una vista por vez— con la advertencia de que **`git status` diciendo *"limpio"* no alcanza**: una vista atrasada se ve limpia. Eso sigue siendo cierto y ahora lo contesta `status`.

**4 · No tenés trabajo sin empujar**, que es el renglón `sin empujar`, y es el que ningún comando de git contesta. Las vistas nacen sin upstream —`git worktree add` no lo configura— así que `git status` **no tiene contra qué compararse** y nunca dice *"ahead by 1"*.

> **Una vista con trabajo sin empujar se ve limpia**, y es peor que una atrasada: lo que no se nota no es que falte bajar algo, es que hay trabajo hecho que nadie más tiene.

Medido el 2026-09-07: cinco commits en una ventana, `git status` diciendo *"el árbol de trabajo está limpio"*.

```bash
worklist push
```

**El bucle es `push` → `pull` → `push`**, y el `pull` del medio no es opcional: el servidor commitea encima de *cada* push que acepta —el `rename`, el `normalize:`, la clave del sprint—, así que tu segundo push choca contra eso si no lo trajiste. El comando lo dice cuando pasa.

**Y la primera vez configura el upstream**, así que a partir de ahí `git status` sí dice *"ahead by N"* sin que haga falta ningún comando del worklist.

**5 · El ítem está `in-progress`, no `open`.** `open` quiere decir *"nadie lo tomó"*, y arrancar sin moverlo deja el trabajo invisible para todo lo demás — el board, el sprint, y cualquiera que pregunte qué se está haciendo. Se cambia **en la vista**, con el resto del trabajo, así que viaja al proveedor por el mismo camino.

### El orden que evita los cinco

```
crear el ítem  →  empujar  →  worklist pull  →  pasarlo a in-progress  →  trabajar
                                                                          ↑
                                                    worklist status ──────┘
                                                    y si algo falta, no se arranca
```

Es el que el método ya pedía —*primero hay una tarea*— con lo que faltaba: **la tarea no está lista cuando se escribe, está lista cuando tiene id, está en tu vista y dice que la estás haciendo.**

**Tres de los cinco los contesta `worklist status`**, y el que los pone al día es `worklist pull`. Los dos que quedan afuera son de leer: que la rama sea `secure/**` y que el ítem no lleve `@`.

De los comandos que faltaban acá, **`sync` no va a existir**: era *"poner todas las ventanas al día de una"*, y eso es `worklist pull --all`. Queda `worklist is-secure`, que es derivar la clase de la rama en vez de creerle al nombre.

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
```

| Tipo | Sufijo | Puede tener de padre |
|---|---|---|
| Epic | `.epic.md` | nada |
| User Story | `.user-story.md` | nada, epic |
| Task | `.task.md` | nada, epic, user story |

**El sprint no es un tipo de ítem, ni un archivo.** Es una entrada de `.metadata/product.yaml`, del servidor — ver § "Dos formas de agrupar" más abajo. Lleva **contador aparte**, distinto del de los ítems: el sprint `21` y el ítem `21` no tienen nada que ver.

Ids de ítem **base-36** (`1…9, a…z, 10…`), de orden de creación, no de prioridad.

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

Nada más. El tipo lo dice la extensión, la pertenencia la dice `parent`, y la asociación con bilinks **se declara desde el bilink, no desde el ítem**.

**Los hijos se calculan**: los hijos de `n` son los ítems cuyo `parent` es `n`. No hay lista que mantener, igual que el backlog.

## Dos formas de agrupar, y no se mezclan

- **Épica → US → task** es *descomposición*: el campo `parent`, en el frontmatter del ítem, editable en tu ventana.
- **Sprint → ítems** es *planificación*: el campo `items` de la entrada del sprint en `.metadata/product.yaml`, **del servidor y no de tu ventana** — ver [`composition.md`](../../../subsystems/worklist/concepts/composition.md).

La diferencia no es sólo qué preguntan —`parent` dice de qué es parte un ítem, el sprint dice cuándo se hace—, es también quién la edita: `parent` lo editás vos, `items` lo arma el board.

**La prosa de un sprint —por qué esos ítems, qué quedó afuera, cómo cerró— ya no tiene dónde escribirse.** Existía en el cuerpo de `.sprint.md`, y se descartó junto con el archivo: ningún campo de la composición la reemplaza. Es un hueco conocido, no una decisión que resolvió algo.

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

**Y vale igual para un comentario o doc-comment de código.** El argumento es el mismo —el id cambia por diseño, y una cita queda apuntando a algo que ya no existe— así que "ver `ACC-305`" o "es la segunda mitad de `ACC-332`" en un comentario tiene el mismo problema que tenerlo en una spec. La razón de un cambio va en el comentario en términos del código; el ítem que lo motivó vive en el prefijo del commit y en su propio archivo, no en el código.

Las referencias ya escritas **se corrigen al tocarlas**, no de una barrida.

## Cómo contestar "qué sigue": dos pasos, y el orden es obligatorio

> **En el panorama se ve qué hay. En la vista segura se constata que sea lo último.**

No es prolijidad: son las dos promesas, y **son excluyentes**. El panorama eligió estar completo, y por eso mismo *"no puede prometer que estén actualizados"*. Preguntarle si algo está al día es preguntarle lo único que declaró no poder contestar.

**Paso 1 — el panorama, para el inventario.** Está en el servidor, en `.metadata/product.yaml`:

```bash
git -C $SRV show insecure/all:.metadata/product.yaml
```

1. Buscar el sprint con `status: in-progress`. Si no hay, el próximo `open` por número.
2. Sus `items` son el compromiso de la iteración. Bajar a la US y de ahí a sus tasks, que la composición no enumera —van con el padre por la regla del ancestro—, con `git -C $SRV show insecure/all:<id>.<tipo>.md`.
3. Cada task dice **qué specs toca**, no qué archivos de código: el código sale de los bilinks que se rompan.

**Paso 2 — la vista segura del sprint, para constatar.** Cortada o refrescada, es la única que puede verificarse entera contra el proveedor. El `status` que vale es el de ahí.

**Y no se puede invertir.** La vista no sabe lo que no tiene: preguntarle *"qué sigue"* devuelve lo de su sprint y **calla el resto sin decir que calló**.

### Cuando la vista no está cortada

Lo primero es que casi siempre se puede cortar: son las dos líneas de § [Abrir una ventana](#abrir-una-ventana), y el segundo paso deja de faltar. Acá decía que la respuesta era *"esto es lo que hay, y no está constatado"* y punto — eso convertía en limitación permanente algo que se arregla en una invocación.

La respuesta honesta sigue siendo necesaria en el caso que **no** tiene comando: un ítem que no está en ningún sprint, porque no hay de dónde cortarle una ventana. Ahí sí:

> **"Esto es lo que hay, y no está constatado."**

Decirlo **es** el paso. La alternativa es presentar como actual algo que nadie verificó, que es el mismo error de forma que el resto de las reglas de acá evitan. Lo que no vale es decirlo cuando cortar era una línea.

**El backlog no es un archivo.** Se calcula, no se mantiene: tenerlo escrito obligaría a editar dos lugares al mover algo. **Y se calcula donde está el todo**, que es el servidor: sobre una ventana la misma cuenta da mal, no da menos. Y el cálculo va sobre el subárbol, que es lo que un sprint referencia — **un ítem está en el backlog si el tope de su rama no lo nombra ningún sprint**. Las tasks de una user story planificada no se cuentan aparte, y una user story que ningún sprint nombra está en el backlog con todas sus tasks, sin importar cuántas alguien haya querido adelantar.

## Al crear o mover

Los ítems **se escriben a mano hoy**: `worklist new` está especificado pero no implementado. No se les inventa un id: nacen con un nombre provisorio `@<slug>` y el servidor los renombra a su clave cuando cruzan al proveedor.

```
@reescribir-la-parte-de-ventanas-de-la-skill.task.md    ← lo que escribís
ACC-352.task.md                                         ← lo que el servidor deja
```

El renombre queda anotado en el log del servidor, así que el `@<slug>` es un id de verdad mientras dura — y por eso el chequeo 2 dice que un ítem con `@` todavía no puede prefijar un commit: la clave llega con el push.

Y en la ventana donde se va a trabajar — ver § "Se trabaja en una vista segura". Si el ítem pertenece a un sprint, en su ventana; si no pertenece a ninguno, se escribe igual en una ventana y sube con ella, porque el panorama ya no es un lugar donde se pueda escribir.

Mover de padre es editar **un solo campo**, nunca un archivo: cambia `parent`. El archivo no se mueve, así que su path no cambia y ningún bilink que lo apunte se entera.

**Mover de sprint no se hace en la ventana.** `items` es del servidor y lo arma comparando contra el board — ver § "Dos formas de agrupar" arriba —, así que mover un ítem de un sprint a otro es moverlo en el board; `worklist pull` trae el resultado. No hay un archivo ni un link que editar acá para lograrlo.

> Y eso vale también para un ítem recién nacido: el push le da clave y sube su archivo al panorama, pero la pasada de membresía le pregunta al board **y el issue nuevo todavía no está en el sprint de allá**. Así que queda en ningún sprint, y el recorte siguiente le saca el archivo de la ventana donde se escribió. Se vuelve a ver después de agregarlo al sprint en el board y correr `worklist pull`.
