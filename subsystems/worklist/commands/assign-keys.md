# Comando: `worklist assign-keys`

Resuelve los pedidos que un push dejó en una ventana: les pide una clave al proveedor, los renombra y reescribe lo que los nombraba. Pensado para correr desde `hooks/post-receive`.

**No puede vivir en `pre-receive`.** Ese hook sólo acepta o rechaza lo que llega; no puede reescribirlo. Ver [`concepts/sync.md`](../concepts/sync.md#dos-pasos-no-uno).

## Firma

```
worklist assign-keys --project <clave> --board <id> [--stdin] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--project` | La clave del proyecto en el proveedor. |
| `--board` | El id del board donde vive el sprint. Es de la instalación, no del worklist: ver [`concepts/sync.md`](../concepts/sync.md#la-correspondencia-con-el-sprint-del-proveedor-se-guarda-no-se-busca). |
| `--stdin` | Lee `<viejo> <nuevo> <ref>` por línea — el protocolo de un hook de recepción. |
| `--dry-run` | No llama al proveedor ni escribe nada. Imprime qué asignaría, en qué orden. |

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

Al final mueve la ref al commit resultante.

**La pasada 5 corre aunque no haya nada que asignar.** Una ventana ya resuelta no tiene pedidos ni cambios, y es justamente la que tiene sus issues creados y su sprint sin existir. Ver [`concepts/sync.md`](../concepts/sync.md#corre-aunque-no-haya-nada-que-asignar).

## Corre sobre un repo bare, sin working tree

Un hook de recepción no tiene árbol de trabajo, y los pasos 4 y 5 necesitan uno. La salida es un **worktree temporal en `--detach`** sobre `<nuevo>`: ahí se hacen los renombres, y al terminar se mueve la ref al `HEAD` resultante y se descarta el worktree.

**En `--detach` y no sobre la rama**, porque una rama con worktree asignado no acepta pushes — y el hook no puede dejar el repo en un estado donde el próximo push falle.

## Salida

```
$ worklist assign-keys --project ACC --board 701 --stdin <<< "a1b2c3d e4f5g6h refs/heads/secure/sprint/10"
refs/heads/secure/sprint/10: 2 pedido(s)
  orden: agregar-b, agregar-a
  agregar-b -> ACC-101
  agregar-a -> ACC-102  (1 refs reescritas)
  sprint 10 -> 6512 (creado): 2 issue(s) agregados, 0 ya estaban
refs/heads/secure/sprint/10: e4f5g6h -> 9z8y7x6
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
