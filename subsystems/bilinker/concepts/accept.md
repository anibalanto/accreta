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
    n:
      1:
        link: capture fe74f8b4e9fd72eeae03ea41ce520155 1b06e7c6750d68696653c9112925a54e
    accepted:
    - agree:
      - pablo
      link: capture 67ba7217e0334051becd4921b55a7872
      hash: c00e07602bd560755096b57df1ddb9ed49d816fb8af58a4ec9cde82f21f38db3
      hash_ast: 1b9e44a2f0c8d3e7a5b1c9d4e2f6a8b0c3d5e7f9a1b3c5d7e9f1a3b5c7d9e1f3
      n:
        1:
          link: capture fe74f8b4e9fd72eeae03ea41ce520155 1b06e7c6750d68696653c9112925a54e
          hash: 96c765b9a3f1e4d7c2b8a5f0e3d6c9b2a7f4e1d8c5b0a3f6e9d2c7b4a1f8e5d0
          hash_ast: 88e834c4b1a7f2e5d0c3b6a9f4e7d2c5b8a1f6e3d0c7b4a9f2e5d8c1b6a3f0e7
  1:
    link: path >impl
```

| Campo | |
|---|---|
| `agree` | Quiénes aprobaron **estos** valores. Ver "Quiénes aprobaron". |
| `link` | La ubicación aprobada. Ausente en un endpoint `issue` o `abstract`, que no tienen capture. |
| `hash` | SHA-256 del fragmento aprobado — la concatenación de los `@target`, ver [capture](capture.md) § "El fragmento son los `@target`". |
| `hash_ast` | SHA-256 de su s-expression, y de las de todos sus nodos unidas por `\n` cuando hay más de uno. Opcional: ausente donde no hay gramática. |
| `n` | El **vecindario**, por nivel: un capture por vecino y sus dos folds. Tres estados, ver abajo. |

**`accepted` es una lista, y el eje del vecindario tiene la misma forma que el del fragmento** — declaración afuera, decisión adentro. Ver [`bilink.md`](bilink.md#más-de-un-accepted-es-un-estado-no-una-forma-de-trabajar) para por qué más de una entrada es un estado y cómo colapsa.

### `n` es un campo con tres estados

El vecindario tiene **una** respuesta, y por eso es **un** campo — con los niveles adentro:

```yaml
n:                       # se adquirió
  1:
    link: capture fe74f8b4… 1b06e7c6…   # un capture por vecino
    hash: 96c765b9…
    hash_ast: 88e834c4…  # opcional adentro del nivel, y todo-o-nada

n: declined              # alguien renunció, a propósito

                         # ausente: el fragmento no tiene firma resoluble
```

Escrito como campos sueltos —`hash_n1`, `hash_ast_n1` y una marca aparte— quedaban representables combinaciones que no significan nada: un fold de ASTs sin el fold de textos que lo acompaña, y una renuncia conviviendo con el valor al que se renunció. **Que ningún código las produzca no es lo mismo que que no se puedan escribir**, y el YAML lo escribe cualquiera a mano.

Plegado, `hash_ast` no puede estar sin su `hash` porque vive adentro del mismo fold, y `declined` no puede convivir con un nivel porque son variantes del mismo campo.

### La renuncia va en el contenedor, no adentro de un nivel

**Es una sola decisión, y es de 1 para arriba.** El nivel 2 son los campos de los tipos que el 1 resuelve, así que está definido *a través* del 1 — renunciar al 1 deja al 2 sin base sobre la cual existir. Y por eso no hay un `--no-n2` que tenga sentido solo: el corte que la renuncia nombra no es un número de nivel, es **lo que bilinker calcula por su cuenta contra lo que pide un language server**, y el 1 es el primero del segundo lado.

Escribirla adentro del nivel 1 —`n1: declined`— decía *"el nivel 1 fue renunciado"* cuando quiere decir *"el vecindario fue renunciado"*. Con un solo nivel las dos frases coinciden y no se nota; con dos, obliga a preguntar qué pasa con el 2. **En el contenedor esa pregunta no llega a existir**, porque no hay dónde escribir la respuesta equivocada.

### El nivel 0 no entra

`hash` y `hash_ast` —el fragmento— se quedan afuera del mapa, aunque se hasheen igual que un vecino y la escalera quede tentadora:

| | nivel 0 | niveles ≥1 |
|---|---|---|
| es obligatorio | **sí** — `accepted` sin `hash` se rechaza | no |
| se puede renunciar | **no** — aceptar *es* hashearlo | sí |
| de dónde sale | tree-sitter y git, siempre | un language server, que puede no estar |

Adentro del mapa, `n: {}` sería una aceptación sin contenido aprobado y `n: {0: declined}` sería escribible sin querer decir nada. La regularidad se paga con dos estados inválidos, y no vale.

**`accepted` está o no está.** Su ausencia *es* `PENDING`, literalmente — no hay que enunciar que los campos de aceptación están presentes juntos o ausentes juntos, porque el bloque no se puede escribir a medias. Lo verifica el tipo: `accepted` sin `hash` es rechazado, y un `hash` suelto afuera del bloque también.

`hash` y `hash_ast` van separados y no hasheados juntos porque `RESTYLED` necesita compararlos por separado. Donde `hash_ast` no está, `RESTYLED` no existe y todo cambio de texto es `ALTERED` — que en prosa es lo correcto.

## El cierre de firma

Un fragmento no es sólo un árbol de sintaxis: **devuelve un tipo**. Con el capture de contrato de [`capture.md`](capture.md), la firma entra en `hash` y un cambio de tipo de retorno se detecta. Lo que no se ve es que el tipo **siga llamándose igual y tenga otros campos** — que es exactamente lo que rompió al consumidor que motivó todo esto.

`n` cubre esa fila, y una sola: **el vecindario de nivel 1.**

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
orden = los vecinos por su id de capture, byte-wise. Es el orden en que se escriben
        en `n.1.link`, así que la clave de orden ya está en el archivo.

n.1.hash     = H( hash(v)     de cada vecino, en ese orden )
n.1.hash_ast = H( hash_ast(v) de cada vecino, en ese orden )
```

**La clave de orden tiene que ser identidad, nunca contenido.** Ordenando por el texto, un reformateo le cambiaría el puesto a un vecino, la lista se reordenaría, y el `hash_ast` del nivel se movería sin que ningún AST cambiara — un falso *"cambió de verdad"* producido por el orden.

El id de un capture **es** su identidad: `sha256(file \0 query \0)`. No lleva contenido, así que un reformateo no lo mueve; y cambia exactamente cuando el vecino entra, sale, se muda de archivo o se renombra — que son cambios de contrato.

> Antes la clave era `<layer-root>::<path>` más el nombre del símbolo. Con los vecinos siendo captures esa clave se cae sola: el id ya ordena, ya está escrito, y **es una sola cosa en vez de dos concatenadas**. Tampoco podría ordenar el rango, que lleva offsets y se corre con cualquier edición más arriba del archivo.

**Los vecinos se hashean con el mismo recorte de bordes que un fragmento.** Es la regla de [`capture.md`](capture.md) § "El rango excluye el espacio que rodea al nodo", y vale igual acá: un vecino sin recortar mueve su hash cuando le agregan algo abajo.

### El `hash_ast` de un nivel es todo-o-nada

Si un vecino no tiene gramática no tiene `hash_ast`, y **no puede quedar afuera del fold**: un cambio real en ese vecino movería el `hash` del nivel y no su `hash_ast`, y eso se leería como *"sólo formateo"* cuando no lo fue. Un falso `RESTYLED` es peor que ningún estado.

Así que el campo está presente sólo si **todos** los vecinos tienen gramática. Si a alguno le falta, está ausente, y cualquier cambio en el `hash` del nivel es un cambio real. Es el mismo espíritu que la regla de `hash_ast`: falla hacia reportar, no hacia callarse.

`n` entero también es opcional — ausente donde el fragmento no tiene firma resoluble, que es prosa, un DTO, o un lenguaje sin anotaciones de tipo.

### Los vecinos no son captures

No se acuñan. Nadie los referencia con un `link` ni con un `accepted.link`, así que un capture acuñado para un vecino sería basura que `prune` borra en la pasada siguiente.

Son **ubicaciones** que alguien resuelve y que bilinker hashea al pasar. Con eso desaparecen la conversión rango → query, un modo nuevo de `capture`, una raíz nueva para `prune`, y —la que más importa— el modo de falla del anclaje: `capture` falla cuando no encuentra un ancla única, y con vecinos resueltos por un language server eso pasaría seguido.

### Bilinker no sale a buscarlos

**Es un valor que bilinker guarda y compara sin poder calcularlo por su cuenta.** No es una excepción nueva: es el patrón que el formato ya tiene en [`capture.md`](capture.md) invariante 7, donde un `accepted.link` de endpoint layer o repo lleva una copia opaca de un id ajeno. Se compara, no se resuelve.

Quien los encuentra entra por un puerto que bilinker define y que **no nombra a nadie**:

```rust
pub trait Neighbours {
    fn of(&self, file: &Path, at: &[Position]) -> Result<Option<Vec<Location>>>;
}
```

**Y lo que devuelve se vuelve un capture, no un hash.** Cada ubicación resuelve a un nodo por la regla de siempre —*"la selección sirve para encontrar los nodos, no para recortarlos"*— y lo que queda escrito es un id por vecino, en `n.1.link`.

Hasta acá el vecindario era el único lugar del formato donde **una ubicación externa se hasheaba cruda**, y de esa anomalía salían tres cosas: que el hash cubriera el *nombre* del tipo y no su forma, que un DTO que cambia de archivo mueva el fold sin que nadie sepa por qué, y que no se pudiera preguntar **qué tipos son**. Con captures las tres se caen juntas.

`None` es *"no pude mirar"* y no *"no hay vecinos"* — la distinción de la que sale el estado. El binario le pasa una implementación que le habla a [`lspd`](../../lspd/overview.md); la librería no lo menciona, y **no es para evitar un ciclo** —ya no hay— sino para que bilinker no quede atado a *ese* daemon: mañana puede ser SCIP, un índice propio, o un language server hablado directo.

**El puerto recibe posiciones, no el rango del fragmento.** Es la división que importa: *dónde hay un tipo que preguntar* es gramática, y la gramática es de bilinker; *qué declara ese tipo* es del proveedor. Pasarle el rango lo obligaría a inventar dónde preguntar adentro, y un proveedor que adivina eso está haciendo trabajo que no es suyo con conocimiento que no tiene.

> **No es teórico: fue un defecto.** El puerto recibía rangos y preguntaba en el byte donde cada uno arrancaba. Para un [capture de contrato](capture.md) eso cae justo sobre el tipo de retorno y sobre los parámetros y anda; para un capture de nodo entero cae sobre `pub`, que no declara nada, y el vecindario salía vacío **con el proveedor resolviendo bien**.

**Y el hasheo queda de este lado.** El recorte de bordes es regla de bilinker y vive en el único lugar donde un nodo se convierte en rango. El proveedor devuelve ubicaciones, no hashes.

#### Una lista vacía es una respuesta, y por eso el puerto no se puede defender solo

`Some(vec![])` dice *"miré y esta firma no menciona ningún tipo resoluble"*, y **es legítimo**: una firma de puros primitivos no tiene vecinos. Así que bilinker no puede tratar el vacío como sospechoso — plegarlo da el hash del string vacío y se escribe como vecindario adquirido, que es lo correcto para ese caso.

Lo que no puede es distinguirlo de *"el servidor de atrás todavía no indexó"*, porque **llega igual**. Y ahí el vacío se escribe afirmando una cobertura que no existe, que es exactamente lo que la regla de más abajo prohíbe.

> **La distinción la tiene que dar quien la sabe.** Un proveedor que no puede contestar tiene que decirlo contestando `None`, no un vacío. Bilinker no tiene con qué adivinarlo: si pusiera una guarda contra el vacío, rompería el caso legítimo, y si no la pone, come el ilegítimo. **No hay una tercera opción de este lado del puerto** — es el motivo de que el puerto tenga tres respuestas y no dos.

Del lado de `lspd` eso es [`-32001`](../../lspd/concepts/protocol.md#un-error-es-del-método-no-del-transporte), que el binario traduce a `None`. Un proveedor que ni siquiera pueda saber si está listo —porque el servidor de atrás no lo informa— devuelve lo que tenga: ahí la distinción no se puede dar en ningún lado, y eso es una propiedad del lenguaje, no un defecto que bilinker pueda tapar.

### Cuándo se adquiere el vecindario

El puerto puede contestar `None` —*"no pude mirar"*— y ahí hay que decidir qué se escribe. **La regla es una: una falla de infraestructura no puede reducir la cobertura de un vínculo.**

Que un fragmento *tenga* vecindario se sabe por la gramática y no por el proveedor. Y son **tres** respuestas, no dos:

| | Qué es | Qué se escribe |
|---|---|---|
| **no hay** | prosa, YAML, un lenguaje sin tipos, un DTO, un `enum`, una constante | la ausencia, sin marca y sin pedir nada |
| **hay, y se alcanza** | el fragmento es una firma, o está adentro de una | el `n` calculado sobre sus tipos |
| **hay, y no se alcanza** | el archivo entero, un `impl`, una clase con métodos | `accept` falla; `--no-n1` deja la renuncia escrita |

**Lo que separa la primera de la tercera es si el fragmento contiene firmas que quedan sin cubrir.** Un DTO no tiene ninguna adentro, así que su ausencia es completa y es la correcta — es la misma razón por la que una clase o un DTO nunca estuvieron en la lista de lo que lleva firma. Un archivo entero de Rust tiene muchas y ninguna es la suya: eso no es *"no hay vecindario"*, es *"no puedo recorrer hacia los elementos del próximo nivel"*.

**La tercera es la que faltaba, y su ausencia era una mentira.** Escribirla como ausencia rompía la propiedad que este documento exige más abajo —que la ausencia sin marca tenga **un** significado— sin que nadie se enterara.

Es la misma figura que la readiness de [`lspd`](../../lspd/concepts/language-servers.md#un-servidor-que-no-informa-su-estado-no-se-puede-esperar), y por el mismo motivo: **el tercer valor es el que hace honestos a los otros dos.** Un error que sale ahí tiene que decir *por qué* no se alcanza, porque quien lo lea no tiene cómo deducirlo:

```
Error: el fragmento de crates/bilinker/src/check.rs es el archivo entero, y su
       vecindario no se puede alcanzar: el nivel 1 sale de una firma, y un archivo
       tiene muchas — ninguna es la suya.
       Capturar el contrato con --as, o renunciar al vecindario con --no-n1.
```

### Y el vecindario tiene los mismos dos ejes que el fragmento

Con los vecinos siendo captures, el nivel 1 deja de tener un solo eje:

| eje | se compara | qué dice |
|---|---|---|
| **ubicación** | `n.1.link` contra `accepted[0].n.1.link` | un vecino entró, salió, se mudó de archivo o se renombró |
| **contenido** | el fold de hoy contra `accepted[0].n.1.hash` | la **forma** de un vecino cambió |

Es la misma división que arriba, un nivel más abajo, y con el mismo reparto de escritores: `apply` mantiene `n.1.link`, `accept` escribe la decisión.

#### Y son de verdad independientes: con `unknown` uno tiene valor y el otro no

Que sean dos ejes se ve cuando **se separan**. Un nivel cuyo `link` es [`unknown`](bilink.md#el-link-de-un-nivel-del-vecindario-y-su-tercera-forma) conserva sus dos hashes y no tiene ids:

| eje | Con `link: unknown` |
|---|---|
| **ubicación** | no se puede comparar — no hay ids de un lado. No queda limpio: hay captures que alguien tiene que acuñar |
| **contenido** | **se compara igual**, contra el `hash` conservado. Sin proveedor, no se pudo mirar |

**Y por eso el contrato conservado no es un resto inservible.** Sin ubicación el nivel deja de detectar que un vecino se mudó de archivo o se renombró, y sigue detectando lo que motivó al nivel 1: que **la forma** de un vecino cambió. Es exactamente la mitad que le hacía falta al consumidor que motivó todo esto — *lo que lo rompió fue la forma del tipo, no la firma del método*.

Cuál de los dos ejes nombra el estado, y por qué un cambio real le gana a la ubicación faltante, está en [`check`](../commands/check.md#contract_unlocated-el-contrato-está-y-su-ubicación-no-se-sabe) — es su reparto, no del formato.

**Y por eso `apply` recibe el puerto.** Un vecino cuyo archivo se renombró es un `MOVED` que git resuelve — pero el conjunto también **gana y pierde miembros** cuando la firma cambia, y qué tipo entró sólo lo sabe un language server. La regla queda una:

> Todo comando que toque el eje del vecindario recibe el puerto, y **degrada sin él**.

Vale ya para `check` y `accept`; con `apply` deja de haber excepción. Sin proveedor, `apply` arregla lo del fragmento con git y dice que no pudo tocar el vecindario — lo mismo que hace `accept`. **La frontera del subsistema no se mueve**: la librería sigue siendo git y tree-sitter, y el proveedor entra por el puerto.

De la segunda fila salen **las posiciones que se le pasan al puerto**: se sube hasta la firma que contiene al fragmento y se baja a los campos que llevan tipos —el retorno y los parámetros—, que son los mismos que [`--as interface`](../commands/chain.md#--as-interface-la-firma-sin-el-cuerpo) captura. Un capture de contrato y un capture de la función entera terminan preguntando **en las mismas posiciones**, que es como tiene que ser: el vecindario es de la firma, no de cómo se la haya capturado.

Con eso, sólo la tercera fila es un problema.

Y hay un dato más que se sabe sin daemon: **el conjunto de vecinos lo determina la firma, y la firma está en el fragmento.** Con el [capture de contrato](capture.md) el `hash` del fragmento es el de la firma, así que un `hash` que no se movió es el mismo conjunto de vecinos — aunque su contenido no se haya podido mirar.

**Y el `n` previo tiene tres valores, no dos.** Puede estar adquirido, puede ser una renuncia escrita, o puede no estar — y los tres son entradas distintas, porque una renuncia **es una decisión que alguien tomó** y no la ausencia de una.

| `n` previo | Se pudo resolver | Cambió la firma | Qué se escribe |
|---|---|---|---|
| ausente | sí | — | `n` calculado |
| ausente | **no** | — | nada, y `accept` falla |
| `declined` | sí | — | `n` calculado — la renuncia **se levanta sola** |
| `declined` | **no** | — | **la renuncia que ya estaba**, intacta |
| adquirido | sí | — | `n` recalculado |
| adquirido | **no** | **no** | **el `n` que ya estaba**, intacto |
| adquirido | **no** | **sí** | nada, y `accept` falla |

**Las dos filas de `declined` son la misma idea que las de adquirido**: sin proveedor no hay información nueva con la cual revisar lo que ya se decidió, así que se conserva. Volver a pedirla convertiría la renuncia en algo que se tipea en cada `accept`, y **un pedido que sale siempre no lo lee nadie** — la misma razón por la que el aviso no sale sobre prosa ni sobre un DTO. La guarda se gastaría a fuerza de dispararse cuando no hace falta, y el día que sí baje una cobertura, se la tipea sin leer.

**Y conservarla no encierra a nadie**, que es lo que la hace segura: la tercera fila dice que con proveedor la renuncia se levanta sin que nadie pida nada. La asimetría es la correcta — **subir cobertura es automático, bajarla sigue pidiendo que se declare.**

**La sexta fila es la que evita el daño.** Preservar es estrictamente más seguro que borrar: si un vecino cambió mientras el proveedor estaba caído, el `n` viejo sigue ahí y el próximo cierre con proveedor lo reporta. Borrándolo, ese cambio se absorbe en el baseline nuevo y **deja de ser detectable para siempre**.

**Y la séptima es la única donde preservar sería mentir**: si la firma cambió, el conjunto de vecinos pudo cambiar con ella, y el valor viejo es sobre un conjunto que ya no es el de hoy. No tiene gemela en `declined` porque **una renuncia no es sobre un conjunto**: no dice qué vecinos había, dice que no se vigilan. Que la firma cambie no la vuelve falsa.

No es un mecanismo nuevo: [`--place` y `--content`](#--place-y---content) ya aceptan una dimensión sin tocar la otra. El vecindario es una tercera y se comporta igual. Lo que no puede pasar es que una dimensión se borre **como efecto colateral** de aceptar otra.

### `n: declined` es lo que vuelve determinista la renuncia

Renunciar al vecindario tiene que poder decirse, y las dos filas que fallan se destraban [declarándolo](../commands/accept.md#--no-n1). Eso escribe `n: declined`, y **el campo no es cosmético: sin él se rompe la invariante 4.**

Sin marca, el mismo fragmento en el mismo estado produce un `accepted` **con** `n` adquirido o **sin** él según si había un language server prendido en esa máquina. La determinación la tomaría el ambiente, que no es parte del estado del fragmento. Y se lleva puesta la propiedad que puso a `n` en `accepted` y no en la cache: que *"dos personas que aceptan el mismo vecindario en el mismo estado escriben el mismo valor"*.

Con la marca, quien decide es el flag —igual que `--place` y `--content`— y la convergencia vuelve: dos personas que renuncian escriben lo mismo, y quien no renuncia no puede escribir un baseline mudo sin saberlo.

**Y es lo único que cruza la frontera.** El consumidor recibe una copia opaca del `accepted` del endpoint `repo` y no puede volver a mirar la gramática del fragmento ajeno para reconstruir el motivo. Sin la marca, *"el proveedor no tiene vecindario"* y *"el proveedor renunció a vigilarlo"* le llegan idénticos.

**La ausencia sin marca queda con un solo significado**: el fragmento no tiene firma resoluble. Eso es prosa, un DTO, o un lenguaje sin anotaciones de tipo — derivable de la gramática por cualquiera, siempre igual.

Y por eso el caso *"hay y no se alcanza"* **no puede escribirse como ausencia**: escribiría *"no hay firma"* sobre un archivo lleno de firmas, y le daría a la ausencia un segundo significado que ningún lector puede separar del primero. Ese caso es una renuncia, y va con su marca como cualquier otra.

> **Y sube la versión de formato.** Un parser que no conozca el campo lee esa aceptación como *"este fragmento no tiene vecindario"*, en silencio y sin fallar — que es exactamente el caso que [`format-version.md`](format-version.md) describe. Ver también [frontier.md](frontier.md) § "que siga siendo `abstract`", donde `link: abstract` tiene la misma forma.

### Van en `accepted`, no en la cache

Dos razones, y las dos son las que decidieron dónde va cada campo del formato.

**La frontera.** Lo que cruza entre repos es la copia opaca de `accepted` del endpoint `repo`. Si `n` no está ahí, **no cruza** — y el caso que motiva todo esto se queda sin el mecanismo que lo entrega al consumidor.

**No es recuperable.** Reconstruirlo pediría un language server indexando un checkout histórico, que puede ni buildear. Ahí está la diferencia con [`commit`](cache.md), que se queda en la cache porque su recuperación es *"más lento, nunca no disponible"*. **Un valor cuya reconstrucción depende de infraestructura que puede no estar no es un derivado: es una decisión.**

Y pasa el otro test: **converge.** Dos personas que aceptan el mismo vecindario en el mismo estado escriben el mismo valor, porque sale de hashear contenido — a diferencia de `commit`, que nunca converge. Así que no le mete a [`adopt`](../commands/adopt.md) un campo que diverja siempre.

### Los cuatro cuadrantes

Dos ejes independientes, y las cuatro combinaciones dicen cosas distintas:

| Difiere | Qué significa |
|---|---|
| `hash` | el fragmento se reformateó |
| `hash` + `hash_ast` | el proveedor cambió lo que el fragmento dice |
| `n.1.hash` | el vecindario se reformateó |
| `n.1.hash` + `n.1.hash_ast` | **un vecino cambió: el contrato se movió** |

La última es el caso que motivó todo: el método intacto, el DTO movido.

**Y la segunda dejó de ser un problema por otro lado.** Antes decía *"el proveedor refactorizó por dentro"* y era ruido que le llegaba al consumidor; con el capture de contrato el cuerpo ya no entra en `hash`, así que ese refactor no mueve nada. La resolvió [`capture.md`](capture.md), no el estado.

## Aceptar exige el fragmento commiteado

`accept` **falla sobre un archivo sucio.**

No es una recomendación: es lo que hace posible calcular `commit`, que es el commit en el que el fragmento quedó con el contenido aprobado. Ese commit no existe si el fragmento no está commiteado.

Y es lo que garantiza que lo aprobado sea recuperable. Sin commit no hay `git show <commit>:<file>`, y sin eso `check` no puede recuperar el texto aceptado — que es lo que distingue `EXPANDED` y `REANCHORED` de un `ALTERED` genérico.

## Quiénes aprobaron

`agree` es el set de quienes aprobaron **exactamente estos valores**. Como los valores direccionan por contenido, *"estar de acuerdo"* no es ambiguo: es haber aprobado este hash, esta ubicación **y este vecindario**, y no otros.

**La identidad de una entrada es su tupla entera**: `link`, `hash`, `hash_ast` y `n`. Dos personas que aprueban el mismo fragmento con vecindarios distintos no comparten entrada: son dos contratos, y por lo tanto [dos entradas](bilink.md#una-entrada-es-completa-y-por-eso-lleva-un-solo-agree). Y no hay endoso parcial de una entrada — con firma resoluble y sin proveedor `accept` se niega, así que *"aprobé la firma y el vecindario no lo miré"* no es un estado alcanzable.

> **Y la tupla se nombra entera o no se nombra.** Esta sección decía *"este hash y esta ubicación"*, y el código comparaba cuatro campos desde que el vecindario existe: `hash_ast` y `n` estaban afuera de la enumeración, en dos párrafos, sin que nada lo detectara. **Una enumeración en prosa es lo que envejece cuando se agrega un campo** — la invariante 9 de esta página zafó justamente por decir *"algún valor"* en vez de listar.

**Por endpoint y local, nunca copiado.** En un endpoint estructural están los que aprobaron ese fragmento; en un endpoint `path` o `repo`, los que aprobaron **esa copia**. Quién aprobó del otro lado de la cadena es un hecho de la otra capa, y traerlo acá sería atribuir mal. Los dos endpoints de un bilink pueden tener listas distintas, y es lo normal.

**Quien escribe el `accepted` entra solo.** Si no, el campo significaría "los que *además* aprobaron", que es otra cosa.

**Es un set: ordenado y único al serializar.** Un duplicado o un reordenamiento producirían un diff que no dice nada.

**Y si cambia cualquiera de los cuatro, no se arrastra a nadie: se abre otra entrada.** Es lo que *"estos valores"* significa — quien aprobó los valores anteriores no aprobó los nuevos, y poner su nombre en la entrada nueva sería atribuirle una decisión que no tomó.

Lo que cambió es **qué pasa con la entrada vieja**. Antes se pisaba, y la aprobación anterior sólo quedaba en el commit que la había escrito. Ahora la entrada nueva se **suma** y el endpoint queda [`CONSENSUS_DIVERGED`](bilink.md#más-de-un-accepted-es-un-estado-no-una-forma-de-trabajar): las dos decisiones están, visibles, y `check` falla hasta que alguien resuelva.

**El destino final de la desplazada es el mismo de siempre.** Cuando alguien resuelve, la entrada que aprobaba otros valores se va, y donde queda es donde siempre quedó: en los commits que la escribieron. La lista no es un archivo histórico — es la ventana entre dos aceptaciones, que antes era invisible.

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
11. **Ningún `accept` reduce la cobertura de un endpoint sin que alguien lo haya pedido.** Que el proveedor de vecindario no conteste nunca borra un `n` adquirido: o se preserva, o `accept` falla.
12. Un `accepted` **sin** `n` afirma que el fragmento **no tiene firma resoluble**. La renuncia se escribe, no se omite.
