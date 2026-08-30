# Especificación: comando `bilinker check`

## Propósito

Verifica la consistencia de uno o más bilinks y **no escribe ni un byte en git**.

Opera en dos pasos: resuelve los captures referenciados —localizando cada fragmento en el árbol actual— y compara lo hallado contra `accepted`, en sus **dos dimensiones**: dónde está y qué dice. El resultado va a [`cache/state`](../concepts/cache.md).

Requiere git como dependencia dura. Opera completamente offline — sólo git y tree-sitter, sin language servers ni indexers.

## Firma

```
bilinker check [<path>] [--against <ref>]
```

| Argumento | Descripción |
|---|---|
| `path` | Path a un bilink individual, o a una capa. Default: la capa actual. |
| `--against <ref>` | Toma los `accepted` de otro lado en vez de los del árbol, y **no escribe cache**. |

### `--against`

Compara el árbol actual contra las aceptaciones de otra parte —otra rama, otro commit— sin tocar nada. Sirve para preguntar *"¿qué endpoints quedarían no-OK si mergeo esto?"* antes de mergearlo.

No escribe cache **a propósito**: la cache describe el estado del árbol contra sus propias aceptaciones, y sobrescribirla con el resultado de una comparación hipotética la volvería mentirosa.

Lo que `--against` **no** puede hacer es cruzar versiones de formato: linkea un solo parser. Comparar dos formatos es trabajo de una migración, que depende de los dos.

## Las dos dimensiones

Un endpoint puede desalinearse de dos formas, y `check` las distingue porque se aprueban por separado:

| | Se compara | Da |
|---|---|---|
| **Ubicación** | `link` contra `accepted.link` | `RELOCATED` |
| **Contenido** | el hash del fragmento contra `accepted.hash` | `ALTERED`, `RESTYLED`, … |

La de ubicación es una comparación de dos ids: no hace falta abrir ningún archivo. Por eso sobrevive donde la otra no —cruzando la frontera, con un clon superficial— y por eso se evalúa primero.

## Estados — resolución del capture

Sobre dónde está el fragmento. Se evalúan sin ninguna aceptación.

| Estado | Condición | Fix |
|---|---|---|
| **RESOLVED** | La query matchea. | — |
| **MOVED** | El archivo cambió de path (git rename ≥ 50%). | `apply` |
| **REANCHORED** | Anchor renombrado; el fragmento se localizó bajo otro nombre por similitud. | `apply` |
| **UNANCHORED** | La query no matchea y el anchor no se localiza. | `recapture` |
| **DELETED** | Eliminación rastreable con `git log -S`. | intervención |
| **BROKEN** | Ninguna hipótesis aplica. | intervención |

## Estados — aceptación

Comparan lo hallado contra `accepted`.

| Estado | Condición | Fix |
|---|---|---|
| **PENDING** | `accepted` ausente. | `accept` |
| **OK** | La ubicación y el contenido coinciden con lo aceptado. | — |
| **RELOCATED** | `link` ≠ `accepted.link`. | `accept --place` |
| **DISPLACED** | El texto aceptado está en otro offset del nodo. | `apply` + `accept` |
| **EXPANDED** | El fragmento contiene lo aceptado verbatim y algo más. | `apply` + `accept` |
| **RESTYLED** | El texto difiere pero el AST coincide — sólo formato. Sólo donde el AST discrimina contenido. | `accept` |
| **ALTERED** | El fragmento cambió estructuralmente. | revisar + `accept` |
| **UNRESOLVED** | El capture referenciado no resolvió. | se resuelve en el capture |

`DISPLACED` y `EXPANDED` se detectan acá porque necesitan el texto aceptado, pero lo que producen es una **ubicación nueva**, y eso lo escribe `apply` acuñando un capture.

### `RESTYLED` sólo existe donde el AST discrimina contenido

En prosa el AST no lleva el texto: el s-expression de una sección markdown es el mismo con cualquier párrafo adentro. Comparar ahí diría "sólo formato" de una reescritura entera, que es exactamente el estado que invita a aceptar sin leer.

Así que la pregunta la decide **la gramática, no el archivo**: en markdown y texto plano `accept` no escribe `hash_ast` y `check` no lo compara. Los dos consultan la gramática antes que `accepted`, así que un `hash_ast` guardado por una versión anterior queda inerte en vez de mentir — y `accept` tampoco lo arrastra hacia adelante.

La lista de lenguajes donde el AST discrimina es la de [`capture`](capture.md) § "Lenguajes soportados", menos markdown.

### Y `hash_ast` cubre los tokens, no sólo la forma del árbol

El s-expression de tree-sitter es la forma del árbol: dice `(identifier)`, no *qué* identificador. Hashear eso solo hace invisible todo renombre y todo literal — `("0.1.0", "21e2…")` y `("2.0.0", "3939…")` tienen el mismo árbol, y el estado saldría RESTYLED de un cambio de versión.

`hash_ast` es entonces **la forma del árbol más el texto de cada token hoja**. Dos fragmentos coinciden cuando tienen los mismos tokens en el mismo orden y la misma estructura; lo único que puede diferir es el espacio entre ellos, que es lo que "sólo formato" quiere decir.

Un comentario es un token, así que cambiarlo no es RESTYLED. Es lo correcto: un comentario dice algo, y cambiar lo que dice es un cambio de contenido.

**Un `hash_ast` calculado con la definición anterior simplemente no coincide**, y el endpoint sale ALTERED — que pide revisión, que es la respuesta segura. No hace falta migrar nada.

### La frontera entre EXPANDED y DISPLACED

Ambos se distinguen con un test de subcadena contra el **texto aceptado**, no con un umbral. Siendo `T` el texto aceptado y `F` el fragmento que el capture resuelve hoy:

| Condición | Estado |
|---|---|
| `F == T` | OK |
| `F ⊃ T` — contiene lo aceptado y algo más | **EXPANDED** |
| `F ⊅ T`, pero el nodo contiene `T` en otro offset | **DISPLACED** |
| `T` no aparece y `accepted.hash_ast` coincide | RESTYLED |
| nada de lo anterior | ALTERED |

Sin solapamiento y sin heurística: EXPANDED es *"creció alrededor de lo aceptado"*, DISPLACED es *"se corrió y sigue igual"*.

Que `F` contenga a `T` verbatim implica que nada dentro de lo aceptado cambió, así que la condición de "AST interno sin cambio estructural" se satisface sola.

#### Cuando git no entrega el texto aceptado

La tabla necesita `T`, y `T` sale de git: `accepted.commit` más el path del capture. Eso puede no estar —un endpoint sin commit cacheado, un archivo que en ese commit no existía, una query que ahí no resuelve—, y entonces la comparación por subcadena no se puede hacer.

El respaldo es buscar el **hash** aceptado en cada offset del nodo, del tamaño del fragmento aceptado. Encuentra el mismo caso que la tabla llama DISPLACED, sin necesitar el texto: si el fragmento sigue ahí íntegro pero corrido, su hash aparece en algún offset. Lo que el respaldo no puede distinguir es EXPANDED, que exige comparar contenidos y no hashes; ese caso cae en ALTERED, que pide revisión.

### Estados propios de un endpoint `path`

| Estado | Condición | Fix |
|---|---|---|
| **TODO** | `accepted` ausente y la capa apuntada no existe todavía. | crear la capa + `accept` |
| **CHAIN_DIRTY** | Los valores copiados ≠ los `accepted` del vecino. | `accept` |
| **BROKEN** | La capa ya no existe, o el vecino no tiene endpoint estructural aceptado. | restaurar + `accept` · o · `remove` |

### Por qué REANCHORED usa similitud y no el hash

`accepted.hash` es exacto, y el nombre del anchor está **dentro** del fragmento capturado en la enorme mayoría de los casos — en este proyecto, en todos los captures que existen. Renombrar el anchor cambia el fragmento, así que una comparación por hash no dispararía nunca: detectaría solo el caso raro en que lo renombrado queda fuera de lo capturado.

El texto aceptado se recupera de git y se compara contra cada candidato. Ver "Recuperar el texto aceptado".

**Umbral: 50%**, el mismo que usa `git diff -M` para renames de archivos. La pregunta es la misma —a dónde se fue algo que cambió de nombre— y usar dos criterios distintos para la misma pregunta sería arbitrario.

**Margen sobre el segundo candidato: 15%.** Un archivo con varias funciones de forma parecida produciría un REANCHORED arbitrario. Ante un empate el estado es `UNANCHORED`: que lo mire un humano es mejor que reanclar al nodo equivocado.

La medida es el coeficiente de Dice sobre líneas, con bigramas de caracteres como respaldo para fragmentos de una sola línea, donde las líneas no discriminan nada.

### La incertidumbre está acotada

Introducir una medida difusa en un sistema construido sobre hashes exactos necesita un límite claro, y lo tiene: **`REANCHORED` nunca cierra solo**. `apply` corrige la ubicación pero el endpoint queda no-OK hasta que un humano ejecute `accept`. La similitud sirve para *encontrar* el fragmento, nunca para afirmar que su contenido sigue siendo válido — eso lo sigue decidiendo un hash exacto.

## Recuperar el texto aceptado

Varias detecciones —EXPANDED, DISPLACED y REANCHORED— necesitan el texto del fragmento **tal como quedó aceptado**, no solo su hash. Se recupera de git:

```
git show <commit>:<file>   →  contenido en el momento de aceptar
ejecutar la query sobre él   →  el nodo
aplicar offset               →  el fragmento aceptado
verificar sha256 == accepted.hash   →  o descartar
```

**No se recorta por el `range` cacheado.** `check` lo reescribe en cada corrida, así que apunta a dónde está el fragmento *ahora*; recortar contenido viejo con una posición nueva da bytes arbitrarios. Resolver la query contra el contenido viejo es lo correcto, y además se autoverifica.

Si la verificación falla —no hay `commit`, el archivo no existía en ese commit, la query no resuelve ahí— el texto se descarta y esas detecciones no corren. Es preferible no detectar nada que razonar sobre el texto equivocado; queda el respaldo por hash para DISPLACED.

## Optimización por diff de git

Antes de parsear o hashear un archivo, `check` determina si tiene cambios desde la última aceptación:

```
git diff --name-only <commit> -- <file>
```

Sin `..HEAD`: la comparación es contra el **árbol de trabajo**, no contra HEAD. Con `..HEAD` se comparan dos commits y los cambios sin commitear quedan invisibles — que es el caso más común mientras alguien trabaja, y el fast-path devolvería un estado cacheado obsoleto.

Si el output está vacío → el archivo no cambió desde `commit`. Eso alcanza para conservar un `state.N` de **OK**, y nada más.

Cualquier `state.N` no-OK cacheado se recalcula. La cache la escribe `check` leyendo el **árbol de trabajo**, no el commit, así que un estado no-OK pudo haberse calculado sobre una edición que después se revirtió: el archivo vuelve a coincidir con `commit`, el diff sale vacío, y el fast-path conservaría un estado que describe un contenido que ya no está. Recalcular lo no-OK cuesta proporcionalmente a lo que está roto, que siempre es poco.

Si git falla —commit inexistente, repo sin historial— se asume **cambiado**. No poder comparar no es evidencia de que nada cambió.

## Algoritmo de detección por tipo de endpoint

### Endpoint estructural

Los pasos 1–2 resuelven el **capture**; los pasos 3–9 comparan contra `accepted`.

```
1. ¿El archivo existe en el path conocido?
   NO → git diff -M --name-status HEAD
        ¿rename ≥ 50%?
        SÍ → MOVED
        NO → git log -S "<accepted.hash>" -- <file>
             SÍ → DELETED
             NO → BROKEN

2. Ejecutar query tree-sitter.
   SIN MATCH → relajar la query (quitar los predicados #eq?) y puntuar
               cada candidato por similitud contra el texto aceptado:
               ¿el mejor supera el umbral y le saca margen al segundo?
               SÍ → REANCHORED
               NO → git log -S "<accepted.hash>" -- <file>
                    SÍ → DELETED
                    NO → UNANCHORED

   (los pasos 1–2 son del capture; si no resuelve, todos los endpoints
    que lo referencian quedan UNRESOLVED y se corta acá)

3. ¿accepted ausente?  → PENDING
4. ¿link ≠ accepted.link?  → RELOCATED     ← ubicación: dos ids,
                                              sin abrir ningún archivo
5. ¿Hash matchea en el range?  → OK
6. Recuperar el texto aceptado T (ver "Recuperar el texto aceptado").
   ¿F contiene T verbatim y es más grande?  → EXPANDED
7. ¿accepted.hash_ast presente y el hash_ast actual coincide?
   SÍ → RESTYLED  (sólo formato; AST idéntico)
8. ¿T aparece en otro offset del nodo?  → DISPLACED
9. → ALTERED
```

**El paso 4 va antes que el 5 y no cuesta nada.** Comparar dos ids no abre ningún archivo, así que la dimensión de ubicación se decide siempre — incluso cruzando la frontera, donde el clon superficial no permite recuperar el texto aceptado y la de contenido degrada a `ALTERED`.

Un mismo capture se resuelve **una sola vez por `check`**, aunque lo referencien varios endpoints. Los pasos 3–9 sí corren por endpoint, porque cada uno tiene su propio `accepted`.

### Endpoint `path`

```
1. Resolver path: ../<stratum-path>/.bilink/<uuid>.yaml
2. ¿La capa o el archivo no existen?
   accepted ausente  → TODO   (la capa todavía no existe)
   accepted presente → BROKEN (nodo de la cadena eliminado)
3. Leer el `accepted` del endpoint **estructural** del bilink adyacente.
   ausente → PENDING (el otro extremo nunca se aceptó)
4. ¿accepted propio ausente? → PENDING
5. Comparar las dos copias guardadas contra las del vecino.
   las dos coinciden → OK
   alguna difiere    → CHAIN_DIRTY
```

Se comparan **dos** valores, no uno: `accepted.link` y `accepted.hash`. Cada uno cambia por una sola razón —la ubicación aprobada del vecino, o su contenido aprobado— y los dos son inmunes a etiquetas, comentarios y reordenamientos de su archivo.

## Escritura de cache

`check` escribe en **un solo archivo**, `.bilink/cache/state`:

- **`range`** — byte range absoluto del fragmento, cuando la resolución lo encuentra.
- **`state`** — estado de resolución del capture.
- **`state.N`** — estado de aceptación, por endpoint.

Y no escribe nada más. **Ni el bilink ni el capture se tocan.**

Ésa es la diferencia más visible con el comportamiento anterior, donde `check` escribía `state.N` y `resolved_at` en archivos versionados: verificar producía un diff. Al escribir [ADR-0003](../.stratum/impl/docs/adr/0003-formato-captures-y-aceptacion.md), `git status` sobre accreta mostraba 16 bilinks modificados y el diff completo de cada uno era una línea de `resolved_at`.

**Y `check` no propaga.** Refrescar la cache no cambia ningún valor aceptado, así que el vecino de la cadena no ve nada. La cadena la mueve `accept`, que es quien escribe una decisión.

## Fuente del cambio

Para endpoints estructurales no-OK:

| Condición git | Fuente en salida |
|---|---|
| `git diff -- <file>` tiene hunks solapando el fragmento | `[UNSTAGED]` |
| `git diff --cached -- <file>` tiene hunks solapando el fragmento | `[STAGED]` |
| `git log <commit>..HEAD -- <file>` tiene commits | `[commit <hash> "<msg>"]` |

El baseline es `commit` —el commit en que el fragmento quedó con el contenido aceptado— y no un timestamp: `git log` recorre por ancestría y es exacto, mientras que las fechas se desordenan con rebases y cherry-picks.

### Intersección hunk / fragmento

```
fragmento: líneas F_start–F_end  (derivadas del range, en bytes)
hunk:      @@ -H_start,H_count +...

H_start + H_count < F_start  → BEFORE  (posible causa de DISPLACED)
H_start > F_end              → AFTER   (irrelevante)
se superpone                 → WITHIN  (causa de EXPANDED, ALTERED, REANCHORED)
```

## Fix disponible

`MOVED`, `DISPLACED`, `REANCHORED` y `EXPANDED` los repunta [`apply`](apply.md), que usa el estado cacheado sólo para elegir candidatos y recalcula cada fix re-resolviendo contra git y el AST actuales. Nunca se aplican solos.

Y ninguno cierra solo: `apply` repunta y deja el endpoint en `RELOCATED`, que sale con 1 hasta que alguien acepte.

## Salida

Los endpoints en `OK` se omiten por defecto. Con `--verbose` se muestran todos.

**Qué se imprime y qué código de salida se devuelve son dos preguntas distintas.** Un endpoint que no está `OK` **se imprime**, siempre, porque hay trabajo que hacer. Cuál de esos trabajos hace **fallar** a `check` es otra cosa. Confundir las dos deja estados que existen en disco y no aparecen en ninguna parte.

```
$ bilinker check .

7f3d8e9a  (OK, CHAIN_DIRTY)
  endpoint.1  → path >impl   el vecino fue re-aceptado
  → inspeccionar: bilinker chain status 7f3d8e9a-…

3a4b5c6d  (RELOCATED, ALTERED)
  endpoint.0  la ubicación cambió y nadie la aprobó
    aceptado: capture 67ba7217…  specs/voting.yaml
    ahora:    capture 9f8e7d6c…  specs/domain/voting.yaml
  → revisar y aprobar: bilinker accept 3a4b5c6d.0 --place
  endpoint.1  java-demo::Persona#vote  el AST interno cambió
    - Comparator.comparingInt(String::length)
    + (a, b) -> a.length() - b.length()
    source: commit c7d3e9f "Inline comparator" (2026-05-19)

f1e2d3c4  (EXPANDED, OK)
  endpoint.0  specs/reporter.yaml#generate  el fragmento creció — AST sin cambios
    + log.info("called");  [commit a3f2b1c "Add audit log"]
  → fix disponible: bilinker apply
```

## Código de salida

| Código | Condición |
|---|---|
| 0 | Todos los captures resuelven y todos los endpoints están en `OK`, `DISPLACED`, `EXPANDED` o `RESTYLED`. |
| 1 | Algún capture en `UNANCHORED`, `DELETED` o `BROKEN`, o algún endpoint en `RELOCATED`, `ALTERED`, `UNRESOLVED`, `PENDING` o `CHAIN_DIRTY`. |

**`RELOCATED` sale con 1.** Antes `MOVED` y `DISPLACED` salían con 0 porque `apply` los cerraba solo; ahora repuntar no aprueba, y un vínculo apuntando a un fragmento que nadie miró es trabajo pendiente, no un detalle de mantenimiento.
