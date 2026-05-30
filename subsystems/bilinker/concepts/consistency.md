# Especificación: modelo de consistencia

## Principio fundamental

La consistencia de un bilink se evalúa por extremo. `check` retorna una tupla `(state_link0, state_link1)` — cada extremo puede estar en un estado diferente. Ejemplos: `(OK, MOVED)`, `(DISPLACED, ALTERED)`, `(OK, OK)`.

Los estados disponibles dependen del tipo de endpoint:
- **Endpoint estructural**: 10 estados (PENDING, OK, MOVED, DISPLACED, REANCHORED,
  EXPANDED, UNANCHORED, ALTERED, DELETED, BROKEN).
- **Endpoint layer**: 5 estados (TODO, PENDING, OK, CHAIN_DIRTY, BROKEN).
- **Endpoint task**: mismos estados que estructural (es un archivo en worklist).

Un bilink es "saludable" si ambos extremos están en {OK, MOVED, DISPLACED, REANCHORED, EXPANDED, TODO}. Requiere acción si alguno está en {PENDING, UNANCHORED, ALTERED, DELETED, BROKEN, CHAIN_DIRTY}.

## Estado TODO (endpoint layer)

Cuando `link.N` apunta a una layer que todavía no existe y `hash.N` está ausente, el estado es `TODO`. Indica una intención declarada — no es un error. Se resuelve creando la layer destino y ejecutando `bilinker accept`.

Si la layer existía (tiene `hash.N`) pero desapareció, el estado es `BROKEN` (regresión).

## Estado CHAIN_DIRTY (endpoint layer)

Cuando `link.N` apunta a una layer, `hash.N` contiene una **copia del `hash.N` del endpoint estructural del bilink adyacente** en el momento de la última aceptación. Si ese valor difiere del actual, el estado es CHAIN_DIRTY.

Esto evita dependencia circular: aceptar un endpoint layer nunca modifica el archivo adyacente, por lo que no hay cascadas. La propagación es unidireccional desde el endpoint estructural que cambió.

CHAIN_DIRTY no tiene auto-fix directo: se resuelve ejecutando `bilinker accept` en el endpoint layer.

## Persistencia de `state.N`

El estado de cada endpoint se guarda en el archivo `.bilink` como `state.N`. Esto tiene dos consecuencias:

1. **Propagación reactiva**: si `state.N` cambia, el archivo cambia, su hash cambia,
   y el nodo adyacente en la cadena detecta CHAIN_DIRTY en el próximo `check`.
2. **Estado visible sin re-check**: leer el archivo alcanza para saber si el bilink
   está saludable, sin re-ejecutar toda la verificación.

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

`hash.N` está ausente — el bilink fue creado pero aún no tiene ningún estado aceptado. Se sale con `bilinker accept`.

### OK

El hash SHA-256 del texto actual del fragmento (en el offset guardado, o en el nodo completo si no hay `start~end`) == `hash.N`. No se requiere acción.

### CHAIN_DIRTY *(solo endpoint layer)*

El `hash.N` del endpoint estructural del bilink adyacente ≠ `hash.N` almacenado. El contenido estructural en el otro extremo de la cadena cambió y fue re-aceptado. No hay auto-fix — requiere ejecutar `bilinker accept` en el endpoint layer.

### MOVED

Condiciones necesarias:
- El archivo en `link.N.file` no existe en su path conocido.
- `git diff -M --name-status HEAD` detecta rename con similaridad ≥ 50%.
- El hash del fragmento matchea en el nuevo path.

Auto-fix: actualizar `file` en `link.N` al nuevo path.

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
| `[commit <hash> "<msg>"]` | `git log --since=<resolved_at> -- <file>` |

### Algoritmo de intersección hunk / fragmento

```
fragmento: líneas F_start–F_end  (derivadas de range.N en bytes)
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
| BROKEN | `git log --since=<resolved_at> -- <file>` (sin resultado útil) |
| CHAIN_DIRTY | comparación del `hash.N` estructural del bilink adyacente |
