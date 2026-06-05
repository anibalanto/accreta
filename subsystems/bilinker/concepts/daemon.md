# Concepto: bilinker daemon

## Propósito

El daemon de bilinker es un proceso de fondo que gestiona conexiones vivas a language servers (LSP) y responde queries de call graph para cualquier lenguaje soportado. Elimina la necesidad del índice SCIP como fuente de verdad del subgrafo — el daemon es el índice vivo.

## Responsabilidades

1. **Gestión de language servers**: arranca, mantiene vivos y detiene los LSP servers de cada lenguaje detectado en el workspace.
2. **Call graph queries**: responde `callees(file, symbol)` usando `callHierarchy/outgoingCalls` del LSP correspondiente.
3. **IPC con el CLI**: expone un socket Unix local para que los comandos bilinker lo consulten.

## Arquitectura

```
bilinker CLI
     │  Unix socket  (~/.bilinker/daemon.sock)
     ▼
bilinker daemon
     ├── LanguageServerManager
     │     ├── rust-analyzer  (stdio) ← lenguajes .rs
     │     ├── typescript-language-server (stdio) ← .ts .tsx .js
     │     ├── jedi-language-server (stdio) ← .py
     │     └── eclipse.jdt.ls (stdio) ← .java
     └── RequestRouter
           └── despacha al LSP correcto según extensión del archivo
```

El daemon corre como proceso hijo persistente, desacoplado del ciclo de vida de cualquier comando CLI.

## Ciclo de vida de un language server

```
Primera query para un lenguaje
  → detectar LSP instalado para ese lenguaje
  → spawn del proceso LSP vía stdio
  → handshake LSP (initialize / initialized)
  → listo para queries

Queries siguientes
  → reusar conexión existente (sin overhead de arranque)

Idle timeout (configurable, default: 10 min sin queries)
  → shutdown del LSP
  → el proceso se cierra

bilinker daemon stop
  → shutdown de todos los LSPs activos
  → daemon termina
```

## Detección de language server instalado

| Extensión | Lenguaje | Ejecutable buscado |
|-----------|----------|--------------------|
| `.rs` | Rust | `rust-analyzer` |
| `.ts` `.tsx` `.js` `.jsx` | TypeScript/JS | `typescript-language-server` |
| `.py` | Python | `jedi-language-server`, `pylsp` |
| `.java` | Java | `jdtls`, `eclipse.jdt.ls` |

Si el ejecutable no está en PATH, el daemon retorna error claro: `LSP for rust not found: install rust-analyzer`.

## Implementación

Basado en [`async-lsp`](https://github.com/oxalica/async-lsp):
- Spawn del proceso LSP via `async_process::Command` con stdio pipeado
- Comunicación JSON-RPC async sobre los pipes
- Un `LspClient` por lenguaje, compartido entre todas las queries de ese lenguaje
- El daemon es un binario separado (`bilinker-daemon`) incluido en el mismo workspace Cargo

## Persistencia del socket

```
~/.bilinker/
  daemon.sock    ← Unix socket (creado al start, eliminado al stop)
  daemon.pid     ← PID del proceso daemon
  daemon.log     ← log del daemon
```

Si `daemon.sock` existe pero el proceso no responde → socket stale, arrancar nuevo daemon automáticamente.
