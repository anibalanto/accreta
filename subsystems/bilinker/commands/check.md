# Especificación: comando `bilinker check`

## Propósito

Verifica la consistencia de uno o más bilinks comparando el estado actual de los archivos referenciados contra la cache del `.bilink`. Por cada bilink devuelve una tupla de estados `(state_link0, state_link1)` y actualiza `state.N` en el archivo.

Requiere git como dependencia dura.

## Firma

```
bilinker check <path>
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `path` | path | Path a un `.bilink` individual, a una carpeta `.bilink/`, o a una layer (procesa recursivamente todos los `.bilink` dentro). |

## Estados — endpoints estructurales (9 estados)

| Estado | Condición | Auto-fix |
|---|---|---|
| **OK** | Hash matchea en el offset guardado. | — |
| **MOVED** | Archivo cambió de path (git rename ≥ 50%); hash matchea en nuevo path. | ✓ Actualiza `file` en `link.N`. |
| **DISPLACED** | Query matchea; hash en offset diferente dentro del nodo. | ✓ Actualiza `start~end`. |
| **REANCHORED** | Anchor renombrado/movido; nueva posición detectada en AST. | ✓ Actualiza predicados de query. |
| **EXPANDED** | Fragmento creció; AST interno sin cambio estructural. | ✓ Amplía `start~end`. |
| **UNANCHORED** | Query no matchea; anchor no localizado. | — Requiere intervención. |
| **ALTERED** | Fragmento encontrado; AST interno cambió estructuralmente. | — Requiere intervención. |
| **DELETED** | Eliminación rastreable en git con `git log -S`. | — Requiere intervención. |
| **BROKEN** | Ninguna hipótesis aplica. | — Requiere intervención. |

## Estados — endpoints layer (2 estados adicionales)

| Estado | Condición | Auto-fix |
|---|---|---|
| **PENDING** | `hash.N` ausente — sin estado aceptado. | — Ejecutar `bilinker accept`. |
| **CHAIN_DIRTY** | Hash actual del `.bilink` referenciado ≠ `hash.N`. | — Inspeccionar nodo origen. |

## Algoritmo de detección por tipo de endpoint

### Endpoint estructural

```
1. ¿El archivo existe en el path conocido?
   NO → git diff -M --name-status HEAD
        ¿rename ≥ 50%?
        SÍ → MOVED
        NO → git log -S "<hash_fragmento>" -- <file>
             SÍ → DELETED
             NO → BROKEN

2. Ejecutar query tree-sitter.
   SIN MATCH → buscar anchor en AST actual (mismo tipo de nodo):
               ¿anchor encontrado con nombre diferente?
               SÍ → REANCHORED
               NO → git log -S "<texto_anchor>" -- <file>
                    SÍ → DELETED
                    NO → UNANCHORED

3. ¿Hash matchea en offset guardado?  → OK
4. ¿Hash matchea en otro offset del nodo?  → DISPLACED
5. ¿Texto guardado es subcadena del nodo?
   SÍ → comparar AST interno:
        igual → EXPANDED
        distinto → ALTERED
   NO → git log -S "<hash>" -- <file>
        SÍ → DELETED
        NO → BROKEN
```

### Endpoint layer

```
1. ¿hash.N ausente?  → PENDING
2. Resolver path: ../<link.N>/.bilink/<uuid>.bilink
3. ¿El archivo existe?
   NO → BROKEN (nodo de la cadena eliminado)
4. Calcular SHA-256 del archivo completo.
   == hash.N → OK
   ≠  hash.N → CHAIN_DIRTY
```

## Escritura de cache tras resolución

Después de evaluar cada endpoint, `check` actualiza el archivo `.bilink`:

- **`state.N`** — nuevo estado calculado.
- **`hash.N` / `commit.N`** — no se modifican. Solo se actualizan con `bilinker accept`.
- **`range.N`** — byte range absoluto del fragmento en su archivo (`start~end`). Solo para endpoints estructurales; se actualiza siempre que la resolución encuentra el fragmento (OK, DISPLACED, REANCHORED, EXPANDED, ALTERED).
- **`resolved_at`** — timestamp UTC de esta verificación.

Si `state.N` cambió respecto al valor anterior, el archivo cambia y su hash cambia, disparando CHAIN_DIRTY en el nodo adyacente de la cadena en el próximo `check`.

## Fuente del cambio

Para endpoints estructurales no-OK:

| Condición git | Fuente en salida |
|---|---|
| `git diff -- <file>` tiene hunks solapando el fragmento | `[UNSTAGED]` |
| `git diff --cached -- <file>` tiene hunks solapando el fragmento | `[STAGED]` |
| `git log --since=<resolved_at> -- <file>` tiene commits | `[commit <hash> "<msg>"]` |

### Intersección hunk / fragmento

```
fragmento: líneas F_start–F_end  (derivadas de range.N en bytes)
hunk:      @@ -H_start,H_count +...

H_start + H_count < F_start  → BEFORE  (posible causa de DISPLACED)
H_start > F_end               → AFTER   (irrelevante)
se superpone                  → WITHIN  (causa de EXPANDED, ALTERED, REANCHORED)
```

## Auto-fix staging

Los estados con auto-fix (MOVED, DISPLACED, REANCHORED, EXPANDED) generan un archivo en `.bilink/.pending/<uuid>-<N>.fix`. Nunca se aplican automáticamente.

## Salida

Bilinks OK se omiten por defecto. Con `--verbose` se muestran todos. Para cadenas, se recomienda usar `bilinker chain status <uuid>`.

```
$ bilinker check .bilink/

7f3d8e9a  (OK, CHAIN_DIRTY)
  link.1  → .stratum/tech-decisions  archivo cambió
  → inspeccionar: bilinker chain status 7f3d8e9a-...

3a4b5c6d  (DISPLACED, ALTERED)
  link.0  specs::voting.yaml#impl  offset 5~42 → 8~45  [UNSTAGED]
  → fix disponible: bilinker apply
  link.1  java-demo::Persona#vote  AST interno cambió
    - Comparator.comparingInt(String::length)
    + (a, b) -> a.length() - b.length()
    source: commit c7d3e9f "Inline comparator" (2026-05-19)

f1e2d3c4  (EXPANDED, OK)
  link.0  specs::reporter.yaml#generate  fragmento creció — AST sin cambios
    + log.info("called");  [commit a3f2b1c "Add audit log"]
  → fix disponible: bilinker apply
```

## Código de salida

| Código | Condición |
|---|---|
| 0 | Todos los extremos en {OK, MOVED, DISPLACED, REANCHORED, EXPANDED}. |
| 1 | Al menos un extremo en {UNANCHORED, ALTERED, DELETED, BROKEN, CHAIN_DIRTY}. |
