# Comando: `worklist-server install-hooks`

Escribe los hooks del bare apuntando al binario que los está escribiendo. El corte y su motivo: [`concepts/distribution.md`](../concepts/distribution.md#los-hooks-los-escribe-el-servidor).

## Firma

```
worklist-server install-hooks --repo <bare> --project <clave> --board <id>
                              [--provider-file <archivo>] [--base <url>] [--force] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--repo` | El bare del servidor. Los hooks van en `<repo>/hooks/`. |
| `--project` | La clave del proyecto de Jira. La usa el `post-receive`, que es el que crea y actualiza. |
| `--provider-file` | El proveedor **de prueba** para el `pre-receive`. Sin él, el compare-and-swap se instala contra el proveedor real. Ver § "Con qué proveedor queda el `pre-receive`". |
| `--board` | El id del board. **Obligatorio y sin default**, por lo mismo que en [`assign-keys`](assign-keys.md): una pasada que se saltea sola porque falta configuración es la peor forma de enterarse de que falta. |
| `--base` | Base del proveedor, para traducir los links. Default `https://lamansys.atlassian.net`. |
| `--force` | Sobrescribe un hook que ya existe. Sin él, un hook presente es un error. |
| `--dry-run` | Imprime los dos scripts y no escribe nada. |

## Comportamiento

1. Resuelve **su propio path** con `current_exe`, y lo canonicaliza. Ese path es lo que va escrito en los scripts.
2. Verifica que `<repo>` sea un bare — que tenga `hooks/`. Si no, falla sin escribir.
3. Escribe `hooks/pre-receive` y `hooks/post-receive`, con permiso de ejecución.
4. Imprime los dos paths y a qué binario quedaron apuntando.

**El path lo resuelve el binario y no lo tipea nadie**, que es la única parte que puede estar mal. Un hook apuntó una vez a un `target/debug` viejo, y eso es un fix verde en los tests y ausente en producción: los tests corren la lib, no el hook.

**Y un hook que ya existe no se pisa sin pedirlo.** Instalar sobre un servidor andando es la operación en la que menos conviene descubrir que había algo escrito a mano; `--force` es lo que vuelve la sobrescritura una decisión y no un efecto.

### Con qué proveedor queda el `pre-receive`

> **Contra el real, el compare-and-swap rechaza todas las ventanas de entrada.**

No es un defecto del comando: el `status` del worklist y el del proveedor **no son el mismo campo**, así que compararlos difiere siempre. El mapeo es otra task, y mientras no exista, la instalación corre contra el proveedor de prueba — un archivo `clave -> status`.

Por eso `--provider-file` existe acá y no es una opción de desarrollo: **es la configuración de hoy.** El día que el mapeo esté, se reinstala sin ese flag y el hook pasa a `--project`.

Y es la mitad del problema que generar el path no resuelve: el path lo pone bien el binario, pero **el comando tiene que ser el que la instalación necesita**. Un generador que no sabe esto instala un hook que rechaza todo.

### Lo que quedan siendo los hooks

```sh
#!/bin/sh
# generado por worklist-server install-hooks — no editar
exec /usr/local/bin/worklist-server check-push --stdin --provider-file /…/provider.json
```

```sh
#!/bin/sh
# generado por worklist-server install-hooks — no editar
exec /usr/local/bin/worklist-server assign-keys --stdin --project ACC --board 701 --base https://lamansys.atlassian.net
```

**El `post-receive` no llama a `propagate`, y no es un olvido**: [`assign-keys`](assign-keys.md) ya propaga al final, con el motivo escrito ahí — *"lo que más falta arriba son las claves y las acaba de escribir la pasada 1"*. Un `propagate` aparte correría la propagación dos veces.

> **Un generador de hooks no puede saber menos que el hook que reemplaza.**

## Salida

```
$ worklist-server install-hooks --repo ~/.local/share/accreta/worklist-sync/worklist.git --project ACC --board 701
hooks/pre-receive  -> /usr/local/bin/worklist-server
hooks/post-receive -> /usr/local/bin/worklist-server

$ worklist-server install-hooks --repo ~/.local/share/accreta/worklist-sync/worklist.git --project ACC --board 701
error: hooks/pre-receive ya existe — `--force` para sobrescribirlo
```

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | los dos hooks escritos |
| `1` | `<repo>` no tiene `hooks/`: no es un bare |
| `1` | un hook ya existe y no se pasó `--force` |
| `1` | `current_exe` no resuelve, así que no hay path que escribir |
