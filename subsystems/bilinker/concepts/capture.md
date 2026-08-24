# Especificación: Formato del archivo `.capture`

Un **capture** es una referencia mantenida a un fragmento de un archivo: dónde está, y cómo volver a encontrarlo cuando el código se mueve. No contiene ningún estado aceptado — no sabe qué versión del fragmento alguien aprobó.

Un bilink no describe un fragmento: **referencia un capture**. Varios bilinks pueden referenciar el mismo.

## Por qué existe

Antes, cada endpoint estructural de cada `.bilink` guardaba su propia copia de `file`, `query` y `range`. Cuando dos bilinks apuntaban al mismo fragmento, esa descripción estaba duplicada: dos resoluciones tree-sitter en cada `check`, dos rangos que `apply` podía dejar desincronizados.

No es hipotético. En la capa impl de bilinker, sobre 73 endpoints estructurales hay **56 distintos** — el 23% son duplicados. El enum `Command` está capturado cuatro veces; `capture()` y `main()`, tres cada uno.

Separar el capture del bilink hace que la ubicación de un fragmento se describa y se mantenga **una sola vez**, sin importar cuántas relaciones lo involucren.

## Ubicación

```
<layer-root>/
  .bilink/
    <uuid>.bilink              ← relaciones
    capture/
      <uuid>.capture           ← ubicaciones
    index/
      index
```

Un capture vive en la capa cuyo archivo referencia; su `file` es relativo a la raíz de esa capa. Su UUID es propio e independiente de los UUID de cadena.

## Formato

```
file:   crates/bilinker/src/check.rs
query:  (function_item
  name: (identifier) @n0 (#eq? @n0 "check_structural")) @target
offset: 42~118

# Cache
range:       5100~7300
state:       RESOLVED
resolved_at: 2026-08-24T10:00:00Z
```

| Campo | Descripción |
|---|---|
| `file` | Path relativo a la raíz de la capa. |
| `query` | Query tree-sitter con captura `@target`. Ausente = el archivo completo. |
| `offset` | Sub-rango **relativo al inicio del nodo** matcheado. Opcional; ausente = el nodo entero. |
| `range` | Byte range **absoluto** del fragmento en el archivo, de la última resolución. Cache. |
| `state` | Estado de resolución. Cache. |
| `resolved_at` | Timestamp UTC de la última resolución. Cache. |

`offset` y `range` son cosas distintas y conviene no confundirlas: `offset` es parte de la referencia — lo que el humano seleccionó dentro del nodo — y solo cambia con `apply`. `range` es dónde cayó eso la última vez que se resolvió.

Un capture **no guarda hashes**. El hash del fragmento se calcula leyendo el archivo en `range` cuando hace falta. Persistir un hash acá volvería a mezclar ubicación con aceptación, que es justo lo que este archivo separa.

## Referencia desde un bilink

```
link.0: capture 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a
link.1: .stratum/impl
```

El prefijo `capture ` identifica el tipo de endpoint, igual que `task `. El resto de los tipos de endpoint —layer, task, bilink— no cambia.

Los campos de aceptación siguen en el bilink, por endpoint: `hash.N`, `hash_ast.N`, `commit.N`, `state.N`.

Esa es la razón de que el hash no baje al capture: **dos bilinks pueden haber aceptado versiones distintas del mismo fragmento.** Si A aceptó la v1 y B la v2, y el código está en v2, `check` debe reportar `ALTERED` para A y `OK` para B. Con un solo hash compartido eso sería imposible de expresar.

## Estados

La separación parte la tabla de estados por una línea que antes estaba mezclada: *¿puedo encontrar el fragmento?* es distinto de *¿lo que hay coincide con lo que acepté?*

### Resolución — en el capture

Se evalúan sin ningún estado aceptado.

| Estado | Condición | Auto-fix |
|---|---|---|
| **RESOLVED** | La query matchea; `range` actualizado. | — |
| **MOVED** | El archivo cambió de path (git rename ≥ 50%). | ✓ actualiza `file` |
| **REANCHORED** | Anchor renombrado; nodo del mismo tipo con nombre distinto. | ✓ actualiza los predicados de `query` |
| **UNANCHORED** | La query no matchea y no se localiza el anchor. | — |
| **DELETED** | El archivo no existe; eliminación rastreable en git. | — |
| **BROKEN** | El archivo no se puede leer o parsear. | — |

### Aceptación — en el bilink

Comparan el contenido actual contra `hash.N`.

| Estado | Condición | Auto-fix |
|---|---|---|
| **PENDING** · **TODO** | Sin estado aceptado. | — |
| **OK** | El hash coincide en `range`. | — |
| **DISPLACED** | El hash coincide en otro offset del nodo. | ✓ actualiza `offset` del capture |
| **EXPANDED** | El fragmento creció; AST interno sin cambio estructural. | ✓ actualiza `offset` del capture |
| **RESTYLED** | `hash.N` difiere pero `hash_ast.N` coincide. | — |
| **ALTERED** | El AST interno cambió. | — |
| **CHAIN_DIRTY** | El nodo adyacente de la cadena cambió. | — |

### La asimetría, dicha explícitamente

`DISPLACED` y `EXPANDED` **se detectan con datos del bilink** — hace falta `hash.N` para saber que el fragmento se corrió o creció — pero **el fix se escribe en el capture**, porque lo que se corrige es la ubicación.

O sea que la regla no es "apply escribe captures porque los estados son de capture", sino:

> **`apply` corrige ubicación; `accept` fija contenido aceptado.**
>
> `apply` escribe captures, y en el bilink solo puede repuntar un `link.N` cuando forkea. Nunca escribe `hash.N`, `hash_ast.N` ni `commit.N`.
> `accept` escribe esos tres campos y nada más.

Sin importar en cuál de las dos tablas se detectó el problema. Antes era una convención enunciada al pie de `apply.md`; ahora la separación de archivos la hace casi imposible de violar por accidente — el único cruce es el repunte del fork, que es deliberado y visible.

## Copy-on-write al aplicar un fix

Un capture compartido no puede corregirse siempre en el lugar: hay fixes cuya resolución **depende de datos que son propios de cada bilink**, y otros que son ambiguos y donde dos referentes podrían decidir distinto.

| Fix | De qué depende la corrección | Acción de `apply` |
|---|---|---|
| **MOVED** | El archivo se renombró — hecho objetivo. | Muta el capture en el lugar. |
| **EXPANDED** | El nodo creció — hecho objetivo sobre el nodo. | Muta el capture en el lugar. |
| **DISPLACED** | De `hash.N`, que es propio de cada bilink. | Fork si hay más de un referente. |
| **REANCHORED** | De una inferencia sobre qué nodo "es el mismo". | Fork si hay más de un referente. |

**Fork** significa: crear un capture nuevo con la corrección aplicada, repuntar únicamente el `link.N` del bilink que se está corrigiendo, y dejar el capture original intacto para los demás.

Con un solo referente el fork es un no-op — el caso común no paga nada. `apply` cuenta los referentes escaneando los bilinks de la capa, que ya escanea de todos modos.

### Por qué `DISPLACED` obliga a forkear

Se detecta buscando `hash.N` en otro offset del nodo. Si el bilink A aceptó la v1 del fragmento y el B la v2, **el hash de cada uno aparece en un offset distinto**. Un solo capture no puede tener dos `offset` a la vez.

### Por qué `REANCHORED` obliga a forkear

Si `vote()` se partió en `voteA()` y `voteB()`, el bilink de la spec de validación debería seguir a uno y el de la spec de registro al otro. `apply` no puede saber cuál corresponde a cada uno, y aplicar cualquiera de los dos es incorrecto para el otro.

### El capture que no se forkeó no queda mintiendo

Un referente que se queda con el capture original no obtiene una respuesta silenciosamente equivocada: ese capture pasa a `UNANCHORED` o `REANCHORED` y lo reporta en cada `check`. Ese bilink *tiene* una situación sin resolver, y que la reporte es el comportamiento correcto.

Lo que el copy-on-write evita es que la corrección de un bilink se la resuelva por decreto a los otros.

## Quién escribe qué

| Comando | Escribe |
|---|---|
| `bilinker capture` | Crea un `.capture`. Devuelve su UUID. |
| `bilinker check` | `range`, `state`, `resolved_at` del capture · `state.N` del bilink |
| `bilinker apply` | `file`, `query`, `offset` del capture — o crea uno nuevo y repunta un `link.N`, si forkea |
| `bilinker accept` | `hash.N`, `hash_ast.N`, `commit.N` del bilink |

## Compartición y ciclo de vida

Un capture no conoce a sus referentes. `bilinker remove` sobre un bilink no borra los captures que referenciaba: puede haber otros usándolos.

Un capture sin referentes es basura inofensiva — ocupa un archivo y se resuelve en cada `check` sin que nadie lea el resultado. `bilinker capture prune` elimina los no referenciados en la capa.

Dos captures pueden describir el mismo fragmento con queries distintas. No se deduplican: son baratos y su identidad es el UUID. La deduplicación es una consecuencia de reusarlos, no una regla que el formato imponga.

## Relación con las cadenas

Las cadenas no cambian. Un bilink sigue viviendo en su capa, con endpoints layer hacia las capas vecinas y la misma propagación por copia de hash. Lo único que cambia es que sus **endpoints estructurales pasan a ser referencias a captures locales**.

Un bilink nunca referencia un capture de otra capa: eso rompería la propiedad de que aceptar en una capa nunca escribe en el repo de otra, que es lo que evita la cascada circular. Las conexiones entre capas siguen siendo endpoints layer.

## Relación con lattice

Un capture es, casi literalmente, un nodo del grafo de [lattice](../../lattice/concepts/node.md): su forma canónica es `<layer-root>::<file>#<range>`. Un bilink es una arista sobre captures.

Con esta separación, `bilinker graph --format json` deja de ser una transformación y pasa a ser casi un volcado: captures → nodos, bilinks → aristas.

## Invariantes

1. Un capture describe ubicación, nunca aceptación. No contiene hashes ni commits.
2. `file` es relativo a la raíz de la capa donde vive el capture.
3. `offset` es relativo al nodo matcheado; `range` es absoluto en el archivo.
4. Un bilink solo referencia captures de su propia capa.
5. Un capture puede ser referenciado por cualquier cantidad de bilinks, incluido cero.
6. `apply` nunca escribe `hash.N`, `hash_ast.N` ni `commit.N`. Su único efecto sobre un bilink es repuntar un `link.N` al forkear.
7. `accept` escribe únicamente `hash.N`, `hash_ast.N` y `commit.N`. Nunca toca un capture.
8. Un fix que dependa de `hash.N` o de una inferencia ambigua forkea el capture si tiene más de un referente.
9. Borrar un bilink nunca borra un capture.
10. El estado de resolución de un capture es idéntico para todos sus referentes; el estado de aceptación es propio de cada bilink.

## Migración desde el formato anterior

Cada endpoint estructural de un `.bilink` existente se convierte en un `.capture`:

- `file`, `query` y el `start~end` de la referencia → `file`, `query`, `offset` del capture.
- `range.N` → `range` del capture.
- `hash.N`, `hash_ast.N`, `commit.N`, `state.N` → se quedan en el bilink.
- `link.N` pasa a `capture <uuid>`.

Los endpoints idénticos pueden colapsarse a un capture compartido, pero no es obligatorio: la conversión uno-a-uno es correcta y el `prune` puede venir después.
