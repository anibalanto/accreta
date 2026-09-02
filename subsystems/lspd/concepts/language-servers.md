# Los language servers

`lspd` no habla ningún lenguaje. Despacha por extensión a un language server, le habla LSP por stdio, y traduce la respuesta al [protocolo](protocol.md).

## La tabla

| Extensión | Ejecutable buscado |
|---|---|
| `.rs` | `rust-analyzer` |
| `.ts` `.tsx` `.js` `.jsx` | `typescript-language-server` |
| `.py` | `jedi-language-server`, `pylsp` |
| `.java` | `jdtls` |

**Agregar un lenguaje es agregar una entrada.** Es todo el conocimiento de lenguajes que hay acá, y es a propósito: cualquier cosa más grande que una tabla sería modelo propio, y este daemon no tiene modelo propio.

Si el ejecutable no está en PATH, la respuesta es un error explícito y no un vacío: `LSP for rust not found: install rust-analyzer`. Un vacío se leería como *"no hay llamadas"*.

### Y la tabla tiene una columna más: qué pide cada servidor en el handshake

La tabla dice qué ejecutable buscar. **Lo que no dice, y hace falta, es qué necesita cada uno para arrancar de verdad**, porque LSP dejó ese pedazo abierto: `initializationOptions` es un campo libre y cada servidor pone ahí lo suyo.

| Servidor | Qué pide, y qué pasa sin eso |
|---|---|
| `rust-analyzer` | `experimental.serverStatusNotification` en las capabilities. Sin eso no avisa cuándo terminó de indexar |
| `jdtls` | `initializationOptions.workspaceFolders`. **Sin eso no importa el proyecto**: cae a su *proyecto invisible*, se queda sin classpath, y resuelve `[]` con el servidor en `READY` |

**Sigue siendo una tabla, y por eso entra acá.** Es un dato por servidor —una constante, no una decisión—, y agregar un lenguaje sigue siendo agregar una fila. Lo que cambia es que la fila tiene dos casillas en vez de una.

**Lo que no entra es interpretar lo que el servidor conteste**: eso sí sería modelo propio. `lspd` le dice a cada uno lo que ese servidor necesita oír para arrancar, y de ahí en adelante todos se hablan igual.

> **Es la clase de dato que sólo se descubre corriéndolo.** Ninguno de los dos está en la especificación de LSP, y los dos aparecieron con el servidor prendido devolviendo respuestas bien formadas y vacías.

## Se levantan a demanda y se reusan

```
primera query de un lenguaje
  → detectar ejecutable → spawn vía stdio → handshake LSP → indexando → listo

queries siguientes
  → reusar la conexión

shutdown
  → shutdown de todos los language servers → el daemon termina
```

Un daemon recién arrancado no tiene ninguno levantado, y eso es normal: `status` lo dice.

## Terminar el handshake no es estar listo

**Entre el handshake y la primera respuesta útil hay un tramo, y durante ese tramo el servidor contesta vacío.** Medido: `rust-analyzer` sobre un workspace mediano tarda siete minutos en dejar de estar ocupado, y en todo ese rato `definitions` devuelve `[]` con el proceso vivo y el handshake terminado.

Un vacío ahí no significa lo que significa después. Es la misma regla que esta página ya aplica al ejecutable que falta —*"la respuesta es un error explícito y no un vacío; un vacío se leería como no hay llamadas"*— y el mismo tramo que [lattice](../../lattice/concepts/provider.md) llama `Degraded`. Lo que cambia es de qué lado se resuelve: **lattice lo infiere de que acaba de arrancar el daemon, y eso sólo sirve para quien lo arrancó.** Quien encuentra un daemon ya prendido no tiene de dónde inferirlo, así que la señal la tiene que dar `lspd`.

### Y quien la tiene es el servidor

No se cronometra ni se adivina: los dos servidores que importan lo dicen, cada uno con su extensión, y son exactamente las notificaciones que [`2o`](../../../.stratum/worklist-accreta/2o.task.md) dejó ignoradas para no morirse con ellas.

| Servidor | Notificación | Dice que está listo cuando |
|---|---|---|
| `rust-analyzer` | `experimental/serverStatus` | `quiescent: true` |
| `jdtls` | `language/status` | `type: ServiceReady` |

La de `rust-analyzer` **hay que pedirla**: sin `experimental.serverStatusNotification` en las capabilities del `initialize`, no la manda. La de `jdtls` viene sola.

### Un servidor que no informa su estado no se puede esperar

`typescript-language-server` y los de Python no mandan ninguna de las dos, y `lspd` no puede inventar la señal — cronometrar el arranque sería adivinar, y adivinar mal en la dirección cara.

Así que la readiness tiene **tres** valores y no dos, y el tercero es el honesto:

| `state` | Qué dice |
|---|---|
| `INDEXING` | el servidor dijo que todavía no está listo |
| `READY` | el servidor dijo que sí |
| `RUNNING` | está arriba, y **este servidor no informa readiness** |

`RUNNING` no es un estado degradado ni un error: es lo que hoy vale para todos, y seguir llamándolo así deja dicho que para ese lenguaje la distinción no se puede dar. **Es la información que el consumidor necesita para saber cuánto vale un vacío**, y esconderla atrás de un `READY` optimista sería volver al problema con otro nombre.

**Que se reusen es la razón de que exista un daemon.** Un `rust-analyzer` tarda decenas de segundos en indexar un proyecto mediano; arrancarlo por consulta haría que preguntar por el call graph cueste más que leer el código.

> Idle timeout por language server: no implementado.

## Lo que deja en el disco

```
~/.lspd/
  daemon.sock    ← el socket, en Unix: creado al arrancar, borrado al terminar
  daemon.pid     ← el pid del proceso
```

En Windows el endpoint es un named pipe y no un archivo, así que sólo queda el `.pid`. Ver [transporte](transport.md).

**Nada de esto es del proyecto.** El daemon no escribe en el árbol que indexa: arrancarlo desde un comando de sólo lectura es un efecto, y vale la pena que el efecto esté acotado a un directorio del usuario.

Un socket que existe pero no contesta está stale, y quien lo encuentre así arranca uno nuevo.

> `daemon.log`: no implementado — stderr va a `/dev/null`. Para ver logs, lanzar el binario a mano redirigiendo stderr.
