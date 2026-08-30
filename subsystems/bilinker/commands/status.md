# Especificación: comando `bilinker status`

## Propósito

Muestra el estado de todos los bilinks en la capa actual, agrupados por archivo. Equivalente a `git status` pero para el grafo de bilinks.

## Firma

```
bilinker status [<path>]
```

| Argumento | Descripción |
|-----------|-------------|
| `<path>` | Directorio de la capa a inspeccionar. Por defecto: directorio actual. |

## Salida

```
$ bilinker status

commands/
  accept.md   f715f67e  (OK, OK)
              6f9fa32c  (OK, OK)
  graph.md    49218eae  (OK, OK)
              298e1b92  (OK, OK)

concepts/
  bilink.md   1e318d3a  (OK, OK)
```

Cada línea muestra:
- Nombre del archivo (solo en la primera aparición)
- UUID corto (8 chars)
- Estado de ambos endpoints: `(state.0, state.1)`

Los bilinks se agrupan por el directorio del endpoint estructural. Un bilink sin endpoint estructural —los dos son `path`— aparece bajo `(layer)`.

## Códigos de salida

| Código | Condición |
|--------|-----------|
| 0 | Éxito. |

## Con la cache fría no hay nada que mostrar

`status` lee [`cache/state`](../concepts/cache.md); no resuelve ninguna query. Y la cache no está en git, así que **un clon fresco no tiene estados**:

```
$ bilinker status

sin estados: la cache está fría.
  Correr `bilinker check .` para calcularlos.
```

No es un error: la cache fría es un estado normal —clon fresco, cambio de rama, otra máquina— y `check` es offline y barato. Lo que `status` no hace es inventar: mostrar `OK` sin haber verificado sería peor que no mostrar nada.

Si la cache corresponde a otra rama, se descarta sola y el resultado es el mismo.

## Propiedades

- **Sólo lectura**: no modifica ningún archivo, ni siquiera la cache.
- No re-ejecuta queries. Para actualizar los estados, `bilinker check` primero.
