---
title: El saldo de cobertura que la task `o` no cerró
status: open
created_at: 2026-08-29T23:53:53Z
updated_at: 2026-08-30T00:37:31Z
---

# El saldo de cobertura que la task `o` no cerró

La task `o` cruzó bilinker y la parte de lattice que [ADR-0003](../../../subsystems/bilinker/.stratum/impl/docs/adr/0003-formato-captures-y-aceptacion.md) toca. Queda esto, que está fuera de lo que los ADRs mueven y por eso no bloqueaba el sprint 1.

## El core de lattice está mudo

De los archivos `.rs` de `lattice>impl`, tienen cero captures: `graph.rs` salvo `covering`, `daemon_client.rs`, `lib.rs` y **el crate `lattice-daemon` entero** (`ipc.rs`, `language.rs`, `lsp_client.rs`, `lsp_manager.rs`, `main.rs`, `types.rs`). `model.rs` y `provider.rs` ya no: la task `o` les puso los suyos.

Del lado de las specs, siguen en cero: `lattice/architecture.md`, `lattice/overview.md`, `lattice/concepts/edge.md`, `lattice/concepts/provider.md`, `lattice/commands/daemon.md`, `lattice/integration/impact.md`.

`concepts/node.md` e `integration/bilinker.md` ya no: la task `o` les puso cuatro y dos bilinks porque ADR-0003 los reescribe.

## `bilinker-lsp` no tiene spec

`crates/bilinker-lsp/` implementa un language server —`Backend`, `impl LanguageServer`, hover y navegación— que funciona y que consume la extensión de VS Code. No hay ningún archivo de spec que lo describa: `commands/watch.md` es sobre `bilinker watch`, otra cosa.

No es un hueco de cobertura sino el hueco de al lado: **implementación sin especificación**. El criterio de `o` —"si una especificación tiene implementación, está bilinkeada"— no lo alcanza, porque no hay spec de la cual salir.

## `integration/worklist.md` describe un archivo que no existe

[`bilinker/integration/worklist.md`](../../../subsystems/bilinker/integration/worklist.md) § "Trabajo pendiente sobre un endpoint" muestra un `accreta/.stratum/worklist/<uuid>.tasks`. [`worklist/concepts/bilink-tasks.md`](../../../subsystems/worklist/concepts/bilink-tasks.md) inv. 3 dice explícitamente que ese archivo no existe. Gana `bilink-tasks.md`.

## Exclusiones deliberadas

No llevan bilink, y no es un pendiente:

| | Por qué |
|---|---|
| `bilinker/proposals/bilink-endpoint.md` | especificado y no implementado, por definición |
| `bilinker/overview.md`, `bilinker/integration/{acreta,impact}.md` | narrativa y remisiones, sin código que las implemente |
| `crates/bilinker/src/bin/debug_ast.rs` | herramienta de desarrollo, sin spec ni usuario |

## Cuándo está hecha

Ninguna spec de lattice con implementación queda muda, el LSP tiene spec o una decisión escrita de no tenerla, y `integration/worklist.md` no describe archivos inexistentes.

## Prioridad

No es urgente. Nada de lo que enumera está en el camino de los cuatro ADRs: es cobertura de superficie que hoy no bloquea ningún cambio.
