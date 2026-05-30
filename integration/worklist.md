# Integración con worklist

Worklist es la capa de tracking de trabajo de Accreta. La relación es bidireccional.

## De Accreta hacia worklist

Cuando una Iteration es aceptada, puede generar automáticamente Tasks en worklist linkedeadas a los fragmentos de spec que cambiaron:

```
Iteration aceptada → Tasks en .stratum/worklist/ con bilinks a spec actualizada
```

## De worklist hacia Accreta

Un ítem `done` en worklist puede generar una Iteration en Accreta sobre la spec o el ADR correspondiente — cerrando el ciclo de trazabilidad.

## Tasks como WorkItems

En la visión completa, los epics, stories y tasks de worklist son WorkItems de Accreta de tipo `Task`. La persistencia en el filesystem es la capa local; Accreta agrega firma criptográfica, gobernanza y distribución P2P.
