# Integración con bilinker

bilinker es la infraestructura de linking de worklist. Cada ítem nace con un bilink al fragmento que lo origina — puede estar en cualquier capa o repo.

## Creación del bilink en `worklist new`

Cuando `worklist new` recibe un selector, ejecuta `bilinker capture` sobre el fragmento referenciado (resuelto desde cwd) y crea un bilink:

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
accreta/.stratum/worklist/<id>.task          ← source_bilink: <uuid-bilink>
accreta/.bilink/<uuid-bilink>.bilink         ← bilink: ítem ↔ fragmento
accreta/.stratum/worklist/<uuid-bilink>.tasks ← "<id>\n" (creado o actualizado)
```

## Archivo `.tasks` por bilink

Un bilink puede estar referenciado por múltiples ítems. El servidor worklist mantiene `<bilink-uuid>.tasks` en la raíz del worklist — un ID por línea:

```
# 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.tasks
3
1a
2f
```

Verificar si un bilink tiene trabajo pendiente = comprobar existencia del archivo (O(1)). Obtener las tasks = leer el archivo (O(1)).

La dirección inversa también es O(1): cada `.task` tiene `source_bilink` en su frontmatter.

## El bilink como ancla de trazabilidad

El `source_bilink` conecta el trabajo pendiente con su origen exacto. Cuando el fragmento cambia:

1. `bilinker check` detecta `ALTERED` o `CHAIN_DIRTY` en el bilink
2. El ítem en worklist sigue apuntando al fragmento, pero el fragmento cambió
3. El desarrollador evalúa si el ítem sigue siendo válido o debe actualizarse

## Drift en ítems de worklist

El drift no modifica el ítem automáticamente — es una señal. El desarrollador decide si actualizar el ítem, completarlo, o eliminarlo.

## Al completar el trabajo

Cuando el trabajo está listo, el ítem se marca `done`. Si el trabajo consistió en crear una conexión entre dos fragmentos, se establece el bilink real con `bilinker chain new`:

```
fragmento origen (cualquier capa)
    ↕ bilink (source_bilink, creado por worklist new)
accreta/.stratum/worklist/<id>.task  →  done
    ↕ bilink (creado por bilinker chain new al terminar)
fragmento destino (cualquier capa)
```
