# Especificación: modelo de consistencia

## Principio fundamental

La consistencia de un bilink se evalúa por extremo. `check` retorna una tupla `(state_link0, state_link1)` — cada extremo puede estar en un estado diferente. Ejemplos: `(OK, MOVED)`, `(DISPLACED, ALTERED)`, `(OK, OK)`.

Los estados disponibles dependen del tipo de endpoint:
- **Endpoint estructural**: 10 estados (PENDING, OK, MOVED, DISPLACED, REANCHORED,
  EXPANDED, UNANCHORED, ALTERED, DELETED, BROKEN).
- **Endpoint layer**: 5 estados (TODO, PENDING, OK, CHAIN_DIRTY, BROKEN).
- **Endpoint issue**: mismos estados que estructural (es un archivo en worklist).

Un bilink es "saludable" si ambos extremos están en {OK, MOVED, DISPLACED, REANCHORED, EXPANDED, TODO}. Requiere acción si alguno está en {PENDING, UNANCHORED, ALTERED, DELETED, BROKEN, CHAIN_DIRTY}.

## Estado TODO (endpoint `path`)

Cuando un `link` apunta a una capa que todavía no existe y `accepted` está ausente, el estado es `TODO`. Indica una intención declarada — no es un error. Se resuelve creando la layer destino y ejecutando `bilinker accept`.

Si la capa existía —el endpoint tiene `accepted`— pero desapareció, el estado es `BROKEN` (regresión).

## Estado CHAIN_DIRTY (endpoint `path`)

Cuando un `link` apunta a una capa, su `accepted` contiene una **copia del `accepted` del endpoint estructural del bilink adyacente** —su `link` y su `hash`— al momento de la última aceptación. Si alguno difiere del actual, el estado es CHAIN_DIRTY.

Esto evita dependencia circular: aceptar un endpoint `path` nunca modifica el archivo adyacente, por lo que no hay cascadas. La propagación es unidireccional desde el endpoint estructural que cambió.

CHAIN_DIRTY no tiene auto-fix directo: se resuelve ejecutando `bilinker accept` en el endpoint layer.

## Dónde vive `state.N`

En [la cache](cache.md), no en el bilink. Es un derivado: `check` lo recalcula resolviendo la query y comparando contra `accepted`.

Que esté afuera del archivo versionado tiene dos consecuencias:

1. **`check` no ensucia el repo.** Verificar deja de producir un diff, que era el defecto (b) de [ADR-0003](../.stratum/impl/docs/adr/0003-formato-captures-y-aceptacion.md).
2. **No propaga.** Refrescar la cache no cambia ningún valor aceptado, así que el vecino no ve nada. La cadena la mueve `accept`, que es quien escribe una decisión.

Con cache fría el estado **no está disponible** y hay que correr `check`. Es distinto de `commit`, que con cache fría sólo cuesta más — ver [la cache](cache.md) § "Estar fría es normal".

## Jerarquía de evidencia (endpoints estructurales)

Para determinar el estado, bilinker aplica hipótesis en orden de especificidad:

```
1. Hash exacto en offset guardado          → OK
2. Archivo movido (git rename)             → MOVED
3. Hash exacto en otro offset del nodo    → DISPLACED
4. Anchor renombrado/movido (AST search)  → REANCHORED
5. Fragmento creció, AST interno igual    → EXPANDED
6. Fragmento encontrado, AST interno ≠    → ALTERED
7. Borrado determinístico (git log -S)    → DELETED
8. Sin evidencia                          → BROKEN / UNANCHORED
```

## Definición detallada de cada estado

### PENDING

`accepted` está ausente — el bilink fue creado pero nadie aprobó nada todavía. La ausencia del bloque *es* el estado. Se sale con `bilinker accept`.

### OK

Las dos dimensiones coinciden: `link` == `accepted.link` y el hash del fragmento actual == `accepted.hash`. No se requiere acción.

### RELOCATED

`link` ≠ `accepted.link`: el endpoint apunta a una ubicación que nadie aprobó. Llega por `bilinker apply`, que repunta pero no bendice. Se sale con `bilinker accept --place`, después de mirar que el fragmento nuevo sea el que corresponde.

### CHAIN_DIRTY *(sólo endpoint `path`)*

El `accepted` del endpoint estructural del bilink adyacente ≠ la copia guardada. El otro extremo de la cadena cambió y fue re-aceptado. No hay fix automático — hay que ejecutar `bilinker accept`.

### MOVED

Condiciones necesarias:
- El archivo del capture no existe en su path conocido.
- `git diff -M --name-status HEAD` detecta rename con similaridad ≥ 50%.
- El hash del fragmento matchea en el nuevo path.

Fix: `apply` acuña el capture del path nuevo y repunta el `link`. El endpoint queda en `RELOCATED`.

### DISPLACED

Condiciones necesarias:
- La query matchea el nodo target en el archivo actual.
- El hash NO matchea en el offset guardado.
- El hash SÍ matchea en algún otro offset dentro del nodo.

Causa típica: texto insertado dentro del mismo nodo antes de la selección.

Auto-fix: actualizar `start~end` al nuevo offset donde se encontró el hash.

### REANCHORED

Condiciones necesarias:
- La query no matchea ningún nodo en el archivo actual.
- Buscando en el AST actual con el mismo tipo de nodo anchor, se encuentra un
  nodo con nombre diferente que contiene el hash del fragmento.

Ejemplo: `vote` renombrado a `castVote`.

Auto-fix: actualizar los predicados de la query con el nuevo nombre.

### EXPANDED

Condiciones necesarias:
- La query matchea el nodo target.
- El hash no matchea en ningún offset del nodo.
- El texto del fragmento guardado ES subcadena del texto actual del nodo.
- El AST interno del fragmento guardado y del texto actual son estructuralmente iguales.

Causa típica: se añadieron líneas al método sin cambiar la parte seleccionada.

Auto-fix: ampliar `start~end` para cubrir el nuevo rango.

### UNANCHORED

Condiciones necesarias:
- La query no matchea ningún nodo.
- No se detecta el anchor en otra posición del AST.
- No hay evidencia de borrado determinístico en git.

Requiere intervención humana.

### ALTERED

Condiciones necesarias:
- La query matchea el nodo target.
- El hash no matchea en ningún offset del nodo.
- El texto del fragmento guardado ES subcadena del texto actual del nodo.
- El AST interno cambió estructuralmente.

El fragmento fue encontrado pero su semántica cambió. Requiere intervención humana.

### DELETED

Condiciones necesarias:
- El contenido fue eliminado.
- `git log -S "<texto_fragmento>" -- <file>` localiza de forma determinística el
  commit que lo eliminó.

Requiere intervención humana.

### BROKEN

- Ninguna hipótesis anterior aplica.
- Estado residual cuando toda evidencia falla.

Requiere intervención humana.

## Fuente del cambio

Para estados no-OK en endpoints estructurales, bilinker reporta el origen:

| Fuente | Detección |
|---|---|
| `[UNSTAGED]` | `git diff -- <file>` tiene hunks solapando el fragmento |
| `[STAGED]` | `git diff --cached -- <file>` tiene hunks solapando el fragmento |
| `[commit <hash> "<msg>"]` | `git log <commit>..HEAD -- <file>`, con el `commit` del endpoint |

**El baseline es `commit`, no un timestamp.** `commit` es el commit en que el fragmento quedó con el contenido aceptado, así que `commit..HEAD` deja la ventana exactamente en lo que pasó después. Un timestamp sería un baseline peor: las fechas se desordenan con rebases, cherry-picks y skew de reloj, mientras que `git log` recorre por ancestría y es exacto. Por eso `resolved_at` desapareció del formato — ver [la cache](cache.md) § "Por qué `resolved_at` no está".

`commit` vive en [la cache](cache.md) y con cache fría se re-deriva. Como esta atribución sólo corre sobre endpoints ya no-OK, el costo está acotado por lo que está roto.

### Algoritmo de intersección hunk / fragmento

```
fragmento: líneas F_start–F_end  (derivadas del range del capture, en bytes)
hunk del diff: @@ -H_start,H_count +...

H_start + H_count < F_start  → BEFORE  (posible causa de DISPLACED)
H_start > F_end               → AFTER   (irrelevante)
se superpone con [F_start, F_end]  → WITHIN  (causa de EXPANDED, ALTERED, etc.)
```

## Relación entre estados y git

| Estado | Comando git principal |
|---|---|
| MOVED | `git diff -M --name-status HEAD` |
| DISPLACED | `git diff -U0 -- <file>` (intersección hunk) |
| REANCHORED | `git diff -- <file>` + búsqueda en AST actual |
| EXPANDED | `git diff -U0 -- <file>` + comparación de AST |
| ALTERED | `git log -S "<hash>" -- <file>` |
| DELETED | `git log -S "<hash_o_texto>" -- <file>` |
| BROKEN | `git log <commit>..HEAD -- <file>` (sin resultado útil) |
| CHAIN_DIRTY | comparación del `accepted` estructural del bilink adyacente |
