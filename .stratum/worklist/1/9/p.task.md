---
title: Ejecutar el corte de formato (`004`) en los 4 repos
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
---

# Ejecutar el corte de formato (`004`) en los 4 repos

El primero de los dos cortes. Deja vivo el formato nuevo **sin tocar dónde viven los bilinks**: siguen en las ramas, con `git status` y `git diff` funcionando normal.

```
1. Generar (o regenerar) .bilink-migrate-003…/ en todas las capas   [sin git]
2. Reemplazar .bilink/ por esa carpeta en el árbol
3. UN commit en la rama del proyecto con el .bilink/ nuevo
4. Ledger: 002, 003, 004
```

Cuatro veces: raíz `accreta` y los tres `.stratum/impl`. `--recursive` desde la raíz no alcanza para los otros tres — están gitignoreados por sus padres y son repos independientes.

Escribe también `.bilink/version` con la versión de formato, que es lo que impide que un binario viejo lea estos archivos y los vacíe en silencio.

**Es la puerta de la US siguiente.** Hasta que acá no pasen los tests, el `check` sobre los 63+63 y las aceptaciones, no se toca la ref.
