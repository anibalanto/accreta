# Propuesta: el cierre de firma como dimensión aceptada

**Estado:** en discusión. No especificado, no implementado. Vive acá para que el razonamiento no se pierda.

La forma quedó cerrada: **dos campos plegados**, `hash_n1` y `hash_ast_n1`. La variante que guardaba los vecinos uno por uno se descartó por complejidad y queda registrada acá abajo, porque sigue siendo el upgrade aditivo si la atribución llega a doler.

Un fragmento no es sólo un árbol de sintaxis: devuelve un tipo, recibe parámetros, llama a otras funciones. **¿Tiene sentido que la aceptación cubra algo de eso, y hasta dónde?**

El documento arranca por la respuesta general. Si lo que se busca es sólo que **quien usa una API se entere de que cambió**, hay dos caminos que no necesitan nada nuevo: ver [Tres formas de contestar la misma pregunta](#tres-formas-de-contestar-la-misma-pregunta).

## El antecedente: ya se intentó, y se revirtió

`subgraph.N` y los sciplinks persistían el call graph en archivos versionados. Se sacaron del formato, y [`lattice/concepts/edge.md`](../../lattice/concepts/edge.md) § "Frescura" registra por qué:

> Un índice derivado bajo control de versiones se desincroniza del código y reporta drift que no existe.

**Cualquier propuesta en esta dirección arranca teniendo que explicar por qué no es eso otra vez.**

## Lo que la evidencia dice que importa

El caso de [`1d`](../../../.stratum/worklist/1d.task.md): `retinar` consume un endpoint de `hsi` con el DTO equivocado. Lo que hay que notar es **qué habría detectado cada cosa**:

| Qué se hashea | ¿Detecta el problema? |
|---|---|
| el texto del método | **no** — el método no cambió |
| los tokens del método | **no** — por lo mismo |
| la forma que devuelve | **sí** — es exactamente lo que difiere |

Lo que rompió al consumidor no fue el cuerpo. Fue el **tipo de retorno y sus campos**. Un hash de contenido es ciego justo a la falla que ocurrió.

## Tres direcciones, y sólo una termina

Desde un fragmento salen tres vecindarios, y conviene no tratarlos como uno:

| Dirección | Qué es | Tamaño |
|---|---|---|
| **hacia afuera** — tipo de retorno, tipos de los parámetros, y sus campos transitivamente | **el contrato** | cerrado y chico: termina en primitivas |
| **hacia adentro** — a qué llama el cuerpo | la implementación | **no termina**: todo alcanza a todo |
| **hacia arriba** — quién lo llama | el alcance de un cambio | no termina, y es otra pregunta |

Sólo el primero es finito, y es el único que un consumidor necesita. *Al que lo usa le interesa la forma que devuelve, no la semántica interna* — y eso, dicho con precisión, es **el cierre de la firma**.

> El "transitivamente" de esa primera fila es lo que [la forma concreta](#la-forma-concreta) termina cortando en un salto. Que el cierre sea finito no quiere decir que convenga recorrerlo entero: la profundidad 1 cubre el caso que motivó el documento y evita tener que decidir dónde para.

Guardar el cuerpo tiene tres problemas encadenados: no termina, obliga a decidir sobre código que nadie leyó, y dispara `CHAIN_DIRTY` en refactors que no cambiaron nada observable. Ese ruido es exactamente lo que mató a `subgraph.N`.

## Por qué el revert no lo prohíbe

Lo que se revirtió fue **persistir un índice**. Un índice afirma algo sobre el código *actual*, y por eso se desincroniza.

Un **hash** no afirma nada sobre el código actual: registra **qué se aprobó**. Es lo que `accepted.hash` ya es — un valor derivado del código, congelado en el momento de una decisión, que justamente sirve porque el código puede alejarse de él.

La diferencia no es de grado: un índice que envejece miente; un hash que envejece **es el mecanismo**.

## La forma concreta

**Dos campos** en `accepted`, al lado de los dos que ya hay:

```yaml
accepted:
  link: capture 8f2a4c6e…
  hash: c4e1770b…          # el texto del fragmento
  hash_ast: 1b9e44a2…      # sus tokens
  hash_n1: 96c765b9…       # el texto de sus vecinos directos      ← nuevo
  hash_ast_n1: 88e834c4…   # los tokens de sus vecinos directos    ← nuevo
```

### Nivel 1: no hay cierre, hay vecindario

El `_1` es la decisión más importante de esta forma: **no recursa.** Los vecinos son los tipos que la firma menciona, un salto, y nada más. Los campos de esos tipos son nivel 2 y no entran.

Cubre menos, y es a propósito, porque con eso tres problemas dejan de existir:

- **Dónde para el cierre** — para en 1, y está escrito en el nombre del campo.
- **La terminación con tipos recursivos** — no hay recursión que terminar.
- **El recorrido** — no hay BFS ni visited-set: se resuelven los tipos que la firma menciona y listo.

Y la profundidad queda **explícita y opt-in por bilink**, en vez de ser una política global escondida. Si alguna vez hace falta, un `hash_n2` es aditivo y no invalida nada.

Qué ve y qué no:

| Cambio | ¿Lo ve? |
|---|---|
| a `PublicAuthorityDto` le agregan un campo | **sí** — cambió su texto |
| `String name` → `AuthorityKind name` en el DTO | **sí** — cambió su texto |
| el DTO se muda de archivo | **sí** — cambió el conjunto |
| el método devuelve otro tipo | **sí** — cambió el conjunto |
| a `AuthorityKind` le cambian los valores | **no** — es nivel 2 |

La primera fila es [`1d`](../../../.stratum/worklist/1d.task.md). **El nivel 1 detecta el caso que motivó todo esto**, que es lo que hay que pedirle a la forma más barata.

### Los vecinos no son captures

No se acuñan. Nadie los referencia con un `link` ni con un `accepted.link`, así que un capture acuñado para un vecino sería basura que `prune` borra en la pasada siguiente.

Son ubicaciones que lattice resuelve y que bilinker hashea al pasar. Con eso desaparecen cuatro cosas: la conversión rango → query, un modo nuevo de `bilinker capture`, cualquier raíz nueva para `prune`, y —la que más importa— **el modo de falla del anclaje**: `capture` falla cuando no encuentra un ancla única, y con vecinos resueltos por LSP eso iba a pasar seguido (clases anónimas, código generado).

Lo único que hay que respetar es el recorte de bordes de [`capture.md`](../concepts/capture.md) § "El rango excluye el espacio que rodea al nodo". Un vecino hasheado sin recortar mueve su hash cuando le agregan algo abajo, que es exactamente el falso drift que esa regla existe para evitar.

### El fold

Un solo orden, y dos folds sobre ese orden:

```
orden = los vecinos ordenados por su identidad — <layer-root>::<path> + nombre del símbolo,
        byte-wise sobre el UTF-8. Esa clave ordena y no entra en ningún hash.

hash_n1     = H( hash(v)     de cada vecino, en ese orden )
hash_ast_n1 = H( hash_ast(v) de cada vecino, en ese orden )
```

**La clave de orden tiene que ser identidad, nunca contenido.** Si se ordenara por el texto del vecino, un reformateo podría cambiarle el puesto a uno, la lista de `hash_ast` se reordenaría, y `hash_ast_n1` se movería sin que ningún AST cambiara: un falso *"cambió de verdad"* producido por el orden y no por el código. Ordenando por identidad nadie se mueve de puesto salvo que un vecino entre, salga o se renombre — y esas tres cosas **son** cambios de contrato.

Tampoco puede ordenar el rango: lleva offsets de bytes, que se corren con cualquier edición más arriba del archivo.

### `hash_ast_n1` es todo-o-nada

Si un vecino no tiene gramática no tiene `hash_ast`, y **no puede simplemente quedar afuera del fold**: un cambio real en ese vecino movería `hash_n1` y no `hash_ast_n1`, y eso se leería como *"sólo formateo"* cuando no lo fue. Un falso `RESTYLED` es peor que ningún estado.

Así que el campo está presente sólo si **todos** los vecinos tienen gramática. Si a alguno le falta, está ausente, y cualquier cambio en `hash_n1` es un cambio real. Es el mismo espíritu que la regla que ya existe —*"donde `hash_ast` no está, `RESTYLED` no existe y todo cambio de texto es `ALTERED`"*— y falla hacia reportar, no hacia callarse.

`hash_n1` también es opcional: ausente donde el fragmento no tiene firma resoluble, que es prosa, un DTO, o un lenguaje sin anotaciones.

### Los cuatro cuadrantes

Dos ejes independientes, y las cuatro combinaciones dicen cosas distintas:

| Difiere | Qué significa |
|---|---|
| `hash` | el fragmento se reformateó |
| `hash` + `hash_ast` | **el proveedor refactorizó por dentro y no rompió a nadie** |
| `hash_n1` | el vecindario se reformateó |
| `hash_n1` + `hash_ast_n1` | **un vecino cambió: el contrato se movió** |

La segunda fila es la que hace adoptable la frontera: el cuerpo cambió, el vecindario no, y el consumidor no se entera de un refactor que no le incumbe. Hoy eso le llega como `CHAIN_DIRTY` y lo obliga a mirar algo que no le incumbe.

La cuarta es `1d`: el método intacto, el DTO movido.

Y en los endpoints layer y repo los dos campos se copian como los otros dos: **dos escalares más** en el valor opaco que se toma del vecino. No hay regla nueva de propagación.

### Lo que se descartó en el camino: el mapa de vecinos

Antes de llegar a los dos hashes, esta propuesta pasó por una forma que guardaba **los vecinos uno por uno** en el bilink: un conjunto declarado al lado de `link`, y un mapa de aceptación adentro de `accepted`, con `hash` y `hash_ast` por vecino.

```yaml
endpoint:
  0:
    link: capture bed60efb…
    vecinos:                       # el conjunto declarado · lo escribe apply
      50e81a05…:
      b86a579f…:
    accepted:
      link: capture bed60efb…
      hash: 96c765b9…
      vecinos:                     # la aceptación, por vecino · la escribe accept
        50e81a05…: { hash: 311285da… }
        b86a579f…: { hash: 1d44d20e…, hash_ast: 975b8302… }
```

Funcionaba, y daba algo que los dos hashes no dan: **atribución**. `check` puede decir *"`PublicAuthorityDto` quedó `ALTERED`"* en vez de *"el vecindario se movió"*.

**Se descartó por complejidad**, y no por una razón sino por seis que se acumulan:

- **La clave no puede ser posicional.** `0, 1, 2` sobre un conjunto dinámico hace que la posición 1 pase a nombrar otra cosa cuando el conjunto cambia, y la aceptación queda comparando el hash de un fragmento contra el contenido de otro. No es un falso positivo: es una **atribución equivocada** — `check` reporta drift en el vecino que no cambió y calla el que sí.
- **Lo cual obliga a keyear por capture id, y eso obliga a acuñar un capture por vecino**, con su query, su verificación de unicidad y su modo de falla cuando no hay ancla estable.
- **`prune` gana una clase de raíz.**
- **YAML no tiene set usable.** El `!!set` de YAML 1.1 no lo soporta `serde_yaml_ng`, así que la unicidad tiene que salir de las claves de un mapa: el conjunto declarado queda como un mapa de valores vacíos, o como una lista que **puede representar un duplicado** y hay que validar al leer.
- **La regla de copiado de los endpoints layer se reescribe entera**: el valor opaco que se copia del vecino pasa de tres escalares a tres escalares más N sub-bloques que nunca se resuelven localmente.
- **El bilink pasa de doce líneas a ochenta**, y cada aceptación produce un diff grande. El archivo deja de leerse de un vistazo, que hoy es una propiedad real del formato.

Los dos hashes cuestan **dos filas en una tabla de campos**, y cubren exactamente los mismos cambios: se hashea lo mismo, sólo que plegado.

**Y el mapa queda disponible como upgrade aditivo.** Como el fold se define sobre los *valores* y no sobre las claves, `hash_n1` es el mismo valor calculado desde el mapa: un bilink que sólo tiene los dos hashes sigue siendo válido, y lo único que no tiene es atribución hasta que alguien lo re-acepte. **El barato es un prefijo válido del caro**, así que no hay ninguna razón para no empezar por el barato.

Lo que se pierde mientras tanto, dicho sin maquillaje: cuando `hash_n1` difiere sabés que el vecindario se movió, no cuál vecino. Corrés lattice y ves los vecinos de hoy con sus hashes, pero no tenés los aceptados de cada uno para diffear. En teoría se re-derivan desde `accepted.commit`; en la práctica eso pide **un LSP indexando un checkout histórico**, que no es la misma clase de costo que re-derivar `commit` con tree-sitter sobre un blob de `git show`. Con tres vecinos es cómodo; con quince, incómodo.

## Quién calcula, quién decide, quién guarda

Resolver un tipo hasta su declaración es trabajo de language server, no de tree-sitter. Y la frontera de bilinker es explícita: *sólo git y tree-sitter; el call graph vive en lattice, el alcance de un cambio en impact.*

**Bilinker no le pregunta a nadie.** La tentación es que `bilinker check` llame a lattice, y hay que resistirla: bilinker hoy funciona con git y tree-sitter, siempre, offline, en cualquier repo. Un `check` que necesita un language server indexando queda condicionalmente degradado, y eso contamina el subsistema entero por un campo. Además invierte las capas — lattice consume bilinker vía `bilinker graph`, no al revés.

El reparto que sí cierra:

| | Quién | Qué hace |
|---|---|---|
| encontrar los vecinos | **lattice** | `textDocument/definition` sobre los tipos de la firma. Un salto. No persiste nada. |
| componer y decidir | **impact** | es el único que ya consume los dos; recorre, evalúa, e invoca |
| hashear y escribir | **bilinker**, invocado | recibe ubicaciones, aplica el recorte de bordes, foldea, escribe `accepted` |
| comparar en `check` | **bilinker** | compara el valor guardado contra el que le traen. Nunca resuelve un tipo. |

Bilinker recibe las ubicaciones; no sale a buscarlas. El hasheo queda de su lado porque el recorte de bordes es su regla y vive *"en el único lugar donde un nodo se convierte en rango"*.

### Para bilinker es un valor opaco

`hash_n1` es un campo que bilinker **guarda y compara sin poder calcularlo por su cuenta**. No es una excepción nueva: es el patrón que el formato ya tiene en [`capture.md`](../concepts/capture.md) invariante 6, donde un `accepted.link` de endpoint layer o repo contiene *"una copia opaca de un id ajeno, que no se resuelve localmente"*. Se compara, no se resuelve.

De ahí sale el estado que hay que nombrar: cuando nadie le pasa el valor de hoy, `check` **no dice OK ni dice drift — dice no verificado.** Es la familia de `LAYER_UNREACHABLE` y `REMOTE_UNREACHABLE`: no pude ver el otro lado. Y es la distinción que impact ya defiende para sí mismo — *"un reporte que no distingue 'no encontré impacto' de 'no pude buscarlo' es peor que no tener reporte"*.

### Y por eso va en `accepted`, no en la cache

Se podría pensar en dejar el vecindario del lado de impact y que bilinker no se enterara. Dos razones para no hacerlo:

**La frontera.** `retinar` consume `hsi` cruzando una frontera de repos, y lo que atraviesa esa frontera es la copia opaca de `accepted` del endpoint `repo`. Si `hash_n1` no está ahí, no cruza — y el caso que motivó este documento se queda sin el mecanismo que lo entrega al consumidor.

**No es re-derivable en la clase de costo de la cache.** `commit` vive en la cache y se re-deriva con tree-sitter sobre un blob de `git show`: *"más lento, nunca no disponible"*. Recuperar el `hash_n1` aceptado pediría **un LSP indexando un checkout histórico**, que puede ni buildear. No es un derivado recuperable: es una decisión, y va versionada.

### Lo que le falta a lattice

El proveedor `lsp` y la forma canónica de nodo ya están. Falta un `kind` que **no** es `call`: algo que exprese *"este fragmento menciona este tipo en su firma"*.

Y el recorrido es un salto, no un grafo. Las tres menciones de `Persona` en `Persona hijoMenor(Persona padre, Persona madre)` resuelven a la misma declaración, así que **la dedup la hace el LSP** y no una regla del formato — el vecindario tiene un `Persona` y no tres.

**Ésa es la composición que el modelo ya permite y nadie usó todavía:** un proveedor derivado alimentando una aceptación. Hoy `accepted` sólo se nutre de lo que bilinker mismo puede ver.

## El revisor es otra pregunta, y ya tiene dueño

Alguien revisando *"¿este cambio rompe el significado?"* sí necesita la semántica interna. Pero eso no es *"¿qué aprobé?"* sino *"¿hasta dónde llega esto?"* — y el ecosistema ya se lo asigna a **impact**.

Dicho corto: **el contrato se acepta; el alcance se consulta.** Meter los dos en `accepted` es lo que vuelve infinito el cierre.

## Lo que no está resuelto, y es el problema difícil

**Igualdad no es compatibilidad.**

Agregarle un campo a un DTO **no rompe** a un consumidor que deserializa JSON: es aditivo, y el consumidor viejo lo ignora. Pero un hash de igualdad dispara igual, y el consumidor tiene que ir a mirar un cambio que no le hace nada.

Si eso no se resuelve, `hash_n1` reproduce el ruido que mató a `subgraph.N`, sólo que un nivel más arriba — y peor, porque ahora el ruido cruza la frontera y llega a gente de otro equipo.

**Es la única pregunta que decide si esto vale la pena, y no la resuelve ninguna variante.** Los dos hashes y el mapa de vecinos disparan con la misma frecuencia ante un cambio aditivo: lo que el mapa agrega es atribución, no una noción de compatibilidad. La diferencia entre ellos es cuántos pasos hay entre el estado y entender qué pasó, no si el estado aparece.

Lo demás que queda por decidir es chico:

1. **El vocabulario de los dos estados nuevos.** `RESTYLED` y `ALTERED` ya nombran las dos filas del fragmento; hacen falta dos nombres para las del vecindario que digan que el eje es otro.
2. **Qué cuenta como vecino directo donde el fragmento no es una función** — un DTO, una interfaz, prosa. La respuesta por defecto es que el campo está ausente, pero conviene decirlo antes de que alguien lo descubra.

### Preguntas que esta forma retira

- ~~**¿Dónde para el cierre?**~~ En 1, y está en el nombre del campo. Con eso también se cae la terminación con tipos recursivos, que era su consecuencia peor.
- ~~**¿Un cierre o dos?**~~ No se puede tener dirección: en `Persona hijoMenor(Persona padre, Persona madre)` el mismo tipo es retorno **y** parámetro, así que un conjunto de tipos no distingue lo que se devuelve de lo que se recibe. Y no hace falta — la asimetría de compatibilidad (agregar un campo a un retorno es seguro, agregar un parámetro requerido no) no es una propiedad del tipo sino de cómo lo usa el consumidor, y eso lo decide quien mira, como en todo el resto del diseño.
- ~~**¿Y los lenguajes sin tipos?**~~ Donde no hay firma resoluble el campo está ausente, igual que `hash_ast` donde no hay gramática.

## Tres formas de contestar la misma pregunta

*«Que quien usa la API se entere de que cambió»* tiene tres respuestas de costo muy distinto, y la más cara es la de arriba.

### A — Publicar el DTO. Funciona hoy, sin nada nuevo.

**En un DTO el problema no existe**, y es el detalle que vuelve barata la opción: un DTO no tiene cuerpo. `PublicAuthorityDto` es tres campos y anotaciones de Lombok — capturarlo *es* capturar la forma, sin una sola línea de implementación que genere ruido.

O sea que la tercera dimensión de esta propuesta **no hace falta donde el fragmento ya es sólo firma**. Interfaces, DTOs, records, declaraciones de tipo: ahí la aceptación de contenido que bilinker ya tiene alcanza y sobra.

Lo que **no** cubre: la ruta. Si `hsi` mueve el endpoint sin tocar el DTO, el consumidor no se entera — y ése fue justamente el error de `retinar`.

### B — Publicar el contrato ya generado

El `openapi.json` de springdoc **es** el contrato: lo emite el mismo framework que sirve las rutas, así que no puede divergir de lo que se sirve — a diferencia de un DTO espejado a mano, que es exactamente lo que se desincronizó en `retinar`.

Y tiene una ventaja que el código Java no puede dar: **ahí la ruta es una clave literal.** En el fuente, `/public-api/user/permissions/from-token` no existe como string — se compone de un `@RequestMapping` de clase y un `@GetMapping` de método, y por eso [`1d`](../../../.stratum/worklist/1d.task.md) lo dio por inexistente en la primera pasada. En el JSON generado es una clave de objeto, que es un ancla estable de las que [`reference.md`](../concepts/reference.md) ya recomienda.

**El costo no es la gramática.** Agregar tree-sitter-json es barato. El costo es que el `openapi.json` **hay que commitearlo**, y hoy no lo está: springdoc lo genera en runtime.

Y ahí reaparece el fantasma de `subgraph.N` — un derivado bajo control de versiones. Pero **no es el mismo caso**, y la diferencia importa:

| | `subgraph.N` | `openapi.json` commiteado |
|---|---|---|
| Quién lo derivaba | bilinker, de callado, en sus propios archivos | el build del proveedor |
| Quién nota que quedó viejo | nadie | el CI del proveedor, si lo chequea |
| De quién es el problema | de la herramienta | de quien publica su contrato |

Sigue siendo un artefacto generado que alguien tiene que mantener al día. La diferencia es que pasa a ser **trabajo declarado de alguien** en vez de un índice escondido, y eso es un patrón conocido y verificable en CI.

> **La trampa: `$ref`.** En un OpenAPI normal el objeto de una ruta no *contiene* su esquema, lo referencia por string. Capturar la ruta no cubre la forma, y estamos en el mismo problema de cierre, ahora en JSON.
>
> **Y tiene solución fuera de bilinker:** generar el spec *dereferenciado*, con los `$ref` resueltos inline. Ahí el objeto de la ruta contiene su esquema literalmente, y **una sola captura cubre ruta y forma**. El cierre lo resuelve el generador, que es quien sabe hacerlo.

### C bis — Que lattice **proponga** el conjunto de fragmentos

Una variante de C que llega a lo mismo por otro lado, y que en un punto es mejor.

En vez de plegar el cierre en **un hash opaco**, que el recorrido de lattice produzca **N capturas** —una por fragmento alcanzado— y que cada una tenga su bilink, su `accepted` y su estado.

**No es un campo del capture.** Un capture es `{file, query}` y su id es el hash de esos dos campos: agregarle algo le cambia la identidad y se cae la inmutabilidad por construcción y la deduplicación. Lo que lattice aporta es **un modo del comando** —de dónde sale la lista de fragmentos a capturar— y no un dato adentro del artefacto.

Y encaja con lo que el formato ya dice: *"la multiplicidad la aporta el capture"*. Que un recorrido produzca varias no rompe nada; lo que hay que cuidar es quién decide.

#### La ventaja sobre C: se ve qué se movió

| | C — el vecindario plegado | C bis — N capturas |
|---|---|---|
| Cuando el contrato cambia | *"el vecindario se movió"* | **`PublicAuthorityDto` quedó `ALTERED`** |
| Para saber qué pasó | hay que ir a buscarlo | ya está dicho |
| Estados | uno | uno por fragmento |

Un hash de cierre dice *que* algo se movió; N capturas dicen **cuál**. Para alguien que tiene que decidir si su código sigue andando, la segunda es la útil.

#### El riesgo, y cómo se evita

Si un recorrido acuña 20 capturas y una aceptación las bendice a todas, alguien acaba de aprobar 20 fragmentos que no leyó. Eso es exactamente lo que vuelve inaceptable guardar el cuerpo entero.

**La salida es la que el proyecto ya usa en otro lado: `apply` propone, `accept` dispone.** El recorrido produce una *propuesta* —*«tu firma alcanza estos 6 fragmentos, ¿cuáles capturo?»*— y la persona elige. El conjunto es derivado; la decisión sigue siendo humana, fragmento por fragmento.

**Y el conjunto no se persiste nunca.** Se recalcula cuando alguien lo pide. Si mañana la firma alcanza un tipo más, eso aparece como una propuesta nueva —*«hay un fragmento más que antes no estaba»*— y no como un índice que creció solo. Es la única forma de no repetir `subgraph.N`: lo que persiste son las capturas individuales y sus aceptaciones, que es lo que ya persiste hoy.

#### Lo que le falta al modelo

**Las N capturas no tienen dónde agruparse.** Un bilink tiene exactamente dos endpoints, y ésa es la invariante más defendida del formato. Seis fragmentos son seis bilinks, y hoy nada dice que los seis son *un* contrato.

Expresarlo pide el endpoint de tipo bilink de [`bilink-endpoint.md`](bilink-endpoint.md) —*"D gobierna el vínculo entre A y B"*— que está especificado y no implementado. Sin eso, el conjunto existe sólo en la cabeza de quien lo creó, y el consumidor ve seis vínculos sueltos en vez de un contrato.

**Es la dependencia real de esta variante**, y conviene saberlo antes de empezar: sin agrupación, C bis da más información y menos sentido.

#### Y es la ergonomía de A

Vale notar que esto no compite con [A](#a--publicar-el-dto-funciona-hoy-sin-nada-nuevo): **es lo que la vuelve practicable.**

A dice *«publicá el DTO, anda hoy»*. Lo que A no resuelve es **cuáles** DTOs — hoy hay que salir a buscarlos a mano, que es exactamente lo que esta sesión hizo a fuerza de `grep` y se equivocó una vez. Un `capture --propose` contesta eso.

O sea: A es qué se captura, C bis es cómo se encuentra. **Se pueden hacer en ese orden**, y la primera no necesita a la segunda para empezar.

### C — El vecindario de nivel 1 vía lattice

Lo de arriba: `hash_n1` y `hash_ast_n1`. Es la única que sirve donde **no hay un spec generado** — código que no expone una API HTTP, contratos entre módulos, cualquier cosa que no tenga un OpenAPI detrás.

Y ya no es *"el cierre de firma"* en sentido estricto: al clavar la profundidad en 1 dejó de ser un cierre transitivo y pasó a ser un vecindario. Ahí está casi todo el ahorro — y el nombre del documento quedó de cuando la idea era la otra.

### Cuál conviene

| | Cubre la forma | Cubre la ruta | Cuesta |
|---|---|---|---|
| **A** publicar el DTO | sí | **no** | nada — anda hoy |
| **B** publicar el openapi | sí | **sí** | gramática JSON + commitear un generado |
| **C** vecindario de nivel 1 | sí, plegado | no aplica | lattice + dos campos en `accepted` |
| **C bis** lattice propone N capturas | sí, **y dice cuál se movió** | no aplica | lattice + agrupación (endpoint bilink) |

**A para empezar mañana; B cuando la ruta importe; C donde no haya spec generado; C bis como el upgrade aditivo de C, si la atribución llega a doler.**

Y una corrección a lo que este documento afirmaba antes: **C bis no resuelve la compatibilidad, y C no la empeora.** Los dos disparan igual ante un cambio aditivo; lo que C bis agrega es atribución. El problema difícil nunca fue el plegado, así que no se disuelve al desplegar — ver [Lo que no está resuelto](#lo-que-no-está-resuelto-y-es-el-problema-difícil). Y B no reemplaza a C: sirve donde hay un spec generado, que es una parte del problema y no todo.

## Qué haría falta antes de decidir

Medirlo sobre el caso que ya tenemos. `hsi` publica `PublicAuthorityDto`; `retinar` lo espeja. Con la historia de los dos repos se puede contestar, con datos y no con opinión:

- ¿Cuántas veces cambió el cuerpo de esos métodos sin cambiar la forma? *(cada una es ruido que esto evitaría)*
- ¿Cuántas veces cambió la forma de manera aditiva? *(cada una es ruido que esto agregaría)*

Si lo primero supera claramente a lo segundo, la propuesta se defiende sola. Si no, la respuesta correcta es no hacerla.
