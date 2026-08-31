# Propuesta: el cierre de firma como tercera dimensión aceptada

**Estado:** en discusión. No especificado, no implementado. Vive acá para que el razonamiento no se pierda.

Un fragmento no es sólo un árbol de sintaxis: devuelve un tipo, recibe parámetros, llama a otras funciones. **¿Tiene sentido que la aceptación cubra algo de eso, y hasta dónde?**

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

## Qué haría falta antes de decidir

Medirlo sobre el caso que ya tenemos. `hsi` publica `PublicAuthorityDto`; `retinar` lo espeja. Con la historia de los dos repos se puede contestar, con datos y no con opinión:

- ¿Cuántas veces cambió el cuerpo de esos métodos sin cambiar la forma? *(cada una es ruido que esto evitaría)*
- ¿Cuántas veces cambió la forma de manera aditiva? *(cada una es ruido que esto agregaría)*

Si lo primero supera claramente a lo segundo, la propuesta se defiende sola. Si no, la respuesta correcta es no hacerla.
