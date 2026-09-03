# Draft: el worklist se sincroniza por ramas, y el proveedor es la autoridad

**Estado:** borrador. La forma está propuesta y nada está decidido — salió de una conversación de diseño y de una primera subida a Jira que se frenó antes de crear nada.

El worklist vive en un repo propio y no habla con nadie. [`worklist new`](../commands/new.md) está especificado y no se puede implementar: *"el servidor asigna el próximo ID base-36"*, y **ese servidor no existe**.

Esta propuesta dice que el servidor ya existe y es el tracker —Jira, en el caso de accreta—, que el transporte es git, y que la unidad de sincronización es una **ventana**: un sprint, o el backlog.

## La forma

Ramas de verdad, en `refs/heads/`, en el repo del worklist:

```
closed              ← el trunk. Épicas y todo lo entregado.
  ├ sprint/<id>     ← una ventana por sprint abierto
  └ backlog         ← la ventana de lo que ningún sprint tomó
```

Cada ventana sale del trunk. Al cerrarse un sprint, su rama **se mergea** al trunk: no junta estados divergentes, cierra la ventana — esos ítems dejan de estar gobernados por una y pasan a ser historia.

**Y son ramas, no un namespace propio.** El worklist es un repo aparte, así que el argumento por el que [los bilinks viven fuera de `refs/heads/`](../../bilinker/concepts/ref.md#fuera-de-refsheads) —*que un clon del proyecto no los arrastre*— acá no aplica: no hay proyecto del cual esconderse. Y esto quiere branchear, mergear y diffear, que es exactamente la maquinaria de una rama. Un namespace propio se justifica cuando la ref **no** es una rama: append-only, sin merge, verificada del lado del servidor. `refs/bilink` es eso; esto no.

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

### La regla que evita la cascada de renombres

Renombrar un archivo rompe todo lo que lo apuntaba. En el worklist de accreta eso es, hoy:

| | |
|---|---|
| referencias entre ítems | **554**, en 128 archivos |
| ítems con `parent:` | 79 |
| **referencias desde las specs de accreta** | **21** |

Las últimas 21 deciden la forma: **viven en otro repo.** Un push al worklist no puede reescribirlas, y renombrar igual las rompe en silencio — la peor manera, la que `2n` describe cuando dice que *"la ambigüedad no falla"*.

> **No se puede referenciar un ítem que todavía no tiene id.**

Con eso el hook **sólo renombra archivos que nadie apunta**, y la cascada no existe. No queda prohibida por una guarda: queda irrepresentable.

El costo es chico y concreto: crear una task y meterla en un sprint son **dos pushes**. Y una task nueva cuyo padre también es nuevo cae bajo la misma regla — primero el padre.

### El nombre provisorio tiene que ser único

`new.task.md` colisiona: dos personas creando a la vez chocan en el mismo path, y una sola no puede crear dos tasks en un push. El hook no necesita saber qué dice el nombre, sólo que dice *sin asignar* — así que un slug del título alcanza, y además se lee.

**Y un ítem no tiene nombre real hasta que un push entra.** Offline, o con el proveedor caído, se trabaja con el provisorio. Está bien mientras nadie lo referencie, que es la regla de arriba otra vez.

## Qué lleva una ventana

Un subárbol entero, **más la épica de la que cuelga, de sólo lectura.**

La primera mitad es del formato y no de esta propuesta: [*"lo que entra a un sprint es un subárbol entero"*](../concepts/hierarchy.md#la-regla-del-ancestro) — una user story entra con todas sus tasks o no entra. Así que **una ventana nunca parte una user story**, y la cadena `parent` de cualquier miembro cierra adentro de la ventana… hasta la épica.

La épica es la única excepción, y por una razón que ya estaba escrita: [*"una épica no entra a un sprint"*](../concepts/hierarchy.md#épicas). Sin ella, un checkout de `sprint/10` tendría user stories cuyo `parent` no resuelve. Con ella la cadena cierra siempre. Es un ancestro y no un miembro: viaja para que el árbol se lea, no para editarse ahí.

> Una versión anterior de este borrador decía que una ventana **sí** podía partir un subárbol, apoyándose en un párrafo de `hierarchy.md` que contradecía a la regla del ancestro. Ese párrafo era el error y se corrigió en [`4u`](../../../.stratum/worklist-accreta/4u.task.md). Lo que queda acá es más simple: el único ancestro que una ventana hereda es la épica.

## Las épicas viven en el trunk

Una épica sobrevive a todas las ventanas: la `1` de accreta tiene quince user stories repartidas en nueve sprints. Y [`hierarchy.md`](../concepts/hierarchy.md#épicas) ya dice que *"una épica no entra a un sprint"*.

Así que no tiene ventana propia: vive en el trunk, y cada ventana la hereda de sólo lectura, como cualquier otro ancestro.

## La membresía se deriva, no se escribe dos veces

Qué ítems tiene una ventana **sale de los links del `.sprint.md`**, que ya es el único lugar donde eso está escrito. Vos editás el link; quién recorta la ventana es el bot.

Si en cambio la membresía la definiera *qué archivos hay en la rama*, habría dos fuentes para el mismo hecho, y el día que discrepen **no falla nada: miente**. Es la misma razón por la que [el backlog no es un archivo](../concepts/hierarchy.md#el-backlog-no-es-un-archivo): *"mantenerlo como archivo obligaría a editar dos lugares para mover algo, y los dos podrían divergir"*.

Y es lo que conserva la propiedad que hoy tiene mover un ítem de sprint: **una sola escritura.**

## El bot es la pieza que no existe

**Atlassian no habla git.** No hay hook de Atlassian sobre una rama tuya, así que entre el push y Jira hay un servicio propio, que hay que escribir y correr. Es el único componente nuevo de todo esto, y conviene tenerlo a la vista antes que cualquier detalle de formato: dónde vive, con qué credencial, y qué pasa cuando está caído.

Lo que ya se sabe de la herramienta que usaría: `acli` está autenticado y **tiene sprints** —`acli jira sprint create --name … --board … --goal …`—, que es justo lo que el MCP de Atlassian no expone.

## Lo que queda por decidir

**Dónde se edita el `.sprint.md`.** Si vive en el trunk, mover un ítem de sprint es un push al trunk que obliga a recortar dos ventanas. Si vive en la ventana, hay una copia por rama y vuelve la divergencia que la sección anterior evita.

**Qué pasa con un ítem que no está en ninguna ventana.** El backlog es *"lo que ningún sprint referencia"*, calculado. Como rama tiene que ser un conjunto concreto de archivos, y hay que decir quién lo recorta y cuándo.

**Si el trunk sólo avanza por merge.** Prohibir el push directo hace que `git log --merges closed` sea la lista de sprints entregados sin que nadie la mantenga. El costo es que arreglar un typo en una épica necesita una rama.

**Qué gana un conflicto que el compare-and-swap no ve.** Dos personas editando la *prosa* del mismo ítem no mueven nada en el proveedor, así que el hook acepta las dos y el conflicto es de git, no de Jira. Está bien, y hay que decirlo.

**Si esto es de worklist o del proyecto.** El diseño no nombra a Jira en ninguna regla: nombra *un proveedor que asigna ids y tiene estado*. Vale la pena mantenerlo así, y que `ACC` sea una configuración.

## Qué haría falta antes de decidir

El bot, en su versión más chica: una ventana, un sprint, y un push que asigne un id. Todo lo demás de esta página es razonamiento, y esa parte es la única que no se puede razonar de antemano.
