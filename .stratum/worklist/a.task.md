---
title: Escribir `bilinker-002-file-partition`
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
parent: 9
---

# Escribir `bilinker-002-file-partition`

Spec primero: `commands/migrate.md` registra la migración nueva junto a `bilinker-001-capture-split`.

Renombres a `_accepted`, `commit.N` y `state.N` a `cache/state`, `resolved_at` descartado, `kind`/`name.N` preservados, y `link_accepted.N` sembrado desde `link.N` donde hay `hash.N`. Puramente sintáctica: sin resolver queries ni consultar git (`migration.md` inv. 5).
