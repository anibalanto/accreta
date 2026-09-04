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

### Una dependencia que cruza la ventana se reporta, no rompe el push

Los `relation.depends` que apuntan **adentro** de la ventana llegan a la pasada 3 ya traducidos: el renombre los reescribió junto con el resto de las referencias. Los que apuntan **afuera** quedan como slug del worklist, y un slug no es una clave del proveedor.

> **Un vínculo que no se puede traducir se informa. No aborta la ventana.**

Una ventana es acotada por diseño y las dependencias no respetan sus bordes: que una user story de un sprint dependa de otra del anterior es lo normal en un backlog, y arrastrar la dependencia adentro dejaría de estar acotada. Exigir que todas caigan adentro sería pedirle al backlog que se ordene por el recorte.

**Y es un síntoma de la propagación que falta.** El ítem de afuera ya tiene clave —se la puso el push de su propia ventana—; lo que no la tiene es el panorama, porque las claves quedan en la rama de cada ventana y nadie las lleva de vuelta. Con la propagación andando, la ventana re-cortada traería el `depends` ya traducido y el vínculo se crearía solo.

### La jerarquía entra hasta donde el proveedor la tiene

El worklist descompone en tres escalones —`epic` → `user-story` → `task`— y Jira **no tiene tres**. Sus tipos viven en niveles numerados, y en `ACC`:

```
nivel  1   Epic
nivel  0   Historia   Tarea   Error   Mejora
nivel -1   Subtask
```

`Historia` y `Tarea` están **en el mismo nivel**, y Jira no admite `parent` entre pares. Medido contra `ACC`, no supuesto:

| Relación | |
|---|---|
| `Historia` bajo `Epic` | ✔ |
| `Tarea` bajo `Epic` | ✔ |
| `Tarea` bajo `Historia` | ✘ *"Selecciona una incidencia principal válida"* |
| `Subtask` bajo `Historia` | ✔ |
| `Subtask` bajo `Epic` | ✘ salta un nivel |

> **El `parent` de un issue es la épica de la que cuelga, por lejos que quede. El escalón del medio es un link `Relates`.**

Así que un ítem con `parent` sube con dos cosas: `--parent <clave de su épica ancestro>`, subiendo la cadena hasta el primer `epic`; y, si su padre directo **no** es esa épica, un `Relates` hacia él.

**Por qué no `Subtask`.** Es la única forma de tener el anidado exacto, y cuesta dos cosas. La primera: `Subtask` no cuelga de un `Epic`, así que una task hija de épica —o suelta— tendría que seguir siendo `Tarea`, y **el tipo de Jira dejaría de ser función del tipo del worklist** — lo contrario de lo que § "El tipo es del worklist, no de Jira" decidió. La segunda: una subtarea de Jira no lleva sprint propio ni aparece en el backlog como tarjeta, así que las tasks dejarían de ser lo que se mueve en el board. Un ítem del worklist tiene **un solo** tipo de Jira posible, y esa regla no se negocia por el anidado.

**Y `Relates` porque es lo que hay.** Los tipos de link de `ACC` son `Blocks`, `Cloners`, `Duplicate`, `Relates` y `Work item split`: **no hay ningún `Parent/Child`**. Es el mismo mecanismo que § "Los vínculos" usa para `relation.depends`, con otro tipo.

**Lo que se pierde, dicho:** en Jira el árbol tiene dos niveles donde el worklist tiene tres, y una user story deja de ser el contenedor de sus tasks — pasa a ser un issue hermano que las referencia. La descomposición completa vive en git, que es donde `parent` es autoritativo.

### El padre se pone al crear, y después no

`acli` acepta `--parent` en `create` y **no en `edit`** — ni por flag ni en el JSON de `--generate-json`.

Dos consecuencias, y las dos son limitaciones reales y no decisiones. Un issue que ya existe sin padre **no se puede corregir** por esta vía: hay que borrarlo y dejar que el próximo push lo cree. Y **recolgar un ítem de otra user story no se propaga**, que es justo la operación que [jerarquía](hierarchy.md) § "IDs secuenciales y jerarquía" describe como barata: cambia un campo del ítem, no el nombre de su archivo. Barata en git, imposible en el proveedor.

Por eso `create_or_find` que **encuentra** en vez de crear no puede prometer la jerarquía, y lo tiene que decir en vez de callarlo.

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

### El schema del proveedor poda, y la poda es de la frontera

> **En ADF el mark `code` es exclusivo: no convive con `strong` ni con `em`.**

GFM sí permite anidarlos, y `` **`bilinker`** `` —negrita sobre un identificador— es el idioma de la casa. La conversión produce entonces un nodo con `strong` y `code` juntos, y **Jira rechaza el documento entero**: no ese nodo, todo. Un solo `` **`x`** `` deja la descripción sin subir.

Medido: ese nodo solo, en un documento de un párrafo, es rechazado con `INVALID_INPUT`; el mismo texto con `code` solo pasa. El primer cuerpo real que se intentó subir traía **ocho casos**, todos nombres de archivo o de herramienta.

**Cuando `code` viene con otros marks de formato, los otros se van.** El `code` es el que lleva la información —dice que eso es un identificador—; la negrita es énfasis, y el énfasis es lo que se puede perder sin cambiar lo que la frase significa.

Va **en la frontera y no en el conversor**: qué marks se pueden combinar es del vocabulario del proveedor, y `body.rs` sólo sabe de markdown y de ADF. Y la pérdida no se esconde: el paso 3 convierte de vuelta, así que el markdown que aterriza dice `` `bilinker` `` sin negrita y **la poda se ve en el `git diff` del `pull`**, como cualquier otra normalización.

### Un ítem que ya tiene clave se actualiza, no se saltea

Resolver una ventana empezó siendo una sola cosa —**crear** lo que no existe— y eso dejaba un agujero: editar un ítem que ya tiene clave y empujar **no llegaba al proveedor**. El push entraba, el servidor guardaba el cambio, y el hook informaba `sin pedidos`.

Cierto en su propio vocabulario y engañoso donde importa: no había ítems sin clave, pero sí había un cambio que no viajó. Es el mismo defecto de forma que § "El éxito se lee de la salida" —confundir *"no había trabajo"* con *"el trabajo no se hizo"*—, y deja a git y al proveedor divergiendo sin que nadie avise.

> **Lo que se resuelve de una ventana son dos conjuntos: los pedidos, y lo que cambió.**

**Qué cambió lo dice el push**, no el proveedor: el diff entre el tip anterior y el que llega nombra los archivos tocados, y de ahí salen las claves. No hay que preguntarle nada a nadie, y **un push que no toca un ítem no lo re-sube** — actualizar uno no puede costar ochenta llamadas.

Y va **después** del compare-and-swap, que ya corrió: *"si git está actualizado, puede ir al proveedor"* es una garantía cobrada un paso antes, sobre el mismo push.

#### Qué sube, y qué no

| Campo | | |
|---|---|---|
| el cuerpo | ✔ | por el mismo camino de siempre: `round_trip`, la poda de marks, los links traducidos |
| el título | ✔ | es el `summary`, y es lo que se ve en el board |
| el `status` | ✘ | **no es el mismo campo** que el del proveedor — ver § "La jerarquía entra hasta donde el proveedor la tiene" y la task `5y` |

**Y subir el título tiene un costo que hay que decir.** La identidad de un ítem sin clave es su título: así lo encuentra `create_or_find` para no duplicar. Si el título cambia en el proveedor y alguien vuelve a cortar la ventana desde el panorama —que tiene el título viejo y sin clave—, la búsqueda no encuentra nada y **crea un issue nuevo**. Es el defecto de la task `5l` por otra puerta.

Hoy está acotado porque re-cortar una ventana viva está prohibido, y deja de estarlo el día que la propagación al panorama exista — que es cuando el panorama va a tener las claves y esto se arregla solo.

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
