# Especificación: comando `lspd`

## Propósito

El ciclo de vida del daemon. Es lo único que `lspd` expone como comando: las preguntas van por el [protocolo](../concepts/protocol.md), no por la línea de comandos.

## `lspd start`

```
lspd start [--workspace <path>]
```

| Argumento | Default | Descripción |
|---|---|---|
| `--workspace` | cwd | Raíz del workspace. Los language servers se inicializan con este directorio. |

Arranca el daemon en background. Si ya hay uno corriendo, **no hace nada y retorna 1** — arrancar dos sobre el mismo socket dejaría al segundo sin poder escuchar, y decirlo es más útil que fallar al bindear.

```
lspd started  pid=12345  endpoint=~/.lspd/daemon.sock
```

`endpoint` se imprime porque es lo que un consumidor va a mirar cuando algo no conecta, y cambia por sistema operativo. No se pasa: se deriva. Ver [transporte](../concepts/transport.md).

## `lspd stop`

Manda `shutdown` a todos los language servers activos y termina el proceso. Si no hay daemon, lo dice y retorna 1.

## `lspd status`

```
$ lspd status

lspd  pid=12345  endpoint=~/.lspd/daemon.sock

language servers:
  rust-analyzer               RUNNING  queries=147
  typescript-language-server  RUNNING  queries=32
```

Un daemon recién arrancado no tiene ninguno: se levantan **por lenguaje y a demanda**, la primera vez que llega una pregunta sobre un archivo de ese lenguaje. `(ninguno arrancado todavía)` es un estado normal y no un problema.

## Arrancarlo no es del daemon

`lspd start` existe para la persona que quiere arrancarlo a mano. **Un consumidor no lo usa**: pregunta `ping`, y si no hay nadie levanta el binario con la política que le convenga. Lattice lo hace apenas el proveedor `lsp` hace falta; bilinker no lo hace nunca —degrada a *no verificado*— y las dos son decisiones de ellos y no de acá.

**Y por eso el binario tiene que estar donde el consumidor pueda encontrarlo**: al lado de su propio ejecutable —el caso de un build local— o en PATH. Es la única cosa que `lspd` le pide al que lo usa, y es consecuencia de vivir en su propia capa: antes compartía el `target/` de lattice y aparecía solo.

Ver [`lattice daemon`](../../lattice/commands/daemon.md), que es el mismo ciclo de vida con el nombre que los usuarios de lattice ya tenían.
