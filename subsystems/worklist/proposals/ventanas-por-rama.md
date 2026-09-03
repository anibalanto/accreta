# Draft: el worklist se sincroniza por ramas, y el proveedor es la autoridad

**Estado:** borrador, con el primer slice en ejecución en [`51`](../../../.stratum/worklist-accreta/51.user-story.md). Salió de una conversación de diseño y de una primera subida a Jira que se frenó antes de crear nada.

El worklist vive en un repo propio y no habla con nadie. [`worklist new`](../commands/new.md) está especificado y no se puede implementar: *"el servidor asigna el próximo ID base-36"*, y **ese servidor no existe**.

Esta propuesta dice que el servidor ya existe y es el tracker —Jira, en el caso de accreta—, que el transporte es git, y que hay **dos clases de rama**, porque completo y verificable son excluyentes.

## Completo o verificable, nunca las dos

| | contiene | se puede verificar contra el proveedor | se puede empujar |
|---|---|---|---|
| **insegura** | todo | no | **no** |
| **segura** | una vista parcial, de menos de `n` ítems | sí, en cualquier momento | sí |

Una rama insegura sirve para **tener un panorama de todos los issues**, y por eso mismo no puede prometer que estén actualizados: verificarlos a todos, todo el tiempo, es exactamente el costo que este diseño existe para no pagar. Como no se puede verificar, **no se puede empujar desde ahí** — y ésa es la propiedad que hace que el nombre se gane: *insegura* no describe un riesgo abstracto, describe qué no podés hacer.

Una rama segura se hace responsable de poder verificarse entera contra el proveedor en cualquier momento. **Por eso no puede tener todos los elementos**: la responsabilidad es lo que la acota, no al revés.

> **La seguridad es una propiedad del tamaño, no del significado.** Un sprint es la ventana más común, no la única: cualquier conjunto acotado y enumerable puede ser seguro.

## La forma

`.worklist/` en la raíz del proyecto —no una capa de stratum, ver § "Por qué no es una capa"—, con un worktree por rama:

```
.worklist/.git                    ← bare
.worklist/insecure/all            ← todo. El panorama.
.worklist/insecure/backlog        ← lo que ningún sprint tomó. Crece sin techo.
.worklist/secure/sprint/10        ← una ventana con sentido
.worklist/secure/ACC-40~ACC-60    ← una ventana ad hoc, acotada por rango
```

**El path espeja el nombre de la rama, y dice la capacidad.** Parado en un directorio sabés si podés empujar sin preguntarle a git, y una IA que trabaja ahí lo sabe por su `cwd` sin que nadie se lo diga.

**Y son ramas, no un namespace propio de refs.** El worklist es un repo aparte, así que el argumento por el que [los bilinks viven fuera de `refs/heads/`](../../bilinker/concepts/ref.md#fuera-de-refsheads) —*que un clon del proyecto no los arrastre*— acá no aplica. Esto quiere branchear, mergear y diffear, que es la maquinaria de una rama.

### Cada commit de una segura es también un commit de `insecure/all`

Venga del cliente o del proveedor, todo commit que entra a una rama segura se propaga a `insecure/all`. Sin eso, el panorama sólo avanzaría cuando un sprint cierra, y estaría semanas viejo; con eso, **`all` está siempre tan al día como el último push aceptado en cualquier ventana — lo que no está es verificado.** Ésa es exactamente la promesa que una insegura puede hacer y la otra no.

### Y el estado del proveedor va a todas las ventanas donde el ítem exista

Un ítem puede estar en más de una ventana a la vez. Cuando llega estado nuevo del proveedor para `ACC-45`, se escribe **en todas las que lo tengan**, no sólo en la que disparó la consulta.

**Y eso no necesita ninguna política de conflicto**, que es lo que lo hace distinto de propagar una edición del cliente: el proveedor es **una sola fuente**, así que todas las ventanas reciben el mismo valor. No hay dos verdades que arbitrar — no hay competencia.

### Las ventanas ad hoc no son permanentes

`secure/sprint/10` dura lo que dura el sprint: está planificada, tiene nombre propio y un `.sprint.md` que la declara. **`secure/ACC-40~ACC-60` no**: se recorta para un trabajo, se usa, y se descarta cuando sus commits ya propagaron a `insecure/all`.

Las dos son seguras por la misma razón —están acotadas— pero **tienen ciclos de vida opuestos**, y la diferencia importa: una ventana ad hoc que sobreviviera indefinidamente acumularía divergencia contra las otras que comparten sus ítems, sin que nada la cierre. La no-permanencia es lo que le pone techo a ese problema en vez de dejarlo crecer.

## Por qué una ventana y no todo

Porque mantener coherente *todo* obliga a preguntarle al proveedor por todo, todo el tiempo. Con la ventana, empujar tres ítems al sprint 10 obliga a preguntar por los ítems del sprint 10 y por ninguno más.

**Es la misma forma que el sprint que está corriendo.** [`4i`](../../../.stratum/worklist-accreta/4i.task.md) dice *"`check <un archivo>` verifica la capa entera — 98 bilinks y 391 queries al proveedor para preguntar por uno"*. Acá es idéntico un piso más arriba, y la respuesta también: **acotar el alcance de la pregunta al alcance del cambio.**

> **La ventana no acota el estado, acota el chequeo.** No existe *"el status de `4j` en `sprint/10`"* distinto del que tiene en el backlog: el estado es uno, el del proveedor. Lo que la ventana delimita es de qué ítems hay que confirmar que no se movieron antes de aceptar un push.

## El push es un compare-and-swap, y git ya lo sabe hacer

El hook, antes de aceptar:

1. Le pregunta al proveedor por los ítems **de esa ventana**.
2. Si el proveedor se movió desde el commit sobre el que empujás, **rechaza** y escribe el estado nuevo en la rama.
3. Si no, aplica el cambio al proveedor y acepta.

**Eso es el chequeo de fast-forward**, con el proveedor como la cosa que movió el remoto. No hay mecanismo nuevo: del lado del cliente el flujo es `fetch`, rebase, empujar de nuevo — el que ya conoce cualquiera.

Y la propiedad que importa: **un push rechazado no deja nada en la historia.** La rama es siempre lo que el proveedor aceptó, sin cicatrices. Una versión anterior de este diseño hacía que el bot commiteara un revert con el motivo; sobra, y ensucia.

> El costo de que el proveedor mande es uno solo: tu edición local se rechaza y la rehacés sobre lo nuevo. Es el costo de `git pull --rebase`.

## Los ids son del proveedor

Un ítem nuevo se escribe con un nombre provisorio y **el hook lo renombra al aceptarlo**:

```
desacuerdo-como-entrada.task.md   →   ACC-235.task.md
```

`git mv`, así que la historia del archivo no se corta. Un id y no dos: no hay tabla de mapeo, y *"de qué board es este ítem"* lo contesta el nombre.

**Esto le pasa por encima a [`2n`](../../../.stratum/worklist-accreta/2n.task.md)**, que decidió `<proyecto>-<id>` con base-36 local. `ACC-235` cumple el propósito de `2n` mejor —el board está en el id por construcción y no por convención— pero `2n` queda para reescribir, no para archivar: la convención de commits y el formato del endpoint `issue` cambian los dos, y la regla de lectura de los commits viejos sigue haciendo falta.

### Y un pedido no se distingue por prefijo conocido, sino por el alfabeto

Un id base-36 es `[0-9a-z]` y [`2n`](../../../.stratum/worklist-accreta/2n.task.md) ya fijó que **nunca lleva guión**. Una clave de proveedor tiene mayúsculas y un guión, y ninguna de las dos cosas cabe en un id. Así que *"esto es un pedido"* no se decide contra una lista de prefijos: lo decide una partición que el alfabeto garantiza, y nada que alguien nombre a mano puede colisionar con algo que asignó el proveedor.

**De ahí sale el bootstrap, y es gratis.** Los 153 ítems que hoy tiene el worklist de accreta no llevan clave de proveedor, así que **son todos pedidos**. La migración no es un script: es empujar el repo como está y recibir 153 moves de vuelta.

### El renombre y la reescritura son un solo commit

Renombrar `agregar-funcionalidad-1.task.md` a `ACC-101.task.md` rompe todo lo que lo nombraba. **Repararlo es del hook**, y va en el mismo commit que el move:

```diff
- relation.depends: agregar-funcionalidad-1
+ relation.depends: ACC-101
```

Si el move entrara sin la reescritura, la rama quedaría con una referencia colgada y el [invariante 9 de `item.md`](../concepts/item.md#invariantes) violado. **O entran las dos cosas o no entra ninguna.**

**Y la reescritura toca sólo posiciones delimitadas**: valores de frontmatter —`parent`, `relation.*`—, destinos de link —`](…)`— e ids entre backticks, que es como la prosa los nombra. Nunca una subcadena suelta: un nombre provisorio es un string libre, así que `agregar-funcionalidad-1` es prefijo de `agregar-funcionalidad-10` y un reemplazo ciego rompe el segundo al arreglar el primero.

### El orden es para el proveedor, no para los números

El hook ordena topológicamente los pedidos del push por `parent` y `relation.*` antes de crear nada. **No es para que los ids salgan correlativos** —que salgan 101 y 102 es consecuencia, no requisito, y nadie debería leer orden en un número—: es porque **no se puede vincular a un issue que todavía no existe**, ni colgar un hijo de un padre que no está.

El orden va sobre el subgrafo de los pedidos y nada más: una dependencia hacia un ítem que ya tiene clave no impone nada, porque ya está creado. Y un ciclo se rechaza, que es el invariante 9 otra vez.

### El nombre provisorio tiene que ser único

`new.task.md` colisiona: dos personas creando a la vez chocan en el mismo path, y una sola no puede crear dos tasks en un push. El hook no necesita saber qué dice el nombre, sólo que dice *sin asignar* — así que un slug del título alcanza, y además se lee.

**Y un ítem no tiene nombre real hasta que un push entra.** Offline, o con el proveedor caído, se trabaja con el provisorio.

### Crear tiene que ser idempotente, y ahí ya hay experiencia

Si el hook crea el issue en el proveedor y falla antes de commitear el move, un reintento crea un duplicado. La defensa es buscar antes de crear, y **el modo de falla conocido no es olvidarse de buscar sino que la búsqueda falle en silencio**: en `bmad-issue-tracking` una comilla sin escapar adentro de un título rompía el JQL, la búsqueda no devolvía nada, y el reintento duplicaba. Encontrado sobre un título real.

## Lo que el renombre no puede reparar

Adentro del repo la reescritura alcanza. Afuera no: hoy son **15 referencias desde las specs de accreta** hacia ítems del worklist —ADRs que nombran la épica que ejecutan, propuestas que nombran el ítem que las abrió— que un push al worklist no puede tocar. Con el bootstrap se rompen todas juntas.

**No es un costo nuevo de esta propuesta: ya existe hoy**, y este diseño no lo agrava más de lo que el simple paso del tiempo ya lo iba a agravar. La épica `1` apunta a un arreglo posible —un endpoint `issue` del lado de la spec, que sobrevive al renombre porque bilinker lo sigue— pero **queda deliberadamente afuera de acá**: no se factorean bilinks en este diseño por ahora. Se anota como trabajo futuro y no bloquea nada de lo de arriba.

## Qué lleva una ventana

Un subárbol entero, **más la épica de la que cuelga, de sólo lectura.**

La primera mitad es del formato y no de esta propuesta: [*"lo que entra a un sprint es un subárbol entero"*](../concepts/hierarchy.md#la-regla-del-ancestro) — una user story entra con todas sus tasks o no entra. Así que **una ventana nunca parte una user story**, y la cadena `parent` de cualquier miembro cierra adentro de la ventana… hasta la épica.

La épica es la única excepción, y por una razón que ya estaba escrita: [*"una épica no entra a un sprint"*](../concepts/hierarchy.md#épicas). Sin ella, un checkout de `sprint/10` tendría user stories cuyo `parent` no resuelve. Con ella la cadena cierra siempre. Es un ancestro y no un miembro: viaja para que el árbol se lea, no para editarse ahí.

> Una versión anterior de este borrador decía que una ventana **sí** podía partir un subárbol, apoyándose en un párrafo de `hierarchy.md` que contradecía a la regla del ancestro. Ese párrafo era el error y se corrigió en [`4u`](../../../.stratum/worklist-accreta/4u.task.md). Lo que queda acá es más simple: el único ancestro que una ventana hereda es la épica.

## Las épicas no tienen ventana propia

Una épica sobrevive a todas las ventanas: la `1` de accreta tiene quince user stories repartidas en nueve sprints. Y [`hierarchy.md`](../concepts/hierarchy.md#épicas) ya dice que *"una épica no entra a un sprint"*.

Así que vive en `insecure/all` como todo lo demás, y cada ventana segura la hereda de sólo lectura, como cualquier otro ancestro — para que la cadena `parent` cierre, no para editarse ahí.

## La membresía se deriva, no se escribe dos veces

Qué ítems tiene una ventana **sale de los links del `.sprint.md`**, que ya es el único lugar donde eso está escrito. Vos editás el link; quién recorta la ventana es el bot.

Si en cambio la membresía la definiera *qué archivos hay en la rama*, habría dos fuentes para el mismo hecho, y el día que discrepen **no falla nada: miente**. Es la misma razón por la que [el backlog no es un archivo](../concepts/hierarchy.md#el-backlog-no-es-un-archivo): *"mantenerlo como archivo obligaría a editar dos lugares para mover algo, y los dos podrían divergir"*.

Y es lo que conserva la propiedad que hoy tiene mover un ítem de sprint: **una sola escritura.**

## Por qué no es una capa

Una capa de stratum es **un cuerpo de specs o código que otras capas referencian por path** — `*/subsystems/bilinker>impl`. El worklist no es eso: es el aparato de seguimiento *del* proyecto, más pariente de `.bilink/` que de `subsystems/`. Y con un worktree por rama deja de ser siquiera un directorio de documentos: pasa a ser un contenedor de checkouts, que no entra en el modelo de capas de ninguna forma.

**Y disuelve una ambigüedad concreta.** Un worktree tiene un `.git` —archivo, no directorio— en su raíz, y stratum y bilinker resuelven raíces caminando hacia arriba buscando `.git` o `.bilink`. Adentro de `.stratum/`, cada ventana se vería como una capa. En `.worklist/` nadie espera capas, y el problema no llega a existir.

Lo que cuesta, medido: [`2m`](../../../.stratum/worklist-accreta/2m.task.md) queda sin objeto —decidía declarar *cuál capa* es el board, y si no es una capa no hay nada que declarar—; `stratum pull` deja de traerlo y pasa a ser un `git clone`; bilinker cambia su resolución del endpoint `issue`; y el `.gitignore` de accreta necesita una línea propia, porque hoy `**/.stratum/*/` lo cubría de arriba. Los links relativos de ítems a specs no entran en la cuenta: [`4y`](../../../.stratum/worklist-accreta/4y.task.md) ya los convirtió a permalinks absolutos, salvo los 11 que apuntan a `impact`, que no tiene repo publicado.

## El bot es la pieza que no existe

**Atlassian no habla git.** No hay hook de Atlassian sobre una rama tuya, así que entre el push y Jira hay un servicio propio, que hay que escribir y correr. Es el único componente nuevo de todo esto, y conviene tenerlo a la vista antes que cualquier detalle de formato: dónde vive, con qué credencial, y qué pasa cuando está caído.

Lo que ya se sabe de la herramienta que usaría: `acli` está autenticado y **tiene sprints** —`acli jira sprint create --name … --board … --goal …`—, que es justo lo que el MCP de Atlassian no expone.

## Lo que ya se decidió

**`.sprint.md` vive en su propia ventana.** Un sprint no es una épica: es el plan de *esa* iteración, así que vive en la rama que esa iteración gobierna.

**`backlog` es insegura, no una ventana.** Es *"todo lo que ningún sprint tomó"* y crece sin techo, así que no puede prometer verificarse entera. Para trabajar sobre ítems del backlog se recorta una ventana segura acotada — `secure/ACC-40~ACC-60` — y se empuja ahí.

**Nadie empuja al panorama.** `insecure/all` no acepta pushes porque no se puede verificar, y por lo tanto sólo avanza por propagación desde ramas seguras. La vieja pregunta de *"si el trunk sólo avanza por merge"* deja de ser una regla que alguien tiene que respetar: **no hay forma de violarla.**

**El worklist no es una capa de stratum**, y por eso vive en `.worklist/`. Ver § "Por qué no es una capa".

## Lo que queda por decidir

**Cuánto vale `n`.** Verificar una ventana **no cuesta una query por ítem**: `key in (ACC-1, ACC-2, …)` trae todas de una, con `status` y `summary`, y tolera claves inexistentes — probado contra `ACC`. Así que el techo no es la latencia sino el largo del JQL y la paginación, y `n` en el orden de las centenas es plausible. Falta medirlo, no estimarlo.

**Quién escribe el estado del proveedor en una rama segura cuando el push se rechaza.** Sin eso, el `pull` del que fue rechazado no baja nada, vuelve a empujar lo mismo, y entra en loop. No puede ser `pre-receive`, que sólo acepta o rechaza — es la misma partición que obligó a poner el renombre en `post-receive`.

**Si una ventana puede recibir trabajo del cliente hecho en otra.** El estado del proveedor va a todas las ventanas donde el ítem exista, y eso no tiene conflicto posible. Una **edición de prosa** hecha por alguien en `sprint/8` no: si `ACC-45` también está en un rango ad hoc, la otra ventana no se entera. Las dos salidas son opuestas — propagar también las ediciones del cliente entre ventanas, o **recortar las ventanas desde `insecure/all`** y que `all` sea el único punto de convergencia. La segunda es más simple y hace de las ventanas vistas derivadas; falta confirmarlo.

**Qué gana un conflicto que el compare-and-swap no ve.** Dos personas editando la *prosa* del mismo ítem no mueven nada en el proveedor, así que el hook acepta las dos y el conflicto es de git, no de Jira. Está bien, y hay que decirlo.

**Si esto es de worklist o del proyecto.** El diseño no nombra a Jira en ninguna regla: nombra *un proveedor que asigna ids y tiene estado*. Vale la pena mantenerlo así, y que `ACC` sea una configuración.

## Qué haría falta antes de decidir

El bot, en su versión más chica: una ventana, un sprint, y un push que asigne un id. Todo lo demás de esta página es razonamiento, y esa parte es la única que no se puede razonar de antemano.
