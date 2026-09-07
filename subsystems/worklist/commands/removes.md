# Comando: `worklist-server removes`

Saca del árbol los ítems cuya clave el proveedor ya no tiene, **sin borrarlos**.

```
worklist-server removes [--ref <rama>] (--provider-file <archivo> | --project <clave> …) [--dry-run]
```

Por defecto `--ref` es el panorama, y no es un default de comodidad: **ahí están las dos cosas que hay que tocar** —el archivo del ítem y el `items` que lo nombra— y las ventanas se regeneran solas con el próximo [recorte](window-open.md).

## Por qué hace falta

Un ítem puede tener clave acá y no existir del otro lado. Medido el 2026-09-07 sobre las 301 claves del panorama: **una**, `ACC-268`, con su `rename 6m -> ACC-268` en el log y un 404 de Jira.

Y eso **traba el ítem para siempre**: `is_unassigned` es falso, así que ninguna pasada vuelve a mirarlo — no se recrea, no se corrige, y su clave muerta [se lleva el lote del sprint entero](assign-keys.md#una-clave-que-el-board-no-tiene-no-puede-llevarse-el-lote) en cada push. No había salida.

## Se decide por el código, nunca por el mensaje

El proveedor contesta *"la incidencia no existe **o no tienes permiso para verla**"* — **una sola frase para dos casos que no se parecen en nada.**

| | |
|---|---|
| **404** | no existe → sale del árbol |
| **403 / 401** | no se puede ver → **no se toca**, y se reporta |
| cualquier otro | **no es una respuesta**: un 500 no dice que el ítem no esté, y tratarlo así lo borraría por un mal día del servidor |
| el proveedor no lo puede contestar | *no se preguntó* — el de prueba no tiene con qué |

Sacar un ítem porque una credencial perdió permiso sería el mismo error de forma que confundir `sin verificar` con `coincide`, **con el costo subido a destruir**.

**Y se pregunta clave por clave**, al contrario de [`snapshot`](../concepts/sync.md#tener-clave-es-del-ítem-verificarse-es-de-la-rama), que toma el lote entero: lo que se decide acá es mover un archivo, y una respuesta de lote no distingue *no existe* de *no se pudo ver*.

## Sale del árbol y del `items`, las dos

**Sacar el archivo sin sacar la referencia deja el sprint roto**: el próximo recorte falla con *"la composición nombra a `ACC-268`, y no está"*. No es una mejora del comando — es un requisito.

```
$ worklist-server removes --project ACC …
refs/heads/insecure/all: 301 clave(s)
  ACC-268  sale del arbol: ACC-268.task.md -> .metadata/removes/ACC-268.task.md
resumen: 1 sacada(s)
```

Y el commit que deja es exacto:

```
 .metadata/product.yaml                               | 1 -
 ACC-268.task.md => .metadata/removes/ACC-268.task.md | 0
```

**Un rename puro** —cero líneas cambiadas— y una línea menos en la composición.

## Por qué lógico y no físico

*"Se recupera de git"* es cierto y no alcanza. Un borrado físico deja una ausencia, y **una ausencia no dice por qué**: quien la encuentra tiene que sospechar que alguna vez hubo algo, y recién ahí buscar. Un archivo en `.metadata/removes/` contesta las dos preguntas sin arqueología — qué era y por qué se fue.

Y hace la vuelta barata: **devolverlo es moverlo de nuevo.** Si el 404 fue un error, restaurar es un `git mv`, no un rescate.

## Lo que no hace

**No libera la clave**, y el ítem no vuelve a nacer con una nueva. Recrearlo automáticamente sería el sistema discutiéndole al board sobre algo que alguien borró a mano allá. Si el trabajo hace falta, se escribe un ítem nuevo — que es una decisión de una persona.

**Y no arregla las citas.** Otros ítems pueden nombrar al que salió, en prosa, y eso queda apuntando a un archivo que se movió. Es markdown, así que nada lo detecta — la misma razón por la que [una spec no cita un ítem](../concepts/item.md).

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | se pudo preguntar, sacara o no |
| `1` | no se pudo preguntar, o una respuesta no fue ni sí ni no |
