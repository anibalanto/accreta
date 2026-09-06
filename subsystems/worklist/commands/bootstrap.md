# Comando: `worklist bootstrap`

Le da clave del proveedor a lo que no la tiene, sobre el panorama, **sin prometer que la rama se verifique**. Es el camino que el backlog no tenía.

Es la operación que separa las dos cosas que el modelo trataba como una — ver [`concepts/sync.md`](../concepts/sync.md#tener-clave-es-del-ítem-verificarse-es-de-la-rama).

## Firma

```
worklist bootstrap --project <clave> [--ref <rama>] [--base <url>] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--project` | El proyecto del proveedor: `ACC`. |
| `--ref` | Sobre qué rama. Por defecto `refs/heads/insecure/all`, que es donde vive todo. |
| `--base` | Base del proveedor, para traducir los links a otros ítems al convertir el cuerpo. |
| `--dry-run` | Dice a qué le pediría clave, sin hablar con nadie ni mover ninguna ref. |

## Qué hace, y sobre todo qué no

1. Lista los `*.md` del árbol cuyo nombre **no** es una clave de proveedor: ésos son los pedidos. Los `_sprints/*.sprint.md` no entran — un sprint no es un issue.
2. Los ordena topológicamente por `parent`, para que una épica exista antes que lo que cuelga de ella.
3. Por cada uno: [`create-or-find`](create-or-find.md), el renombre con su reescritura, y el `--parent` de su épica ancestro.
4. Y el cuerpo, **sólo donde el issue se creó**.

**Y nada más.** Ni vínculos, ni sprint, ni actualizar lo que ya tenía clave: todo eso es sincronización, y sincronizar es lo que esta operación explícitamente no promete. Las cinco pasadas de [`assign-keys`](assign-keys.md) son para una ventana, que sí lo promete.

### Encontrado no es creado

`create_or_find` puede encontrar en vez de crear — una corrida anterior que se cayó después de crear y antes de comitear el renombre deja exactamente eso. Ahí el cuerpo **no** viaja.

Crear un issue con su descripción escribe sobre algo que no existía: no hay nada que pisar. Escribírselo a uno que ya estaba es una escritura sobre contenido ajeno, y no hubo compare-and-swap que probara que partimos de su estado actual. La salida lo dice en vez de callarlo:

```
  5e -> ACC-207  (ya existia: el cuerpo no se toco)
```

## Se corre donde el panorama vive, y hoy eso es tu worktree

`insecure/**` [rechaza el push](../concepts/sync.md#dos-clases-de-rama-y-el-nombre-dice-qué-se-puede-hacer) — **del cliente.** Escribir en el árbol donde uno está parado es otra cosa, y no pasa por ningún hook.

Y dónde está parado quien lo corre decide cómo se escribe:

| | |
|---|---|
| la rama **está checkouteada acá** | se trabaja en el árbol y se commitea, como cualquiera |
| no lo está | worktree temporal en `--detach`, y `update-ref` al final |

**Lo que no hace es mover una rama que otro worktree tiene abierta.** Es el defecto que [`window open`](window-open.md#y-no-se-mueve-una-rama-que-alguien-tiene-abierta) ya evita: `update-ref` la mueve igual y deja ese worktree con el índice del árbol anterior — y acá serían ciento y pico de renombres. Sobre eso no hay `--force` que valga.

```
$ worklist bootstrap --project ACC
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

## Salida

```
$ worklist bootstrap --project ACC
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
