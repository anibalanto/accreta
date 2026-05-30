# Integración con Accreta

worklist es el sistema de tracking de trabajo de Accreta. La relación entre ambos es bidireccional.

## De Accreta hacia worklist

Cuando una Iteration es aceptada en Accreta, puede generar automáticamente tasks en worklist que representen el trabajo de implementación:

```
Iteration aceptada → genera tasks en worklist con bilinks a la spec actualizada
```

## De worklist hacia Accreta

Un item `done` en worklist puede generar una Iteration en Accreta:

```
task done → propone Iteration sobre la spec o el ADR correspondiente
```

## Items como WorkItems de Accreta

En la visión completa del ecosistema, los epics, stories y tasks de worklist son WorkItems de Accreta con tipo `Task`. La persistencia en el filesystem es la capa local; Accreta agrega gobernanza, firma criptográfica y distribución P2P.

## Agentes y worklist

Los agentes de Accreta pueden:
- Crear tasks automáticamente a partir de análisis de impacto
- Proponer resoluciones para tasks existentes
- Marcar tasks como `done` cuando la implementación es verificada

Los humanos retienen la aprobación final — el agente propone, el humano acepta.
