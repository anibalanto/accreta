---
title: Índice git propio y refspecs de `refs/bilink/*`
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
---

# Índice git propio y refspecs de `refs/bilink/*`

Spec primero: `commands/init.md` y `commands/sync.md`, que no existen.

`GIT_INDEX_FILE` propio sobre el mismo árbol de trabajo, contra `refs/bilink/<branch>`; construcción del commit con `read-tree` del absorbido más `update-index` de `.bilink/`; las dos verificaciones previas —disyunción sobre el **árbol** del commit, no sobre su diff, y fidelidad por tree oids—.
