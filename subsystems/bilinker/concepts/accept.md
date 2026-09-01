# La aceptación

Aceptar es decir *"revisé esto y lo apruebo"*. Es el único acto del sistema que no se puede derivar de nada, y por eso es lo único —junto con la declaración— que queda versionado.

## Las dos dimensiones

Un endpoint puede desalinearse de dos formas distintas, y hay que poder aprobarlas por separado:

| Dimensión | Cambió | Se aprueba |
|---|---|---|
| **Ubicación** | el fragmento se movió: otro archivo, otro nodo | `accepted.link` |
| **Contenido** | el fragmento sigue donde estaba y dice otra cosa | `accepted.hash` |

Antes había una sola: `hash.N` respondía por el contenido, y la ubicación se corregía sola cuando `apply` reescribía el capture. Eso hacía que **mover un fragmento no requiriera aprobación de nadie**: `apply` decidía a qué otro nodo apuntaba el vínculo y lo dejaba en `OK`.

Con las dos dimensiones separadas, `apply` sigue proponiendo la ubicación nueva pero no la bendice. El endpoint queda en `RELOCATED` hasta que alguien acepte.

> **`apply` propone, `accept` dispone.** Es la misma división que ya había entre corregir y aprobar, extendida a la dimensión que se había quedado afuera.

## El bloque

```yaml
endpoint:
  0:
    link: capture 67ba7217e0334051becd4921b55a7872
    accepted:
      agree:
      - pablo
      link: capture 67ba7217e0334051becd4921b55a7872
      hash: c00e07602bd560755096b57df1ddb9ed49d816fb8af58a4ec9cde82f21f38db3
      hash_ast: 1b9e44a2f0c8d3e7a5b1c9d4e2f6a8b0c3d5e7f9a1b3c5d7e9f1a3b5c7d9e1f3
      hash_n1: 96c765b9a3f1e4d7c2b8a5f0e3d6c9b2a7f4e1d8c5b0a3f6e9d2c7b4a1f8e5d0
      hash_ast_n1: 88e834c4b1a7f2e5d0c3b6a9f4e7d2c5b8a1f6e3d0c7b4a9f2e5d8c1b6a3f0e7
  1:
    link: path >impl
```

| Campo | |
|---|---|
| `agree` | Quiénes aprobaron **estos** valores. Ver "Quiénes aprobaron". |
| `link` | La ubicación aprobada. Ausente en un endpoint `issue` o `abstract`, que no tienen capture. |
| `hash` | SHA-256 del fragmento aprobado — la concatenación de los `@target`, ver [capture](capture.md) § "El fragmento son los `@target`". |
| `hash_ast` | SHA-256 de su s-expression, y de las de todos sus nodos unidas por `\n` cuando hay más de uno. Opcional: ausente donde no hay gramática. |
| `hash_n1` | SHA-256 plegado del **vecindario**: los tipos que la firma menciona. Opcional. Ver § "El cierre de firma". |
| `hash_ast_n1` | Ídem sobre las s-expressions de esos vecinos. Opcional, y **todo-o-nada**. |
| `n1` | `declined` cuando alguien aceptó renunciando al vecindario. Presente sólo en ese caso. Ver § "Cuándo se adquiere el vecindario". |

**`accepted` está o no está.** Su ausencia *es* `PENDING`, literalmente — no hay que enunciar que los campos de aceptación están presentes juntos o ausentes juntos, porque el bloque no se puede escribir a medias. Lo verifica el tipo: `accepted` sin `hash` es rechazado, y un `hash` suelto afuera del bloque también.

`hash` y `hash_ast` van separados y no hasheados juntos porque `RESTYLED` necesita compararlos por separado. Donde `hash_ast` no está, `RESTYLED` no existe y todo cambio de texto es `ALTERED` — que en prosa es lo correcto.

## El cierre de firma

Un fragmento no es sólo un árbol de sintaxis: **devuelve un tipo**. Con el capture de contrato de [`capture.md`](capture.md), la firma entra en `hash` y un cambio de tipo de retorno se detecta. Lo que no se ve es que el tipo **siga llamándose igual y tenga otros campos** — que es exactamente lo que rompió al consumidor que motivó todo esto.

`hash_n1` y `hash_ast_n1` cubren esa fila, y una sola: **el vecindario de nivel 1.**

### No recursa, y está en el nombre

Los vecinos son **los tipos que la firma menciona, un salto**. Los campos de esos tipos son nivel 2 y no entran.

| Cambio en el proveedor | ¿Lo ve? |
|---|---|
| al DTO le agregan un campo | **sí** — cambió su texto |
| `String name` → `AuthorityKind name` en el DTO | **sí** |
| el DTO se muda de archivo | **sí** — cambió el conjunto |
| el método devuelve otro tipo | **sí** — cambió el conjunto |
| a `AuthorityKind` le cambian los valores | **no** — es nivel 2 |

Clavar la profundidad en 1 retira tres preguntas de una vez: dónde para el cierre, cómo termina con tipos recursivos, y cómo se recorre. No hay recorrido: se resuelven los tipos de la firma y listo. Si alguna vez hace falta más, un `hash_n2` es aditivo y no invalida nada.

### El fold, y por qué el orden es por identidad

Un solo orden, y dos folds sobre ese orden:

```
orden = los vecinos por su identidad — <layer-root>::<path> más el nombre del símbolo,
        byte-wise sobre el UTF-8. Esa clave ordena y no entra en ningún hash.

hash_n1     = H( hash(v)     de cada vecino, en ese orden )
hash_ast_n1 = H( hash_ast(v) de cada vecino, en ese orden )
```

**La clave de orden tiene que ser identidad, nunca contenido.** Ordenando por el texto, un reformateo le cambiaría el puesto a un vecino, la lista se reordenaría, y `hash_ast_n1` se movería sin que ningún AST cambiara — un falso *"cambió de verdad"* producido por el orden. Ordenando por identidad nadie se mueve de puesto salvo que un vecino **entre, salga o se renombre**, y esas tres cosas son cambios de contrato.

Tampoco puede ordenar el rango: lleva offsets de bytes, que se corren con cualquier edición más arriba del archivo.

**Los vecinos se hashean con el mismo recorte de bordes que un fragmento.** Es la regla de [`capture.md`](capture.md) § "El rango excluye el espacio que rodea al nodo", y vale igual acá: un vecino sin recortar mueve su hash cuando le agregan algo abajo.

### `hash_ast_n1` es todo-o-nada

Si un vecino no tiene gramática no tiene `hash_ast`, y **no puede quedar afuera del fold**: un cambio real en ese vecino movería `hash_n1` y no `hash_ast_n1`, y eso se leería como *"sólo formateo"* cuando no lo fue. Un falso `RESTYLED` es peor que ningún estado.

Así que el campo está presente sólo si **todos** los vecinos tienen gramática. Si a alguno le falta, está ausente, y cualquier cambio en `hash_n1` es un cambio real. Es el mismo espíritu que la regla de `hash_ast`: falla hacia reportar, no hacia callarse.

`hash_n1` también es opcional — ausente donde el fragmento no tiene firma resoluble, que es prosa, un DTO, o un lenguaje sin anotaciones de tipo.

### Los vecinos no son captures

No se acuñan. Nadie los referencia con un `link` ni con un `accepted.link`, así que un capture acuñado para un vecino sería basura que `prune` borra en la pasada siguiente.

Son **ubicaciones** que alguien resuelve y que bilinker hashea al pasar. Con eso desaparecen la conversión rango → query, un modo nuevo de `capture`, una raíz nueva para `prune`, y —la que más importa— el modo de falla del anclaje: `capture` falla cuando no encuentra un ancla única, y con vecinos resueltos por un language server eso pasaría seguido.

### Bilinker no sale a buscarlos

**Es un valor que bilinker guarda y compara sin poder calcularlo por su cuenta.** No es una excepción nueva: es el patrón que el formato ya tiene en [`capture.md`](capture.md) invariante 7, donde un `accepted.link` de endpoint layer o repo lleva una copia opaca de un id ajeno. Se compara, no se resuelve.

Quien los encuentra entra por un puerto que bilinker define y que **no nombra a nadie**:

```rust
pub trait Neighbours {
    fn of(&self, file: &Path, ranges: &Ranges) -> Result<Option<Vec<Location>>>;
}
```

`None` es *"no pude mirar"* y no *"no hay vecinos"* — la distinción de la que sale el estado. El binario le pasa una implementación que le habla a [`lspd`](../../lspd/overview.md); la librería no lo menciona, y **no es para evitar un ciclo** —ya no hay— sino para que bilinker no quede atado a *ese* daemon: mañana puede ser SCIP, un índice propio, o un language server hablado directo.

**Y el hasheo queda de este lado.** El recorte de bordes es regla de bilinker y vive en el único lugar donde un nodo se convierte en rango. El proveedor devuelve ubicaciones, no hashes.

### Cuándo se adquiere el vecindario

El puerto puede contestar `None` —*"no pude mirar"*— y ahí hay que decidir qué se escribe. **La regla es una: una falla de infraestructura no puede reducir la cobertura de un vínculo.**

Que un fragmento *tenga* vecindario se sabe por la gramática y no por el proveedor: es si su firma es resoluble. Así que bilinker distingue sin daemon los dos motivos por los que no habría `hash_n1` que calcular, y sólo el segundo es un problema.

Y hay un dato más que se sabe sin daemon: **el conjunto de vecinos lo determina la firma, y la firma está en el fragmento.** Con el [capture de contrato](capture.md) el `hash` del fragmento es el de la firma, así que un `hash` que no se movió es el mismo conjunto de vecinos — aunque su contenido no se haya podido mirar.

| Ya hay `hash_n1` aceptado | Se pudo resolver | Cambió la firma | Qué se escribe |
|---|---|---|---|
| no | sí | — | `hash_n1` calculado |
| no | **no** | — | nada, y `accept` falla |
| sí | sí | — | `hash_n1` recalculado |
| sí | **no** | **no** | **el `hash_n1` que ya estaba**, intacto |
| sí | **no** | **sí** | nada, y `accept` falla |

**La cuarta fila es la que evita el daño.** Preservar es estrictamente más seguro que borrar: si un vecino cambió mientras el proveedor estaba caído, el `hash_n1` viejo sigue ahí y el próximo cierre con proveedor lo reporta. Borrándolo, ese cambio se absorbe en el baseline nuevo y **deja de ser detectable para siempre**.

**Y la quinta es la única donde preservar sería mentir**: si la firma cambió, el conjunto de vecinos pudo cambiar con ella, y el valor viejo es sobre un conjunto que ya no es el de hoy.

No es un mecanismo nuevo: [`--place` y `--content`](#--place-y---content) ya aceptan una dimensión sin tocar la otra. El vecindario es una tercera y se comporta igual. Lo que no puede pasar es que una dimensión se borre **como efecto colateral** de aceptar otra.

### `n1: declined` es lo que vuelve determinista la renuncia

Renunciar al vecindario tiene que poder decirse, y las dos filas que fallan se destraban [declarándolo](../commands/accept.md#--no-n1). Eso escribe `n1: declined`, y **el campo no es cosmético: sin él se rompe la invariante 4.**

Sin marca, el mismo fragmento en el mismo estado produce un `accepted` **con** `hash_n1` o **sin** él según si había un language server prendido en esa máquina. La determinación la tomaría el ambiente, que no es parte del estado del fragmento. Y se lleva puesta la propiedad que puso a `hash_n1` en `accepted` y no en la cache: que *"dos personas que aceptan el mismo vecindario en el mismo estado escriben el mismo valor"*.

Con la marca, quien decide es el flag —igual que `--place` y `--content`— y la convergencia vuelve: dos personas que renuncian escriben lo mismo, y quien no renuncia no puede escribir un baseline mudo sin saberlo.

**Y es lo único que cruza la frontera.** El consumidor recibe una copia opaca del `accepted` del endpoint `repo` y no puede volver a mirar la gramática del fragmento ajeno para reconstruir el motivo. Sin la marca, *"el proveedor no tiene vecindario"* y *"el proveedor renunció a vigilarlo"* le llegan idénticos.

**La ausencia sin marca queda con un solo significado**: el fragmento no tiene firma resoluble. Eso es prosa, un DTO, o un lenguaje sin anotaciones de tipo — derivable de la gramática por cualquiera, siempre igual.

> **Es aditivo y sube la versión de formato igual.** Un parser viejo ignora `n1` y lee esa aceptación como *"este fragmento no tiene vecindario"*, en silencio y sin fallar — que es exactamente el caso que [`format-version.md`](format-version.md) describe. Ver también [frontier.md](frontier.md) § "que siga siendo `abstract`", donde `link: abstract` tiene la misma forma.

### Van en `accepted`, no en la cache

Dos razones, y las dos son las que decidieron dónde va cada campo del formato.

**La frontera.** Lo que cruza entre repos es la copia opaca de `accepted` del endpoint `repo`. Si `hash_n1` no está ahí, **no cruza** — y el caso que motiva todo esto se queda sin el mecanismo que lo entrega al consumidor.

**No es recuperable.** Reconstruirlo pediría un language server indexando un checkout histórico, que puede ni buildear. Ahí está la diferencia con [`commit`](cache.md), que se queda en la cache porque su recuperación es *"más lento, nunca no disponible"*. **Un valor cuya reconstrucción depende de infraestructura que puede no estar no es un derivado: es una decisión.**

Y pasa el otro test: **converge.** Dos personas que aceptan el mismo vecindario en el mismo estado escriben el mismo valor, porque sale de hashear contenido — a diferencia de `commit`, que nunca converge. Así que no le mete a [`adopt`](../commands/adopt.md) un campo que diverja siempre.

### Los cuatro cuadrantes

Dos ejes independientes, y las cuatro combinaciones dicen cosas distintas:

| Difiere | Qué significa |
|---|---|
| `hash` | el fragmento se reformateó |
| `hash` + `hash_ast` | el proveedor cambió lo que el fragmento dice |
| `hash_n1` | el vecindario se reformateó |
| `hash_n1` + `hash_ast_n1` | **un vecino cambió: el contrato se movió** |

La última es el caso que motivó todo: el método intacto, el DTO movido.

**Y la segunda dejó de ser un problema por otro lado.** Antes decía *"el proveedor refactorizó por dentro"* y era ruido que le llegaba al consumidor; con el capture de contrato el cuerpo ya no entra en `hash`, así que ese refactor no mueve nada. La resolvió [`capture.md`](capture.md), no el estado.

## Aceptar exige el fragmento commiteado

`accept` **falla sobre un archivo sucio.**

No es una recomendación: es lo que hace posible calcular `commit`, que es el commit en el que el fragmento quedó con el contenido aprobado. Ese commit no existe si el fragmento no está commiteado.

Y es lo que garantiza que lo aprobado sea recuperable. Sin commit no hay `git show <commit>:<file>`, y sin eso `check` no puede recuperar el texto aceptado — que es lo que distingue `EXPANDED` y `REANCHORED` de un `ALTERED` genérico.

## Quiénes aprobaron

`agree` es el set de quienes aprobaron **exactamente estos valores**. Como los valores direccionan por contenido, *"estar de acuerdo"* no es ambiguo: es haber aprobado este hash y esta ubicación y no otros.

**Por endpoint y local, nunca copiado.** En un endpoint estructural están los que aprobaron ese fragmento; en un endpoint `path` o `repo`, los que aprobaron **esa copia**. Quién aprobó del otro lado de la cadena es un hecho de la otra capa, y traerlo acá sería atribuir mal. Los dos endpoints de un bilink pueden tener listas distintas, y es lo normal.

**Quien escribe el `accepted` entra solo.** Si no, el campo significaría "los que *además* aprobaron", que es otra cosa.

**Es un set: ordenado y único al serializar.** Un duplicado o un reordenamiento producirían un diff que no dice nada.

**Y si los valores cambian, la lista se vacía.** Es lo que *"estos valores"* significa: quien aprobó el hash anterior no aprobó el nuevo, y arrastrar su nombre sería atribuirle una decisión que no tomó. Un `accept` que cambia `hash` o `link` deja `agree` con una sola persona — la que acabó de aceptar — y los aprobadores anteriores quedan donde siempre estuvieron, en los commits que escribieron los valores anteriores.

Vale también para `--place` y `--content`: lo que se compara es el par, no cada dimensión por su lado. Aprobar una ubicación nueva sobre un contenido que nadie volvió a mirar produce un par que nadie más aprobó.

### Un nombre por línea, y por eso no guarda su commit

Se escribe en bloque, nunca en flow:

```yaml
agree:
- ana
- pablo
```

La secuencia va sin sangría extra bajo la clave — es lo que el serializador emite, y las dos formas son el mismo YAML. Lo que importa es que sea **bloque y no flow**: una línea por nombre.

**Porque `git blame` sólo puede atribuir una línea a un commit.** En una sola línea, `- ana, - pablo` colapsa N actos distintos en un lugar, y blame devuelve el commit del último que la tocó: el primer aprobador se pierde. Con un nombre por línea, cada endoso queda atribuible por separado — autor, fecha y firma — y de ahí sale que **el campo no necesite guardar el commit de nadie**: git ya lo sabe, y con `blame` se llega en un salto.

Es la misma razón por la que `commit` no está en `accepted`: lo que git puede contestar no se duplica en el archivo.

**Y el orden es alfabético, no cronológico.** Cronológico sería tentador —diría quién aprobó primero— pero después de una unión el orden dependería del orden del merge, que no es un hecho sobre nada: dos repos con el mismo set escribirían archivos distintos. El alfabético es canónico, y no pierde nada: quién fue primero lo dice la fecha del commit, que blame entrega igual.

Insertar un nombre más arriba no rompe la atribución de los demás: blame sigue el contenido de la línea, no su número.

### Es lo que le da algo que escribir al segundo

Sin `agree`, aprobar algo que ya está `OK` no cambia ningún byte: no hay diff, no hay commit, no hay firma, no hay registro. **El endoso explícito era inexpresable.** Con la lista, es un diff de una línea sobre un commit firmado.

Por eso el campo no duplica la historia: **es lo que la crea.** Parece derivable —caminar el log del bilink juntando autores— y sin él no habría log del cual derivarlo.

### Y sale de la convergencia byte a byte

Dos personas que aceptan el mismo contenido escriben los mismos hashes y **listas distintas**, así que `agree` sale de la fila *"ya coincidía"* de la que depende [`adopt`](../commands/adopt.md).

La diferencia con un campo que no puede estar acá —`commit`, que el mismo contenido aceptado en dos ramas resuelve a dos valores sin forma de elegir— es que **acá la resolución es correcta y única: unión**. `adopt` es un merge campo por campo, así que une los dos sets sin preguntarle a nadie. Cuesta una fila en su tabla, no una suposición rota.

### Sin firma verificada es atribución, no atestación

`pablo` es texto, y **cualquiera puede escribir `agree: [ana]` sin que Ana se entere**. Un campo que parece atestación y es la afirmación de un tercero es peor que no tenerlo.

Vale lo que [`ref.md`](ref.md#autoría-atestación-y-autorización) ya dice: *"el autor de git es auto-declarado; lo que constituye atestación es la firma, no el campo."*

Lo que la convierte en algo en que apoyarse son **dos reglas que se verifican del lado que puede rechazar**, y ninguna necesita traducir un nombre a una clave:

1. El commit está **firmado** por una clave de la allowlist, lo que lo ata al autor que declara.
2. Los nombres que ese commit **agregó** a algún `agree` son exactamente su autor.

Con las dos, `- ana` sólo puede haberlo escrito un commit firmado cuya autora es Ana. Las verifica [`verify-ref`](../commands/verify-ref.md). Sin ellas —en un clon, o contra un remoto sin hook— `agree` se lee como lo que es: **una declaración local, sin más peso que quien la escribió.**

## Los valores son deterministas; la lista se une

Dos personas que aceptan el mismo fragmento en el mismo estado escriben **los mismos valores**. `accepted` no lleva cuándo se aceptó ni desde qué HEAD: eso es del commit de la ref.

De ahí que `commit` sea el commit **del contenido** y no el HEAD de quien acepta. Con el HEAD, el mismo acto daba distinto según quién y cuándo lo hiciera, y el valor no describía nada del fragmento.

Lo único que no converge es `agree`, y a propósito: **es el campo cuyo contenido *es* la diferencia entre las personas.** Reconcilia por unión, que es la única regla posible y no requiere que nadie decida.

## `--place` y `--content`

Por defecto `accept` aprueba las dos dimensiones. Los dos flags permiten aprobar una sola:

```sh
bilinker accept <uuid>.<N>              # ubicación y contenido
bilinker accept <uuid>.<N> --place      # sólo la ubicación
bilinker accept <uuid>.<N> --content    # sólo el contenido
```

Aceptar la ubicación de un fragmento cuyo contenido no se revisó es una situación real —el archivo se renombró y el código no cambió, o cambió y hay que mirarlo aparte— y sin los flags habría que aprobar de más para poder avanzar.

## Un endpoint layer copia las dos

El `accepted` de un endpoint layer o repo copia los **dos** valores del endpoint estructural del bilink adyacente: su `link` y su `hash`. **`agree` no se copia**: es de esta capa, no del vecino.

Cada uno cambia por una sola razón y por ninguna otra: `hash` cuando cambia el contenido publicado, `link` cuando cambia su ubicación aprobada. Los dos son inmunes a etiquetas, comentarios y reordenamientos del archivo vecino — que es la razón de copiar los valores y no hashear el archivo entero.

## Lo que no se acepta a ciegas

Un `accept .` sobre una capa recién cambiada fabrica aprobaciones que nadie miró. Cada estado no-OK es un puntero al fragmento que hay que revisar, y **el inventario de trabajo de un cambio es esa lista**; vaciarla para dejar el árbol verde es tirar justamente lo que hacía falta.

`accept .` existe para el caso en que ya se revisó todo, no para el caso en que no se revisó nada.

## Invariantes

1. `accepted` está completo o ausente. Su ausencia es `PENDING`.
2. `accept` es el único que escribe `accepted`.
3. `accept` falla si el fragmento no está commiteado.
4. Aceptar el mismo fragmento en el mismo estado produce siempre los mismos **valores** de `accepted`. `agree` es la excepción, y reconcilia por unión.
5. `accepted.link` de un endpoint layer o repo es una copia opaca del id ajeno: se compara, no se resuelve.
6. Aprobar una ubicación y aprobar un contenido son actos separables.
7. `accepted.agree` es un set, incluye a quien escribió el `accepted`, y es local: nunca se copia de un vecino.
8. `agree` no participa de ninguna comparación de estado ni de ningún hash. `OK` no depende de cuántos aprobaron.
9. Un `accept` que cambia algún valor deja `agree` con quien aceptó y nadie más; uno que no los cambia lo agrega al set que había.
10. Un commit sobre la ref sólo agrega a **su propio autor** a un `agree`. Sacar no está restringido: agregar es lo único que afirma algo sobre otra persona.
11. **Ningún `accept` reduce la cobertura de un endpoint sin que alguien lo haya pedido.** Que el proveedor de vecindario no conteste nunca borra un `hash_n1` aceptado: o se preserva, o `accept` falla.
12. Un `accepted` sin `hash_n1` y sin `n1` afirma que el fragmento **no tiene firma resoluble**. La renuncia se escribe, no se omite.
