---
title: Índice git propio y refspecs de `refs/bilink/*`
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T18:21:24Z
parent: c
---

# Índice git propio y refspecs de `refs/bilink/*`

Spec primero: `commands/init.md` y `commands/sync.md`, que no existen.

`GIT_INDEX_FILE` propio sobre el mismo árbol de trabajo, contra `refs/bilink/<branch>`; construcción del commit con `read-tree` del absorbido más `update-index` de `.bilink/`; las dos verificaciones previas —disyunción sobre el **árbol** del commit, no sobre su diff, y fidelidad por tree oids—.

## Lo que quedó hecho

`concepts/ref.md` es donde vive lo que todo comando que escribe sobre la ref tiene que cumplir; `commands/init.md` y `commands/sync.md` son los dos comandos. `crates/bilinker/src/bilink_ref.rs` lleva el índice propio, la construcción del commit y las dos verificaciones, y `config.rs` las dos líneas por clon.

**Y arrastró `track`**, que no estaba en esta task: el paso 4 del corte —el commit que crea `●0`— no lo puede escribir `sync`, porque sin nada que absorber `sync` no escribe ningún commit. Es el caso "ningún candidato califica" de `track`. Ver la enmienda en [ADR-0004](../subsystems/bilinker/.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md) § `005` y [`commands/track.md`](../subsystems/bilinker/commands/track.md).

**El freno del recorrido, dos veces.** Ni buscar el commit absorbido ni listar los commits de la ref se detienen solos: el corte tiene un commit del proyecto como padre único, así que `--first-parent` sigue por la historia del proyecto. El freno es la disyunción —los commits de la ref llevan `.bilink/` en su árbol y los del proyecto no— y quedó dicho una sola vez, en `Repo::ref_chain`. Sin él `track` elige mal de forma sistemática: el commit más viejo del proyecto es ancestro de cualquier rama.
