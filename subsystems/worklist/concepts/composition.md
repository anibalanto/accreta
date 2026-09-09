# La composición del producto

Qué ítems lleva cada sprint, en qué orden está el backlog, y qué coordenada tiene cada sprint en el proveedor. **Vive en un solo archivo, en el panorama, y es del servidor.**

```yaml
# .metadata/product.yaml
sprints:
  - id: 21
    titulo: "La estructura del worklist se muda al servidor"
    status: in-progress
    key: 6525
    items: [ACC-315, ACC-316, …]
backlog:
  - ACC-96
  - …
```

`.metadata/` **con punto adelante** por lo mismo que `_sprints/` llevaba `_`: garantiza que nunca choque con un id. Y el panorama ya tiene `.bilink/` en su raíz, así que la forma no es nueva.

## El `.sprint.md` era cuatro cosas mezcladas

|  | Dónde va |
|---|---|
| **la composición** — qué ítems, el orden del backlog, `key`, `status` | acá |
| **el título** — lo que hace falta para nombrar el sprint del otro lado | acá, en `titulo` |
| **la prosa** — por qué esos ítems, en qué orden, qué quedó afuera, cómo cerró | se descarta: no es operativa para un sprint, y nada la preserva |
| el archivo | **desaparece** |

Y no es una mudanza por prolijidad: **la composición y el título tienen un dueño — el servidor —, y la prosa tiene otro — una persona.** Mezclarlas en un archivo hacía que las dos direcciones de la propagación se pelearan por el mismo texto — el `.sprint.md` era [el único archivo que las dos direcciones tocaban](propagation.md#y-el-tercero-es-la-clave-del-sprint-por-un-motivo-que-no-es-de-alcance-sino-de-autoría), y por eso el único donde hubo que partir la regla campo por campo.

## Por qué en el servidor y no como campo del ítem

Hubo una decisión anterior que proponía lo contrario —sacar `items` del sprint y ponerlo como campo del ítem— con el argumento de que *"tenerlo escrito obligaría a editar dos lugares al mover algo"*.

**Esto la revierte y respeta su principio.** El problema no era el contenedor: era la **duplicación** entre el `items` del sprint y el ítem. Con un solo archivo hay **un solo lugar** — y uno que el cliente no necesita.

> **Quién arma los sprints y el backlog es responsabilidad del servidor.** Al cliente le interesa ver su carpeta, esté o no configurado el proveedor.

## Y la membresía deja de ser una propiedad de la rama

Es el cambio que más cosas destraba, y conviene decirlo solo.

> **Hoy un ítem está en un sprint porque su archivo está en esa rama.** Con la composición en un campo, está porque una lista lo nombra.

| | Sacar un ítem de un sprint era | Pasa a ser |
|---|---|---|
| en el worklist | **borrar un archivo de una rama** — un commit que alguien hace | **editar una lista** en el panorama, y la ventana se regenera |
| desde el proveedor | no tenía cómo entrar: nadie compara ramas contra sprints | **comparar dos listas**, que es lo que [`absorb`](../commands/absorb.md) ya sabe hacer con los otros campos |

De ahí salen tres cosas que estaban trabadas:

**La invariante de [sólo edición](sync.md#una-vista-sólo-puede-aportar-ediciones) deja de tener excepción.** Que un archivo desaparezca de una vista pasa a ser **consecuencia de regenerar** y no un commit que alguien hace.

**Y sincronizar la membresía deja de ser una traducción y pasa a ser un espejo.** Jira ya modela sprints con issues adentro; comparar `items` contra lo que el board tiene es comparar dos listas de claves. Antes había que preguntarle a la forma del recorte, que no es una lista de nada.

**Y absorber un cambio de membresía deja de ser peligroso.** Traer *"el board sacó este ítem del sprint"* significaba sacar un archivo de una ventana —cambiar el recorte, con trabajo adentro— y pasa a significar editar una línea del YAML: lo que sigue es el recorte de siempre, que es idempotente y ya está probado.

## El `titulo` es el que se tipeó, y no hay otro nombre

> **`titulo` es el texto tal cual, sin normalizar.** No hay un segundo campo con una forma slugificada: guardar las dos sería guardar la misma información dos veces, y una de las copias quedaría vieja el día que alguien edite el título y no la otra.

Antes esto proponía un `name` derivado —minúscula, con guiones medios, capado a 20 caracteres— para tener algo corto y sin acentos que mostrar. **Se descartó entero**: lo único que un `name` así resolvía era una limitación de `.sprint.md`, que necesitaba un nombre con forma de slug porque en algún momento fue candidato a nombre de archivo. `product.yaml` no tiene esa restricción — sus entradas son elementos de una lista, no nombres de archivo — así que no hay nada que el `titulo` crudo no pueda hacer.

Quien necesite una forma corta o slugificada para mostrar la deriva al vuelo con una función pura; no hace falta persistirla.

### Y no es el nombre que ve el proveedor

| | |
|---|---|
| **`titulo`** | el texto tal cual, sin tope ni normalizar |
| **el nombre del sprint en Jira** | `<id> <titulo>`, **recortado a 29** porque Jira no acepta 30 |

La maquinaria del recorte es del proveedor, y sigue existiendo por eso — no porque `titulo` la necesite.

**Y el nombre del sprint no es su clave.** En la interfaz de Jira el id de un sprint no se muestra, no se puede buscar y nadie lo escribe: un sprint renombrado a `6524` queda imposible de encontrar justo para la persona que iba a usar ese nombre. La clave vive en el YAML como **coordenada**, que es lo que es.

## La mudanza terminó: `.sprint.md` se retira con `DropSprintFiles`

**El `items` sale de acá**, y también el `titulo` y el `key`: no queda ningún campo operativo que las pasadas de sprint —resolver el sprint del proveedor, anotarle la clave— necesiten leer de otro lado. Retirar el archivo es `worklist-server drop-sprint-files`: le agrega el `titulo` a cada sprint leyéndolo de su `.sprint.md` una última vez, y borra `_sprints/` entero.

**Y edita, no reconstruye.** A diferencia de la migración original —que armaba `product.yaml` desde cero a partir de los `.sprint.md`—, esto parte de la composición que ya existe y le agrega un campo: `key`, `status` e `items` pueden haber divergido del `.sprint.md` desde que la composición es la que manda, y reconstruir los pisaría con la copia vieja.

**Y el renombre necesitó una regla propia**, mientras `.sprint.md` convivió con la composición. En un `.md` una referencia a un ítem es un link —`](@o.task.md)`— y se reescribe con el texto. En la composición es una **entrada de una lista**, sin sintaxis alrededor: un renombre que sólo mire markdown deja el sprint nombrando un slug que ya no existe, y el próximo recorte falla con *"la composición nombra a `@o`, y no está"*. Se reescribe sobre la estructura, porque `@o` como texto también aparece adentro de `@otro`. Esa regla queda — es la que mantiene la composición renombrada, tenga o no `.sprint.md` al lado.

## Y lo que el proveedor perdió se saca, pero no se borra

Un ítem puede tener clave acá y no existir del otro lado. Medido el 2026-09-07: `ACC-268` tenía su `rename 6m -> ACC-268` en el log y Jira contestaba **404**.

Y hoy eso **traba el ítem para siempre**: `is_unassigned` es falso, así que ninguna pasada vuelve a mirarlo — no se recrea, no se corrige, y su clave muerta [se lleva el lote del sprint entero](../commands/assign-keys.md#una-clave-que-el-board-no-tiene-no-puede-llevarse-el-lote) en cada push.

> **Sale del árbol y entra a `.metadata/removes/`, entero.**

```
.metadata/removes/ACC-268.task.md
```

El archivo se mueve, no se destruye: **el ítem deja de ser un ítem y su contenido queda a la vista.** No hay que reconstruir nada de la historia de git para saber qué decía, ni por qué no está.

### Por qué lógico y no físico

*"Se recupera de git"* es cierto y no alcanza. Un borrado físico deja una ausencia, y **una ausencia no dice por qué**: quien la encuentra tiene que sospechar que alguna vez hubo algo, y recién ahí buscar. Un archivo en `removes/` contesta las dos preguntas sin arqueología — qué era y por qué se fue.

Y hace la vuelta barata: **devolverlo es moverlo de nuevo.** Si el 404 fue un error —alguien borró de más en el board— restaurar es un `git mv`, no un rescate.

### Se decide por el código, nunca por el mensaje

El proveedor contesta *"la incidencia no existe **o no tienes permiso para verla**"* — **una sola frase para dos casos que no se parecen en nada**. Sacar un ítem porque una credencial perdió permiso sería el mismo error de forma que confundir `sin verificar` con `coincide`, con el costo subido a destruir.

|  |  |
|---|---|
| **404** | no existe → se saca |
| **403** | no se puede ver → **no se toca**, y se reporta |

Se decide por el status HTTP, que sí los distingue. La frase no.

### Y no se libera la clave

`ACC-268` queda muerta y no se reasigna. El ítem tampoco vuelve a nacer con clave nueva: **salió**, y si el trabajo hace falta se escribe uno nuevo, que es una decisión de una persona.

Recrearlo automáticamente sería el sistema discutiéndole al board sobre algo que alguien borró a mano allá.

## Lo que no contesta

**Dónde está el archivo de un ítem.** El YAML dice **en qué sprint está**, que es otra pregunta — y es la que un endpoint `issue` de bilinker necesita.

**Y la composición no baja al cliente**, por lo mismo que el panorama no baja: es del servidor, y una copia que se queda vieja es una fuente de verdad falsa. Lo que el cliente ve es su ventana, que sale de acá por el recorte.

## La membresía la manda el proveedor

> **Si el board sacó un ítem del sprint, sale del `items`.** El proveedor manda.

Y hasta acá pasaba lo contrario, medido el 2026-09-07: alguien sacó varias tareas del sprint 21 en Jira y **volvieron en el push siguiente**.

```
sprint 21 -> 6525: 6 issue(s) agregados, 12 ya estaban
```

La pasada de sprint lee qué tiene el sprint allá para *"mandar sólo lo que falta"* — nunca para detectar que algo salió. Así que **una baja hecha del otro lado no sobrevivía a un push**, y el sistema le discutía a la persona.

### Un ítem que el board no tiene no siempre falta

Son dos casos, y hasta acá los dos se resolvían agregando:

| | |
|---|---|
| **acaba de recibir su clave** en este push | **falta de verdad** — nunca estuvo allá |
| **ya la tenía** | **el board lo sacó**, y eso es una decisión de alguien |

Lo que los distingue es lo que la propia corrida asignó, que ella sabe. Con eso la pasada mete las nuevas y **reporta** las otras.

**Y `bootstrap` es el tercer caso**, que se dice aparte: resuelve sprints que **nunca tuvieron ventana**, así que su membresía no viajó nunca y la ausencia del board no es una baja. Ahí entran todas.

### Y el alta también, aunque primero se dijo que no

Acá decía que un ítem que está en el sprint del board y no en el `items` *se reporta y no se agrega*, con el argumento de que entrar a un sprint es **planificar** y eso se hace de este lado.

**Era acotar la regla.** Si el proveedor manda, mover un ítem *hacia* un sprint es el proveedor moviendo algo igual que sacarlo — y quien lo movió allá espera que baje. Se descubrió del modo más directo: un ítem se devolvió al sprint en Jira y el `pull` no lo trajo.

> **Una sola regla: la membresía es la del board.** Sale lo que él sacó y entra lo que él tiene.

**Y entrar a un sprint es salir del anterior**, que es como el proveedor lo modela: un issue está en un sprint a la vez. Así que el alta lo saca de cualquier otro `items` y del backlog.

#### Y un ítem sin clave no se compara

> **Un `@<slug>` no puede estar en el sprint del board**, así que su ausencia no dice nada: es una tarea nueva que todavía no cruzó.

Comparar sin filtrarla la trataba como una baja y **la sacaba del `items`**: crear una tarea y correr `pull` la hacía desaparecer del sprint. Es la misma regla que el resto —*no se preguntó* no es *no está*— y acá el costo era perder trabajo recién escrito.

Se reporta, para que se vea que hay algo esperando cruzar:

```
  @la-tarea-nueva  todavia no cruzo — esperando clave para el sprint 21
```

#### Salvo que no haya archivo, y eso no es una elección

Un issue que el board tiene en el sprint y **no es un ítem de este lado** no entra: el `items` nombraría algo que no está, y el próximo recorte falla. Es el caso de un issue creado directamente en Jira.

Se reporta, y con eso queda a la vista lo que hoy no tiene camino: **un issue que nace del otro lado no se vuelve un ítem** — eso es adopción, y es otra pregunta.

### La guarda que más importa: un board vacío no vacía nada

Si el board contesta **cero** sobre un sprint que el `items` dice que tiene ítems, **no se toca ninguno**.

> Sacar todos es de otra magnitud que sacar uno, y **una lista vacía no se distingue de una lectura que no anduvo**: un 200 con lista vacía se ve igual que un sprint que existe y está vacío de verdad.

### Y el `status` sigue la misma regla

Acá decía que **no** la seguía, con el argumento de que un `status` que difiere no es *"el board moviendo algo"* sino que *"el push nunca llegó"*.

**Ese argumento era la premisa de que el worklist es la fuente, otra vez.** [Y la spec ya la había marcado como una que nadie decidió](sync.md#asignar-una-clave-crear-o-encontrar): la propuesta que origina todo esto se titula *"el proveedor es la autoridad"*.

> **El proveedor es la autoridad, y eso no se parte por campo.** Si hay proveedor, el `status` que vale es el suyo.

**Y el caso que motivaba la excepción sigue siendo real, con otro nombre.** Un ítem `done` de este lado que el board tiene en `Tareas por hacer` porque su transición nunca se propuso es [`ACC-323`](.) — **un defecto con número**, no una razón para invertir la autoridad. Lo que corresponde es arreglarlo, no protegerse de él absorbiendo menos.

**Lo que sí queda afuera, y no por autoría:** cuando la vuelta del mapeo **no es una función** —`Finalizada` vuelve a `done` y a `dropped`— no hay qué absorber. Elegir sería inventar, y eso no es el worklist discutiéndole al board: es que el board no dijo lo suficiente.
