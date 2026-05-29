# Stratum

Stratum es un modelo de organización de conocimiento en capas para proyectos de
software. Define cómo estructurar especificaciones, decisiones técnicas e
implementación de forma que cada nivel de abstracción tenga su lugar y las
referencias entre capas sean verificables.

## Problema que resuelve

El conocimiento de un proyecto existe en múltiples niveles — specs funcionales,
ADRs, implementación, tests — que típicamente viven en herramientas inconexas.
Cuando la implementación diverge de la especificación, nadie lo sabe hasta que
el daño ya está hecho.

Stratum define una convención de estructura que hace ese conocimiento navegable
y composable, sin imponer herramientas ni formatos.

## Posición en el ecosistema

```
stratum      ← estructura y navegación entre capas
bilinker     ← referencias verificables entre fragmentos de capas
impact       ← análisis de cambios entre capas
worklist     ← trabajo pendiente entre capas
accreta       ← gobernanza del sistema completo
```

## Principios de diseño

- **Specs primero**: las especificaciones funcionales son el artefacto de mayor
  jerarquía. El código, los ADRs y los tests existen para realizarlas.
- **Sub-proyectos como carpetas planas**: un proyecto relacionado vive como
  subcarpeta directa. No hay convención especial de nombre.
- **`.stratum/` solo para capas internas**: la carpeta `.stratum/` aparece dentro
  de un proyecto únicamente cuando ese proyecto tiene sus propias capas inferiores.
- **Git por repositorio**: cada proyecto y cada capa interna es un repo git
  independiente. Historial separado, releases independientes, sin acoplamiento.
- **Navegación por paths Stratum**: los vínculos entre capas usan una sintaxis
  concisa (`>tech-decisions`, `<<`) en lugar de rutas relativas largas.
