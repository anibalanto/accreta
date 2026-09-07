# Comando: `worklist-server bootstrap`

Le da clave del proveedor a lo que no la tiene, sobre el panorama, **sin prometer que la rama se verifique**. Es el camino que el backlog no tenía.

Es la operación que separa las dos cosas que el modelo trataba como una — ver [`concepts/sync.md`](../concepts/sync.md#tener-clave-es-del-ítem-verificarse-es-de-la-rama).

## Firma

```
worklist-server bootstrap --project <clave> [--ref <rama>] [--base <url>] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--project` | El proyecto del proveedor: `ACC`. |
| `--ref` | Sobre qué rama. Por defecto `refs/heads/insecure/all`, que es donde vive todo. |
| `--base` | Base del proveedor, para traducir los links a otros ítems al convertir el cuerpo. |
| `--limit` | Crea sólo los primeros N y para. Sin él, todos. |
| `--dry-run` | Dice a qué le pediría clave, sin hablar con nadie ni mover ninguna ref. |

## Qué hace, y sobre todo qué no

1. Lista los `*.md` del árbol cuyo nombre **no** es una clave de proveedor: ésos son los pedidos.
2. Los ordena topológicamente por `parent`, para que una épica exista antes que lo que cuelga de ella.
3. Por cada uno: [`create-or-find`](create-or-find.md), el renombre con su reescritura, y el `--parent` de su épica ancestro.
4. Y el cuerpo, **sólo donde el issue se creó**.

5. Y **los sprints**, al final: el que no tiene `key` se crea del otro lado, y a todos se les meten adentro los issues que les falten.

**Y nada más.** Ni vínculos, ni actualizar lo que ya tenía clave: eso es sincronización, y sincronizar es lo que esta operación explícitamente no promete. Las cinco pasadas de [`assign-keys`](assign-keys.md) son para una ventana, que sí lo promete.

### El sprint entra, y lo que cambia es la llamada

Decía que *"los `_sprints/*.sprint.md` no entran — un sprint no es un issue"*, y la conclusión no se seguía de la premisa. Es cierto que no es un issue: se pide con `create_or_find_sprint` y no con [`create-or-find`](create-or-find.md). **Pero la operación es la misma** — darle identidad del proveedor a lo que no la tiene, sin prometer que la rama se verifique— y un sprint sin `key` es exactamente eso.

Dejarlo afuera producía el agujero que el board mostró: **22 sprints en el worklist y 17 en Jira**, y entre los que faltaban, el que estaba en curso.

> **Va al final, y no es preferencia.** Meter los issues adentro del sprint necesita que sus ítems ya tengan clave. Es la misma restricción topológica que ordena a la épica antes que sus tasks, un escalón más arriba.

#### Y se miran **todos**, no sólo los que no tienen clave

> **La clave se pide una vez; los miembros se reconcilian siempre.**

Un sprint no es sólo una identidad: es también una **membresía**, y la membresía cambia mientras el sprint vive — eso es lo normal en el que está en curso, no la excepción. Filtrar por *"no tiene `key`"* trataba al sprint como a un ítem, y dejaba pasar el caso más frecuente.

Medido: un ítem agregado veinte minutos después de que su sprint cruzara quedó en el worklist y no en el board, con el sprint mostrando cuatro donde había cinco.

**La idempotencia no la da el filtro, la da la pasada**: lee qué hay adentro y manda sólo lo que falta, así que volver a correrlo cuesta una lectura por sprint y cero escrituras.

##### Lo que sigue sin hacer: sacar

Un ítem que **sale** del `items` de un sprint no se saca del otro lado. La pasada agrega lo que falta y nunca quita lo que sobra, así que un ítem movido entre sprints queda en los dos.

No es un olvido: **quitar es una escritura destructiva sobre el board**, y merece su propio argumento antes que su propia línea de código.

### Y no hay bandera para pedir sólo los sprints

Elegirlos de antemano obligaría a saber qué falta antes de mirar, y **el estado ya lo dice**: lo que no tiene clave es lo que se pide. `--limit` alcanza para ir de a poco, y cuando no queda nada la corrida contesta que no hay nada que hacer.

### Encontrado no es creado

`create_or_find` puede encontrar en vez de crear — una corrida anterior que se cayó después de crear y antes de comitear el renombre deja exactamente eso. Ahí el cuerpo **no** viaja.

Crear un issue con su descripción escribe sobre algo que no existía: no hay nada que pisar. Escribírselo a uno que ya estaba es una escritura sobre contenido ajeno, y no hubo compare-and-swap que probara que partimos de su estado actual. La salida lo dice en vez de callarlo:

```
  5e -> ACC-207  (ya existia: el cuerpo no se toco)
```

## Se corre donde el panorama vive, y eso es el bare del servidor

`insecure/**` [rechaza el push](../concepts/sync.md#dos-clases-de-rama-y-el-nombre-dice-qué-se-puede-hacer) — **del cliente.** Escribir en el árbol donde uno está parado es otra cosa, y no pasa por ningún hook.

**Y ya no hay dos lugares donde pueda vivir.** El título decía *"y hoy eso es tu worktree"*, que era cierto mientras el panorama estuviera checkouteado en el clon; [ya no lo está](../concepts/sync.md#el-panorama-vive-en-un-solo-lado-y-la-ventana-en-los-dos). Lo de abajo se queda porque es genérico —vale para cualquier `--ref`— y no porque al panorama le queden dos caminos: le queda uno.

Y dónde está parado quien lo corre decide cómo se escribe:

| | |
|---|---|
| la rama **está checkouteada acá** | se trabaja en el árbol y se commitea, como cualquiera |
| no lo está | worktree temporal en `--detach`, y `update-ref` al final |

**Lo que no hace es mover una rama que otro worktree tiene abierta.** Es el defecto que [`window open`](window-open.md#y-no-se-mueve-una-rama-que-alguien-tiene-abierta) ya evita: `update-ref` la mueve igual y deja ese worktree con el índice del árbol anterior — y acá serían ciento y pico de renombres. Sobre eso no hay `--force` que valga.

```
$ worklist-server bootstrap --project ACC
error: refs/heads/insecure/all esta checkouteada en otro worktree y no se puede mover:
  /home/…/.worklist/insecure/all

  moverla dejaria ese worktree con el indice del arbol anterior, y aca son
  126 renombres. Corre esto parado ahi, o saca el worktree primero.
```

### Y el árbol tiene que estar limpio

Sólo en el primer caso, y por una razón mecánica: el renombre commitea con `add -A`, así que **lo que hubiera sin commitear se colaría adentro**. Se rechaza antes de pedirle nada al proveedor.

### Y un bare nunca es "acá"

Decía que el único lugar posible era el worktree del panorama, porque el servidor no tenía `insecure/all`. **Ya lo tiene**, así que corre ahí — y ahí aparece el caso que la pregunta por el `HEAD` no cubría.

Un bare puede tener el `HEAD` apuntando a la rama y **no tener dónde escribir**. Preguntar sólo por el `HEAD` daba *"acá"* y mandaba el chequeo de árbol limpio a fallar con *"esta operación debe ser realizada en un árbol de trabajo"*.

> **Sin árbol de trabajo no hay "acá".** Un repo bare va siempre por el worktree temporal, apunte su `HEAD` a donde apunte.

## Cada clave conseguida se guarda a medida

> **La unidad de atomicidad es el ítem, no la corrida.**

Crear un issue es un efecto **afuera**: irreversible y pago. Una corrida que se cae después de haber creado ochenta y tres y antes de terminar no puede descartar el árbol entero — las claves quedarían **sólo del otro lado**, y el panorama no sabría que las pidió.

Medido, y es de donde sale esta sección: una corrida contra el board real creó **23 issues** y se cortó en el ítem 24 de 109. El panorama no registró ninguno.

Así que la ref se mueve **después de cada ítem**, no al final. Una caída deja arriba exactamente lo que se hizo, y la corrida siguiente arranca desde ahí.

### Y no es lo mismo que la propagación, que sí descarta

[`propagate`](propagate.md) hace lo contrario a propósito — *"si algo no aplica, el panorama no avanza"*— y ahí está bien: un cherry-pick que no se aplicó **no dejó rastro en ningún lado**, así que descartar devuelve al estado de antes.

**La diferencia es si lo descartado salió del repo.** Acá salió.

### Que el ancla falle no tira abajo la corrida

Mover la ref puede fallar —otro proceso la movió, el `update-ref` no pudo escribir— y eso **no interrumpe nada**: se sigue, y el intento siguiente vuelve a probar. Un ancla que se pierde deja el estado que había sin ella, que es el de antes de esta sección; una que interrumpiera convertiría un problema de contabilidad local en la pérdida que esto viene a evitar.

### Parado en la rama no hace falta

Ahí cada commit del renombre **ya la mueve**. El ancla existe para el caso del worktree temporal, que es donde la ref no se entera hasta el final — y es el caso del servidor.

## De a lotes, con `--limit`

> **Un lote no necesita ser una transacción. Sólo un corte.**

Que una caída no pierda nada ya lo resuelve el ancla de arriba. `--limit` resuelve otra cosa: **poder mirar**. Noventa y un issues en un board real es una escritura que conviene ver a la décima, no a la nonagésima primera.

```
worklist-server bootstrap --project ACC --limit 10
```

Toma los **primeros N del orden topológico** y para. Los que quedan siguen sin clave, así que la corrida siguiente los toma **sin ninguna contabilidad extra**: lo que falta es lo que no tiene clave, y eso se lee del árbol.

**No hay archivo de progreso ni marca de "iba por acá".** Guardar una posición sería una segunda fuente de la verdad para algo que ya se calcula, y una que puede diferir de la primera.

### El corte respeta el orden, y por eso es seguro

Un lote puede dejar una épica creada y sus tasks sin crear: es un estado válido, porque el `--parent` se le pone a cada task **al crearse** y la épica ya tiene clave. Al revés no puede pasar — el orden topológico lo impide.

**Y para que eso sea cierto, el ancestro se busca en el árbol y no entre los pedidos.** Quién tiene clave decide qué se *pide*, no qué se puede *nombrar*: una épica ya resuelta sigue siendo la épica ancestro de sus tasks, y es la clave que va en el `--parent`. Buscarla sólo entre los pedidos de la corrida la vuelve invisible en cuanto cruzó, y el issue se crea suelto — con `--limit` a partir del segundo lote, y sin él en cuanto el panorama tiene claves de antes.

Con `--dry-run`, `--limit` recorta el listado: sirve para ver cuál sería el próximo lote sin pedir nada.

## Salida

```
$ worklist-server bootstrap --project ACC
refs/heads/insecure/all: resolvio 126 item(s)
  51 -> ACC-205  (7 refs reescritas)  [parent ACC-14]
  5e -> ACC-206  (1 refs reescritas)  [parent ACC-14]
  5z -> ACC-207
refs/heads/insecure/all: a1b2c3d -> 9z8y7x6
```

Y la segunda corrida:

```
refs/heads/insecure/all: no hay ningun item sin clave
```

**No es idempotencia por suerte**: los ítems que ya tienen clave no son pedidos, así que la lista del paso 1 sale vacía sin preguntarle nada al proveedor.

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | los pedidos quedaron con clave, o no había ninguno |
| `1` | un ciclo entre pedidos, o el proveedor falló — la ref queda como estaba |
| `1` | la rama está abierta en otro worktree, o el árbol tiene cambios sin commitear |
