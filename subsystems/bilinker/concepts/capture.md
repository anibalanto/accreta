# Especificación: Formato del archivo `.capture`

Un **capture** es una ubicación: qué archivo y qué nodo del AST. Nada más. No sabe qué versión del fragmento alguien aprobó, ni dónde cayó la última vez que se lo resolvió.

Un bilink no describe un fragmento: **referencia un capture**. Varios bilinks pueden referenciar el mismo.

## El id es el hash de la ubicación

El nombre del archivo es el hash de lo único que contiene: **cada campo seguido de un `\0`**.

```
id = sha256( file "\0" query "\0" )[..32]
```

El terminador va después de cada campo y no entre campos, y eso importa: así el id no cambia cuando un campo desaparece del formato. Es lo que permitió sacar el `offset` sin re-acuñar los 316 captures que existían.

De ahí salen tres propiedades que antes había que sostener con reglas:

**Un capture es inmutable.** Cambiarle la ubicación le cambiaría el nombre, así que no se cambia: se acuña otro. Un capture, una vez escrito, dice lo mismo para siempre.

**La deduplicación es por construcción.** Dos referencias a la misma ubicación producen el mismo id y por lo tanto el mismo archivo. No hay que buscar un capture equivalente antes de crear uno, que es lo que hoy hace `capture_to_file` escaneando la capa.

**Mover un fragmento pasa a ser un cambio visible.** Antes `apply` reescribía el capture en el lugar y el bilink no se enteraba; ahora repunta el `link` a un capture nuevo, y ese repunte es un cambio en el bilink que alguien tiene que aprobar. Ver [aceptación](accept.md) § "Las dos dimensiones".

### El hash del contenido no entra en el id

El id sale de la ubicación y **sólo** de la ubicación. Si entrara el hash del fragmento, cada edición del código produciría un capture nuevo y el vínculo se rompería en cada commit — la referencia dejaría de sobrevivir a los cambios, que es su razón de ser.

Por el mismo motivo tampoco entra `commit`: la procedencia de una decisión no es parte de dónde está un fragmento.

## Ubicación

```
<layer-root>/
  .bilink/
    <uuid>.yaml              ← las relaciones
    capture/
      <id>.yaml              ← las ubicaciones, inmutables
    cache/
      state                  ← lo derivable · no versionado
    version                  ← la versión de formato
    .gitignore               ← qué de acá adentro no se versiona
    index/
      index                  ← lookup O(1) · no versionado
```

Un capture vive en la capa cuyo archivo referencia; su `file` es relativo a la raíz de esa capa.

**La extensión es `.yaml` y nada más.** El tipo lo dice la carpeta que lo contiene, así que repetirlo en el nombre sería redundante.

## Formato

```yaml
# .bilink/capture/<id>.yaml
file: subsystems/bilinker/commands/check.md
query: |-
  (section (atx_heading (inline) @n0 (#eq? @n0 "Especificación: comando `bilinker check`"))
    (section (atx_heading (inline) @n1 (#eq? @n1 "Estados — endpoints layer"))) @target)
```

| Campo | Descripción |
|---|---|
| `file` | Path relativo a la raíz de la capa. |
| `query` | Query tree-sitter con captura `@target`. Ausente = el archivo completo. |

### No hay sub-rango

Un capture nombra un nodo entero. **No se puede referenciar un rango de bytes adentro de uno**, y la selección con la que se lo crea sirve para *encontrar* el nodo, no para recortarlo.

Un rango adentro de un nodo se corre con cualquier edición encima suya dentro del mismo nodo: su granularidad es ilusoria, se rompe todo el tiempo y hay que repuntarlo a mano. Un ancla de nodo entero es estable, y sus falsas alarmas son honestas — *"esto cambió, fijate si tu spec sigue valiendo"*.

Lo que se pierde es **atribución**, no detección: dos fragmentos de spec que describen dos partes de la misma función pasan a compartir capture. `hash` dice que cambió y `hash_ast` si fue sólo espaciado; cuál parte cambió lo dice `bilinker get --diff`, que es trabajo de quien mira.

**Si hace falta más precisión, la respuesta es una query que nombre algo más chico**, no un recorte sobre una que nombra algo más grande. Por eso una fila de tabla markdown se ancla por el texto de su primera celda: es un nodo, y tiene con qué distinguirse.

### El rango excluye el espacio que rodea al nodo

Dónde empieza un nodo depende de qué hay alrededor. En YAML el mismo item de secuencia empieza en el `-` cuando es el último y en la indentación de su línea cuando lo sigue otro: agregar un item más abajo le cambiaba los bytes —y con ellos el hash— a un item que nadie tocó.

Eso contradice la propiedad central, así que **el rango resuelto se recorta en los dos bordes**. El fragmento es su contenido; el espacio que lo separa de sus vecinos es de los dos y no de él.

Va en el único lugar donde un nodo se convierte en rango, así que no hay forma de obtener uno sin recortar.

Dos campos, y los dos entran en el id. **No hay más**: ni `range`, ni `state`, ni `resolved_at`. Todo eso se puede reconstruir resolviendo la query, así que vive en [`cache/state`](cache.md) y no en git.

Que el archivo tenga exactamente los campos que lo nombran es lo que hace verificable el id: cualquiera puede recalcularlo leyendo el archivo.

## Referencia desde un bilink

```yaml
endpoint:
  0:
    link: capture 67ba7217e0334051becd4921b55a7872
  1:
    link: path >impl
```

El prefijo `capture ` identifica el tipo de endpoint. Ver [tipos de endpoint](reference.md).

**El capture no guarda nada de la aceptación.** Eso vive en el bloque `accepted` del bilink, por endpoint, porque dos bilinks pueden haber aceptado versiones distintas del mismo fragmento: si A aceptó la v1 y B la v2, y el código está en v2, `check` debe reportar drift para A y no para B. Con un solo valor compartido eso sería imposible de expresar.

## Un capture no cambia; los `link` cambian de capture

Cuando un fragmento se mueve —el archivo se renombra, el anchor cambia de nombre— la ubicación nueva es **otro** capture. `apply` lo acuña y repunta el `link` del bilink que está corrigiendo. El capture anterior queda intacto para los demás referentes.

Eso reemplaza el copy-on-write que el formato anterior necesitaba:

| Antes | Ahora |
|---|---|
| `apply` mutaba el capture en el lugar, salvo que estuviera compartido | `apply` nunca muta: acuña |
| había que decidir por tipo de fix si forkear | no hay decisión que tomar |
| un fix se le imponía a los demás referentes | ninguno se entera |

La regla de fork por tipo de fix —MOVED en el lugar, REANCHORED forkeando— desaparece con ella. Existía porque mutar era el caso normal y forkear la excepción; con captures inmutables no hay caso normal que proteger.

**Y repuntar ya no es gratis.** `apply` repunta el `link` pero **no** devuelve el endpoint a `OK`: la ubicación cambió, y aprobar una ubicación es una decisión humana como aprobar un contenido. El endpoint queda en `RELOCATED` hasta que alguien acepte. Ver [aceptación](accept.md).

## Quién escribe qué

| Comando | Escribe |
|---|---|
| `bilinker capture` | Un `.capture` nuevo, si no existía. Devuelve su id. |
| `bilinker check` | Nada en el capture. Escribe `range` y `state` en [`cache/state`](cache.md). |
| `bilinker apply` | Acuña captures y repunta un `link`. Nunca modifica uno existente. |
| `bilinker accept` | Nada en el capture. Escribe `accepted` en el bilink. |

Ningún comando modifica un `.capture` existente. La única operación sobre el conjunto es agregar, y `prune` sacar los que ya no referencia nadie.

## Ciclo de vida

Un capture no conoce a sus referentes. `bilinker remove` sobre un bilink no borra los captures que referenciaba: puede haber otros usándolos.

Un capture sin referentes es basura inofensiva — ocupa un archivo y nadie lo lee. `bilinker capture prune` los elimina.

**`prune` es mark & sweep sobre dos clases de raíz**, no una sola. Un capture está vivo si lo referencia:

1. algún `link` — la ubicación vigente de un endpoint, o
2. algún `accepted.link` — la ubicación que alguien aprobó.

Barrer sólo por la primera borraría el capture que dice **dónde estaba lo que se aceptó**, y con él la capacidad de decidir si una ubicación cambió. Es la clase de raíz que el formato anterior no tenía porque no distinguía las dos cosas.

### El fan-out vive del lado del capture

Un bilink tiene **siempre exactamente dos endpoints**. La multiplicidad la aporta el capture: un fragmento puede tener N bilinks asociados.

```
                    ┌── bilink → spec de validación
capture(vote) ──────┼── bilink → ADR de auditoría
                    └── bilink → issue 3a
```

Esto cierra la alternativa de darle aridad variable al bilink. Un archivo llamado bilink con `link.0` … `link.4` sería una contradicción, y la aridad variable obligaría a redefinir la topología de cadena —hoy lineal, con exactamente dos tips— y el copiado de valores aceptados de los endpoints layer, que asume un único endpoint estructural adyacente.

**Lo que esta forma no expresa** es una relación conjunta entre tres cosas. Una estrella dice *"D se relaciona con A"* y *"D se relaciona con B"* por separado; no dice *"D gobierna el vínculo entre A y B"*. Para eso hace falta que un bilink apunte a otro bilink — el endpoint de tipo bilink, que está especificado y no implementado, en [`proposals/bilink-endpoint.md`](../proposals/bilink-endpoint.md).

## Relación con las cadenas

Las cadenas no cambian. Un bilink sigue viviendo en su capa, con endpoints layer hacia las capas vecinas y la misma propagación por copia. Lo único que cambia es que sus **endpoints estructurales referencian captures locales**.

Un bilink nunca referencia un capture de otra capa: eso rompería la propiedad de que aceptar en una capa nunca escribe en el repo de otra, que es lo que evita la cascada circular. Las conexiones entre capas siguen siendo endpoints layer.

La excepción aparente es `accepted.link` de un endpoint layer o repo, que **copia** el id del capture ajeno. Es una copia opaca: se compara, no se resuelve. Nadie va a buscar ese archivo en la capa local.

## Relación con lattice

Un capture es, casi literalmente, un nodo del grafo de [lattice](../../lattice/concepts/node.md): su forma canónica es `<layer-root>::<file>#<range>`. Un bilink es una arista sobre captures.

El `range` sale de la cache, no del capture. Un clon fresco no lo tiene hasta que corra un `check`.

## Invariantes

1. El nombre de un capture es el hash de sus campos, cada uno seguido de un `\0`, y el archivo contiene exactamente esos campos.
2. Un capture es inmutable. Ningún comando modifica uno existente.
3. Un capture describe ubicación, nunca aceptación. No contiene hashes ni commits.
4. `file` es relativo a la raíz de la capa donde vive el capture.
5. Un capture nombra un nodo entero: no hay sub-rango. El rango absoluto en el archivo es derivado y vive en la cache.
6. Un `link` sólo referencia captures de su propia capa. Un `accepted.link` de endpoint layer o repo puede contener una copia opaca de un id ajeno, que no se resuelve localmente.
7. Un capture puede ser referenciado por cualquier cantidad de bilinks, incluido cero.
8. `apply` acuña captures y repunta un `link`; nunca escribe `accepted`.
9. `accept` escribe `accepted`; nunca toca un capture.
10. Borrar un bilink nunca borra un capture.
11. `prune` conserva todo capture alcanzable desde un `link` **o** un `accepted.link`.

## Migración desde el formato anterior

Una migración, `bilinker-002-file-partition`. Ver [`bilinker migrate`](../commands/migrate.md).

Acuña cada capture bajo el hash de sus campos —no del archivo— y repunta cada `link` y cada `accepted.link`. **Los sub-rangos del formato 1 se descartan y se cuentan**: el formato ya no los tiene, y reubicarlos exigiría resolver la query, cosa que una migración no hace. Dos captures con la misma ubicación colapsan en uno: la dedup por construcción, aplicada de una vez a lo que ya existía.
