# Comando: `worklist propagate`

Sube al panorama lo que una ventana resolvió. Es la dirección de ida de [la propagación](../concepts/propagation.md); la de vuelta es regenerar la ventana, y ésa la hace [`window open`](window-open.md).

[`assign-keys`](assign-keys.md) lo corre al final de cada ventana, así que **el flujo normal no lo invoca nadie a mano**. Existe suelto por una razón: si la propagación se cae, la ventana queda adelantada del panorama, y de ahí se sale reintentando.

## Firma

```
worklist propagate [--window <id>]… | [--all-windows] | [--stdin] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--window` | Una ventana por su id. Se puede repetir. |
| `--all-windows` | Todas las `refs/heads/secure/**` del repo, en orden numérico. |
| `--stdin` | Lee `<viejo> <nuevo> <ref>` por línea — el protocolo del hook. |
| `--dry-run` | Dice qué subiría, sin tocar ninguna ref. |

Sin ninguno de los tres, propaga la rama del `HEAD` actual.

## No pide credencial, y eso no es un detalle

**La propagación es entre ramas de git y no habla con ningún proveedor.** No hay `preflight` que pase: corre con el token puesto y sin él.

Es lo que la vuelve el reintento útil. La corrida que se cae en el medio se cae casi siempre del lado del proveedor —una llamada que no volvió, una credencial que faltaba—, y si subir al panorama pidiera lo mismo, el reintento se caería en el mismo lugar.

## Qué sube

Lo que la ventana tiene y el panorama todavía no: desde `refs/worklist/propagated/<rama>`, o desde el **commit del corte** la primera vez.

**El corte nunca sube.** Es un commit que borra los ítems que el recorte dejó afuera; aplicado al panorama, le borraría los 233 que esa ventana no tiene. Por eso el comando se niega cuando no lo encuentra, en vez de propagar desde la base:

```
$ worklist propagate --window 1
error: no encuentro el corte de esta ventana: el primer commit sobre el panorama es
  a2b035d edito o

  sin el corte no se sabe donde empieza el trabajo de la ventana, y propagarlo
  entero le borraria al panorama todo lo que el recorte dejo afuera.
```

Pasa con una rama `secure/**` que nació de un `checkout -b` en vez de un recorte — que es exactamente lo que [`window open`](window-open.md) existe para evitar.

### El renombre se rehace; todo lo demás se cherry-pickea

Ver [`concepts/propagation.md`](../concepts/propagation.md#el-renombre-es-el-único-que-no-se-copia-y-el-motivo-es-de-alcance). El `rename <slug> -> <clave>` de la ventana corrigió las referencias que la ventana veía; en el panorama hay más, así que se vuelve a hacer ahí y la salida dice cuántas tocó de este lado.

## Salida

```
$ worklist propagate --window 10
refs/heads/secure/sprint/10: sube 3 commit(s) al panorama
  rename agregar-b -> ACC-101  (rehecho)  (4 refs reescritas en el panorama)
  9z8y7x6 normalize: ACC-101
  a1b2c3d edito la descripcion de ACC-101
  panorama: a1b2c3d -> 7f6e5d4
```

Y la segunda corrida sobre lo mismo:

```
refs/heads/secure/sprint/10: el panorama ya tiene todo lo suyo
```

Un commit que el panorama ya tenía por otra vía **no es un error**: se informa y se sigue.

```
  9z8y7x6 normalize: ACC-101  (el panorama ya lo tenia)
```

## Cuando algo no aplica

**El panorama no avanza, y la marca no se mueve.** Las dos cosas juntas son lo que hace que el reintento vuelva a pasar por lo mismo en vez de saltearlo.

```
$ worklist propagate --window 10
error: el panorama no recibe a2b035d edito ACC-14
  choca en: ACC-14.epic.md

  el panorama no avanzo y la ventana queda adelantada — se reintenta.
  Lo que el proveedor arbitra no llega hasta aca: si esto choco, dos
  ventanas escribieron algo que el no ve.
```

**El mensaje dice contra qué archivo**, y no *"conflicto"*, porque el archivo es la información: lo que llega hasta acá es lo que el compare-and-swap no arbitró, y en la práctica es un ancestro de sólo lectura que alguien editó adentro de su ventana.

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | el panorama recibió lo que faltaba, o ya lo tenía |
| `1` | algo no aplicó, o la ventana no tiene corte — ninguna ref se movió |
