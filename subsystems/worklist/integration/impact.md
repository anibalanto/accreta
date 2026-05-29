# Integración con impact

impact y worklist se complementan: impact detecta que algo cambió y abre
un hilo de discusión; worklist registra el trabajo concreto que hay que hacer
como consecuencia de ese cambio.

## Flujo

```
bilinker check → ALTERED en bilink-format.md
impact scan    → genera Impact Report con commits intercedidos
impact thread  → abre hilo de discusión
                 → el hilo concluye: "hay que actualizar la implementación"

# Desde spec, creando tarea hacia abajo
worklist new task "implementar accepted.N en BiLinkFile" bilink-format.md:104:1

# O desde impl, creando tarea hacia arriba
worklist new task "actualizar spec en base a cambio en accept()" lib.rs:88:1
```

El ítem creado queda linkedeado exactamente al fragmento que cambió — el
mismo que impact analizó.

## Trazabilidad completa

```
git commit (cambió fragmento en cualquier capa)
    ↓ detectado por
impact report
    ↓ origina
worklist task  →  source_bilink → fragmento
    ↓ completado con
bilinker chain new (si corresponde crear un bilink entre capas)
    ↓ cierra
worklist done <task>
```

Cada paso es trazable: desde el commit que introdujo el cambio hasta el
ítem de worklist que registra el trabajo y el bilink que certifica la alineación.
