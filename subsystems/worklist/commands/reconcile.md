# Comando: `worklist-server reconcile`

Adopta los issues que ya existen del otro lado y el panorama no registró. Es la primera pasada de [`bootstrap`](bootstrap.md) **sin la creación**.

## No puede crear nada, y ésa es la propiedad

> **Reparar no debería poder crear.** Un comando que repara se corre cuando algo salió mal, que es exactamente cuando menos se quiere estar decidiendo si además se va a escribir.

`bootstrap` recupera lo perdido de paso —`create_or_find` encuentra por título antes de crear— pero **para recuperar veintitrés claves hay que correr el comando que además crea ochenta y seis issues**. Eso no es una reparación: es volver a intentar la operación entera y esperar que esta vez llegue al final.

Y en el caso general se degrada solo: si reparar es *"corré otra vez lo que se cayó"*, cada recuperación es una escritura nueva contra el proveedor, y una que falle a mitad deja más para recuperar que antes.

**No hay flag que lo habilite.** Un `--create` volvería esto `bootstrap` con otro nombre, y la garantía se sostiene por no existir, no por estar apagada.

## Firma

```
worklist-server reconcile --project <clave> [--ref <rama>] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--project` | El proyecto del proveedor: `ACC`. |
| `--ref` | Sobre qué rama. Por defecto `refs/heads/insecure/all`. |
| `--dry-run` | Pregunta al proveedor y dice qué adoptaría, sin escribir nada local. |

**No lleva `--base`.** El cuerpo no viaja: adoptar es escribir la clave de este lado, y nada del otro. Es la misma razón por la que [`propagate`](propagate.md#no-pide-credencial-y-eso-no-es-un-detalle) no pide credencial — un comando de reintento no puede depender de más de lo que arregla.

## Qué hace

1. Lista los ítems del árbol cuyo nombre **no** es una clave de proveedor. Los `_sprints/*.sprint.md` no entran.
2. Por cada uno pregunta si existe un issue cuyo `summary` sea **exactamente** ese título — la misma búsqueda de [`create-or-find`](create-or-find.md), y la misma comparación literal después.
3. Al que existe lo adopta: renombra el archivo a la clave y reescribe las referencias, igual que la pasada 1.
4. Al que no existe **no le hace nada**, y lo nombra.

El orden es topológico por `parent`, por la misma razón que en `bootstrap`: el renombre de una épica reescribe lo que cuelga de ella.

## Lo que no encuentra se nombra, no se cuenta

> **Un total no sirve acá.** Quien corre esto está reparando, y necesita saber *cuáles* quedaron afuera para decidir si es lo esperado o es otro problema.

```
$ worklist-server reconcile --project ACC
refs/heads/insecure/all: adopto 23 de 109 item(s)
  11 -> ACC-182  (4 refs reescritas)  [parent ACC-14]
  12 -> ACC-183  (1 refs reescritas)
  …
  y 86 sin issue del otro lado:
    2x  Error de reporte: `graph --format json` no imprime nada cuando no hay resultados…
    38  Error de aceptación: `--no-n1` se ignora cuando hay proveedor…
    …
refs/heads/insecure/all: a1b2c3d -> 9z8y7x6
```

**No encontrar es el caso normal**, no un error: un ítem del backlog que nunca cruzó no tiene contraparte, y eso es lo que `bootstrap` existe para cambiar. Por eso el código de salida es `0`.

## La identidad sigue siendo el título, y se dice

Buscar por `summary` es la identidad implícita de [`create-or-find`](create-or-find.md#comportamiento) —la query busca de más y decide la comparación exacta— con todo lo que arrastra: **un título editado del lado del proveedor desde que el issue se creó hace que la reconciliación no lo encuentre**, y el ítem queda huérfano igual que antes.

No es de este comando resolverlo. Sí es suyo **no fingir que lo resolvió**: por eso lo que no encuentra se reporta uno por uno, con su título, y no como un número.

### Y la otra mitad de la deriva no la mira nadie

Este comando va de local hacia el proveedor: recorre los ítems sin clave y pregunta. Al revés —un issue que ningún ítem del worklist reclama— **queda fuera de alcance**, y no porque sea difícil sino porque es otra pregunta: acá se repara una correspondencia que se perdió, no se audita el board.

Y no está escrito en ningún lado todavía. No es [`concepts/drift.md`](../concepts/drift.md), que es sobre un ítem que se separó de su fragmento; es sobre el board que se separó del worklist, y hoy nadie lo mira.

## Dónde se corre

Lo mismo que [`bootstrap`](bootstrap.md#se-corre-donde-el-panorama-vive-y-hoy-eso-es-tu-worktree), y por lo mismo: parado donde el panorama está checkouteado, o con worktree temporal si no lo está; nunca moviendo una rama que otro worktree tiene abierta; y con el árbol limpio, porque el renombre commitea con `add -A`.

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | se adoptó lo que existía, hubiera o no ítems sin contraparte |
| `1` | un ciclo entre ítems, o el proveedor falló — la ref queda como estaba |
| `1` | la rama está abierta en otro worktree, o el árbol tiene cambios sin commitear |
