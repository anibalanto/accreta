# La propagación entre el panorama y sus ventanas

> **Lo que se hace en una ventana llega al panorama por cherry-pick, y la ventana se pone al día regenerándose.**

Decidido y confirmado por eliminación contra git: el merge con revert anda una vez y choca a la segunda con `agregar/agregar`, `sparse-checkout` deja que la ventana escriba lo ajeno con un solo comando, y rebasear el corte **ensancha la ventana sola**. Las pruebas de cada alternativa están en el trabajo que lo decidió; esta página es la spec de lo decidido.

El panorama es `insecure/all`, y **nadie le empuja**: [`sync.md`](sync.md#dos-clases-de-rama-y-el-nombre-dice-qué-se-puede-hacer) rechaza el push a una rama insegura, así que la propagación no es una política que alguien pueda saltearse — es la única forma que tiene de avanzar.

## Por qué hace falta: dos ventanas sólo se cruzan en el panorama

Una ventana se corta una vez y no vuelve a mirar. Si alguien edita en su ventana un archivo que también está en la mía, **yo nunca lo veo**: las dos divergen en silencio sobre el mismo ítem, y ninguna tiene forma de enterarse.

Y no es hipotético: la épica viaja como ancestro para que la cadena `parent` cierre adentro del recorte, así que `ACC-14.epic.md` está en 8 de las 10 ventanas.

```
1. la ventana trabaja
2. se empuja; el hook resuelve
3. el panorama recibe lo resuelto
4. la ventana se regenera desde el panorama, que ya tiene su trabajo
```

**Regenerar descarta, y descartar está bien acá**: lo que se descarta ya subió en el paso 3.

## Hacia arriba: sube lo resuelto, no lo que llegó

El panorama recibe el rango entero que el push dejó en la ventana — **el commit del cliente y los del servidor encima**, porque las claves son lo que más falta arriba. Sube al final del `post-receive`, cuando el servidor ya escribió lo suyo.

| Qué | Cómo sube |
|---|---|
| las ediciones del cliente | cherry-pick |
| `provider: <clave> …` | cherry-pick |
| **`rename <slug> -> <clave>`** | **se rehace**, no se copia |
| **`normalize: <clave>`** | **se rehace**, no se copia |
| **`sprint: <id> -> <clave>`** | **se rehace**, no se copia |

### Lo que depende del árbol que lo vio no se copia, y el motivo es de alcance

[Renombrar y reescribir son un solo commit](sync.md#el-renombre-y-la-reescritura-son-un-solo-commit): el `git mv` va con la corrección de todo lo que nombraba al slug. **Esa reescritura recorre el árbol donde corre**, y el árbol de una ventana tiene 19 archivos donde el panorama tiene 243.

Así que el commit de renombre de una ventana corrige las referencias que esa ventana veía, y ninguna más. Cherry-pickeado tal cual al panorama, deja el archivo movido y **las referencias de afuera del recorte apuntando al slug que ya no existe**:

```
en la ventana:   m.task.md ─▶ ACC-93.task.md,  y las 3 referencias que tenía adentro
en el panorama:  ACC-93.task.md,  y 5e.task.md sigue diciendo `m`
```

El commit no es incorrecto: es **correcto en su alcance**. Lo que no se puede es mudar un commit cuyo contenido depende del árbol que lo vio, a un árbol más grande.

> **Sube la clave, no el commit que la escribió.** El renombre se vuelve a hacer en el panorama, con el mismo par `<slug> -> <clave>` y la reescritura recalculada sobre los 243.

Es la misma invariante de siempre —o entran el `git mv` y la reescritura, o no entra ninguna— **sostenida en cada rama donde se aplica** en vez de una sola vez donde se originó.

#### Y el renombre no es el único: `normalize:` es el mismo commit por otra puerta

`normalize:` es la [pasada 2](../commands/assign-keys.md) — convierte el cuerpo a ADF *traduciendo los links a otros ítems*, y escribe de vuelta la forma canónica. Esa traducción lee los nombres del árbol donde corre, así que **el texto que deja es el renombre parcial de la ventana**, congelado como contenido.

Cherry-pickeado al panorama vuelve a meter lo que el renombre acababa de arreglar. Medido sobre una épica que ocho ventanas normalizaron: las ocho versiones difieren, y **cada línea que difiere lleva un id** — la prosa es idéntica, lo único que cambia es cuántas referencias alcanzó a resolver cada recorte.

Que sean ocho parciales incompatibles y no dos ediciones en disputa importa, porque **no hay nada que arbitrar**: no se elige una, se rehace la pasada arriba y sale la unión.

> **Se rehace, no se copia** — la misma regla y el mismo motivo. `round_trip` es función pura del texto, así que recalcular la forma canónica sobre el árbol del panorama está tan bien definido como recalcular la reescritura del renombre.

Y el orden es el que ya estaba: primero el renombre rehecho, que deja los 243 nombres finales, y recién después la normalización sobre ellos.

#### Y el tercero es la clave del sprint, por un motivo que no es de alcance sino de autoría

La pasada 5 escribe **un campo** —`key`— sobre el `.sprint.md` de la ventana. Copiarlo como parche arrastra sus líneas de contexto, y ahí está el problema: **el `.sprint.md` del panorama es el que se planifica.** Se cierra el sprint, se mueven ítems al backlog, se corrige el `items`. Así que el contexto difiere por trabajo legítimo, y el parche choca sobre algo que no estaba tratando de cambiar.

```
ventana:    status: in-progress   items: [ACC-94, ACC-95, ACC-96]
panorama:   status: done          items: [4h, 4j]        ← el sprint cerró y `4i` se fue al backlog
```

> **Sube la clave, no el `.sprint.md`.** `status` e `items` no viajan hacia arriba: la ventana no los sabe mejor que el panorama, y la clave es lo único que la ventana tiene y el panorama no.

Es la primera vez que la asimetría se dice en un campo y no en una regla: **la planificación se edita arriba y baja regenerando; lo que el proveedor arbitra se escribe abajo y sube.** El `.sprint.md` es el único archivo que las dos direcciones tocan, y por eso es el único donde había que partirlo.

### Hasta dónde subió se anota en una ref, no se deduce

```
refs/worklist/propagated/secure/sprint/17
```

Apunta al último commit de esa ventana que el panorama ya tiene. La próxima propagación es el rango de ahí al tip nuevo, y **una que se cayó por la mitad se reintenta sin repetir nada**: la ref no se mueve hasta que el panorama avanzó.

Deducirlo comparando los árboles sería preguntar *"¿este cambio ya está?"* sobre 243 archivos por cada push, y contestarlo mal en cuanto dos ventanas tocaran lo mismo. Es el mismo trato que [`key` en el `.sprint.md`](sync.md#la-correspondencia-con-el-sprint-del-proveedor-se-guarda-no-se-busca): un dato que alguien tiene que sostener se guarda, no se busca.

Va en `refs/worklist/**` y no en una rama: es contabilidad del servidor, no contenido. Nadie la clona y nadie la mira.

## Qué pasa cuando el cherry-pick no aplica

### Casi todo el conflicto ya lo previno el compare-and-swap

Dos ventanas que editan el mismo ítem **no llegan a chocar en el panorama**, porque chocan un paso antes: la primera sube su edición al proveedor, y el push de la segunda encuentra que el cuerpo en vivo ya no coincide con el de su tip y [se rechaza](sync.md#la-ventana-y-el-compare-and-swap).

> El panorama era el único lugar donde dos ventanas se cruzaban. Desde que el cuerpo viaja, **el proveedor también es uno** — y llega primero.

Así que el conflicto que queda es el de lo que **no tiene contraparte allá**: el `.sprint.md`, que no es un issue, y un pedido todavía sin clave.

Del `.sprint.md` se decía que *"el de cada ventana es suyo"*, y no alcanzaba: es suyo el archivo, no el campo. La ventana escribe `key` y el panorama escribe `status` e `items`, así que [la clave se rehace](#y-el-tercero-es-la-clave-del-sprint-por-un-motivo-que-no-es-de-alcance-sino-de-autoría) y el resto no sube. Con eso, el caso que queda es el de abajo.

### El que queda es el ancestro que nadie protege

Un ancestro viaja de sólo lectura para que la cadena `parent` cierre — y *"de sólo lectura"* no lo hace cumplir nadie: el archivo está en el árbol y se puede editar. Si dos ventanas editan la misma épica, **el cherry-pick es lo primero que se entera**.

No es un defecto de la propagación: es el único detector que hay hoy de una regla que sólo estaba escrita. Que aparezca ahí es información, y el mensaje tiene que decir qué archivo y contra qué ventana, no *"conflicto"*.

**Y se parece a un caso que no es éste.** Un ancestro normalizado por ocho ventanas también choca, y ahí no hay ninguna regla violada: nadie lo editó, dicen todas lo mismo con nombres distintos. La diferencia se lee en el conflicto — si cada línea en disputa lleva un id, es alcance y [se rehace](#y-el-renombre-no-es-el-único-normalize-es-el-mismo-commit-por-otra-puerta); si hay prosa distinta, alguien escribió.

### Y el panorama nunca guarda un conflicto

> **De `all` se corta todo. Un marcador de conflicto escrito ahí entra en el próximo recorte de cada ventana.**

Un conflicto anotado en una rama de trabajo espera a que alguien lo mire. Anotado en el tronco, se reparte. Así que la salida no es escribirlo:

1. El `pre-receive` **prueba** el cherry-pick de los commits del cliente contra el panorama. Si no aplica, rechaza el push — la misma forma que ya tiene el compare-and-swap, y por la misma razón: una escritura tiene que probar que parte del estado actual. Se prueba con `merge-tree`, **en memoria**: un hook que rechaza no puede dejar un worktree ni objetos atrás, y esto corre en todos los pushes y no sólo en los que fallan.
2. El `post-receive` lo **aplica**, después de sus propios commits. Es la [misma partición en dos pasos](sync.md#dos-pasos-no-uno) que todo lo demás: `pre-receive` sólo puede aceptar o rechazar, y escribir es de después.

Lo que el paso 1 no puede probar son los commits que el servidor todavía no escribió. Si uno de ésos no aplica, **el panorama no avanza y el hook lo reporta**: la ventana queda adelantada, que es un estado del que se sale reintentando la propagación, y no una pérdida.

**Que la ventana quede adelantada del panorama es recuperable; que el panorama quede inválido no.** Por eso el orden es ése y no el inverso.

## Hacia abajo: regenerar, y el rebase es sólo el rescate

Acá había una contradicción aparente: está medido que el rebase **ensancha la ventana sola**, y también está decidido que una vista se pone al día rebaseando. Las dos son ciertas, y hablan de rebasear cosas distintas.

**Lo que ensancha es re-aplicar el corte.** El corte es un commit que borra una lista fija de rutas; re-aplicado sobre un panorama que creció, no menciona lo nuevo — así que lo nuevo entra:

```
antes del rebase:   ACC-1
después:            ACC-1  ACC-9      ← ACC-9 no es de esta ventana
```

Y no es un caso de borde: **cada ítem que se cree en el panorama se filtra a todas las ventanas** en su próximo rebase, y la promesa de que una ventana lleva sólo su sprint se cae sin que nadie toque nada.

> **El corte se recalcula. Lo que está encima del corte se re-aplica.**

```
antes:    all(viejo) ─▶ corte(viejo) ─▶ W1 ─▶ W2
después:  all(nuevo) ─▶ corte(nuevo) ─▶ W1' ─▶ W2'
```

El corte nuevo se computa como lo computa [`window open`](../commands/window-open.md#qué-entra): contra el `items` de hoy, no contra la lista de rutas que se borró la vez pasada. Y `W1`/`W2` se replantan encima, **de a un cherry-pick y no con `git rebase --onto`.**

### Por qué de a uno, y no es preferencia

El descarte por patch-id de `rebase` compara contra el **upstream** que se le nombra, que acá es el corte viejo. Y lo que ya subió no vive ahí: vive en el panorama, que es ancestro del corte **nuevo**. Medido — con `rebase --onto` el trabajo ya propagado se replanta igual, y la ventana termina con una copia de lo que el panorama ya le estaba dando.

Un cherry-pick pregunta contra el árbol que tiene delante, que es el único lugar donde la respuesta es la correcta.

### Y el descarte deja de ser una promesa

En el caso sano no hay nada que replantar: `W1` y `W2` **ya subieron al panorama** en el paso 3, así que el corte nuevo los contiene, el cherry-pick queda vacío y se deja caer. Nadie tiene que acordarse de propagar antes de regenerar — **si se propagó, el replante es vacío; si no, replanta y no se pierde nada.**

Es una sola operación cubriendo los dos casos, y es lo que le da un mecanismo a lo que hasta acá era una advertencia: la guarda de [`window open --force`](../commands/window-open.md#y---force-baja-de-categoría) decía *"descartar es correcto sólo si el trabajo se propagó"* y le pedía a quien corre el comando que lo supiera.

**`--force` no desaparece**, y baja de categoría: queda para el caso en que el replante conflictúa y alguien decide tirar lo local. Deja de ser el flujo del `items` que cambió, que ahora es regenerar y ya.

**Y el replante conflictúa por una razón sola**: el trabajo toca un ítem que el `items` de hoy ya no lleva, así que el corte nuevo no lo tiene y el parche no encuentra dónde apoyarse. Es información, no un accidente — dice que alguien trabajó sobre algo que se fue de la ventana.

## Cuándo se dispara

| Dirección | Quién | Cuándo |
|---|---|---|
| **arriba** | el servidor | al final del `post-receive`, después de sus propios commits |
| **abajo** | quien tiene la ventana | **a pedido**, nunca solo |

**Arriba es en cada push aceptado, y no al cerrar el sprint.** Propagar una sola vez al final evitaría la segunda vuelta —era una de las tres salidas que `64` enumeró— y evitaría también lo único que la propagación existe para dar: que dos ventanas se vean. Al cerrar el sprint ya es tarde por definición.

**Abajo nunca es automático**, por dos razones distintas. Regenerar reemplaza el árbol de la ventana, y reemplazar lo que alguien tiene abierto no es algo que se haga por su cuenta — es la misma razón por la que [`window open` se niega](../commands/window-open.md#y-no-se-mueve-una-rama-que-alguien-tiene-abierta) sobre una rama con worktree activo. Y una ventana que se pusiera al día sola cambiaría lo que estás mirando en el medio de mirarlo.

## Lo que esto destraba

- **El backlog** — subir el backlog deja de pedir un camino de escritura aparte: con el panorama teniendo claves, es una operación normal.
- **Las dependencias que cruzan** — una [dependencia que cruza la ventana](sync.md#una-dependencia-que-cruza-la-ventana-se-reporta-no-rompe-el-push) llega ya traducida, y el vínculo se crea solo.
- **La correspondencia slug ↔ clave** — las ramas `secure/*` dejan de ser la única copia de la correspondencia slug ↔ clave.
- **La identidad de un ítem** — la identidad de un ítem deja de ser su título: con clave en el panorama no hay nada que buscar, y buscar por título es lo único que puede duplicar.
- **`window open --force`** — el caso del `items` que cambió pasa a ser regenerar, en vez de una salida de emergencia.

Y de paso se cierra el costo diario: hoy el mismo ítem se llama `4i` parado en el panorama y `ACC-96` parado en su ventana, así que **todo lo que se planifica se nombra en base-36 y todo lo que se muestra se nombra `ACC-`**.
