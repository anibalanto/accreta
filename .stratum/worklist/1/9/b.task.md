---
title: Escribir `bilinker-003-immutable-captures`
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
---

# Escribir `bilinker-003-immutable-captures`

Renombrar cada `.capture` a `H(file, query, offset)` y repuntar `link.N` y `link_accepted.N`. Sin fan-out: el id nunca dependió del hash. Dos captures con la misma ubicación colapsan en uno.

Va después de `a`: mientras `range`, `state` y `resolved_at` sigan dentro del `.capture`, no se le puede calcular un id estable.
