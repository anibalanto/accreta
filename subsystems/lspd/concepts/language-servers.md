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

## Se levantan a demanda y se reusan

```
primera query de un lenguaje
  → detectar ejecutable → spawn vía stdio → handshake LSP → listo

queries siguientes
  → reusar la conexión

shutdown
  → shutdown de todos los language servers → el daemon termina
```

Un daemon recién arrancado no tiene ninguno levantado, y eso es normal: `status` lo dice.

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
