# Comando: `worklist-server install-hooks`

Escribe los hooks del bare apuntando al binario que los está escribiendo. El corte y su motivo: [`concepts/distribution.md`](../concepts/distribution.md#los-hooks-los-escribe-el-servidor).

## Firma

```
worklist-server install-hooks --repo <bare> --project <clave> --board <id> [--base <url>] [--force] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--repo` | El bare del servidor. Los hooks van en `<repo>/hooks/`. |
| `--project` | La clave del proyecto de Jira. Va escrita en los dos hooks, porque es lo que reciben como argumento. |
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

### Lo que quedan siendo los hooks

```sh
#!/bin/sh
# generado por worklist-server install-hooks — no editar
exec /usr/local/bin/worklist-server check-push --stdin --project ACC
```

```sh
#!/bin/sh
# generado por worklist-server install-hooks — no editar
set -e
/usr/local/bin/worklist-server assign-keys --stdin --project ACC --board 701 --base https://lamansys.atlassian.net
/usr/local/bin/worklist-server propagate --stdin --base https://lamansys.atlassian.net
```

El `post-receive` corre las dos cosas en orden y **para en la primera que falle**: propagar lo que `assign-keys` no llegó a resolver subiría al panorama un estado a medias.

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
