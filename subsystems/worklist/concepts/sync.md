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

## La ventana y el compare-and-swap

Una ventana es un sprint o el backlog — un subconjunto de ítems sobre el que un push se valida. Antes de aceptar:

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

## Asignar una clave: crear o encontrar

Un pedido se resuelve preguntándole al proveedor si ya existe —por título— antes de crear. Sin esto, un hook que crea el issue y falla antes de comitear el renombre duplica en el reintento.

```
Creator::create_or_find(titulo, tipo, descripcion) -> clave
```

**Buscar mal es peor que no buscar.** Una búsqueda que falla en silencio —una comilla sin escapar rompiendo el JQL— vuelve indistinguible *"no existe"* de *"no se pudo preguntar"*, y el reintento duplica. El título se escapa siempre antes de interpolarlo en cualquier consulta.

**Y el título de búsqueda no es el mismo string que el título del issue.** `summary ~` de JQL no compara texto literal: es una búsqueda de texto completo, y algunos caracteres —`[` y `]`, confirmado— rompen su parser incluso adentro de comillas, sin que JQL tenga forma de escaparlos (`\[` es una secuencia inválida). La búsqueda usa una versión del título con esos caracteres neutralizados; **crear usa el título real, intacto** — `--summary` no pasa por JQL y no tiene ese problema. Buscar con una versión más laxa puede traer falsos positivos; no buscar nada, por un carácter que rompe el parser, produce un duplicado seguro. Lo primero es el riesgo que vale correr.

**La descripción es siempre una línea plana**, nunca el cuerpo del ítem: `Fuente: <ruta>`. Este diseño trata a git como la fuente de verdad y a Jira como quien asigna la clave y trackea el estado — no como un espejo del contenido. Una línea sin formato es ADF válido sin convertir nada; el cuerpo tiene tablas, listas y blockquotes que `acli --description` no interpreta si llegan como Markdown crudo.

### El tipo es del worklist, no de Jira

`Creator::create_or_find` recibe el tipo en el vocabulario del worklist —`task`, `user-story`, `epic`— nunca el nombre que Jira le da. Quién habla con Jira es quien sabe traducirlo, y la traducción es por proyecto: en `ACC` los tipos están localizados —`Tarea`, `Historia`, `Epic`, más `Subtask`, `Error`, `Mejora` que el worklist no usa—, y un `--type Task` en inglés no existe ahí y falla.

```
task       -> Tarea
user-story -> Historia
epic       -> Epic
```

Confirmado contra `ACC` real. Otro proyecto de Jira puede tener otro vocabulario — la tabla es de esta capa, no universal.

## Dos pasos, no uno

`pre-receive` sólo puede aceptar o rechazar lo que llega — no puede reescribirlo. El compare-and-swap vive ahí. El renombre y la reescritura son un commit *nuevo*, agregado después de aceptar: viven en `post-receive`. El cliente que empujó tiene que hacer `fetch` para ver los ids reales.
