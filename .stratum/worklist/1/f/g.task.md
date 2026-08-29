---
title: Especificar e implementar `init`, `sync`, `track` y `adopt`
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
---

# Especificar e implementar `init`, `sync`, `track` y `adopt`

Los cuatro comandos no tienen spec: `commands/init.md`, `sync.md`, `track.md` y `adopt.md` se crean acá, y recién después se implementan. Más `log` y `diff` sobre la ref propia, `.bilink/version` escrita por `init`, y `.bilink/head` con su guarda, y la materialización automática al cambiar de rama.

**Cambio de una línea en `task.rs`:** `format!("{task_id}.task")` pasa a `format!("{task_id}.task.md")`. Es lo único del código que fija la extensión de un ítem de worklist. **Es la única excepción al método en toda la épica**, y conviene decir por qué: nace de `worklist/concepts/item.md`, una spec de otro subsistema que ningún bilink de la raíz cubre. Si algún día worklist tiene sus propias cadenas, deja de ser excepción.
