# Especificación: Modelo de capas

## Dos tipos de relación entre carpetas

Stratum distingue dos tipos de relación entre carpetas de un proyecto:

| Tipo | Carpeta | Relación |
|---|---|---|
| **Sub-proyecto** | carpeta plana | proyecto hermano con sus propias specs |
| **Capa interna** | `.stratum/<nombre>/` | nivel inferior de implementación del mismo proyecto |

La distinción es fundamental: los sub-proyectos son entidades independientes que comparten un contexto de más alto nivel. Las capas internas son partes del mismo proyecto en distintos niveles de abstracción.

## Sub-proyectos: carpetas planas

Un sub-proyecto es simplemente una subcarpeta con sus propias especificaciones. No requiere ninguna convención de nombre especial.

```
accreta/
  stratum/        ← sub-proyecto con sus specs
  bilinker/       ← sub-proyecto con sus specs
  README.md
  general/
  specific/
```

`stratum/` y `bilinker/` son proyectos independientes dentro del contexto de `accreta/`. Cada uno tiene su propio README, sus propias specs, su propio git.

No existe una carpeta `.stratum/` en `accreta/` para agruparlos — eso mezclaría dos conceptos distintos.

## Capas internas: `.stratum/`

La carpeta `.stratum/` aparece dentro de un proyecto cuando ese proyecto necesita separar sus propias capas de abstracción: decisiones técnicas, implementación, tests, etc.

```
bilinker/                           ← specs funcionales (capa más alta)
  general/
  specific/
  .stratum/
    impl/                           ← implementación Rust
      crates/
      docs/adr/
      tests/
```

La carpeta `.stratum/` en `bilinker/` no tiene nada que ver con los sub-proyectos de `accreta/`. Es exclusiva del proyecto `bilinker/` y contiene sus capas de implementación.

## Regla de uso de `.stratum/`

`.stratum/` aparece dentro de una carpeta de specs **si y solo si** esa carpeta necesita sus propias capas inferiores de implementación.

**Correcto:**
```
bilinker/
  .stratum/           ← bilinker tiene capas propias
    impl/
```

**Incorrecto:**
```
accreta/
  .stratum/           ← INCORRECTO: no agrupa sub-proyectos
    stratum/
    bilinker/
```

## Ejemplo completo

```
accreta/                             (git — specs Accreta)
  general/
  specific/
  stratum/                          (git — specs Stratum)
    general/
    specific/
    .stratum/                       ← capas internas de Stratum (si las tiene)
      impl/
  bilinker/                         (git — specs Bilinker)
    general/
    specific/
    .stratum/                       ← capas internas de Bilinker
      tech-decisions/               (git — ADRs)
        adr/
        .stratum/
          impl/                     (git — código Rust)
            crates/
            tests/
```

## Profundidad de anidamiento

No hay límite de profundidad. Una capa interna puede tener a su vez sus propias capas internas con `.stratum/`. La regla es la misma en cada nivel: `.stratum/` solo para capas propias, carpetas planas para sub-proyectos hermanos.

## Git por repositorio

Cada proyecto (carpeta de specs) y cada capa interna es un repositorio git independiente. Esto permite:

- Historial separado por nivel de abstracción.
- Que un proyecto dependa de otro sin acoplar su versionado.
- Que las capas internas (impl, tests) evolucionen en su propio ritmo.

Los vínculos entre layers de distintos repositorios se gestionan con **bilinker**: archivos `.bilink/<uuid>.bilink` que mantienen referencias estructurales verificables entre fragmentos de código o documentación en capas distintas.
