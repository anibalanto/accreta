# Integración con impact

Impact alimenta el ciclo de gobernanza de Accreta: cuando detecta drift entre
capas, genera los artefactos que disparan Iterations y discusiones.

## Flujo

```
bilinker check  →  ALTERED en fragmento linkedeado
impact scan     →  Impact Report con commits intercedidos
impact thread   →  hilo de discusión
                →  agentes de Accreta analizan el impacto
                →  Iteration propuesta sobre la spec afectada
                →  consenso → Iteration aceptada
```

## Impact Reports como evidencia

Los Impact Reports generados por impact son la evidencia técnica que alimenta
las Opinions de los agentes en Accreta. El `context_refs` del agente apunta
exactamente al commit analizado.
