# El protocolo

JSON-RPC 2.0 con **framing newline-delimited**: cada mensaje es un objeto JSON en una línea, terminado en `\n`. Va sobre el [transporte](transport.md) que corresponda al sistema operativo, y no sabe cuál es.

```json
{"jsonrpc":"2.0","id":1,"method":"callers",
 "params":{"file":"/abs/path/src/check.rs","line":261,"col":4}}
```

## Los métodos

| Método | Params | Resultado |
|---|---|---|
| `callees` | `{file, line, col}` | `[CalleeInfo]` — llamadas salientes |
| `callers` | `{file, line, col}` | `[CalleeInfo]` — llamadas entrantes |
| `symbol_at` | `{file, line, col}` | `SymbolInfo` o `null` |
| `definitions` | `{file, line, col}` | `[DefinitionInfo]` — dónde está declarado lo que se menciona ahí |
| `status` | — | `[LspStatus]` |
| `ping` | — | `"pong"` |
| `shutdown` | — | `null`, y cierra la conexión |

`CalleeInfo` es `{symbol, name, file, line, col}`; `callees` y `callers` comparten esquema. `SymbolInfo` es `{symbol, name, kind}`. `LspStatus` es `{name, state, queries}`. `DefinitionInfo` es `{name, file, line, col, end_line, end_col}`.

**`definitions` es la única pregunta que no es del call graph**, y la pide bilinker para el [cierre de firma](../../bilinker/concepts/accept.md#el-cierre-de-firma): dónde están declarados los tipos que una firma menciona.

Es **un salto y ninguna interpretación**. `lspd` devuelve ubicaciones; no lee el contenido, no lo hashea y no sabe para qué se lo piden. Las tres formas que LSP permite en la respuesta —una, varias, o links— se aplanan a ubicaciones antes de salir: cuál usó el servidor de atrás no es un dato de quien pregunta.

Y la dedup la hace el language server: las tres menciones de `Persona` en `Persona hijoMenor(Persona padre, Persona madre)` resuelven a la misma declaración, así que el vecindario tiene **un** `Persona` y no tres.

**`line` y `col` son 0-based**, como en LSP y a diferencia del resto del ecosistema. La conversión es de quien pregunta: traducirla acá sería ponerle al daemon una convención que no es suya.

## `ping` no es diagnóstico

Es cómo se decide si hay daemon. Un consumidor que quiera arrancarlo si no está pregunta `ping` y mira si contesta; no hay archivo de pid que consultar ni proceso que enumerar.

**Y contesta antes de que los language servers estén listos.** Eso es deliberado y es la distinción que [lattice](../../lattice/concepts/provider.md) llama `Degraded`: el daemon está, pero el servidor de atrás sigue indexando, así que `callers` puede devolver vacío por *"todavía no sé"* y no por *"no hay"*. Confundir las dos es la equivocación más cara que puede cometer un consumidor, y por eso el protocolo no las junta en un booleano.

## Un error es del método, no del transporte

Una respuesta con `error` es una respuesta: el daemon está vivo y esa pregunta no se pudo contestar. Que el socket no exista o que se corte es otra cosa, y la distingue el cliente.

| | |
|---|---|
| `-32700` | el JSON no parsea |
| `-32601` | no existe ese método |
| `-32602` | los params no tienen la forma que el método espera |
| `-32000` | el language server falló o no hay soporte para ese lenguaje |

## Qué no está en el protocolo

**Nada de streaming ni de suscripciones.** Una pregunta, una respuesta, y el consumidor pregunta cuando necesita. Un grafo de llamadas se expande nodo a nodo —no se enumera— así que no hay para qué empujar.

**Ni el workspace.** Se lo fija al arrancar y no viaja en cada pregunta: un daemon indexa un workspace, y preguntarle por otro es arrancar otro.
