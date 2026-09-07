# Comando: `worklist-server assign-keys`

Resuelve los pedidos que un push dejó en una ventana: les pide una clave al proveedor, los renombra y reescribe lo que los nombraba. Pensado para correr desde `hooks/post-receive`.

**No puede vivir en `pre-receive`.** Ese hook sólo acepta o rechaza lo que llega; no puede reescribirlo. Ver [`concepts/sync.md`](../concepts/sync.md#dos-pasos-no-uno).

## Firma

```
worklist-server assign-keys --project <clave> --board <id>
                     [--stdin | --window <id>… | --all-windows] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--project` | La clave del proyecto en el proveedor. |
| `--board` | El id del board donde vive el sprint. Es de la instalación, no del worklist: ver [`concepts/sync.md`](../concepts/sync.md#la-correspondencia-con-el-sprint-del-proveedor-se-guarda-no-se-busca). |
| `--stdin` | Lee `<viejo> <nuevo> <ref>` por línea — el protocolo de un hook de recepción. |
| `--window` | Una ventana por su id: `--window 1`. Se puede repetir. |
| `--all-windows` | Todas las ventanas del repo, en orden numérico. |
| `--dry-run` | No llama al proveedor ni escribe nada. Imprime qué asignaría, en qué orden. |

**`--stdin` es para el hook; `--window` es para una persona.** Los tres son excluyentes entre sí.

### Nombrar una ventana es pasar el mismo sha de los dos lados

`--window 1` arma la tripla `(sha, sha, refs/heads/secure/sprint/1)`, y eso **no es un truco para engañar al comando**: es lo que significa *"mirá esta ventana, no traigo nada nuevo"*.

Lo que las pasadas tienen que hacer no depende de que algo se haya movido en git — depende de que git y el proveedor puedan diferir. Con `<viejo>` igual a `<nuevo>`, las pasadas 1 a 4 no encuentran trabajo, que es correcto, y la 5 reconcilia el sprint, que es el punto.

**Sin esto, reconciliar una ventana ya resuelta obliga a imitar el hook a mano** —un `rev-parse`, un `echo` con el sha repetido, y saber el nombre de la ref—, contra el proveedor de producción.

**Y el orden es numérico, no lexicográfico**: `1, 2, … 16`, no `1, 10, 11, 2`. Un listado de refs viene ordenado como texto y el que lo lee espera lo otro.

**`--board` es obligatorio y no tiene default.** Sin él la pasada 5 no puede correr, y una pasada que se saltea sola porque falta configuración es la peor forma de enterarse de que falta.

## Comportamiento

Para cada `ref` que sea una ventana —`refs/heads/secure/**`—:

1. Lista los `*.md` del árbol de `<nuevo>` cuyo nombre **no** es una clave de proveedor: ésos son los pedidos.
2. Los ordena topológicamente por `parent` y `relation.*`, sobre el subgrafo de pedidos. Un ciclo aborta sin escribir nada.

Y después, **cinco pasadas** — ver [`concepts/sync.md`](../concepts/sync.md#nada-viaja-al-proveedor-hasta-que-el-estado-local-esté-completo):

| | |
|---|---|
| **1** | por cada pedido: pide su clave con [`create-or-find`](create-or-find.md), renombra, y reescribe sus referencias. Un commit por ítem. |
| **2** | con todos los nombres finales puestos: convierte cada cuerpo a ADF —traduciendo los links a otros ítems— y lo manda como descripción. |
| **3** | crea los vínculos que `relation.*` declara. |
| **4** | los ítems que ya tenían clave y este push cambió: título y cuerpo se actualizan. |
| **5** | el sprint de la ventana: lo crea si no existe, anota su id en el `.sprint.md`, y mete adentro los issues que le falten. |

Al final mueve la ref al commit resultante, y **sube al panorama lo que quedó** — ver [`concepts/propagation.md`](../concepts/propagation.md) y [`propagate`](propagate.md).

**La propagación va última y no antes**, porque lo que más falta arriba son las claves y las acaba de escribir la pasada 1. Y **que falle no deshace lo resuelto**: la ventana queda adelantada del panorama, que es un estado del que se sale reintentando con `worklist-server propagate`. Por eso se informa como una línea más y no como un error del comando.

**La pasada 5 corre aunque no haya nada que asignar.** Una ventana ya resuelta no tiene pedidos ni cambios, y es justamente la que tiene sus issues creados y su sprint sin existir. Ver [`concepts/sync.md`](../concepts/sync.md#corre-aunque-no-haya-nada-que-asignar).

### Que falte el panorama se avisa una vez, no por ventana

Es una propiedad del repo y no de cada ventana, así que se pregunta antes del lote. Repetir la misma línea dieciséis veces convierte un aviso útil en ruido que se scrollea — y el que lo lee termina creyendo que le pasó algo a cada una.

```
aviso: este repo no tiene refs/heads/insecure/all — lo que se resuelva no sube a ningun lado
```

Hoy es el caso del bare de la instalación, que recibe las ventanas y no tiene panorama.

### Y el `--dry-run` no dice cuántos ya estaban

No le preguntó a nadie: decir `0` sería afirmar sobre el board sin haberlo mirado, que es el defecto que la § "Pero decirlo no es afirmar sobre el board" de [`concepts/sync.md`](../concepts/sync.md#pero-decirlo-no-es-afirmar-sobre-el-board) corrigió del otro lado. Lo único cierto sin preguntar es la membresía que se calculó de `items`:

```
  sprint 1 -> (dry-run): 4 miembro(s) calculados; no se pregunto cuantos ya estan
```

**Y no se puede volver a escribir mal**, porque el dato no se puede representar: el campo es un `Option`, y *"no se preguntó"* es `None` y no `0`.

## Corre sobre el repo del directorio actual, y el error lo nombra

No hay flag para decirle cuál: es el del `cwd`. En una máquina con el clon del worklist y el bare de sincronización a un `cd` de distancia, **equivocarse de directorio es el único modo de equivocarse**, así que el error lo dice:

```
$ worklist-server assign-keys --project ACC --board 701 --all-windows
error: no hay ninguna ventana en /home/…/Workspace/accreta: refs/heads/secure/** esta vacio

  este comando corre sobre el repo del directorio actual. Si ese no es
  el del worklist, parate en el que si lo es.
```

Un *"no hay ninguna ventana en este repo"* describe bien el síntoma y manda a revisar el repo equivocado.

## Corre sobre un repo bare, sin working tree

Un hook de recepción no tiene árbol de trabajo, y los pasos 4 y 5 necesitan uno. La salida es un **worktree temporal en `--detach`** sobre `<nuevo>`: ahí se hacen los renombres, y al terminar se mueve la ref al `HEAD` resultante y se descarta el worktree.

**En `--detach` y no sobre la rama**, porque una rama con worktree asignado no acepta pushes — y el hook no puede dejar el repo en un estado donde el próximo push falle.

## Salida

```
$ worklist-server assign-keys --project ACC --board 701 --window 10
refs/heads/secure/sprint/10: 2 pedido(s)
  orden: agregar-b, agregar-a
  agregar-b -> ACC-101
  agregar-a -> ACC-102  (1 refs reescritas)
  sprint 10 -> 6512 (creado): 2 issue(s) agregados, 0 ya estaban
refs/heads/secure/sprint/10: e4f5g6h -> 9z8y7x6
refs/heads/secure/sprint/10: sube 3 commit(s) al panorama
  rename agregar-b -> ACC-101  (rehecho)  (4 refs reescritas en el panorama)
  rename agregar-a -> ACC-102  (rehecho)  (1 refs reescritas en el panorama)
  9z8y7x6 normalize: ACC-101
  panorama: a1b2c3d -> 7f6e5d4
```

Y la segunda corrida sobre lo mismo:

```
  sprint 10 -> 6512: 0 issue(s) agregados, 2 ya estaban
```

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | todos los pedidos resueltos, o no había ninguno |
| `1` | un ciclo entre pedidos, o el proveedor falló — la ref queda como estaba |

## Una clave que el board no tiene no puede llevarse el lote

`jira sprint add` toma hasta 50 claves por llamada y **la rechaza entera si una no existe**. Medido el 2026-09-07 contra el board real:

```
jira sprint add 6525 ACC-268 ACC-303 ACC-305 ACC-320 ACC-321 ACC-322 ACC-323
                     ^^^^^^^ borrada del board

  - ACC-268: La incidencia no existe o no tienes permiso para verla.
jira: Received unexpected response '400 Bad Request'.
```

`ACC-268` tiene clave en el worklist —el log muestra su `rename 6m -> ACC-268`— y el board no la tiene: existió y se borró. Y por esa una, **las otras seis no entraban, en cada push**, con un error que no decía cuál era la culpable.

> **Perder seis por una es peor que decir cuál fue.**

Así que las que el proveedor nombra se sacan del lote y se reintenta con el resto. La pasada no falla: **reporta**.

```
  ! el board rechazo ACC-268: el worklist las tiene y el no
  sprint 21 -> 6525: 6 issue(s) agregados, 12 ya estaban
```

**Y va en su propia línea, no en la cuenta.** Una clave rechazada es *deriva* —el worklist la tiene y el proveedor no— y ésta es la única pasada que la ve. Sumarla a los agregados la taparía; restarla en silencio también.

**Un fracaso que no nombra ninguna de las claves mandadas no es esto**: es otro error —una credencial, un board que no responde— y sube tal cual. Confundirlos dejaría el reintento en un bucle.
