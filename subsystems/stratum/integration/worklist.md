# Integración con worklist

Worklist vive como capa interna dentro del proyecto principal Accreta, siguiendo
la convención `.stratum/` de Stratum:

```
accreta/
  .stratum/
    worklist/      ← capa interna de Accreta
      <uuid>.epic
      <uuid>.task
```

La estructura jerárquica de worklist (epic → user-story → task) usa carpetas
planas dentro de `worklist/`, no sub-capas `.stratum/` — es jerarquía de trabajo,
no de abstracción.
