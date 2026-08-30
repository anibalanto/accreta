# Integración con worklist

Worklist usa bilinker como infraestructura de linking. Cada ítem de worklist nace con un bilink al fragmento que lo origina.

## Creación

`worklist new` ejecuta `bilinker capture` internamente y crea el bilink:

```bash
worklist new task "implementar bilinker accept" concepts/bilink.md:104:1
```

Produce `.bilink/<uuid>.bilink` que conecta el ítem de worklist con el fragmento exacto de la spec.

## Drift

Cuando el fragmento al que apunta un ítem cambia, `bilinker check` detecta `ALTERED` en el `source_bilink`. Es una señal para que el desarrollador evalúe si el ítem sigue siendo válido.

## Trabajo pendiente sobre un endpoint

**Se declara con otro bilink, no con un archivo índice.** Un bilink de tarea conecta el bilink estructural con el ítem del worklist; su forma y su ciclo de vida están en [worklist — asociación tarea ↔ bilink](../../worklist/concepts/bilink-tasks.md), que es la fuente.

Este documento describió un `<bilink-uuid>.tasks` con un id por línea, que nunca existió y que `bilink-tasks.md` invariante 3 descarta explícitamente. Un archivo así sería un índice: derivable de los bilinks de tarea, y por lo tanto una segunda copia de la misma verdad que alguien tendría que mantener sincronizada.

Que la asociación sea un bilink y no un campo es lo que la hace verificable: los dos extremos se chequean, y si el ítem cambia o el fragmento cambia, `check` lo dice. Un índice no chequea nada.

## Ver también

[Worklist — integración con bilinker](../../worklist/integration/bilinker.md)
