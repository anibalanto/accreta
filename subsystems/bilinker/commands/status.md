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

Los bilinks se agrupan por el directorio del endpoint estructural. Si el bilink no tiene endpoint estructural (solo layer endpoints), aparece bajo `(layer)`.

## Códigos de salida

| Código | Condición |
|--------|-----------|
| 0 | Éxito. |

## Propiedades

- **Solo lectura**: no modifica ningún archivo.
- Usa el estado almacenado en los archivos `.bilink` — no re-ejecuta queries. Para actualizar el estado, correr `bilinker check` primero.
