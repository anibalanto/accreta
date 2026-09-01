# Integración con Accreta

Stratum es la estructura sobre la que Accreta opera. Los proyectos de Accreta son proyectos Stratum — la convención de capas y sub-proyectos es la misma.

## Proyectos Accreta como proyectos Stratum

Un proyecto Accreta es una carpeta de specs con sub-proyectos hermanos y capas internas, siguiendo exactamente las reglas de Stratum. Accreta agrega gobernanza (votación, firma criptográfica, distribución P2P) sin cambiar la estructura.

## Worklist como capa de Accreta

`accreta/.stratum/worklist-accreta/` es una capa del proyecto principal Accreta, siguiendo la convención `.stratum/`. Es el lugar donde viven los ítems de worklist del ecosistema completo.

**Tiene repo propio**, así que para stratum es el mismo caso que una capa `impl`: se llega con `*>worklist-accreta`, y si no está clonada la capa falta. Quién la versiona no cambia cómo se navega.
