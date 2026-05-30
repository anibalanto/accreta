# Integración con bilinker

bilinker es la infraestructura de linking de worklist. La relación entre tareas y fragmentos de código/spec se declara mediante **bilinks de tarea** — bilinks que conectan un bilink estructural con un ítem del worklist.

## Creación del bilink de tarea en `worklist new`

Cuando `worklist new` recibe un selector, `bilinker capture` ubica el fragmento referenciado, crea un bilink estructural si no existe, y luego crea el bilink de tarea:

```bash
# Desde spec hacia impl
cd accreta/bilinker/specific
worklist new task "implementar bilinker accept" bilink-format.md:104:1

# Desde impl hacia spec
cd accreta/bilinker/.stratum/impl/src
worklist new task "actualizar spec en base a cambio en accept()" lib.rs:88:1
```

En ambos casos produce:

```
accreta/.bilink/<uuid-struct>.bilink      ← bilink estructural: fragmento ↔ layer
accreta/.bilink/<uuid-task>.bilink        ← bilink de tarea: .bilink/<uuid-struct>.bilink ↔ task <id>
accreta/.stratum/worklist/<id>.task       ← ítem del worklist
```

La asociación vive en el bilink de tarea. El ítem no tiene `source_bilink` en su frontmatter.

## Trazabilidad

El bilink de tarea conecta el trabajo pendiente con su origen exacto. Cuando el fragmento cambia:

1. `bilinker check` detecta `ALTERED` o `CHAIN_DIRTY` en el bilink estructural
2. El bilink de tarea detecta `ALTERED` en `state.0` (hash del bilink estructural cambió)
3. El desarrollador evalúa si la tarea sigue siendo válida

## Al completar el trabajo

Cuando el trabajo está listo, la tarea se marca `done`. El bilink estructural queda como registro del vínculo establecido entre fragmentos. El bilink de tarea queda como registro histórico de la asociación.
