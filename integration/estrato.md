# Integración con Stratum

Los proyectos de Accreta son proyectos Stratum. La convención de capas y sub-proyectos es la misma — Accreta agrega gobernanza sin cambiar la estructura.

## El worklist es una capa, y tiene repo propio

`accreta/.stratum/worklist-accreta/` sigue exactamente la convención `.stratum/` de Stratum. **No es una capa interna**: es un repo aparte clonado adentro de `.stratum/`, igual que las capas `impl` de cada subsistema, y accreta la ignora con la misma regla — un `.gitignore` con `/*/`.

Que sea capa y no sub-proyecto hermano es lo que importa para stratum: se llega con `*>worklist-accreta`, y quién la versiona es otra pregunta.

**Y el nombre lleva el proyecto** porque el worklist es de accreta y la herramienta no. Quien la busque la encuentra por el prefijo `.stratum/worklist*`, no por un nombre fijo.

## Navegación entre capas

El CLI `stratum` funciona dentro de cualquier proyecto Accreta para navegar entre spec, tech-decisions e impl:

```bash
cd $(stratum '>tech-decisions>impl')
```
