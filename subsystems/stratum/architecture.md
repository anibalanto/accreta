# Arquitectura

## Estructura típica de un proyecto Stratum

```
accreta/                             (git — specs Accreta)
  overview.md
  stratum/                          (git — specs Stratum)
    overview.md
    concepts/
    .stratum/
      .impl.toml                    ← config: cómo obtener impl
      impl/                         (git — implementación)
  bilinker/                         (git — specs Bilinker)
    overview.md
    concepts/
    commands/
    .stratum/
      .impl.toml                    ← config: cómo obtener impl
      impl/                         (git — código Rust)
            crates/
```

## Dos tipos de relación entre carpetas

| Tipo | Carpeta | Cuándo |
|------|---------|--------|
| **Sub-proyecto** | carpeta plana | proyecto hermano con sus propias specs |
| **Capa interna** | `.stratum/<nombre>/` | nivel inferior del mismo proyecto |

Los sub-proyectos son entidades independientes. Las capas internas son el mismo proyecto en distintos niveles de abstracción.

## La regla de `.stratum/`

`.stratum/` aparece en una carpeta **si y solo si** esa carpeta tiene sus propias capas inferiores:

```
bilinker/
  .stratum/           ✓ bilinker tiene capas propias

accreta/
  .stratum/           ✗ incorrecto para agrupar sub-proyectos
    bilinker/
    stratum/
```

## Profundidad

No hay límite de profundidad. La regla es la misma en cada nivel.

## Navegación

El CLI `stratum` resuelve paths relativos entre capas:

```bash
cd $(stratum '>impl')   # bajar a impl
cd $(stratum '<')      # subir a specs
stratum '>?'                           # listar capas disponibles
```

## Vínculos entre capas

Los vínculos verificables entre fragmentos de distintas capas se gestionan con bilinker: archivos `.bilink/<uuid>.bilink` que mantienen referencias estructurales y detectan drift cuando el contenido cambia.
