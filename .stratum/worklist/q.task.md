---
title: Reescribir `concepts/migration.md`: proceso vs migración
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
parent: 9
---

# Reescribir `concepts/migration.md`: proceso vs migración

Va primera de esta US: define la maquinaria que las demás usan.

**Distinguir una migración de el proceso de migración.** Una migración es lo que `migration.md` ya define: transformación sintáctica, idempotente, con `--dry-run`. El proceso es la transición completa, y tiene pasos que no son migraciones — los cortes. Entran en la misma secuencia numerada y el mismo ledger; los corre otro comando, así que las inv. 5 (no consulta git) y 6 (el runner no commitea) se salvan enteras.

**El conjunto de migraciones es de sólo-agregar.** Nunca se borra una, ni siquiera cuando "ya nadie está en ese formato": es lo único que permite que alguien en X-2 llegue a X corriendo la cadena entera. Hoy la spec define ids ordenados, ledger e idempotencia, pero no dice esto — y sin decirlo, sacar una migración vieja parece una limpieza razonable.

Corolario a dejar escrito: la historia de formatos **sí se acumula**, en el build y no en el camino de lectura. Cada migración depende de los dos formatos que puentea, así que los parsers viejos quedan en el repo para siempre. Es el costo del esquema y conviene que esté a la vista.

También el recuento de la línea 26 —"cuatro capas en tres repos"; son seis en cuatro—.
