---
title: Especificar e implementar `init`, `sync`, `track` y `adopt`
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T01:59:27Z
parent: f
---

# Especificar e implementar `init`, `sync`, `track` y `adopt`

Los cuatro comandos no tienen spec: `commands/init.md`, `sync.md`, `track.md` y `adopt.md` se crean acá, y recién después se implementan. Más `log` y `diff` sobre la ref propia, `.bilink/version` escrita por `init`, y `.bilink/head` con su guarda, y la materialización automática al cambiar de rama.

> El cambio de una línea en `task.rs` que esta task llevaba —`format!("{task_id}.task")` a `.task.md`— ya no aplica: la task `u` lo resolvió de otra forma, buscando el ítem por prefijo en vez de fijar la extensión, y el archivo se llama `issue.rs` desde la task `z`.
