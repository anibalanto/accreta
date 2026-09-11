# Especificación: comando `bilinker fetch`

## Propósito

Traer el repo de un proveedor declarado. **Es la operación de red de la [frontera](../concepts/frontier.md)**, y es explícita a propósito: [`check`](check.md) corre sobre todos los bilinks y no puede clonar como efecto colateral, así que un proveedor que no está traído se reporta `REMOTE_UNREACHABLE` hasta que alguien corre esto.

## Firma

```
bilinker fetch [<alias>]
```

| Argumento | Descripción |
|---|---|
| `<alias>` | El proveedor a traer, declarado en `.bilink/.<alias>.toml`. Sin argumento, todos los declarados en esta capa. |

## Qué trae: una sola ref

Del proveedor se trae **`refs/bilink/<branch>`**, con la rama que declara su `.toml`, y nada más. Esa ref lleva el árbol del proyecto más `.bilink/` ([la ref](../concepts/ref.md)), así que un solo fetch trae las declaraciones del proveedor y el código al que apuntan, coherentes por construcción.

El clon vive en `.bilink/<alias>/`, al lado de su declaración, y queda fuera de la ref de bilinks del consumidor: no se versiona el checkout de otro repo.

Un proveedor sin `refs/bilink/<branch>` no tiene nada que consumir:

```
$ bilinker fetch hsi
error: no se pudo traer refs/bilink/main de 'hsi'.
  ¿El proveedor ya cortó a la ref? Sin `refs/bilink/<branch>` no hay nada que consumir.
```

## Superficial, y con el sparse calculado

El clon es **superficial**: el tip de la ref, sin historia. Alcanza para `check`. La historia se paga después, y solo donde alguien mira un diff ([profundidad a pedido](../concepts/frontier.md#profundidad-a-pedido)).

Como el clon es superficial, el fetch fuerza la ref: git no tiene la historia para probar que el tip nuevo desciende del viejo. Es seguro porque la ref de bilinks solo va para adelante.

El sparse-checkout arranca en `.bilink/` y, con los bilinks del proveedor ya en mano, se amplía a los archivos que los bilinks de esta capa consumen ([el sparse se calcula](../concepts/frontier.md#el-sparse-se-calcula-no-se-declara)). Correrlo de nuevo después de consumir otra abstracción es lo que saca su archivo al árbol.

## La versión se verifica antes de interpretar

Después de traer la ref, se lee `.bilink/version` del proveedor, y si no se entiende, se niega ([la versión del proveedor](../concepts/frontier.md#la-versión-del-proveedor-se-verifica)):

```
$ bilinker fetch hsi
error: el proveedor 'hsi' no declara versión de formato: es anterior a la frontera.
  No se puede interpretar lo que publica.
```

## Salida

Una línea por proveedor. Medido el 2026-09-11, entre dos repos locales:

```
$ bilinker fetch prov
prov: refs/bilink/main · 0 archivo(s) en el sparse       ← antes de consumir nada

$ bilinker fetch prov
prov: refs/bilink/main · 1 archivo(s) en el sparse       ← con una abstracción consumida
```

Sin proveedores declarados, lo dice y no hace nada:

```
$ bilinker fetch
no hay ningún proveedor declarado (.bilink/.{alias}.toml)
```

## Lo que trae, `check` lo compara

Un cambio que el proveedor re-aceptó no aparece en `check` hasta el siguiente `fetch`: recién ahí la punta `repo` pasa a `CHAIN_DIRTY`, y `get --diff` muestra el cambio a través de la frontera. Es la consecuencia de que `check` no haga red, y es a propósito.

## Códigos de salida

| Código | Condición |
|---|---|
| 0 | Traído, o no hay ningún proveedor declarado. |
| 1 | El alias no está declarado; o el fetch falló; o la versión del proveedor no se entiende. |

## Propiedades garantizadas

- **Es la única operación de la frontera que hace red.**
- Trae una sola ref, `refs/bilink/<branch>`, superficial.
- El clon queda en `.bilink/<alias>/`, fuera de la ref del consumidor.
- El sparse sale de los bilinks de esta capa, y nunca se declara.
- No interpreta un proveedor cuya versión de formato no entiende.
