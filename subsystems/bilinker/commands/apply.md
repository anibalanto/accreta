# Especificación: comando `bilinker apply`

## Propósito

Aplica los auto-fixes generados por `bilinker check` que están acumulados en `.bilink/.pending/`. Cada aplicación se registra como un commit git con un mensaje descriptivo. El humano siempre confirma antes de que los cambios se escriban.

Requiere git como dependencia dura.

## Firma

```
bilinker apply [--dry-run] [--filter <estado>] [-y]
```

| Flag | Descripción |
|---|---|
| `--dry-run` | Muestra los fixes que se aplicarían sin escribir nada. |
| `--filter <estado>` | Aplica solo fixes de un estado específico (e.g., `--filter MOVED`). |
| `-y` | Omite la confirmación interactiva. |

## Área de staging: `.bilink/.pending/`

`bilinker check` escribe un archivo `.fix` por cada auto-fix pendiente:

```
.bilink/.pending/
  7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a-1.fix
  3a4b5c6d-2e3f-4a5b-9c6d-7e8f9a0b1c2d-0.fix
```

El nombre del archivo es `<uuid>-<endpoint>.fix`. Cada `.fix` describe en texto legible el cambio propuesto. El usuario puede leer, editar o eliminar archivos `.fix` antes de correr `apply`.

### Formato de un `.fix`

```
uuid: 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a
endpoint: 1
state: MOVED
change:
  file: specs/persona/voting.yaml → specs/domain/voting.yaml
  git-similarity: R92
```

## Flujo de `apply`

1. Leer todos los archivos `.fix` en `.bilink/.pending/`.
2. Mostrar resumen:
   ```
   Pending fixes (3):
     MOVED      7f3d8e9a…  link.1  specs/domain/voting.yaml
     DISPLACED  3a4b5c6d…  link.0  offset 5~42 → 8~45
     EXPANDED   f1e2d3c4…  link.0  start~end ampliado
   ```
3. Pedir confirmación (o `-y` para omitirla).
4. Para cada fix, modificar el archivo `.bilink` correspondiente:
   - **MOVED**: actualizar `file` en `link.N`.
   - **DISPLACED**: actualizar `start~end` en `link.N`.
   - **REANCHORED**: actualizar los predicados de la query en `link.N`.
   - **EXPANDED**: actualizar `start~end` en `link.N` con el nuevo rango.
5. Actualizar la cache (`range.N`, `state.N`, `resolved_at`). Para EXPANDED y
   REANCHORED, donde el contenido del fragmento cambió, agrega una nueva entrada estableciendo `hash.N` y `commit.N` con el hash actual y el commit actual.
6. Eliminar los `.fix` aplicados de `.bilink/.pending/`.
7. Crear un commit git con todos los `.bilink` modificados.

**Mensaje de commit:**

```
bilinker: auto-fix MOVED + DISPLACED + EXPANDED (2026-05-24)

- 7f3d8e9a… link.1: MOVED → specs/domain/voting.yaml
- 3a4b5c6d… link.0: DISPLACED offset 5~42 → 8~45
- f1e2d3c4… link.0: EXPANDED start~end ampliado
```

## Salida

```
$ bilinker apply

Pending fixes (3):
  MOVED      7f3d8e9a…  link.1  specs/domain/voting.yaml
  DISPLACED  3a4b5c6d…  link.0  offset 5~42 → 8~45
  EXPANDED   f1e2d3c4…  link.0  start~end ampliado

Apply? [y/N] y

Applied 3 fixes.
Committed: a4b5c6d "bilinker: auto-fix MOVED + DISPLACED + EXPANDED (2026-05-24)"
```

## Código de salida

| Código | Condición |
|---|---|
| 0 | Todos los fixes aplicados exitosamente. |
| 1 | Error al aplicar algún fix. |
| 2 | No hay fixes pendientes. |

## Invariantes

- `apply` nunca modifica un `.bilink` que no tenga un `.fix` correspondiente.
- Si el `.bilink` fue modificado desde que se generó el `.fix`, `apply` rechaza
  ese fix y alerta al usuario.
- Solo MOVED, DISPLACED, REANCHORED y EXPANDED generan `.fix`. Los estados
  UNANCHORED, ALTERED, DELETED, BROKEN y CHAIN_DIRTY nunca tienen auto-fix.
