# Especificación: comando `bilinker apply`

## Propósito

Corrige **dónde apunta** un endpoint cuando el fragmento se movió. Calcula el fix en el momento re-resolviendo contra git y el AST actuales — nunca reutiliza un cálculo previo.

`apply` acuña el [capture](../concepts/capture.md) de la ubicación nueva y repunta el `link` del endpoint. **No escribe `accepted`**, así que el endpoint no queda `OK`: queda en `RELOCATED` hasta que alguien apruebe la ubicación nueva.

> **`apply` propone, `accept` dispone.**

Ésa es la diferencia con el comportamiento anterior, donde MOVED y DISPLACED volvían a `OK` solos. Mover un vínculo a otro fragmento es una decisión —el fragmento nuevo puede no ser el que la spec describe— y una decisión sin aprobar es trabajo pendiente, no trabajo hecho.

Requiere git como dependencia dura.

## Firma

```
bilinker apply [--dry-run] [--filter <estado>] [-y]
```

| Flag | Descripción |
|---|---|
| `--dry-run` | Muestra los fixes que se aplicarían sin escribir nada. |
| `--filter <estado>` | Aplica sólo fixes de un estado específico (e.g. `--filter MOVED`). |
| `-y` | Omite la confirmación interactiva. |

## Flujo

1. Escanear los bilinks de la capa actual.
2. Para cada endpoint en un estado con fix, **re-resolverlo** con el mismo algoritmo que usa `check` (ver [`check`](check.md) § "Algoritmo de detección por tipo de endpoint"). Eso produce un estado re-derivado y la ubicación actual del fragmento.
3. Validar el estado re-derivado contra el cacheado (ver "Validación de frescura"). Si difieren, descartar ese endpoint.
4. Calcular la ubicación nueva. Si coincide con la que el `link` ya tiene, es un no-op y se omite.
5. Mostrar el resumen y pedir confirmación (o `-y`).
6. Para cada fix: acuñar el capture de la ubicación nueva —si no existía— y repuntar el `link`.
7. Commitear los archivos escritos.

**Mensaje de commit:**

```
bilinker: repuntar 3 endpoint(s) (2026-08-30)

- 7f3d8e9a… endpoint.1  MOVED       → specs/domain/voting.yaml
- 3a4b5c6d… endpoint.0  DISPLACED   → offset 8~45
- f1e2d3c4… endpoint.0  EXPANDED    → offset ampliado
```

## Cálculo del fix por estado

| Estado | Cómo se encuentra la ubicación nueva |
|---|---|
| **MOVED** | El índice de renames de git: `git diff -M --name-status`. Se verifica que la query resuelva en el destino. |
| **DISPLACED** | El texto aceptado aparece en otro offset del mismo nodo. |
| **EXPANDED** | El nodo contiene lo aceptado y algo más; el offset se amplía. |
| **REANCHORED** | La query relajada matchea un nodo con nombre distinto, por encima del umbral y con margen sobre el segundo. |

Los cuatro producen lo mismo: **un `(file, query, offset)` nuevo**, y con él un capture nuevo. Ver [`check`](check.md) para los criterios de detección.

## No hay fork, porque no hay mutación

Un capture es inmutable y su id es el hash de su ubicación, así que corregir una ubicación **siempre** produce un capture distinto. `apply` lo acuña y repunta un solo `link`: el del endpoint que está corrigiendo. Los demás referentes no se enteran, sin que haya que decidir nada.

Eso reemplaza el copy-on-write del formato anterior, donde `apply` mutaba el capture en el lugar salvo que estuviera compartido, y una tabla decidía por tipo de fix si forkear:

| Antes | Ahora |
|---|---|
| MOVED y EXPANDED mutaban en el lugar | todos acuñan |
| DISPLACED y REANCHORED forkeaban si había más de un referente | no hay caso especial |
| había que contar referentes antes de aplicar | no hace falta |

El capture viejo queda: sigue vivo mientras algún `accepted.link` lo nombre —es la ubicación que alguien aprobó— y lo limpia [`capture prune`](capture.md) cuando ya no lo alcanza nadie.

## Validación de frescura

El estado cacheado lo escribió el último `check`, y el archivo pudo cambiar después. Por eso `apply` nunca aplica un fix derivado de la cache: re-resuelve el endpoint y compara.

| Situación | Acción |
|---|---|
| Estado re-derivado == el cacheado | Aplicar el fix. |
| Estado re-derivado == `OK` | Omitir en silencio — el fix ya no hace falta. |
| Estado re-derivado distinto y no `OK` | Descartar el fix, refrescar la cache y advertir. |

```
warn: 3a4b5c6d… endpoint.0: la cache dice DISPLACED y la resolución actual da ALTERED
      — fix descartado. Correr `bilinker check`.
```

Un endpoint que pasó de tener fix a no tenerlo —DISPLACED → ALTERED— nunca se toca.

## Qué escribe

| Archivo | Qué |
|---|---|
| `capture/<id>.yaml` | El capture de la ubicación nueva, si no existía. |
| el bilink | `link` del endpoint corregido, y nada más. |
| `cache/state` | El estado re-derivado tras el fix: `RELOCATED`. |

**`accepted` no se toca nunca.** Es la invariante que separa proponer de aprobar, y con el bloque aparte es casi imposible de violar por accidente.

## Salida

```
$ bilinker apply

Pending fixes (3):
  MOVED      7f3d8e9a…  endpoint.1  → specs/domain/voting.yaml
  DISPLACED  3a4b5c6d…  endpoint.0  → offset 8~45
  EXPANDED   f1e2d3c4…  endpoint.0  → offset ampliado

Apply? [y/N] y

Repuntados 3 endpoint(s). Los 3 quedan en RELOCATED.
  Revisar con `bilinker get <uuid>.<N>` y aprobar con `bilinker accept --place`.
Committed: a4b5c6d "bilinker: repuntar 3 endpoint(s) (2026-08-30)"
```

Ningún fix cierra solo. La línea final dice qué falta, porque el trabajo no terminó cuando `apply` termina.

## Código de salida

| Código | Condición |
|---|---|
| 0 | Todos los fixes aplicados. |
| 1 | Error al calcular o aplicar algún fix, o algún fix descartado por cache desactualizada. |
| 2 | No hay endpoints con fix disponible. |

## Invariantes

- `apply` nunca escribe `accepted`. Corrige dónde está el fragmento, nunca qué se aprobó.
- `apply` nunca modifica un capture existente: acuña.
- El único efecto de `apply` sobre un bilink es repuntar un `link`.
- Tras `apply`, el endpoint queda en `RELOCATED`. Ningún fix lo devuelve a `OK`.
- `apply` nunca aplica un fix derivado de la cache: cada uno se recalcula re-resolviendo contra el árbol y el índice git actuales.
- Si el estado re-derivado no coincide con el cacheado, `apply` descarta el fix y avisa.
- Si el fix calculado no se puede verificar —el hash no coincide en el path nuevo, el anchor no aparece— `apply` lo rechaza y avisa.
- `apply` es idempotente: un fix ya aplicado se detecta como no-op.
- `ALTERED`, `UNRESOLVED` y `CHAIN_DIRTY` no tienen fix y `apply` no los toca.
