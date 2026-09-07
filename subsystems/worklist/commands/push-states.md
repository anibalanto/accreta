# Comando: `worklist-server push-states`

Cierra las divergencias de estado que un push **ya no puede alcanzar**. El vocabulario y su mapeo: [`concepts/states.md`](../concepts/states.md).

## Firma

```
worklist-server push-states --project <clave> --states-map <archivo>
                            [--ref <rama>] [--base <url>] [--account <email>]
                            [--limit <n>] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--project` | La clave del proyecto de Jira. |
| `--states-map` | El [mapeo de estados](../concepts/states.md) de esta instalación. **Obligatorio, y no un aviso**: sin traducir no hay nada que comparar, así que sin él el comando no tiene trabajo que hacer — no uno más chico. |
| `--ref` | Sobre qué rama. Por defecto el panorama, que es del repo donde el comando corre. |
| `--base`, `--account` | La dirección de REST y el email de la cuenta. Ver [`install-hooks`](install-hooks.md). |
| `--limit` | Mueve sólo las primeras N y para. |
| `--dry-run` | Imprime el plan y no escribe nada. |

## Por qué es otro comando y no una pasada más

> **`assign-keys` mueve lo que el push movió. Éste mueve lo que ya está movido y nadie llevó.**

Las dos preguntas son distintas y las dos son correctas:

| | Qué pregunta | Cuándo sirve |
|---|---|---|
| la pasada de `assign-keys` | *"¿qué cambió en este push?"* | siempre — y **sólo mover eso es lo correcto**: un push que no toca un ítem no puede pisarlo, que es la misma regla del compare-and-swap |
| `push-states` | *"¿en qué difiere el proveedor de lo que el árbol dice, hoy?"* | cuando el mapeo llegó después de los cambios |

**Y la segunda no se puede derivar de la primera esperando.** Un ítem que se cerró antes de que el mapeo existiera no vuelve a cambiar de estado, así que ningún push futuro lo va a alcanzar: su divergencia es permanente. Medido el 2026-09-07, al instalar el mapeo por primera vez: **188 de 288 ítems** divergían, y ninguno iba a moverse solo.

## Qué hace

1. Lee el `status` de cada ítem con clave del árbol de `--ref`.
2. Pide al proveedor el estado en vivo de **todas** esas claves — una sola llamada, la misma que usa `check-push`.
3. Traduce cada estado local con el mapeo y compara contra el vivo.
4. Imprime el plan: clave, status vivo, status destino, y el estado local sin traducir.
5. Salvo `--dry-run`, mueve las primeras `--limit`.

**El estado local va en el plan además del destino**, porque es lo que permite ver que el mapeo está mal: `ACC-9 Tareas por hacer -> Finalizada (local: done)` se lee y se aprueba; sin el `local:`, un mapeo equivocado produce un plan que parece razonable.

### Una clave que el proveedor no informa se reporta, no se saltea

No es una que coincida: es una que **no se vio**. Contarla como coincidente sería afirmar sobre el board sin mirarlo, que es lo que el resto del puerto ya evita — ver [`concepts/sync.md`](../concepts/sync.md#pero-decirlo-no-es-afirmar-sobre-el-board).

Va en un aviso aparte y antes del plan, porque **lo que hay que hacer con ella es distinto**: no se mueve, se averigua por qué no está.

### Un rechazo por regla no corta el lote

Reintentar una transición ilegal la vuelve a rechazar para siempre, así que **no es algo que se vaya a arreglar en la corrida siguiente**. Cortar doscientos movimientos por uno que el workflow no admite deja al resto sin mover por algo que no va a cambiar.

Lo que sí corta es un fallo del transporte: eso no es del ítem, y el que sigue va a fallar igual.

## Los lotes

Cada ítem que se mueve cuesta **dos llamadas** — listar sus transiciones y pedir la elegida por id, ver [`concepts/sync.md`](../concepts/sync.md#y-pedir-por-id-es-lo-que-saca-una-ambigüedad-no-sólo-una-llamada). Sobre 188 divergentes eso es ~376 escrituras contra un board real.

`--limit` corta por lo mismo que en [`bootstrap`](bootstrap.md): *"un lote no necesita ser una transacción —de eso ya se ocupa el ancla— sino un corte"*. Acá el ancla es que **volver a correrlo cuesta una lectura y cero escrituras**: lo que ya coincide deja de aparecer en el plan, así que la segunda corrida arranca donde quedó la primera sin que nadie lleve la cuenta.

## Corre sobre el panorama, y lee

`--ref` apunta por defecto a la rama insegura, y eso **no contradice** que a una insegura no se le empuje: este comando no escribe en git. Lee el árbol y escribe en el proveedor.

Es la misma forma que [`bootstrap`](bootstrap.md), y por la misma razón: el panorama es el único que tiene el inventario completo, y un recorte contestaría sobre sus ítems callando el resto.

### Y el panorama es el del servidor, que es donde este comando corre

> **No alcanzaba con decir "el panorama": había dos, y este comando leyó el equivocado.**

El 2026-09-07, corrido desde el clon, leyó un worktree con dos commits de atraso y **devolvió un ítem al estado viejo en el board**. El plan que imprimió era consistente con lo que esa copia decía, así que no había nada que mirar y sospechar. Es el caso entero de [§ El panorama vive en un solo lado](../concepts/sync.md#el-panorama-vive-en-un-solo-lado-y-la-ventana-en-los-dos), y lo que lo cierra no es un guardia adentro de este comando: es que **la copia que se podía quedar vieja ya no existe**.

Lo que queda es esto, y se dice para que no se lea como una promesa más grande:

| | |
|---|---|
| sin `--ref` | lee el panorama del bare donde corre, que es **la autoridad** — no hay copia de la que dudar |
| con `--ref` a una ventana | lee un recorte, y un recorte **puede estar atrasado**: la propagación va de la ventana al panorama, nunca al revés |

**El segundo caso sigue abierto y se acepta**, porque es un gesto y no un default: nombrar una ventana es escribir su nombre. Lo que se sacó fue el camino que no pedía ningún gesto.

## Salida

```
$ worklist-server push-states --project ACC --states-map ~/.local/share/accreta/worklist-sync/states.json --limit 10 --dry-run
refs/heads/insecure/all: 185 divergen, 10 en este lote
  ACC-100 Tareas por hacer -> Finalizada (local: done)
  ACC-101 Tareas por hacer -> Finalizada (local: done)
  …

$ worklist-server push-states --project ACC --states-map … --limit 10
refs/heads/insecure/all: 185 divergen, 10 en este lote
  …
  movido: ACC-100 -> Finalizada
  movido: ACC-101 -> Finalizada
movidos 10 de 10, quedan 175
```
