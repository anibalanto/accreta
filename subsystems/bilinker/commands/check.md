# Especificación: comando `bilinker check`

## Propósito

Verifica la consistencia de uno o más bilinks. Opera en dos pasos: **resuelve los captures** referenciados —localizando cada fragmento en el árbol actual— y luego **compara** el contenido hallado contra `hash.N`. Escribe `state`, `range` y `resolved_at` en cada capture, y `state.N` en cada bilink.

Requiere git como dependencia dura. Opera completamente offline — solo git y tree-sitter, sin language servers ni indexers externos.

## Firma

```
bilinker check [<path>]
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `path` | path | Path a un `.bilink` individual, o a una layer (directorio que contiene `.bilink/`). Default: layer actual (cwd). |

## Estados — resolución (en el capture)

Se evalúan sin ningún estado aceptado: son sobre dónde está el fragmento.

| Estado | Condición | Auto-fix |
|---|---|---|
| **RESOLVED** | La query matchea; `range` actualizado. | — |
| **MOVED** | Archivo cambió de path (git rename ≥ 50%). | ✓ Actualiza `file` del capture. |
| **REANCHORED** | Anchor renombrado/movido; nueva posición detectada en AST. | ✓ Actualiza los predicados de `query`. |
| **UNANCHORED** | Query no matchea; anchor no localizado. | — Requiere intervención. |
| **DELETED** | Eliminación rastreable en git con `git log -S`. | — Requiere intervención. |
| **BROKEN** | Ninguna hipótesis aplica. | — Requiere intervención. |

## Estados — aceptación (en el bilink, endpoints estructurales)

Comparan el contenido hallado contra `hash.N`.

| Estado | Condición | Auto-fix |
|---|---|---|
| **OK** | Hash matchea en el `range` del capture. | — |
| **DISPLACED** | Hash en otro offset dentro del nodo. | ✓ Actualiza `offset` del capture. |
| **EXPANDED** | Fragmento creció; AST interno sin cambio estructural. | ✓ Amplía `offset` del capture. — *ver nota* |
| **RESTYLED** | `hash.N` difiere pero `hash_ast.N` coincide — solo formato, AST idéntico. | — Advertencia leve; ejecutar `bilinker accept`. |
| **ALTERED** | Fragmento encontrado; AST interno cambió estructuralmente. | — Requiere intervención. |
| **UNRESOLVED** | El capture referenciado no resolvió. | — Se resuelve en el capture. |

`DISPLACED` y `EXPANDED` se detectan acá porque necesitan `hash.N`, pero su fix se escribe en el capture. Ver [capture.md](../concepts/capture.md) § "Copy-on-write al aplicar un fix".

> **`EXPANDED` no está implementado, y tal como está definido no es detectable.**
>
> El paso 5 del algoritmo pregunta *"¿el texto guardado es subcadena del nodo?"*, pero el formato guarda `hash.N` —un hash— y no el texto. Se puede buscar el hash dentro del nodo con el largo del fragmento guardado, pero encontrarlo dice que el fragmento **se movió**, que es `DISPLACED`. No dice que ahora deba **abarcar más**, que es la afirmación de `EXPANDED`.
>
> Distinguirlos requiere una decisión: definir `EXPANDED` de forma estructural —el nodo de la query creció y el fragmento guardado está contenido en él— asumiendo que la frontera con `DISPLACED` es heurística; o eliminar el estado y dejar que esos casos sean `DISPLACED` seguido de `accept`.
>
> Mientras tanto `check` nunca lo devuelve, y la rama de `apply` que lo trata es código inalcanzable.

## Estados — endpoints layer (2 estados adicionales)

| Estado | Condición | Auto-fix |
|---|---|---|
| **PENDING** | `hash.N` ausente — sin estado aceptado. | — Ejecutar `bilinker accept`. |
| **CHAIN_DIRTY** | Hash actual del `.bilink` referenciado ≠ `hash.N`. | — Inspeccionar nodo origen. |

## Optimización por diff de git

Antes de parsear o hashear un archivo, `check` determina si tiene cambios desde la última aceptación:

```
git diff --name-only <commit.N> -- <file>
```

Sin `..HEAD`: la comparación es contra el **árbol de trabajo**, no contra HEAD. Con `..HEAD` se comparan dos commits y los cambios sin commitear quedan invisibles — que es el caso más común mientras alguien trabaja, y el fast-path devolvería un estado cacheado obsoleto.

Si el output está vacío → el archivo no cambió desde `commit.N` → el `state.N` cacheado sigue siendo válido; se omite el resto del algoritmo para ese endpoint.

Si git falla —commit inexistente, repo sin historial— se asume **cambiado**. No poder comparar no es evidencia de que nada cambió.

Si `commit.N` está ausente (endpoint nunca aceptado) → no se puede optimizar; se ejecuta el algoritmo completo.

## Algoritmo de detección por tipo de endpoint

### Endpoint estructural

Los pasos 1–2 resuelven el **capture**; los pasos 3–5 comparan contra `hash.N` del **bilink**.

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

   (los pasos 1–2 escriben state del capture; si no es RESOLVED,
    todos los bilinks que lo referencian quedan UNRESOLVED y se corta acá)

3. ¿Hash matchea en el range del capture?  → OK
4. ¿Hash matchea en otro offset del nodo?  → DISPLACED
5. ¿Texto guardado es subcadena del nodo?
   SÍ → comparar AST interno:
        igual → EXPANDED
        distinto → ALTERED
   NO → ¿hash_ast.N presente y hash_ast actual coincide?
        SÍ → RESTYLED  (solo cambio de formato; AST idéntico)
        NO → ALTERED
```

Un mismo capture se resuelve **una sola vez por `check`**, aunque lo referencien varios bilinks. Los pasos 3–5 sí corren por endpoint, porque cada uno tiene su propio `hash.N`.

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

`check` escribe en dos archivos distintos.

**En el capture**, tras resolverlo:

- **`range`** — byte range absoluto del fragmento, siempre que la resolución lo encuentre.
- **`state`** — estado de resolución.
- **`resolved_at`** — timestamp UTC.
- **`file`, `query`, `offset`** — no se modifican. Solo los cambia `bilinker apply`.

**En el bilink**, tras comparar:

- **`state.N`** — nuevo estado calculado.
- **`resolved_at`** — timestamp UTC.
- **`hash.N` / `hash_ast.N` / `commit.N`** — no se modifican. Solo los establece `bilinker accept`.

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
fragmento: líneas F_start–F_end  (derivadas del range del capture, en bytes)
hunk:      @@ -H_start,H_count +...

H_start + H_count < F_start  → BEFORE  (posible causa de DISPLACED)
H_start > F_end               → AFTER   (irrelevante)
se superpone                  → WITHIN  (causa de EXPANDED, ALTERED, REANCHORED)
```

## Auto-fix

Los estados con auto-fix (MOVED, DISPLACED, REANCHORED, EXPANDED) se resuelven con `bilinker apply`, que usa `state.N` para seleccionar candidatos y recalcula cada fix re-resolviendo el endpoint contra git y el AST actuales. Nunca se aplican automáticamente.

`apply` no lee el `range` cacheado como fuente del fix: refleja el último `check` y el archivo puede haber cambiado desde entonces. Si la re-resolución arroja un estado distinto del cacheado, `apply` descarta el fix y pide correr `check`.

## Salida

Bilinks OK se omiten por defecto. Con `--verbose` se muestran todos. Para cadenas, se recomienda usar `bilinker chain status <uuid>`.

```
$ bilinker check .

7f3d8e9a  (OK, CHAIN_DIRTY)
  link.1  → .stratum/impl  archivo cambió
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
| 0 | Todos los captures en {RESOLVED, MOVED, REANCHORED} y todos los extremos en {OK, DISPLACED, EXPANDED, RESTYLED}. |
| 1 | Algún capture en {UNANCHORED, DELETED, BROKEN} o algún extremo en {ALTERED, UNRESOLVED, CHAIN_DIRTY}. |
