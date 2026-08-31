# Propuesta: el cierre de firma como tercera dimensión aceptada

**Estado:** en discusión. No especificado, no implementado. Vive acá para que el razonamiento no se pierda.

Un fragmento no es sólo un árbol de sintaxis: devuelve un tipo, recibe parámetros, llama a otras funciones. **¿Tiene sentido que la aceptación cubra algo de eso, y hasta dónde?**

El documento arranca por la respuesta general y cara. Si lo que se busca es sólo que **quien usa una API se entere de que cambió**, hay dos caminos mucho más baratos: ver [Tres formas de contestar la misma pregunta](#tres-formas-de-contestar-la-misma-pregunta).

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

Guardar el cuerpo tiene tres problemas encadenados: no termina, obliga a decidir sobre código que nadie leyó, y dispara `CHAIN_DIRTY` en refactors que no cambiaron nada observable. Ese ruido es exactamente lo que mató a `subgraph.N`.

## Por qué el revert no lo prohíbe

Lo que se revirtió fue **persistir un índice**. Un índice afirma algo sobre el código *actual*, y por eso se desincroniza.

Un **hash** no afirma nada sobre el código actual: registra **qué se aprobó**. Es lo que `accepted.hash` ya es — un valor derivado del código, congelado en el momento de una decisión, que justamente sirve porque el código puede alejarse de él.

La diferencia no es de grado: un índice que envejece miente; un hash que envejece **es el mecanismo**.

## La forma concreta

Una tercera dimensión en `accepted`, al lado de las dos que ya hay:

```yaml
accepted:
  link: capture 8f2a4c6e…
  hash: c4e1770b…          # el texto
  hash_ast: 1b9e44a2…      # los tokens
  hash_contract: 7d3e0f91…  # el cierre de la firma      ← nuevo
```

Y los estados componen solos:

| Cambió | No cambió | Estado | Qué significa |
|---|---|---|---|
| texto | tokens | `RESTYLED` | sólo espaciado |
| tokens | **contrato** | *(nuevo)* | **el proveedor refactorizó por dentro y no rompió a nadie** |
| contrato | — | *(nuevo)* | alguien se rompe |

La fila del medio es la que hace adoptable la frontera: **el proveedor puede reescribir el cuerpo con libertad y el consumidor no se entera.** Hoy cualquier cambio en el cuerpo llega al consumidor como `CHAIN_DIRTY` y lo obliga a mirar algo que no le incumbe.

## Por qué lo tiene que aportar lattice

Resolver un tipo hasta su declaración es trabajo de language server, no de tree-sitter. Y la frontera de bilinker es explícita: *sólo git y tree-sitter; el call graph vive en lattice, el alcance de un cambio en impact.*

Así que bilinker **no puede calcular** este hash. Tiene que pedirlo.

Lattice ya tiene el proveedor `lsp` y sabe expandir un nodo. Lo que falta es un `kind` que **no** es `call`: algo que exprese *"este fragmento expone esta forma"*. Y lattice sigue sin persistir nada — el grafo se calcula en el momento; lo único que se guarda es el hash, del lado de bilinker, como parte de una decisión.

**Ésa es la composición que el modelo ya permite y nadie usó todavía:** un proveedor derivado alimentando una aceptación. Hoy `accepted` sólo se nutre de lo que bilinker mismo puede ver.

## El revisor es otra pregunta, y ya tiene dueño

Alguien revisando *"¿este cambio rompe el significado?"* sí necesita la semántica interna. Pero eso no es *"¿qué aprobé?"* sino *"¿hasta dónde llega esto?"* — y el ecosistema ya se lo asigna a **impact**.

Dicho corto: **el contrato se acepta; el alcance se consulta.** Meter los dos en `accepted` es lo que vuelve infinito el cierre.

## Lo que no está resuelto, y es el problema difícil

**Igualdad no es compatibilidad.**

Agregarle un campo a un DTO **no rompe** a un consumidor que deserializa JSON: es aditivo, y el consumidor viejo lo ignora. Pero un hash de igualdad dispara igual, y el consumidor tiene que ir a mirar un cambio que no le hace nada.

Si eso no se resuelve, `hash_contract` reproduce el ruido que mató a `subgraph.N`, sólo que un nivel más arriba — y peor, porque ahora el ruido cruza la frontera y llega a gente de otro equipo.

Las preguntas abiertas, en orden de dificultad:

1. **¿Dónde para el cierre?** Un campo de tipo DTO recursa; uno de tipo `Instant` no. La línea es *"tipos declarados en este repo"*, pero un tipo de una librería compartida entre proveedor y consumidor es un caso real y no obvio.
2. **¿El hash es de igualdad o de compatibilidad?** Lo segundo es semver de formas, y es difícil de verdad. Lo primero es fácil y ruidoso.
3. **¿Un cierre o dos?** Lo que se devuelve y lo que se recibe se rompen distinto: agregar un campo al retorno es seguro, agregar un parámetro requerido no. Tal vez son dos hashes y no uno.
4. **¿Y los lenguajes sin tipos?** El cierre de firma de una función TypeScript existe; el de una Python sin anotaciones, no. La degradación tiene que estar dicha, no descubrirse.

**La 2 es la que decide si esto vale la pena.** Sin una noción de compatibilidad, esta propuesta agrega precisión donde no hace falta y ruido donde duele.

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

| | C — un hash del cierre | C bis — N capturas |
|---|---|---|
| Cuando el contrato cambia | *"el hash del contrato cambió"* | **`PublicAuthorityDto` quedó `ALTERED`** |
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

### C — El cierre de firma vía lattice

Lo de arriba. Es la única que sirve donde **no hay un spec generado** — código que no expone una API HTTP, contratos entre módulos, cualquier cosa que no tenga un OpenAPI detrás.

### Cuál conviene

| | Cubre la forma | Cubre la ruta | Cuesta |
|---|---|---|---|
| **A** publicar el DTO | sí | **no** | nada — anda hoy |
| **B** publicar el openapi | sí | **sí** | gramática JSON + commitear un generado |
| **C** cierre de firma | sí, opaco | no aplica | lattice + resolver compatibilidad |
| **C bis** lattice propone N capturas | sí, **y dice cuál se movió** | no aplica | lattice + agrupación (endpoint bilink) |

**A para empezar mañana; B cuando la ruta importe; C bis como la ergonomía de A; C parked.**

Y una observación sobre C contra C bis: **C bis no necesita resolver la compatibilidad.** Como cada fragmento conserva su propio estado, un campo agregado a un DTO aparece como ese DTO `ALTERED` y quien mira decide en un segundo si le afecta — en vez de un hash de cierre que dispara sin decir por qué. El problema difícil de C se disuelve al no plegar todo en un valor. Y B no reemplaza a C: sirve donde hay un spec generado, que es una parte del problema y no todo.

## Qué haría falta antes de decidir

Medirlo sobre el caso que ya tenemos. `hsi` publica `PublicAuthorityDto`; `retinar` lo espeja. Con la historia de los dos repos se puede contestar, con datos y no con opinión:

- ¿Cuántas veces cambió el cuerpo de esos métodos sin cambiar la forma? *(cada una es ruido que esto evitaría)*
- ¿Cuántas veces cambió la forma de manera aditiva? *(cada una es ruido que esto agregaría)*

Si lo primero supera claramente a lo segundo, la propuesta se defiende sola. Si no, la respuesta correcta es no hacerla.
