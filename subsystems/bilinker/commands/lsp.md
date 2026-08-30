# Especificación: `bilinker-lsp`

Un language server que muestra, dentro del editor, los bilinks que tocan el archivo abierto. No verifica nada y no escribe nada: es una vista.

## Por qué es un servidor y no un comando

`bilinker get` contesta la misma pregunta, y contestarla desde el editor por CLI significaría un proceso por consulta y una integración distinta por editor. Un language server la contesta por un protocolo que los editores ya hablan: la extensión de VS Code que lo consume no sabe nada de bilinker más allá de arrancarlo.

Es también la razón de que no tenga comandos propios. Todo lo que hace sale de `find_by_file` y `get`, que son la misma API que usa el CLI — si el servidor tuviera lógica propia habría dos respuestas posibles a la misma pregunta.

## Transporte

stdio, JSON-RPC de LSP. No abre puertos ni sockets: lo arranca el editor y muere con él.

## Capacidades

Declara exactamente dos, y ninguna más:

| Capacidad | Qué muestra |
|---|---|
| `hover` | Los endpoints que cubren la posición: su uuid, su estado y el fragmento del otro extremo. |
| `codeLens` | Una lente por línea donde empieza al menos un endpoint, con cuántos hay. |

`codeLens` no resuelve —`resolve_provider: false`—: la lente se emite completa. Resolver perezosamente serviría si construirla fuera caro, y no lo es: sale del mismo escaneo que ya se hizo.

La lente emite el comando `bilinker.showBilinks` con la URI del archivo y los ids de esa línea. **El servidor no ejecuta ese comando**: lo implementa la extensión, porque abrir un panel es decisión del editor y no del protocolo. Ver [editors/vscode.md](../../lattice/editors/vscode.md).

## Raíz del proyecto

Se detecta por archivo, no por workspace: cada URI se resuelve con la misma detección de raíz que el CLI —caminar hacia arriba buscando `.bilink/` o `.git/`— y no con el `rootUri` que manda el editor.

Es lo que hace que funcione en un workspace con varias capas: un archivo de `subsystems/bilinker/.stratum/impl` y uno de la raíz se resuelven contra capas distintas en la misma sesión, sin configuración.

Un archivo que no cae bajo ninguna capa no produce hover ni lentes. No es un error: es un archivo sin bilinks.

## Estado

Ninguno. El servidor no cachea nada entre consultas, y por eso no puede quedar desactualizado respecto del disco. Cada hover vuelve a escanear.

Cuando el escaneo pese, el lugar donde arreglarlo es el [índice](../concepts/index.md), que es un derivado compartido con el CLI — no una cache propia del servidor, que sería una segunda verdad que invalidar.

## Lo que no hace

- **No corre `check`.** Muestra lo que la cache ya dice; un estado ausente se muestra ausente. Verificar es una acción del usuario, no un efecto de abrir un archivo.
- **No escribe.** Ni bilinks, ni cache, ni índice.
- **No ofrece acciones de código.** Aceptar o repuntar son decisiones, y una decisión no se toma desde un menú contextual sin ver el diff.

## Código de salida

| Código | Condición |
|---|---|
| 0 | El editor cerró la conexión. |
| 1 | No pudo hablar LSP por stdio. |
