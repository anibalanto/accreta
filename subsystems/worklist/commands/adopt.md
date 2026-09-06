# Comando: `worklist adopt`

Adopta un proveedor sobre un worklist que ya tiene historia: crea del otro lado lo que todavía no existe, y renombra de este lado todo lo que se llamaba con un id local.

```
worklist adopt <proveedor> --project <clave> [--limit <n>] [--dry-run]
```

```bash
worklist adopt jira --project ACC
```

> **Lo corre el servidor.** Habla con el proveedor y escribe sobre el panorama, que son las dos cosas que el cliente no hace. Ver [`distribution.md`](../concepts/distribution.md).

## Cuándo existe, y cuándo no

**Sólo para un worklist que vivió sin proveedor.** Sus ítems tienen ids base-36 —`8y`, `1f`— y hay que llevarlos a claves del proveedor.

Uno que nació con Jira **nunca lo corre**: sus ítems fueron `@<slug>` y después `ACC-…`, sin pasar por base-36 en el medio.

| Cuándo | Qué pasa | Alcance |
|---|---|---|
| **al sincronizar** | `@<slug>` → su id | **un ítem**, el que se empujó |
| **al adoptar un proveedor** | base-36 → clave del proveedor | **por lotes**, todo el corpus |

Las dos son la misma operación con distinto alcance, y la primera ya existe: la pasada 1 de [`assign-keys`](assign-keys.md) es un `adopt` de un ítem. Lo que este comando agrega es el caso de lote.

> **La migración no es un evento: es el estado normal de un ítem al cruzar por primera vez.**

## Es por lotes, y el lote no es una comodidad

`--limit <n>` corta después del orden topológico, igual que [`bootstrap`](bootstrap.md): una épica creada con sus tasks sin crear es un estado válido, y al revés el orden lo impide.

**No hay marca de progreso.** Lo que falta es lo que todavía no tiene clave, así que la corrida siguiente lo toma sin contabilidad extra — y una corrida que se cae deja del lado de acá las claves que alcanzó a conseguir, que ya están pagas del otro.

## `--dry-run` no es una cortesía

> **`adopt` crea issues, y crear un issue es un efecto afuera, irreversible y pago.** Equivocarse de proyecto cuesta tanto como acertarle.

Con `--dry-run` imprime el mapeo entero —qué id local se convierte en qué pedido, y bajo qué épica— sin tocar el proveedor ni el árbol. Es barato de mirar y caro de saltear.

## Qué renombra, y no son sólo los archivos

| | |
|---|---|
| los archivos del worklist | el renombre y la reescritura son un solo commit, y arrastran cada referencia cruzada |
| los bilinks con endpoint `issue` | `issue 8y` → `issue ACC-347`. Cambia el `link`, así que quedan `RELOCATED` y se cierran con `accept --place` |
| las menciones en specs | **no debería haber ninguna**: una spec no cita un ítem, porque su id cambia por diseño. Las que aparezcan son el síntoma de un porqué que le falta a la spec |

## Y hay algo que no puede tocar

> **Los mensajes de commit.** La historia no se reescribe, así que después de adoptar `git log --grep '^8y:'` encuentra commits que el árbol ya no nombra.

**Decidido: siempre para adelante.** No se guarda registro de la migración y no hay un `map-id` que traduzca ids viejos.

**Y no hace falta, porque el log del remoto ya es el mapa**: los `rename <viejo> -> <nuevo>` que el hook escribe cubren cada clave asignada. Un `map-id` sería un índice de algo ya indexado, y una segunda fuente que puede diferir de la primera.

El costo es acotado por una razón que conviene decir: **la historia vieja se lee por su contenido, no por su id.** Lo que se pierde es el salto automático de un commit a su ítem, no la posibilidad de entender qué se hizo.

## Y empieza desde donde esté, no desde cero

Un worklist a medio migrar —parte de los ítems con clave y parte sin— es el caso normal y no un accidente: es lo que deja cualquier adopción por lotes interrumpida. `adopt` **contempla ese estado o empieza por normalizarlo**, y en ningún caso supone que nada cruzó.

## Dejar un proveedor no deshace nada

Si los ids migraron a Jira y mañana se abandona Jira, los ítems siguen llamándose `ACC-…` por algo que ya no existe.

> **El nombre queda como fósil, y no se migra de vuelta.**

Es la misma decisión que los commits, un nivel más arriba: volver atrás rompería cada referencia una segunda vez —archivos, bilinks, historia— a cambio de un nombre más prolijo. Un id es una etiqueta estable, y de dónde salió es una nota al pie.

## Exit codes

| Código | Condición |
|--------|-----------|
| `0` | adoptado, o `--dry-run` completo |
| `1` | ciclo en las dependencias, o proyecto inexistente del otro lado |
| `2` | el proveedor rechazó una creación; lo conseguido queda escrito |
