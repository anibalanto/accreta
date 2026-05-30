# Asociación tarea ↔ bilink

La relación entre una tarea del worklist y un bilink estructural se declara mediante un **bilink de tarea** — un bilink separado que conecta ambos extremos.

## Formato

```
# .bilink/<uuid>.bilink   (vive en la capa donde se ejecuta la tarea)
link.0: .bilink/<uuid-estructural>.bilink   ← bilink estructural como archivo
link.1: task 3a                             ← ítem del worklist
```

`link.0` apunta al archivo `.bilink` del bilink estructural (tratado como endpoint estructural — se hashea su contenido). `link.1` apunta al ítem `3a` en `<project-root>/.stratum/worklist/3a.task`.

`bilinker check` detecta cambios en ambos extremos:
- Si el bilink estructural fue re-aceptado con nuevo hash → `state.0: ALTERED`
- Si el contenido de la tarea cambió → `state.1: ALTERED`

## Ciclo de vida

### Creación

```bash
worklist new task "implementar vote en Persona" capture specs/voting.yaml:104:1
```

Crea:
- `.bilink/<uuid-task-bilink>.bilink` con `link.0: .bilink/<uuid-struct>.bilink` y `link.1: task <id>`
- `worklist/<id>.task`

### Completar

```bash
worklist done 3a
```

Marca la tarea `done`. El bilink de tarea permanece como registro histórico de que la tarea 3a estaba asociada al bilink estructural.

## Tareas con subtareas en subcapas

Una tarea puede vivir en una capa y sus subtareas en subcapas. Cada nivel tiene su propio bilink de tarea apuntando al bilink estructural relevante en esa capa:

```
# spec layer — tarea padre
link.0: .bilink/<uuid-spec>.bilink
link.1: task 3a

# impl layer — subtarea
link.0: .bilink/<uuid-impl>.bilink
link.1: task 3b    ← subtarea en worklist
```

## Invariantes

1. El ID en `task <id>` es un ID base-36 que corresponde a un ítem en `<project-root>/.stratum/worklist/<id>.task`.
2. El bilink de tarea vive en la capa donde se debe ejecutar la tarea.
3. No existe archivo `source_bilink` en el ítem ni archivo `<uuid>.tasks` separado — la asociación vive en el bilink de tarea.
