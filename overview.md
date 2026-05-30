# Accreta

Accreta es una plataforma distribuida y open-source para la especificación colaborativa de software. Modela la co-creación de specs vivas como un proceso de acreción: las contribuciones de humanos y agentes se acumulan capa por capa, con historial completo y gobernanza por consenso.

## Problema que resuelve

Los proyectos de software generan conocimiento en múltiples niveles — specs, decisiones técnicas, implementación, tests — que viven en herramientas inconexas y divergen sin que nadie lo detecte. Las decisiones se toman sin trazabilidad. Los agentes de IA participan sin identidad verificable ni responsabilidad atribuible.

Accreta conecta esos niveles, hace visible el drift entre capas y provee la estructura de gobernanza para que humanos y agentes evolucionen las specs con consenso explícito.

## Posición en el ecosistema

```
stratum    ← estructura de capas y navegación
bilinker   ← referencias verificables entre fragmentos de capas
impact     ← análisis de cambios entre capas
worklist   ← trabajo concreto pendiente
accreta     ← gobernanza: identidad, consenso, distribución P2P
```

## Principios

- **Acreción sobre reemplazo**: las specs evolucionan por Iterations aceptadas —
  nada se borra, todo se acumula con historial preservado
- **Humanos gobiernan, agentes asesoran**: los agentes son participantes de primera
  clase con identidad verificable, pero el poder de voto es exclusivamente humano
- **Trazabilidad total**: cada acción de cada actor — humano o agente — está firmada
  criptográficamente y es permanentemente atribuible
- **Distribución primero**: sin servidor central; red P2P, workspaces federados
- **Git como backbone**: todo el contenido de specs vive en Git; los eventos de
  gobernanza se anclan on-chain
