# Especificación: comando `lattice daemon`

## Propósito

Gestiona el ciclo de vida del daemon que mantiene conexiones vivas a language servers y responde queries de call graph. Es la infraestructura del proveedor `lsp`.

Migrado desde `bilinker daemon`: el crate no tenía nada específico de bilinks — despacha por extensión de archivo, habla `callHierarchy` y devuelve posiciones. Cambia el nombre del binario y del socket.

## Subcomandos

### `lattice daemon start`

```
lattice daemon start [--workspace <path>]
```

| Argumento | Default | Descripción |
|---|---|---|
| `--workspace` | cwd | Raíz del workspace. Los language servers se inicializan con este directorio. |

Arranca el daemon en background. Si ya hay uno corriendo para este workspace, no hace nada y retorna 1.

```
daemon started  pid=12345  socket=~/.lattice/daemon.sock
```

### `lattice daemon stop`

Envía shutdown a todos los language servers activos y termina el proceso.

### `lattice daemon status`

```
daemon  pid=12345  socket=~/.lattice/daemon.sock

language servers:
  rust-analyzer               RUNNING  queries=147
  typescript-language-server  RUNNING  queries=32
```

## Protocolo IPC

Socket Unix en `~/.lattice/daemon.sock`. JSON-RPC 2.0 con framing newline-delimited: cada mensaje es un objeto JSON en una línea, terminado en `\n`.

| Método | Params | Resultado |
|---|---|---|
| `callees` | `{file, line, col}` | `[CalleeInfo]` — llamadas salientes |
| `callers` | `{file, line, col}` | `[CalleeInfo]` — llamadas entrantes |
| `symbol_at` | `{file, line, col}` | `SymbolInfo` o `null` |
| `status` | — | `[LspStatus]` |
| `ping` | — | `"pong"` |
| `shutdown` | — | `null`, cierra la conexión |

```json
{"jsonrpc":"2.0","id":1,"method":"callers",
 "params":{"file":"/abs/path/src/check.rs","line":261,"col":4}}
```

`CalleeInfo` es `{symbol, name, file, line, col}`; `callees` y `callers` comparten esquema. `SymbolInfo` es `{symbol, name, kind}`.

`line` y `col` son 0-based, como en LSP.

## `symbol_at` y el anclaje

`symbol_at` es el **fallback** de la resolución de anclaje (ver [concepts/node.md](../concepts/node.md) § "Anclaje"): dado un nodo localizado por rango de bytes, encontrar la posición del identificador que el LSP necesita para `prepareCallHierarchy`.

El camino principal no lo necesita: la query almacenada en el `.bilink` ya captura el identificador en su predicado de nombre. `symbol_at` cubre los endpoints sin ese predicado — nodos sin campo `name`, o archivos capturados enteros sin query.

En bilinker el método estaba implementado pero sin ningún uso: su único consumidor documentado era `bilinker scip retrofit`, un comando que se eliminó.

## Language servers

| Extensión | Ejecutable buscado |
|---|---|
| `.rs` | `rust-analyzer` |
| `.ts` `.tsx` `.js` `.jsx` | `typescript-language-server` |
| `.py` | `jedi-language-server`, `pylsp` |
| `.java` | `jdtls` |

Un language server se arranca en la primera query de su lenguaje y se reusa después. Si el ejecutable no está en PATH, el daemon responde con error explícito: `LSP for rust not found: install rust-analyzer`.

Agregar un lenguaje es agregar una entrada a esta tabla.

## Ciclo de vida

```
primera query de un lenguaje
  → detectar ejecutable → spawn vía stdio → handshake LSP → listo

queries siguientes
  → reusar la conexión

lattice daemon stop
  → shutdown de todos los language servers → el daemon termina
```

> Idle timeout por language server no implementado en v1.

## Persistencia

```
~/.lattice/
  daemon.sock    ← socket (creado al start, eliminado al stop)
  daemon.pid     ← PID del proceso
```

Si `daemon.sock` existe pero el proceso no responde, el socket está stale y se arranca un daemon nuevo.

> `daemon.log` no implementado en v1 — stderr va a `/dev/null`. Para logs, lanzar el binario manualmente redirigiendo stderr.

## Auto-start

El proveedor `lsp` intenta conectarse al socket antes de cada consulta. Si no responde, arranca el daemon y espera hasta 5s.

| Resultado | Estado del proveedor |
|---|---|
| El daemon ya estaba corriendo | `Available` |
| Se arrancó recién y respondió | `Degraded` — el language server está indexando |
| No se pudo arrancar | `Unavailable` |

La distinción importa: el daemon responde al ping apenas arranca, pero rust-analyzer y sus pares tardan bastante más en indexar un proyecto. Durante esa ventana `callers` devuelve vacío, y reportar `Available` haría pasar "todavía no sé" por "no hay llamadas".

Ningún comando de lattice **requiere** el daemon: su ausencia siempre degrada, nunca aborta.

Arrancar un proceso desde un comando de solo lectura es un efecto, y vale la pena ser explícito: el daemon no escribe nada del proyecto — su socket y su pid viven en `~/.lattice/`.

## Exit codes

| Código | Condición |
|---|---|
| 0 | Éxito |
| 1 | Daemon ya corriendo (`start`) / no corriendo (`stop`, `status`) |
| 2 | Language server no instalado para un lenguaje requerido |
