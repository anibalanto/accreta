# Especificación: comando `bilinker daemon`

## Propósito

Gestiona el ciclo de vida del daemon de bilinker, que mantiene conexiones vivas a language servers LSP para queries de call graph en tiempo real.

## Subcomandos

### `bilinker daemon start`

```
bilinker daemon start [--workspace <path>]
```

| Argumento | Default | Descripción |
|-----------|---------|-------------|
| `--workspace` | cwd | Raíz del workspace a indexar. Los LSPs se inicializan con este directorio. |

Arranca el daemon en background. Si ya hay un daemon corriendo para este workspace, no hace nada y retorna 0.

**Salida:**
```
daemon started  pid=12345  socket=~/.bilinker/daemon.sock
```

### `bilinker daemon stop`

```
bilinker daemon stop
```

Envía shutdown a todos los LSPs activos y termina el proceso daemon.

### `bilinker daemon status`

```
bilinker daemon status
```

Muestra el estado del daemon y los LSPs activos:

```
daemon  pid=12345  uptime=2h14m  socket=~/.bilinker/daemon.sock

language servers:
  rust-analyzer        RUNNING  queries=147  idle=0:23
  typescript-server    RUNNING  queries=32   idle=1:05
  jedi-language-server STOPPED  (no queries yet)
```

---

## Protocolo IPC (Unix socket)

El daemon expone un socket Unix en `~/.bilinker/daemon.sock`. El protocolo es JSON-RPC 2.0 sobre el socket.

### Request: `callees`

Retorna los callees directos de un símbolo dado, usando `callHierarchy/outgoingCalls` del LSP correspondiente.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "callees",
  "params": {
    "file": "/abs/path/to/crates/bilinker/src/check.rs",
    "line": 261,
    "col": 4
  }
}
```

**Respuesta:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": [
    {
      "symbol": "rust-analyzer cargo bilinker 0.1.0 chain/resolve_layer_link().",
      "name": "resolve_layer_link",
      "file": "crates/bilinker/src/chain.rs",
      "line": 102,
      "col": 0
    },
    {
      "symbol": "rust-analyzer cargo bilinker 0.1.0 bilink/impl#[BiLinkFile]load().",
      "name": "load",
      "file": "crates/bilinker/src/bilink.rs",
      "line": 25,
      "col": 4
    }
  ]
}
```

### Request: `symbol_at`

Retorna el símbolo LSP en una posición dada (para `bilinker scip retrofit`).

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "symbol_at",
  "params": {
    "file": "/abs/path/to/crates/bilinker/src/check.rs",
    "line": 261,
    "col": 4
  }
}
```

**Respuesta:**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "symbol": "rust-analyzer cargo bilinker 0.1.0 check/check_layer().",
    "name": "check_layer",
    "kind": "function"
  }
}
```

### Request: `ping`

Health check. Retorna `{"result": "pong"}`.

---

## Integración con otros comandos

### `bilinker check`

Si el daemon está corriendo, usa `callees` para detectar si el call graph del subgrafo cambió. Si un callee nuevo aparece → crea el `.sciplink` correspondiente con query tree-sitter.

Si el daemon no está corriendo → usa el `index.scip` cacheado si existe (fallback). Si tampoco existe → skip subgraph check con advertencia.

### `bilinker scip retrofit`

Usa `symbol_at` para detectar el símbolo SCIP de cada endpoint estructural y agregar `subgraph.N` al `.bilink`. Mucho más preciso que la heurística actual de `find_callable_at`.

### `bilinker chain new`

Usa `symbol_at` para auto-detectar `subgraph.N` en el momento de creación de la cadena.

---

## Auto-start

`bilinker check`, `bilinker chain new` y `bilinker scip retrofit` intentan conectarse al daemon antes de ejecutar. Si no responde en 500ms:

1. Intentan arrancar el daemon automáticamente (`bilinker daemon start`)
2. Esperan hasta 5s a que esté listo
3. Si falla, continúan sin daemon (modo fallback)

---

## Exit codes

| Código | Condición |
|--------|-----------|
| 0 | Éxito |
| 1 | Daemon ya corriendo (`start`) / no corriendo (`stop`, `status`) |
| 2 | LSP no instalado para un lenguaje requerido |
