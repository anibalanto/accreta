# Especificación: comando `bilinker apply`

## Propósito

Aplica los auto-fixes para bilinks en estado auto-fixeable (MOVED, DISPLACED, REANCHORED, EXPANDED). Calcula el fix en el momento re-resolviendo el endpoint contra git y el AST actual — nunca reutiliza un cálculo previo. Cada aplicación se registra como un commit git con un mensaje descriptivo. El humano siempre confirma antes de que los cambios se escriban.

`apply` corrige **dónde** apunta el endpoint (`file`, `start~end`, predicados de la query). Nunca toca `hash.N` ni `commit.N` — eso es exclusivo de `bilinker accept`. Para MOVED y DISPLACED el contenido del fragmento no cambió, así que el endpoint queda `OK` tras el fix; para EXPANDED y REANCHORED el contenido sí cambió y el endpoint sigue no-OK hasta que el humano ejecute `bilinker accept`.

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

## Flujo de `apply`

1. Escanear todos los `.bilink` de la layer actual.
2. Para cada bilink con algún endpoint en {MOVED, DISPLACED, REANCHORED, EXPANDED}, **re-resolver el endpoint** con el mismo algoritmo de detección que usa `check` (ver [`check`](check.md) § "Algoritmo de detección por tipo de endpoint"). Esto produce un estado re-derivado y la posición actual del fragmento.
3. Validar el estado re-derivado contra `state.N` (ver "Validación de frescura"). Si difieren, descartar ese endpoint.
4. Calcular el fix a partir de la re-resolución (ver "Cálculo del fix por estado"). Si el fix es un no-op — el `.bilink` ya apunta al lugar correcto — omitir ese endpoint.
5. Mostrar resumen:
   ```
   Pending fixes (3):
     MOVED      7f3d8e9a…  link.1  specs/domain/voting.yaml
     DISPLACED  3a4b5c6d…  link.0  offset 5~42 → 8~45
     EXPANDED   f1e2d3c4…  link.0  start~end ampliado          → requiere accept
   ```
6. Pedir confirmación (o `-y` para omitirla).
7. Para cada fix, modificar `link.N` y actualizar la cache del `.bilink` (ver "Escritura de cache").
8. Crear un commit git con todos los `.bilink` modificados.

**Mensaje de commit:**

```
bilinker: auto-fix MOVED + DISPLACED + EXPANDED (2026-05-24)

- 7f3d8e9a… link.1: MOVED → specs/domain/voting.yaml
- 3a4b5c6d… link.0: DISPLACED offset 5~42 → 8~45
- f1e2d3c4… link.0: EXPANDED start~end ampliado
```

## Cálculo del fix por estado

### MOVED

```
git diff -M --name-status HEAD -- <link.N.file>
→ R<similarity>  <old_path>  <new_path>
```

`apply` verifica que el hash del fragmento siga coincidiendo en `new_path`, luego actualiza `link.N.file = new_path`.

### DISPLACED

`apply` re-ejecuta la query tree-sitter y busca `hash.N` en los offsets del nodo resultante. El offset donde el hash coincide es el fix: actualiza `start~end` en `link.N`. No lee `range.N` — lo recalcula, porque `range.N` refleja el último `check` y el archivo puede haber cambiado desde entonces.

### EXPANDED

`apply` re-ejecuta la query tree-sitter y toma el byte range del nodo resultante, verificando que el fragmento de `hash.N` siga siendo subcadena de ese nodo y que el AST interno no haya cambiado. Actualiza `start~end` en `link.N` con el rango del nodo.

El fragmento apuntado ahora es más grande que el que `hash.N` describe, así que el endpoint queda en EXPANDED hasta que `bilinker accept` fije el hash del fragmento ampliado. El segundo `apply` sobre ese endpoint es un no-op y se omite (paso 4 del flujo).

### REANCHORED

`apply` re-ejecuta la búsqueda AST: busca en el árbol sintáctico actual un nodo del mismo tipo que el anchor original pero con nombre diferente que contenga el hash del fragmento. Actualiza los predicados de la query en `link.N` con el nuevo nombre encontrado, y `start~end` con el rango del nodo hallado.

Igual que EXPANDED, el endpoint queda no-OK hasta que `bilinker accept` fije el hash bajo el anchor nuevo.

## Validación de frescura

`state.N` es un valor cacheado por el último `check`; el archivo fuente pudo cambiar después. Por eso `apply` nunca aplica un fix derivado de la cache: re-resuelve el endpoint (paso 2) y compara el estado re-derivado contra `state.N`.

| Situación | Acción |
|---|---|
| Estado re-derivado == `state.N` | Aplicar el fix. |
| Estado re-derivado == `OK` | Omitir en silencio — el fix ya no hace falta. |
| Estado re-derivado != `state.N` y != `OK` | Descartar el fix, escribir el estado re-derivado en `state.N` y advertir. |

```
warn: 3a4b5c6d… link.0: state.0 dice DISPLACED pero la resolución actual da ALTERED
      — cache desactualizada, fix descartado. Correr `bilinker check`.
```

Un endpoint que pasó de auto-fixeable a no auto-fixeable (e.g. DISPLACED → ALTERED) nunca se toca: cae bajo la invariante general.

## Escritura de cache

Tras aplicar un fix, `apply` actualiza en el `.bilink`:

- **`link.N`** — el fix propiamente dicho (`file`, `start~end`, o predicados de la query).
- **`range.N`** — byte range del fragmento tras el fix.
- **`state.N`** — estado re-derivado *después* de aplicar el fix: `OK` para MOVED y DISPLACED; `EXPANDED` / `REANCHORED` se mantienen, porque el contenido cambió y solo `accept` puede cerrarlos.
- **`resolved_at`** — timestamp UTC de esta ejecución.
- **`hash.N` / `commit.N`** — nunca se modifican.

## Salida

```
$ bilinker apply

Pending fixes (3):
  MOVED      7f3d8e9a…  link.1  specs/domain/voting.yaml
  DISPLACED  3a4b5c6d…  link.0  offset 5~42 → 8~45
  EXPANDED   f1e2d3c4…  link.0  start~end ampliado          → requiere accept

Apply? [y/N] y

Applied 3 fixes.
  2 resueltos (OK)
  1 requiere `bilinker accept` — el contenido del fragmento cambió:
    f1e2d3c4… link.0  EXPANDED
Committed: a4b5c6d "bilinker: auto-fix MOVED + DISPLACED + EXPANDED (2026-05-24)"
```

## Código de salida

| Código | Condición |
|---|---|
| 0 | Todos los fixes aplicados exitosamente. |
| 1 | Error al calcular o aplicar algún fix, o algún fix descartado por cache desactualizada. |
| 2 | No hay bilinks en estado auto-fixeable. |

## Invariantes

- `apply` nunca modifica un `.bilink` cuyo estado no sea auto-fixeable.
- `apply` nunca escribe `hash.N` ni `commit.N`. Un fix corrige a dónde apunta el endpoint, nunca qué contenido está aceptado.
- `apply` nunca aplica un fix derivado de la cache: cada fix se recalcula re-resolviendo el endpoint contra el árbol y el índice git actuales.
- Si el estado re-derivado no coincide con `state.N`, `apply` descarta ese fix y alerta al usuario.
- Si el fix calculado no puede verificarse (e.g., hash no coincide en el nuevo path para MOVED, o el anchor no se encuentra para REANCHORED), `apply` rechaza ese fix y alerta al usuario.
- `apply` es idempotente: un fix ya aplicado se detecta como no-op y se omite.
- Solo MOVED, DISPLACED, REANCHORED y EXPANDED tienen auto-fix. Los estados UNANCHORED, ALTERED, DELETED, BROKEN y CHAIN_DIRTY nunca son tocados por `apply`.
- Tras `apply`, MOVED y DISPLACED quedan en `OK`; EXPANDED y REANCHORED quedan en su mismo estado y requieren `bilinker accept` para cerrarse.
