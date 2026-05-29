# Integración con worklist

Worklist usa bilinker como infraestructura de linking. Cada ítem de worklist
nace con un bilink al fragmento que lo origina.

## Creación

`worklist new` ejecuta `bilinker capture` internamente y crea el bilink:

```bash
worklist new task "implementar bilinker accept" bilink-format.md:104:1
```

Produce `.bilink/<uuid>.bilink` que conecta el ítem de worklist con el
fragmento exacto de la spec.

## Drift

Cuando el fragmento al que apunta un ítem cambia, `bilinker check` detecta
`ALTERED` en el `source_bilink`. Es una señal para que el desarrollador
evalúe si el ítem sigue siendo válido.

## Trabajo pendiente sobre un endpoint

Un bilink puede tener múltiples tasks asociadas. El worklist mantiene un archivo
`<bilink-uuid>.tasks` con un ID de task por línea:

```
# accreta/.stratum/worklist/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.tasks
3
1a
```

Verificar si un bilink tiene trabajo pendiente = comprobar existencia de
`<bilink-uuid>.tasks` (O(1)). El bilink no almacena ninguna referencia al worklist.

## Ver también

[Worklist — integración con bilinker](../../worklist/integration/bilinker.md)
