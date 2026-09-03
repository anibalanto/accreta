# Draft: `PAUSED` — un `issue` cerrado no vuelve `ALTERED` para siempre a su otro extremo

**Estado:** borrador. Salió de discutir [`bilink-tasks.md`](../../worklist/concepts/bilink-tasks.md) § Completar, que hoy dice que el bilink de tarea *"permanece como registro histórico"* sin decir qué le pasa al reporte.

## El problema, en una frase

Un capture vive mientras la spec vive. Un `issue` vive mientras hay algo que decidir. Si el issue cierra y la spec sigue evolucionando —tiene que, es lo que se le pide—, el otro extremo del bilink queda `ALTERED` **para siempre**, sobre una decisión que nadie va a volver a tomar. Información que nunca deja de sonar es indistinguible de ruido.

## La salida no es escribir nada nuevo — ya está adentro del hash

**[Invariante 7 de `bilink.md`](../concepts/bilink.md#invariantes)**: *"Un endpoint `issue` se hashea como el contenido del archivo del ítem."* El archivo entero, frontmatter incluido. Así que `status: done` **ya es parte de lo que se acepta** el día que alguien acepta ese endpoint por última vez con el ítem cerrado.

No hace falta un campo `paused: true` en ningún lado. La señal es derivada, como todo lo demás en este formato: **`check` la lee comparando el `status` del último `accepted` del endpoint `issue` contra `done` o `removed`** — los dos estados terminales de [`item.md`](../../worklist/concepts/item.md#estados-y-transiciones), los únicos de los que no se sale.

## El efecto: un estado nuevo, no un silencio

> **`PAUSED`** reemplaza a `ALTERED` (y a `RESTYLED`, `EXPANDED`, `RELOCATED`, `CONSENSUS_DIVERGED`) en el **otro** extremo del bilink, cuando el `issue` está aceptado como cerrado.

No colapsa a `OK`. Sería mentir en la otra dirección: hay drift real, y `OK` afirma que no hay nada que hacer. `PAUSED` dice lo que es — *"hay drift, y nadie lo va a mirar porque quien tenía que mirarlo ya cerró"*. Es la misma razón por la que `RESTYLED` no colapsa a `OK` cuando el texto cambió y los tokens no: un estado distinto para un caso distinto, nunca un state existente estirado para cubrirlo.

**Y no falla el comando.** Exit code 0, igual que `TODO` — es el precedente más cercano: *"la capa vecina todavía no está, y eso no es una falla"*. `PAUSED` es la misma figura del otro lado del tiempo: *"ya no hay a quién preguntarle, y eso tampoco es una falla."*

## Qué sí sigue reportando, y por qué

`PAUSED` cubre el eje de **contenido comparado contra lo último aceptado** — eso es lo que nadie va a volver a aprobar. No cubre:

- **La resolución del capture** (`MOVED`, `REANCHORED`, `UNANCHORED`). Que el fragmento se pueda seguir encontrando es independiente de si alguien lo va a aprobar, y un capture roto es candidato a `prune` con o sin issue.
- **El estado del propio endpoint `issue`.** Si el ítem se reabre —`status` vuelve a `open` o `in-progress`—, ese cambio de contenido es en sí mismo `ALTERED` sobre el endpoint `issue`, y alguien tiene que aceptarlo. **Es la salida del pausado, y no hace falta un comando de "unpause": re-aceptar el `issue` con el nuevo `status` listo alcanza.** Mientras ese endpoint siga en `ALTERED`, el otro extremo no puede pausarse todavía — pausar depende de lo *aceptado*, no de lo que el archivo diga en el momento.

## Consecuencia que no buscaba y vale nombrar

`bilink-tasks.md` ata el bilink de tarea a un endpoint de tipo `bilink` —*"apunta a otro `.bilink`"*— que está especificado y no implementado. **Esto no lo necesita.** Un bilink directo entre un `capture` (o `path`) y un `issue` ya existe en la grilla de tipos de hoy, y `PAUSED` se calcula igual sobre él. La indirección de `bilink-tasks.md` sigue teniendo su propio motivo —conectar un bilink entero, no un fragmento—, pero el caso más chico y más común no tiene que esperarla.

## Lo que queda por decidir

**Si `PAUSED` entra a la grilla de estados de `check.md`** con su propia fila, o si es un modificador que se antepone —`PAUSED(ALTERED)`— para no perder cuál habría sido el estado de fondo. La segunda preserva más información y es más fea de leer.

**Si aplica también al vecindario (`n.1`).** Por consistencia con el eje de contenido, sí — un vecindario que cambió tampoco lo va a mirar nadie. Pero el vecindario ya tiene su propio apagado (`declined`), y hay que decir cómo conviven sin que se pise uno con otro.

**Qué pasa con `CONSENSUS_DIVERGED` si alguien sigue proponiendo entradas sobre un issue cerrado.** Es un caso raro —alguien trabajando activamente sobre algo que el tracker dice cerrado— y probablemente no debería pausarse: la divergencia es sobre personas en desacuerdo, no sobre si el issue sigue vivo. Se deja fuera de esta propuesta hasta que aparezca.

**El nombre.** `PAUSED` describe bien la intención pero no dice la causa. `CLOSED_UPSTREAM` es más preciso y peor de leer en una tabla.

## Qué haría falta antes de decidir

Un bilink de tarea real, cerrado, con la spec del otro lado siguiendo viva un tiempo después. Hoy no hay ninguno: `4z` es la primera vez que esto se pensó.
