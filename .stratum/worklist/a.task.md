---
title: Escribir `bilinker-002-file-partition`
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T07:31:11Z
parent: 9
---

# Escribir `bilinker-002-file-partition`

Spec primero: `commands/migrate.md` registra la migración nueva junto a `bilinker-001-capture-split`.

Renombres a `_accepted`, `commit.N` y `state.N` a `cache/state`, `resolved_at` descartado, `kind`/`name.N` preservados, y `link_accepted.N` sembrado desde `link.N` donde hay `hash.N`. Puramente sintáctica: sin resolver queries ni consultar git (`migration.md` inv. 5).

## Se hizo fuera de orden

La migración se escribió en el sprint 3, porque el corte de formato la necesitaba antes de que su sprint llegara. Está en `crates/bilink-migrate/src/partition.rs` y corrió en los cuatro repos — ver la task [`p`](p.task.md).

Lo que quedó pendiente de esta descripción y se cerró después: `kind` y `name.N` no se preservaban, que fue la task [`4`](4.task.md).
