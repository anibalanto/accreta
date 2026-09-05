---
name: worklist
description: "Cómo leer y mover el trabajo de este proyecto — épicas, user stories, tasks y sprints en `.worklist/insecure/all/`. Cargar esta skill cuando haya que saber qué sigue, qué hay en el sprint actual, qué está en el backlog, dónde está una tarea, o al crear o mover un ítem. También al planificar o repartir trabajo."
---

El trabajo vive en `<project-root>/.worklist/insecure/all/`, que es un worktree del repo propio del worklist — no una capa de stratum, así que un clon de accreta no lo trae y `stratum pull` tampoco. Spec completa en [`subsystems/worklist/`](../../../subsystems/worklist/concepts/item.md) — acá va lo operativo.

El repo es local: un bare en `~/.local/share/accreta/worklist-sync/worklist.git`. **El proyecto lo nombra el directorio y no la rama**, porque el worklist es de accreta y la herramienta no; `insecure/all` es el panorama y se llama igual en todos, porque lo nombra la spec de sincronización. Así lo resuelve `bilinker` un endpoint `issue`, y **siempre contra el panorama, nunca contra una ventana**: si resolviera contra la rama abierta, el mismo `issue 3a` resolvería o no según qué sprint tengas cortado.

## Se trabaja en una vista segura. El panorama es para leer

> **`insecure/all` no se toca.** Toda modificación de un ítem se hace parado en una vista segura — `secure/…`.

No es una convención: una rama insegura **no se puede verificar** —crece sin techo— así que no acepta escrituras, y no hay forma de empujarle. Lo que se escriba ahí no cruza al proveedor y no pasa por ningún chequeo. **Trabajar en el panorama es escribir en el único lugar del que nada sale.**

El panorama es lo que se **consulta**: qué sigue, dónde está un ítem, qué lleva un sprint, y contra qué resuelve un endpoint `issue`. Para eso es, y para eso lo tiene todo.

Para trabajar se corta la vista, y se edita ahí:

```bash
worklist window open <sprint-id>          # produce secure/sprint/<id>
```

### Y hay un hueco, que conviene saber antes de chocarlo

**Un ítem que no está en ningún sprint no tiene vista segura donde cortarse.** El backlog es inseguro por definición, y la vista para trabajarlo todavía no existe — es lo que resuelven las tasks `5q` (`to-work`), `5n` y la user story `6j`.

| | |
|---|---|
| el ítem pertenece a un sprint | se crea y se edita **adentro de su ventana** |
| no pertenece a ninguno | hoy no hay dónde, y hacerlo en el panorama es una excepción a la vista |

Decirlo así y no dar una regla que a veces no se puede cumplir: la excepción existe, y lo que importa es que **se note al hacerla** en vez de que sea el camino por defecto.

### Dos chequeos antes de escribir una línea de código

> **Las dos son bandera roja: no se avanza, se arregla primero.**

**Uno: la vista es insegura.** `worklist is-secure` en falso quiere decir que lo que escribas no se puede empujar, no se verifica y no cruza al proveedor. No es una advertencia sobre después: es que el trabajo no tiene dónde ir.

**Dos: el ítem todavía lleva `@`.** Un `@<slug>` es un id que el servidor todavía no reemplazó, así que **no hay con qué prefijar el commit** — y `AGENTS.md` § Commits pide que arranque con el id del ítem. Trabajar antes de sincronizar deja una historia que nombra un id que va a dejar de existir.

La salida es la misma en los dos casos, y es el orden que el método ya pedía con el paso que faltaba:

```
crear el ítem  →  sincronizar  →  cortar la vista  →  trabajar
```

**La tarea no está lista cuando se escribe: está lista cuando tiene id.** Es lo que hace que el `@` viva minutos en vez de meses.

### Cómo saber dónde estás parado

Hoy, mirando el path. El comando que lo contesta —`worklist is-secure`— está decidido en la task `6i` y no existe todavía, así que el chequeo es a ojo — y por eso conviene tenerlo escrito.

## Qué hay

Todos los ítems son **archivos sueltos en la raíz**. No hay carpetas por ítem.

```
1.epic.md                 épica 1
n.user-story.md           parent: 1
o.task.md                 parent: n
q.task.md                 sin parent — suelta
_sprints/                 el otro eje; el `_` garantiza que nunca sea un id
  1.sprint.md … 7.sprint.md
```

| Tipo | Sufijo | Puede tener de padre |
|---|---|---|
| Epic | `.epic.md` | nada |
| User Story | `.user-story.md` | nada, epic |
| Task | `.task.md` | nada, epic, user story |
| Sprint | `.sprint.md` | nada — vive en `_sprints/` |

Ids **base-36** (`1…9, a…z, 10…`), de orden de creación, no de prioridad. Los sprints llevan **contador aparte**: `_sprints/1.sprint.md` es el sprint 1. Por eso `1` solo es ambiguo — desambiguar con el sufijo: `show 1` es el ítem, `show 1.sprint` es el sprint.

Frontmatter, cuatro campos obligatorios y uno opcional:

```yaml
---
title: <string>
status: open | in-progress | done
created_at: <iso8601-utc>
updated_at: <iso8601-utc>
parent: <id>              # opcional — ausente en un ítem de raíz
relation.<tipo>: [<id>, …] # opcional — `relation.depends` es el que se usa
---
```

Un `.sprint.md` lleva además `items: [<id>, …]` en vez de `parent`: los ids que referencia directamente, y ni uno más — la regla del ancestro dice que un ítem entra con su subárbol entero, así que `items` nunca nombra una task cuya user story ya está en la lista.

Nada más. El tipo lo dice la extensión, la pertenencia la dice `parent`, y la asociación con bilinks **se declara desde el bilink, no desde el ítem**.

**Los hijos se calculan**: los hijos de `n` son los ítems cuyo `parent` es `n`. No hay lista que mantener, igual que el backlog.

## Dos formas de agrupar, y no se mezclan

- **Épica → US → task** es *descomposición*: el campo `parent`.
- **Sprint → ítems** es *planificación*: el campo `items` del `.sprint.md`.

Las dos son campos y las dos se editan en un solo lugar. La diferencia es qué preguntan: `parent` dice de qué es parte un ítem, el sprint dice cuándo se hace.

**El cuerpo del sprint sigue siendo prosa**, y es donde va todo lo que no es membresía: por qué esos ítems son un sprint, en qué orden, qué quedó afuera, cómo cerró. `items` reemplaza al link que declaraba pertenencia, no al texto que explica por qué.

**La regla del ancestro:** *lo que entra a un sprint es **un subárbol entero**.* Una user story entra con **todas** sus tasks, o no entra — sus tasks no se enumeran, van con ella. Y una task se nombra sola sólo cuando no cuelga de ninguna user story. La cadena de ancestros se lee siguiendo `parent` hasta que se acaba.

No alcanza con decir *"entra sólo si ninguno de sus ancestros entra"*: eso deja pasar una task suelta cuando su user story no entra a **ningún** sprint, y por esa puerta la user story queda partida igual, con la única diferencia de que nadie la nombró. La unidad es el subárbol, no la ausencia de conflicto.

De ahí sale que **una US no puede atravesar sprints**: si no cabe en una iteración, está mal dimensionada — y eso es un problema de descomposición, no de planificación.

## Cómo se nombra un ítem

> **`<id> <título>`.** Nunca el id solo.

En markdown el link envuelve las dos cosas:

```markdown
[`3z` Error de cobertura: el vecindario de una firma resuelve a la firma misma, y el tipo que importa no se pregunta](3z.task.md)
```

El id solo obliga a abrir el archivo para saber de qué se habla, y **el que lo abre es el que menos contexto tiene**: quien escribió la referencia ya sabía cuál era. Vale en los ítems y en los sprints.

### Y en una spec no se cita un ítem

> **Ninguna spec nombra un ítem.** Ni con un link, ni con un `` `5y` `` en la prosa.

El id de un ítem **cambia por diseño**: `@arreglar-el-hook` pasa a `ACC-347` el día que cruza. El renombre reescribe su propio repo y ninguno más, así que una cita desde una spec queda apuntando a un archivo que ya no existe — y nada lo detecta, porque es markdown, no un bilink.

**Una spec que cita un ítem apuesta a que su id no cambie.**

Y el proyecto ya tiene dónde va cada cosa:

| | |
|---|---|
| **la spec** | lo que es cierto |
| **el ADR**, en `docs/adr/` de la capa impl | la decisión, y por qué se tomó |
| **el ítem** | el trabajo que la ejecuta |

Una spec que necesita justificar algo cita **el ADR**. Si la justificación sólo existe adentro de un ítem, lo que falta es el ADR — la cita es el síntoma.

**Un endpoint `issue <id>` es otra cosa** y sí referencia un ítem: es una referencia verificada y repuntable, que `bilinker` mantiene. La regla es sobre la prosa, no sobre el mecanismo que existe para esto.

**El título va verbatim, no una glosa.** Una glosa envejece cuando el título cambia y nada lo detecta; un título copiado envejece igual, pero se ve al lado del que cambió. Y donde el título no describa al ítem, lo que hay que arreglar es el título.

Las referencias ya escritas **se corrigen al tocarlas**, no de una barrida.

## Cómo contestar "qué sigue"

1. Buscar el sprint con `status: in-progress`. Si no hay, el próximo `open` por número.
2. Sus links son el compromiso de la iteración. Bajar a la US y de ahí a sus tasks.
3. Cada task dice **qué specs toca**, no qué archivos de código: el código sale de los bilinks que se rompan.

**El backlog no es un archivo.** Se calcula, no se mantiene: tenerlo escrito obligaría a editar dos lugares al mover algo. Y el cálculo va sobre el subárbol, que es lo que un sprint referencia — **un ítem está en el backlog si el tope de su rama no lo nombra ningún sprint**. Las tasks de una user story planificada no se cuentan aparte, y una user story que ningún sprint nombra está en el backlog con todas sus tasks, sin importar cuántas alguien haya querido adelantar.

## Al crear o mover

Los ítems **se escriben a mano hoy**: `worklist new` está especificado pero no implementado, y además delega la asignación de ids a un servidor que no existe. Al crear uno, tomar el siguiente id base-36 libre del contador que corresponda, y escribirlo en la raíz de la vista con su `parent`.

**Y en la vista donde se va a trabajar** — ver § "Se trabaja en una vista segura". Si el ítem pertenece a un sprint, en su ventana; si no pertenece a ninguno, hoy no hay dónde y se hace en el panorama sabiendo que es la excepción.

Mover un ítem es editar **un solo campo o un solo link**, nunca un archivo:

- de sprint: el link sale de un `.sprint.md` y entra en otro.
- de padre: cambia `parent`. El archivo no se mueve, así que su path no cambia y ningún bilink que lo apunte se entera.

### El título

> **Infinitivo cuando ya sabés qué hacer. Diagnóstico cuando lo que tenés es el síntoma.**

No son dos gustos: un título en infinitivo **obliga a nombrar la solución**. Sobre un bug recién visto eso es inventarla antes de decidirla, y el título queda casado con una hipótesis. Sobre trabajo ya decidido es al revés — un título diagnóstico hace parecer que la tarea *causa* el problema en vez de resolverlo.

**Infinitivo:** *"Especificar e implementar `bilinker history`"* · *"Proteger `refs/bilink/*`"* · *"Reescribir `concepts/migration.md`"*.

**Diagnóstico**, con la forma `<Categoría>: <síntoma observable> <cuándo pasa>` — el síntoma en términos de lo que se ve, no de la causa sospechada:

> **Error de reporte:** un endpoint no resuelto no informa qué query ni qué anchor buscaba

El vocabulario de categorías **se deja crecer**, no se inventa por adelantado.

En los dos casos el **cuerpo** abre con el diagnóstico. Lo que cambia es el título.
