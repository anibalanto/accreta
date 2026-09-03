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

## Dos pasos, no uno

`pre-receive` sólo puede aceptar o rechazar lo que llega — no puede reescribirlo. El compare-and-swap vive ahí. El renombre y la reescritura son un commit *nuevo*, agregado después de aceptar: viven en `post-receive`. El cliente que empujó tiene que hacer `fetch` para ver los ids reales — ver [`commands/push.md`](../commands/push.md).
