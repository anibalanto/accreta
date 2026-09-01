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
| `query` | Query tree-sitter con una o más capturas `@target`. Ausente = el archivo completo. |

### El fragmento son los `@target`, en orden de archivo

Una query puede llevar **más de una** captura `@target`. El fragmento es la concatenación de sus rangos, en el orden en que aparecen **en el archivo** — no en el que la query los nombra, que es un detalle de cómo se escribió el patrón y no del documento.

Con un solo `@target` el fragmento es ese nodo y nada más: es el caso de siempre, y da exactamente el mismo `hash` que daba antes de que hubiera varios.

Con varios se pueden decir dos cosas que un nodo solo no puede:

| | |
|---|---|
| **menos que un nodo** | la firma de un método sin su cuerpo: se capturan `type`, `name` y `parameters`, y `body` queda afuera |
| **más que un nodo** | una ruta que sale de dos anotaciones —`@RequestMapping` en la clase, `@GetMapping` en el método—, que como texto completo no existe en ningún nodo |

**Y sigue siendo estructural**: son nodos, no rangos de bytes. Si el código se mueve, la query lo vuelve a encontrar, que es la propiedad por la que un capture existe.

**Ninguna parte contiene a otra.** El fragmento es la concatenación, así que una parte adentro de otra se contaría dos veces y el hash pasaría a depender de un solapamiento que nadie quiso. Cuando se señala una posición que resuelve a un nodo que contiene a otro señalado, no hay capture raro que escribir: hay una posición que resolvió más arriba de lo que se apuntó, y decirlo es más útil que capturar cualquier cosa.

#### Una query, no una lista de queries

Un patrón único ancla los fragmentos **entre sí**: dice *"el `@RequestMapping` de la clase que contiene al método `getPermissions`"*. Una lista de queries independientes ancla cada una por su cuenta, y la primera sería *"el `@RequestMapping`"* a secas — que en un archivo con dos clases matchea la equivocada. Se arregla repitiendo el contexto en cada una, o sea reconstruyendo a mano lo que el patrón único ya expresa.

**Y abriría la resolución parcial**, que no existe: tree-sitter matchea el patrón entero o no matchea. Con una lista, dos de tres queries resueltas no serían `RESOLVED` ni `UNANCHORED` sino un fragmento a medias, y habría que inventarle un estado y decidir si se hashea lo que hay.

Menor pero cierto: el id es `sha256(file \0 query \0)`, y `query` sigue siendo un string. Con una lista habría que definir cómo se hashea una.

#### El separador es `\n`

Los rangos no son contiguos, así que hay que decidir qué va entre uno y el siguiente — y eso **entra en el `hash`**:

| | |
|---|---|
| nada | dos capturas pegadas producen un texto que no existe en ningún archivo |
| **`\n`** | elegido: legible, y estable frente a cuánto espacio haya en el medio |
| el texto intermedio | es el archivo tal cual, pero entonces el cuerpo entra por la ventana cuando dos capturas lo tienen en el medio |

**Y no se vuelve a tocar.** Cambiarlo movería de una vez el hash de todos los captures multi-fragmento, y todos pasarían a `ALTERED` sin que nadie tocara el código — drift fabricado por un cambio de opinión.

`hash_ast` sigue la misma regla: las s-expressions de los nodos, en el mismo orden, unidas por `\n`.

### No hay sub-rango

Un capture nombra un **conjunto estructural de nodos** —uno por `@target`, cada uno entero. **No se puede referenciar un rango de bytes adentro de uno**, y la selección con la que se lo crea sirve para *encontrar* los nodos, no para recortarlos.

Un rango adentro de un nodo se corre con cualquier edición encima suya dentro del mismo nodo: su granularidad es ilusoria, se rompe todo el tiempo y hay que repuntarlo a mano. Un ancla de nodo entero es estable, y sus falsas alarmas son honestas — *"esto cambió, fijate si tu spec sigue valiendo"*.

Lo que se pierde es **atribución**, no detección: dos fragmentos de spec que describen dos partes de la misma función pasan a compartir capture. `hash` dice que cambió y `hash_ast` si fue sólo espaciado; cuál parte cambió lo dice `bilinker get --diff`, que es trabajo de quien mira.

**Si hace falta más precisión, la respuesta es una query** —que nombre algo más chico, o que nombre varios nodos y deje el resto afuera—, no un recorte sobre una que nombra algo más grande. Por eso una fila de tabla markdown se ancla por el texto de su primera celda: es un nodo, y tiene con qué distinguirse.

### El rango excluye el espacio que rodea al nodo

Dónde empieza un nodo depende de qué hay alrededor. En YAML el mismo item de secuencia empieza en el `-` cuando es el último y en la indentación de su línea cuando lo sigue otro: agregar un item más abajo le cambiaba los bytes —y con ellos el hash— a un item que nadie tocó.

Eso contradice la propiedad central, así que **el rango resuelto se recorta en los dos bordes**. El fragmento es su contenido; el espacio que lo separa de sus vecinos es de los dos y no de él.

Va en el único lugar donde un nodo se convierte en rango, así que no hay forma de obtener uno sin recortar.

**Con varios `@target` se recorta cada rango por separado**, antes de concatenar. Recortar la concatenación dejaría los bordes internos a merced de dónde termina un nodo y empieza el otro, que es justo el contexto del que el recorte existe para independizar.

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
5. Un capture nombra un conjunto de nodos enteros, uno por `@target`: no hay sub-rango. El fragmento es la concatenación de sus rangos en orden de archivo, separados por `\n`. Los rangos absolutos son derivados y viven en la cache.
6. Ninguna parte de un fragmento contiene a otra: los rangos son disjuntos.
7. Un `link` sólo referencia captures de su propia capa. Un `accepted.link` de endpoint layer o repo puede contener una copia opaca de un id ajeno, que no se resuelve localmente.
8. Un capture puede ser referenciado por cualquier cantidad de bilinks, incluido cero.
9. `apply` acuña captures y repunta un `link`; nunca escribe `accepted`.
10. `accept` escribe `accepted`; nunca toca un capture.
11. Borrar un bilink nunca borra un capture.
12. `prune` conserva todo capture alcanzable desde un `link` **o** un `accepted.link`.

## Migración desde el formato anterior

Una migración, `bilinker-002-file-partition`. Ver [`bilinker migrate`](../commands/migrate.md).

Acuña cada capture bajo el hash de sus campos —no del archivo— y repunta cada `link` y cada `accepted.link`. **Los sub-rangos del formato 1 se descartan y se cuentan**: el formato ya no los tiene, y reubicarlos exigiría resolver la query, cosa que una migración no hace. Dos captures con la misma ubicación colapsan en uno: la dedup por construcción, aplicada de una vez a lo que ya existía.
