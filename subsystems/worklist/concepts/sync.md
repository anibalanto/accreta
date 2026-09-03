# Sincronización con el proveedor

Un ítem sin clave de proveedor es un **pedido**. El proveedor —hoy Jira, vía `acli`— asigna la clave real; git es el transporte. Ver [`proposals/ventanas-por-rama.md`](../proposals/ventanas-por-rama.md) para el razonamiento completo; esta página es la spec de lo ya decidido.

## Qué es un pedido

Un id de ítem es base-36, `[0-9a-z]`, nunca con mayúscula ni guión. Una clave de proveedor tiene las dos cosas. La partición la garantiza el alfabeto: nada escrito a mano puede colisionar con algo que asignó el proveedor.

```
is_unassigned(slug) := no matchea ^[A-Z]+-\d+$
```

## El renombre y la reescritura son un solo commit

Renombrar `<slug>.<tipo>.md` a `<clave>.<tipo>.md` rompe todo lo que lo nombraba. La reescritura corre en el mismo commit que el `git mv` — o entran las dos cosas, o no entra ninguna.

**Sólo se tocan posiciones delimitadas**, nunca una subcadena suelta:

| Posición | Forma |
|---|---|
| destino de link | `](<slug>.<tipo>.md` |
| frontmatter | `parent: <slug>`, `relation.<tipo>: [<slug>, …]` o `relation.<tipo>: <slug>` |
| id en prosa | `` `<slug>` `` |

Un slug es un string libre, así que `agregar-funcionalidad-1` es prefijo de `agregar-funcionalidad-10`: cada posición de arriba exige que el slug no siga con un carácter de identificador, para que renombrar el primero nunca toque al segundo.

## El orden es para el proveedor, no para los números

Cuando un push trae varios pedidos y uno depende de otro —por `parent` o `relation.*`—, se asignan en orden topológico sobre el subgrafo de **pedidos únicamente**: una dependencia hacia un ítem que ya tiene clave no impone nada, porque ya existe. Un ciclo se rechaza sin escribir nada.

El orden no lo necesita la reescritura local — cada renombre ya recorre todo el árbol y corrige cualquier referencia al slug que se está reemplazando, sin importar el orden de llamadas. Lo que sí lo necesita es crear en el proveedor: no se puede pedir un issue con `--parent <clave>` si esa clave todavía no existe.

## Dos clases de rama, y el nombre dice qué se puede hacer

Completo y verificable son excluyentes, así que hay dos clases y no una:

| Prefijo | Contiene | Se verifica | `pre-receive` |
|---|---|---|---|
| `refs/heads/secure/**` | una vista parcial, acotada | sí, entera, en cualquier momento | corre el compare-and-swap |
| `refs/heads/insecure/**` | todo, o un conjunto que crece sin techo | no | **rechaza el push** |
| cualquier otra | no es del worklist | — | no opina |

**Rechazar el push a una insegura es lo que vuelve cierta la palabra.** Una rama que se llama insegura y acepta escrituras miente: no puede verificarse, así que no puede prometer que lo que entra sea consistente con el proveedor. Y de ahí sale, sin que nadie tenga que respetarla, que el panorama sólo avance por propagación — **no hay forma de empujarle.**

`backlog` es insegura: es *"todo lo que ningún sprint tomó"* y crece sin techo. Para trabajar sobre ítems del backlog se recorta una ventana segura acotada.

### Una ventana sobrevive al cierre de su sprint

> **Nada cierra una ventana. Cerrar el sprint no la borra.**

Una rama segura se hace responsable de **poder verificarse entera contra el proveedor en cualquier momento**, y si se borrara al cerrar el sprint esa responsabilidad tendría fecha de vencimiento: lo que prometería no es *"esto se puede verificar"* sino *"esto se pudo verificar mientras duró"*. Quedando, *"¿el sprint 7 sigue coincidiendo con el proveedor?"* es una pregunta contestable un año después.

**Y nada empuja a borrarlas**, porque una ventana es acotada por construcción: en el panorama de accreta la más grande tiene 19 archivos contra 154. Lo que crece sin techo es el panorama, y el panorama no es una ventana.

El commit de la ventana es además el registro de **qué llevaba el sprint**, y eso no se recupera del panorama más tarde: el `items` de un sprint puede cambiar después de cerrado, y el panorama guarda el estado final de cada ítem y no el conjunto que aquella iteración tomó.

Que quede no la vuelve autoritativa: sigue siendo **derivada**, y se puede volver a cortar. Y borrar una a mano sigue siendo posible, como con cualquier rama — lo que se descarta es que el cierre lo haga solo.

## La ventana y el compare-and-swap

Una ventana es una rama `secure/` — un subconjunto de ítems sobre el que un push se valida. Antes de aceptar:

1. Preguntar al proveedor por el estado de los ítems **de esa ventana**, y ninguno más.
2. Si algo cambió desde el commit sobre el que se empuja, rechazar. La rama queda con el estado nuevo, sin commit de más — como un push no fast-forward cualquiera.
3. Si no cambió nada, aplicar.

### Qué se compara, y contra qué

**La creencia es lo que ya está escrito.** El `status` de cada ítem en el **tip actual de la rama** —el del servidor, antes de este push, nunca el que el cliente asume como base— ya es la única fuente de lo que se creía cierto. No hay un archivo de estado aparte que mantener sincronizado.

```
Provider::state(keys) -> { clave -> status }
```

El chequeo no mira el contenido que llega: compara el estado en vivo del proveedor contra lo que el tip actual del servidor tiene escrito, para las claves presentes en ese tip. Coincide → se acepta el push (recién ahí se aplica lo nuevo). Difiere → se rechaza, sin importar qué traiga el push.

**El proveedor es un puerto**, para que la implementación real (Jira, vía `acli`) y una de prueba compartan la misma forma. La de prueba es un archivo `clave -> status`, mutable desde afuera — necesario para poder probar el rechazo: en un sistema cerrado, sin una forma de simular que el proveedor cambió por su cuenta, esa rama nunca se ejercita.

### El éxito se lee de la salida, nunca del código de retorno

> **`acli` devuelve 0 cuando falla.**

Escribe `✗ Failure: …` en `stdout` y sale con éxito, así que un cliente que mire el código de retorno **no se entera de nada**. El primer push real de una ventana creó cinco issues y las cinco descripciones fueron rechazadas por Jira sin una palabra en la salida del push.

La detección no puede ser buscar ese texto: es de una herramienta ajena y está en el idioma de quien la corre. Se pide `--json`, que responde estructurado:

```json
{"results":[{"status":"FAILURE","message":"InvalidPayloadException: INVALID_INPUT","id":"ACC-16"}],
 "totalCount":1,"successCount":0}
```

**Una operación del proveedor se considera hecha cuando su `status` lo dice**, y `successCount` cierra el lote. Es el mismo principio que el resto: fallar hacia reportar, y no confundir *"no se pudo ver"* con *"está bien"*.

Y de ahí sale una regla para cualquier proveedor futuro: **el puerto devuelve el resultado de la operación, no el de haberla intentado.** Un `Creator` que no distingue las dos cosas no sirve, por más que el comando que corra por debajo salga con 0.

## Asignar una clave: crear o encontrar

Un pedido se resuelve preguntándole al proveedor si ya existe —por título— antes de crear. Sin esto, un hook que crea el issue y falla antes de comitear el renombre duplica en el reintento.

```
Creator::create_or_find(titulo, tipo, descripcion) -> clave
```

**Buscar mal es peor que no buscar.** Una búsqueda que falla en silencio —una comilla sin escapar rompiendo el JQL— vuelve indistinguible *"no existe"* de *"no se pudo preguntar"*, y el reintento duplica. El título se escapa siempre antes de interpolarlo en cualquier consulta.

**Y el título de búsqueda no es el mismo string que el título del issue.** `summary ~` de JQL no compara texto literal: es una búsqueda de texto completo, y algunos caracteres —`[` y `]`, confirmado— rompen su parser incluso adentro de comillas, sin que JQL tenga forma de escaparlos (`\[` es una secuencia inválida). La búsqueda usa una versión del título con esos caracteres neutralizados; **crear usa el título real, intacto** — `--summary` no pasa por JQL y no tiene ese problema. Buscar con una versión más laxa puede traer falsos positivos; no buscar nada, por un carácter que rompe el parser, produce un duplicado seguro. Lo primero es el riesgo que vale correr.

**La descripción lleva el cuerpo del ítem, convertido a ADF** — ver § "El cuerpo viaja, y vuelve convertido".

> Una versión anterior de esta página decía que la descripción era una línea `Fuente: <ruta>` y que *"git es la fuente de verdad"*. **Las dos cosas estaban mal.** La segunda contradice el título de la propuesta que la origina —*"el proveedor es la autoridad"*— y nadie la decidió: se coló como justificación de la primera, que a su vez era una limitación técnica de `acli`, no una decisión de diseño. La regla real es una sola y está abajo.

### El tipo es del worklist, no de Jira

`Creator::create_or_find` recibe el tipo en el vocabulario del worklist —`task`, `user-story`, `epic`— nunca el nombre que Jira le da. Quién habla con Jira es quien sabe traducirlo, y la traducción es por proyecto: en `ACC` los tipos están localizados —`Tarea`, `Historia`, `Epic`, más `Subtask`, `Error`, `Mejora` que el worklist no usa—, y un `--type Task` en inglés no existe ahí y falla.

```
task       -> Tarea
user-story -> Historia
epic       -> Epic
```

Confirmado contra `ACC` real. Otro proyecto de Jira puede tener otro vocabulario — la tabla es de esta capa, no universal.

## El cuerpo viaja, y vuelve convertido

> **La verdad viene del proveedor a git. Y si git está actualizado, puede ir al proveedor.**

Una sola regla, en vez de una tabla de qué campo es de quién: el proveedor arbitra la concurrencia, y **cualquier escritura tiene que probar que parte del estado actual** — que es el compare-and-swap de arriba, aplicado a todo el contenido y no sólo al `status`.

### Lo que se guarda no es lo que empujaste

El camino de escritura de un push aceptado:

1. Se toma el cuerpo del ítem —el markdown, sin el frontmatter— y se convierte a ADF.
2. Ese ADF se manda al proveedor.
3. **Se convierte el ADF de vuelta a markdown, y eso es lo que queda guardado** — no el markdown original.

Suena raro y es lo que hace que el resto funcione:

- **No hace falta migrar nada.** La forma canónica no es un estado al que hay que llevar el corpus una vez: es el resultado de escribir. Nada puede salirse de ella, porque todo lo que entra pasa por la conversión.
- **Lo guardado es una proyección fiel de lo que el proveedor tiene.** Comparar deja de necesitar una conversión en cada chequeo, y el compare-and-swap deja de poder dar falsos positivos por diferencias de formato.
- **La pérdida se ve en el primer push.** Si el markdown tenía algo que ADF no expresa, el archivo que aterriza difiere del que se empujó y aparece en el `git diff` del `pull`. Guardar una versión rica local y mandar una pobre dejaría dos verdades divergiendo en silencio.

Se apoya en una propiedad medida, no supuesta: **la conversión converge en una pasada.** Medido con `amdc` sobre cuatro archivos reales —entre ellos uno de 29 KB y otro de 28 KB—, el segundo round-trip es byte a byte idéntico al primero, con cero warnings en las dos direcciones. Las diferencias contra el original son normalizaciones de formato —`*cursiva*` a `_cursiva_`, el separador de tablas— y en un caso el pegado de líneas que CommonMark ya considera un solo párrafo. **Ningún texto se pierde**, y los identificadores `SNAKE_CASE` adentro de code spans sobreviven intactos.

### El servidor anota lo que hizo, no borra lo que hiciste

La conversión **no reescribe el commit que llegó**: se agrega uno encima.

```
<el commit del cliente>     ← lo que se empujó, intacto
normalize: ACC-45           ← lo que el round-trip dejó
```

Es el tercero de la misma familia que ya tienen `rename <slug> -> <clave>` y `provider: <clave> …`: **el servidor deja su trabajo como un commit propio y auditable.** Con esto, un error del conversor es un diff que se ve y se revierte; reescribiendo el commit del cliente sería una pérdida sin contra qué comparar — y eso importa especialmente porque el conversor es la pieza más nueva de todo esto.

### Nada viaja al proveedor hasta que el estado local esté completo

Resolver una ventana son **tres pasadas**, y el orden no es de estilo:

| | Qué hace | Por qué no antes |
|---|---|---|
| **1** | claves y renombres, en orden topológico | — |
| **2** | los cuerpos, convertidos y enviados | un cuerpo enviado antes lleva los nombres **previos** al renombre, y queda congelado así: el ítem ya tiene clave, así que ningún push posterior lo vuelve a mirar |
| **3** | los vínculos entre ítems | un vínculo necesita que **las dos puntas** existan en el proveedor |

La pasada 2 sólo salía bien por accidente cuando la referencia estaba declarada en `relation.*` —el orden topológico ponía al referenciado primero—; una referencia que vive **sólo en la prosa** no participa de ese orden y quedaba vieja.

### Un link a otro ítem se traduce en el borde

En el repo un ítem referencia a otro por su archivo; en el proveedor eso no significa nada. La traducción es mecánica y va en las dos direcciones:

```
<clave>.<tipo>.md   ←→   <base>/browse/<clave>
```

**Y no se convierte en un vínculo del proveedor.** Una cita en prosa no es una dependencia declarada — ésa es la distinción que `relation.*` existe para hacer, y convertir cada mención en un vínculo llenaría el ítem de relaciones que nadie declaró.

De vuelta, el tipo no está en la URL: se resuelve mirando qué `<clave>.*.md` existe en la ventana. **Si no está —una referencia a algo de otra ventana—, la URL se queda como URL**, que es la forma correcta para algo que no vive acá.

### Dos cosas no pasan por el conversor

**El frontmatter.** `title`, `status`, `relation.*` no son cuerpo markdown ni viven en la descripción del proveedor. Se separan antes de convertir y se vuelven a pegar después, intactos.

**Y el título.** Va al `summary` como texto plano, por su propio camino — no es parte de la conversión del cuerpo.

## Dos pasos, no uno

`pre-receive` sólo puede aceptar o rechazar lo que llega — no puede reescribirlo. El compare-and-swap vive ahí. El renombre y la reescritura son un commit *nuevo*, agregado después de aceptar: viven en `post-receive`. El cliente que empujó tiene que hacer `fetch` para ver los ids reales.
