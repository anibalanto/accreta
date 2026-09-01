# Especificación: comando `bilinker relayer`

## Propósito

Mueve los bilinks de una capa a la de arriba.

Existe por un modo de falla concreto: **un `.bilink/` fabrica una raíz de capa**, porque es uno de los marcadores con los que bilinker [resuelve la raíz](../concepts/configuration.md#resolución-de-la-raíz). Si queda en un directorio que stratum no declara como capa, las dos herramientas discrepan sobre dónde termina una — y el `check` de la capa de arriba deja de ver esos bilinks **sin decir nada**, que es la peor forma de no verlos.

## Firma

```
bilinker relayer <capa> [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `<capa>` | La capa a vaciar, relativa a la actual. |
| `--dry-run` | Muestra qué movería sin escribir nada. |

## Lo que se mueve es la ubicación, nunca el contenido

Es la propiedad que lo hace seguro, y sale de cómo está armado el formato:

> El id de un capture es `sha256(file \0 query \0)`, así que **prefijar el `file` le cambia el id**. Los `hash` no se tocan.

De ahí que ningún endpoint pase a `ALTERED`: el fragmento no se movió, y lo aprobado sobre él sigue coincidiendo. Lo que cambia es **desde qué capa se lo nombra**.

Es lo mismo que hacen [`apply`](apply.md) y [`accept --place`](accept.md) juntos —corregir una ubicación y aprobarla— pero **entre capas**, que es lo que ninguno de los dos sabe hacer: los dos operan dentro de una.

## Qué reescribe

| | Qué le pasa |
|---|---|
| cada capture | se reacuña con el `file` prefijado, y **cambia de id** |
| `link` y `accepted.link` de un endpoint `capture` | el id nuevo |
| `link` de un endpoint `path` **relativo a la capa que se vacía** | gana el prefijo: `>impl` pasa a `<capa>>impl` |
| `accepted.link` de un endpoint `path` | **sólo si el id está en el mapa**: es una copia opaca del vecino, y el capture del vecino no se movió |
| los bilinks de las capas que cuelgan de la que se vacía | su `accepted.link` copiaba un id que cambió |
| `hash` y `hash_ast` | **nada** |

El `path <` de las capas de abajo **no se toca**, y es lo que hace que la migración sea de un solo lado: al desaparecer el `.bilink/`, ese `<` pasa a resolver a la capa de arriba por sí solo — porque el marcador que lo detenía era ese mismo directorio.

## Es una decisión, y tiene verbo propio

Su commit sobre la ref es del tipo **decisión** —un padre, sólo `.bilink/` cambia— y su [comando canónico](../concepts/ref.md#el-mensaje-es-el-comando) es `relayer <capa>`.

Hizo falta un verbo nuevo: mover bilinks entre capas no es ninguno de los actos que existían. `apply` repunta un endpoint a **otro fragmento**; acá el fragmento es el mismo. Es la misma situación que el caso 3.b antes de [`pull`](pull.md) — un acto real sin nombre, y la señal de que faltaba era que no se podía commitear.

## Salida

```
$ bilinker relayer subsystems/stratum
subsystems/stratum: 9 capture(s) reacuñados, 10 bilink(s) movidos, 10 vecino(s) con el id actualizado
commit:  refs/bilink/… @ f2a6ca1
```

## Código de salida

| Código | Condición |
|---|---|
| 0 | Movido. |
| 1 | La capa no tiene `.bilink/` propio, o es la capa de destino. |

## Invariantes

- Ningún `hash` ni `hash_ast` cambia.
- Ningún endpoint pasa a `ALTERED`, `UNRESOLVED` ni `RELOCATED`.
- El `.bilink/` de la capa vaciada se borra: si quedara, la capa seguiría fabricada.
- Todo o nada: los captures se reacuñan en memoria y se escribe recién al final.

## Lo que no hace

**No decide si una capa debería serlo.** Eso lo dice stratum, y bilinker no lo consulta — [`1w`](../../../.stratum/worklist-accreta/1w.task.md) deja abierto si `check` debería avisar al encontrar un `.bilink/` bajo la capa actual que no corresponde a una capa declarada. Este comando arregla el caso; no lo detecta.
