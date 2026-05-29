# Integración con Stratum

Los proyectos de Accreta son proyectos Stratum. La convención de capas y sub-proyectos es la misma — Accreta agrega gobernanza sin cambiar la estructura.

## Worklist como capa interna

`accreta/.stratum/worklist/` sigue exactamente la convención `.stratum/` de Stratum:
es una capa interna del proyecto principal Accreta, no un sub-proyecto hermano.

## Navegación entre capas

El CLI `stratum` funciona dentro de cualquier proyecto Accreta para navegar entre
spec, tech-decisions e impl:

```bash
cd $(stratum '>tech-decisions>impl')
```
