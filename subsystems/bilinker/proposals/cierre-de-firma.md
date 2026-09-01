# Propuesta: el cierre de firma como dimensión aceptada

**Estado:** implementado como el **vecindario de nivel 1** — `accepted.hash_n1` y `accepted.hash_ast_n1`, formato `3.4.0`. La forma normativa vive en [`concepts/accept.md`](../concepts/accept.md#el-cierre-de-firma); acá queda el argumento del que salió.

Lo que **no** está resuelto sigue abierto y es lo que decide si esto valió la pena: [igualdad no es compatibilidad](#lo-que-no-está-resuelto-y-es-el-problema-difícil).

> Lo de abajo es el diseño con el que se escribió el mecanismo. Para qué campos hay, cómo se pliegan y qué estados producen, la spec.

Un fragmento no es sólo un árbol de sintaxis: devuelve un tipo, recibe parámetros, llama a otras funciones. **¿Tiene sentido que la aceptación cubra algo de eso, y hasta dónde?**

El documento arranca por la respuesta general. Si lo que se busca es sólo que **quien usa una API se entere de que cambió**, hay dos caminos que no necesitan nada nuevo: ver [Tres formas de contestar la misma pregunta](#tres-formas-de-contestar-la-misma-pregunta).

## El antecedente: ya se intentó, y se revirtió

`subgraph.N` y los sciplinks persistían el call graph en archivos versionados. Se sacaron del formato, y [`lattice/concepts/edge.md`](../../lattice/concepts/edge.md) § "Frescura" registra por qué:

> Un índice derivado bajo control de versiones se desincroniza del código y reporta drift que no existe.

**Cualquier propuesta en esta dirección arranca teniendo que explicar por qué no es eso otra vez.**

## Lo que la evidencia dice que importa

El caso de [`1d`](../../../.stratum/worklist-accreta/1d.task.md): `retinar` consume un endpoint de `hsi` con el DTO equivocado. Lo que hay que notar es **qué habría detectado cada cosa**:

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

## La forma concreta, y dónde está ahora

**Dos campos** en `accepted`, al lado de los dos que ya había:

```yaml
accepted:
  hash: c4e1770b…          # el texto del fragmento
  hash_ast: 1b9e44a2…      # sus tokens
  hash_n1: 96c765b9…       # el texto de sus vecinos directos
  hash_ast_n1: 88e834c4…   # los tokens de sus vecinos directos
```

Lo que era la discusión de esta sección —el nivel 1 y por qué no recursa, que los vecinos no son captures, el fold y por qué se ordena por identidad, el todo-o-nada de `hash_ast_n1`, y los cuatro cuadrantes— **pasó a ser spec**: [`concepts/accept.md`](../concepts/accept.md#el-cierre-de-firma).

Salió como estaba pensado, con **una corrección que la implementación aportó**: el cuadrante *"el proveedor refactorizó por dentro y no rompió a nadie"* ya no hace falta. Con el [capture de contrato](../concepts/capture.md), el cuerpo no entra en `hash`, así que ese refactor no mueve nada. Lo resolvió el capture y no el vecindario.


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

Lo que se pierde mientras tanto, dicho sin maquillaje: cuando `hash_n1` difiere sabés que el vecindario se movió, no cuál vecino. Corrés lattice y ves los vecinos de hoy con sus hashes, pero no tenés los aceptados de cada uno para diffear. En teoría se reconstruyen partiendo del `commit` del endpoint; en la práctica eso pide **un LSP indexando un checkout histórico**, que puede ni buildear. Con tres vecinos es cómodo; con quince, incómodo.

## Quién calcula, quién decide, quién guarda

> **Este cuadro cambió, y vale la pena leer las dos versiones.** Lo que sigue es el reparto que salió; abajo está el que este documento proponía y por qué no fue.

Resolver un tipo hasta su declaración es trabajo de language server, no de tree-sitter. Y la frontera de bilinker es explícita: *sólo git y tree-sitter.*

| | Quién | Qué hace |
|---|---|---|
| encontrar los vecinos | **[`lspd`](../../lspd/overview.md)** | `definitions` sobre la firma. Un salto. No persiste nada. |
| pedirlos | **bilinker**, por un puerto que no nombra a nadie | `Neighbours::of` — y `None` es *no pude mirar* |
| hashear y escribir | **bilinker** | aplica el recorte de bordes, foldea, escribe `accepted` |
| comparar en `check` | **bilinker** | compara lo guardado contra lo que le traen. Nunca resuelve un tipo. |

Bilinker recibe las ubicaciones; no sale a buscarlas con conocimiento propio de tipos. El hasheo queda de su lado porque el recorte de bordes es su regla y vive *"en el único lugar donde un nodo se convierte en rango"*.

### Lo que este documento proponía, y por qué no fue

Decía *"encontrar los vecinos: **lattice**"* y *"componer y decidir: **impact**"*, y de ahí salió [ADR-0001 de impact](../../impact/.stratum/impl/docs/adr/0001-orquesta-no-escribe.md). El argumento era bueno y **la premisa cambió**:

**El daemon no era de lattice.** Su crate nunca dependió del crate `lattice`, y lattice le hablaba por un socket como le hablaría cualquiera. Al salir a su propia capa, bilinker puede pedirle sin invertir nada:

```
      lattice ──┐
                ├──►  lspd    (socket)
     bilinker ──┘
```

**Y no hace falta componer.** Con los dos consumidores pidiéndole al mismo daemon, no queda nada en el medio que juntar — así que el compositor no tiene qué hacer.

Lo que el ADR protegía —que bilinker funcione solo, offline, en cualquier repo— **se conservó**: el puerto no nombra a nadie, la librería no depende de `lspd`, y sin daemon `check` corre igual y no falla. La revisión está anotada en el propio ADR.

### Lo que le faltaba a lspd, y ya tiene

`definitions`: dónde está declarado lo que se menciona en una posición. Es la única pregunta que sabe contestar que no es del call graph, y devuelve **ubicaciones y ninguna interpretación**.

Y el recorrido es un salto, no un grafo. Las tres menciones de `Persona` en `Persona hijoMenor(Persona padre, Persona madre)` resuelven a la misma declaración, así que **la dedup la hace el language server** y no una regla del formato: el vecindario tiene un `Persona` y no tres.


## El revisor es otra pregunta, y ya tiene dueño

Alguien revisando *"¿este cambio rompe el significado?"* sí necesita la semántica interna. Pero eso no es *"¿qué aprobé?"* sino *"¿hasta dónde llega esto?"* — y el ecosistema ya se lo asigna a **impact**.

Dicho corto: **el contrato se acepta; el alcance se consulta.** Meter los dos en `accepted` es lo que vuelve infinito el cierre.

## Lo que no está resuelto, y es el problema difícil

**Igualdad no es compatibilidad.**

Agregarle un campo a un DTO **no rompe** a un consumidor que deserializa JSON: es aditivo, y el consumidor viejo lo ignora. Pero un hash de igualdad dispara igual, y el consumidor tiene que ir a mirar un cambio que no le hace nada.

Si eso no se resuelve, `hash_n1` reproduce el ruido que mató a `subgraph.N`, sólo que un nivel más arriba — y peor, porque ahora el ruido cruza la frontera y llega a gente de otro equipo.

**Sigue abierta, y es la única que decide si esto vale la pena.** No la resuelve ninguna variante. Los dos hashes y el mapa de vecinos disparan con la misma frecuencia ante un cambio aditivo: lo que el mapa agrega es atribución, no una noción de compatibilidad. La diferencia entre ellos es cuántos pasos hay entre el estado y entender qué pasó, no si el estado aparece.

Lo demás que queda por decidir es chico:

1. **El vocabulario de los dos estados nuevos.** `RESTYLED` y `ALTERED` ya nombran las dos filas del fragmento; hacen falta dos nombres para las del vecindario que digan que el eje es otro.
2. **Qué cuenta como vecino directo donde el fragmento no es una función** — un DTO, una interfaz, prosa. La respuesta por defecto es que el campo está ausente, pero conviene decirlo antes de que alguien lo descubra.

### Preguntas que esta forma retira

- ~~**¿Dónde para el cierre?**~~ En 1, y está en el nombre del campo. Con eso también se cae la terminación con tipos recursivos, que era su consecuencia peor.
- ~~**¿Un cierre o dos?**~~ No se puede tener dirección: en `Persona hijoMenor(Persona padre, Persona madre)` el mismo tipo es retorno **y** parámetro, así que un conjunto de tipos no distingue lo que se devuelve de lo que se recibe. Y no hace falta — la asimetría de compatibilidad (agregar un campo a un retorno es seguro, agregar un parámetro requerido no) no es una propiedad del tipo sino de cómo lo usa el consumidor, y eso lo decide quien mira, como en todo el resto del diseño.
- ~~**¿Y los lenguajes sin tipos?**~~ Donde no hay firma resoluble el campo está ausente, igual que `hash_ast` donde no hay gramática.

## Tres formas de contestar la misma pregunta, y por qué se hizo ésta

*«Que quien usa la API se entere de que cambió»* tiene tres respuestas de costo muy distinto. **Se implementó C**, y las otras no quedaron descartadas: dos de ellas **siguen siendo mejores donde aplican**, y conviene que eso esté escrito antes de que alguien use la que se construyó en un caso donde había una más barata.

### A — Publicar el DTO. Funciona hoy, sin nada nuevo.

**En un DTO el problema no existe**, y es el detalle que vuelve barata la opción: un DTO no tiene cuerpo. `PublicAuthorityDto` es tres campos y anotaciones de Lombok — capturarlo *es* capturar la forma, sin una sola línea de implementación que genere ruido.

O sea que la tercera dimensión de esta propuesta **no hace falta donde el fragmento ya es sólo firma**. Interfaces, DTOs, records, declaraciones de tipo: ahí la aceptación de contenido que bilinker ya tiene alcanza y sobra.

Lo que **no** cubre: la ruta. Si `hsi` mueve el endpoint sin tocar el DTO, el consumidor no se entera — y ése fue justamente el error de `retinar`.

### B — Publicar el contrato ya generado

El `openapi.json` de springdoc **es** el contrato: lo emite el mismo framework que sirve las rutas, así que no puede divergir de lo que se sirve — a diferencia de un DTO espejado a mano, que es exactamente lo que se desincronizó en `retinar`.

Y tiene una ventaja que el código Java no puede dar: **ahí la ruta es una clave literal.** En el fuente, `/public-api/user/permissions/from-token` no existe como string — se compone de un `@RequestMapping` de clase y un `@GetMapping` de método, y por eso [`1d`](../../../.stratum/worklist-accreta/1d.task.md) lo dio por inexistente en la primera pasada. En el JSON generado es una clave de objeto, que es un ancla estable de las que [`reference.md`](../concepts/reference.md) ya recomienda.

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

**A donde el fragmento ya es sólo firma; B cuando hay un spec generado; C donde no lo hay; C bis como el upgrade aditivo de C, si la atribución llega a doler.**

### Por qué se hizo C, y no las otras primero

**C es la única que no depende de nada de afuera.** A pide que el proveedor publique el DTO; B pide que commitee un generado. Las dos son mejores donde el proveedor coopera, y ninguna de las dos se puede hacer desde el lado del consumidor — que es donde estaba el problema.

**Y el orden se lo dio otra cosa.** Este documento suponía que C necesitaba a lattice y un compositor, o sea dos subsistemas de acuerdo. Cuando el daemon salió a su propia capa, C pasó a ser dos campos, un puerto y un método nuevo en el daemon: la más barata de las tres en trabajo, y no la más cara como decía el encabezado.

**C bis sigue esperando lo mismo que esperaba:** el endpoint de tipo bilink, para que N capturas puedan decir que son *un* contrato. Y el fold se definió sobre los valores y no sobre las claves justamente para que siga siendo un upgrade aditivo — el barato es un prefijo válido del caro.

Y una corrección a lo que este documento afirmaba antes: **C bis no resuelve la compatibilidad, y C no la empeora.** Los dos disparan igual ante un cambio aditivo; lo que C bis agrega es atribución. El problema difícil nunca fue el plegado, así que no se disuelve al desplegar — ver [Lo que no está resuelto](#lo-que-no-está-resuelto-y-es-el-problema-difícil). Y B no reemplaza a C: sirve donde hay un spec generado, que es una parte del problema y no todo.

## Lo que falta medir

La decisión ya se tomó y el mecanismo está. Lo que sigue pendiente es saber **si el ruido que agrega es menor que el que evita**, y eso no se contesta con opinión.

Medirlo sobre el caso que ya tenemos. `hsi` publica `PublicAuthorityDto`; `retinar` lo espeja. Con la historia de los dos repos se puede contestar, con datos y no con opinión:

- ¿Cuántas veces cambió el cuerpo de esos métodos sin cambiar la forma? *(cada una es ruido que esto evitaría)*
- ¿Cuántas veces cambió la forma de manera aditiva? *(cada una es ruido que esto agregaría)*

Si lo primero supera claramente a lo segundo, se defiende sola. Si no, lo que hay que revisar es [el problema difícil](#lo-que-no-está-resuelto-y-es-el-problema-difícil) — y la salida no es sacar el mecanismo sino darle una noción de compatibilidad, porque el campo ya cruza la frontera y es lo único que se la entrega al consumidor.
