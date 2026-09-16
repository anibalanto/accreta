---
name: parallel-work
description: "Cómo se trabajan varios ítems a la vez con muckpile: cuándo vale el paralelo, una vista por ítem, una sesión por vista, un worktree por repo en la rama del ítem —o la capa traída adentro con stratum pull cuando los bilinks cruzan capas—, y cómo cada rama vuelve a main con sus bilinks. Incluye lo que hoy se hace a mano porque la base con .claude/ y accreta init todavía no existen."
when_to_use: "Al abrir dos o más ítems independientes para trabajarlos en paralelo, al preparar una vista para que una sesión arranque ahí, y al cerrar una rama de code-work llevándola a main con sus decisiones de bilinker."
---

Es cómo se abren, se trabajan y se cierran varios ítems a la vez, hoy. La mecánica de cada comando está en el README y en `docs/specs/` del impl de `muckpile`; la de los bilinks, en la skill `bilinker`. Esta dice el orden, y qué se suple a mano mientras `organizacion` no esté implementada. Lo particular de un lote concreto —qué ítems, qué repos, qué archivo es compartido— no va acá: va en el primer mensaje de cada sesión.

## Cuándo vale el paralelo

**Dos ítems van en paralelo si no escriben los mismos fragmentos.** Cada impl es un repo con su propia ref de bilinks, así que dos ítems sobre impls distintos nunca se pisan. Sobre el mismo repo, lo que decide es el archivo: dos ramas que borran directorios distintos se mergean solas; dos ramas que editan líneas adyacentes de la misma tabla chocan en el merge. Lo que varias ramas tienen que tocar en el mismo lugar —una tabla de avance, un contador— se hace una sola vez al final, desde `main`.

**Una decisión que dice "de a uno" hay que leerla:** si el argumento es contra una barrida en un solo cambio, el paralelo en ramas separadas no la contradice. Si el argumento es que el primero prueba el camino, el paralelo espera a que el primero termine.

## Una vez, antes de abrir nada

1. **Instalar lo que las sesiones van a usar,** antes y no después: un binario anterior al commit que cambia un comportamiento hace que todas las sesiones se topen con el mismo error a la vez.
2. **Abrir todas las vistas desde la raíz del proyecto.** Cada `to-work` trae su ítem.

```sh
cd <organización>/projects/<proyecto>
for k in <clave>...; do muckpile to-work $k; done
```

## En cada vista, antes de abrir la sesión

3. **Un worktree por repo que el ítem toca,** en la rama del ítem, que sale de `commit_prefix` y del número de la clave. En una vista con slug —`@algo`— la clave sale del ítem con clave que la vista tenga; si todavía es un borrador, `code-work add` se niega y la rama se nombra con `--branch <rama>`.

```sh
cd to-work/<clave>
muckpile code-work add <repo>
```

**Salvo cuando el ítem cruza capas.** Si sus bilinks tienen puntas `path` hacia otra capa —`path subsystems/<nombre>>impl`—, resuelven por el anidamiento de Stratum, y un clon suelto en `code-work/<impl>/` deja cada una `LAYER_UNREACHABLE` o `BROKEN` antes de empezar. Ahí el worktree es solo del repo raíz, y la capa se trae adentro, en su lugar, con `stratum pull`. Pasa, por ejemplo, cuando una spec que vive en el repo raíz se muda a su impl.

```sh
muckpile code-work add <raíz>
(cd code-work/<raíz>/subsystems/<nombre> && stratum pull)
```

4. **La ref de bilinks de la rama, en cada repo que se toca.** `track` crea `refs/bilink/<rama>` heredando de la de `main`. Una capa traída con `stratum pull` es un clon fresco: primero `init`, después su rama, y recién ahí `track`. Si `track` falla, se para ahí: todo lo demás depende de esto.

```sh
(cd code-work/<repo> && bilinker track <rama> && bilinker status)
(cd code-work/<raíz>/subsystems/<nombre>/.stratum/impl && bilinker init && git checkout -b <rama> && bilinker track <rama> && bilinker status)
```

`muckpile` no ve una capa anidada: no la sincroniza ni la publica. Es un repo más, que se empuja y se mergea a mano como cualquier otro.

5. **Las skills y el hook, copiados a la vista.** Es lo que la base del registro va a hacer sola cuando `organizacion` § 4 esté implementada; hasta entonces se copia a mano, con `-L` para que los enlaces se vuelvan copias. Queda sin trackear en la vista, y `push` no lo sube.

```sh
cp -rL <organización>/.claude .
```

6. **Marcar el ítem en curso** antes de que la sesión arranque, no cuando termine.

```sh
muckpile transition <clave> "En curso"
```

## La sesión

7. **Arranca en la vista, nunca en la raíz del proyecto.** Así llegan `CLAUDE.md` y `AGENTS.md` de la organización desde arriba, y las skills y el hook desde la copia de la vista.

```sh
cd to-work/<clave> && claude
```

8. **El primer mensaje dice el ítem, dónde está cada cosa, el precedente, y qué no tocar.** El ítem ya tiene el diagnóstico y los criterios de cierre; lo que la sesión no puede deducir es qué otras sesiones están abiertas y qué archivo es compartido. La forma:

> Trabajá <clave>, que está en <clave>.<tipo>.md. <Dónde está lo que toca, y en qué rama>. El precedente es <commits o ítem>. No toques <el archivo compartido>: se actualiza al final, una sola vez. Antes de dar algo por hecho, corré bilinker check . en cada worktree.

**Lo que la sesión deja, en orden:** el código o la spec commiteados, las cadenas creadas y aceptadas, y lo que se va del otro repo borrado con sus bilinks. Es el orden de la skill `accreta-method`: código, después spec y decisión, y después se acepta.

## Cerrar cada ítem

9. **Publicar cada rama con su ref.** Son dos actos por repo, y `git push` solo no alcanza. Una capa anidada se publica desde su propio directorio. Antes, `verify-ref` sobre lo nuevo sale ok, y si rechaza no se empuja: la ref es append-only, y un commit mal formado que llega al remoto se queda para siempre.

```sh
(cd code-work/<repo> && bilinker verify-ref refs/bilink/main..refs/bilink/<rama> && git push -u origin <rama> && bilinker push)
```

10. **Llevar a `main` con merge, nunca con rebase.** Un accept fija un contenido que tiene que existir en la historia, y un rebase reescribe esa historia. Las decisiones aceptadas en la rama se traen con `adopt`. Para un repo de `code-work/` se hace en `base/<repo>/`, que siempre está en `main`; para una capa anidada, en la capa misma, porque `base/` no la conoce.

```sh
cd <organización>/projects/<proyecto>/base/<repo>
git pull --ff-only && git merge --no-ff <rama> && bilinker adopt <rama> && git push && bilinker push
```

11. **Cerrar el ítem desde su vista,** con lo que quedó commiteado.

```sh
cd to-work/<clave>
muckpile push .
muckpile transition <clave> Finalizada
```

## Al final, una sola vez

12. **Lo compartido se escribe desde `main`, en un solo commit:** la tabla de avance de la decisión, su contador, su versión y su historial. Varias ramas que editan filas adyacentes de la misma tabla chocan en el merge, y resolverlo varias veces es más caro que escribirla una.

## Lo que hay que saber antes de arrancar

- **Un bilink que se borra se borra en las dos capas que ata,** con `bilinker remove` en cada una: un borrado de un solo lado deja al otro `BROKEN`, y `remove` es lo único que lo publica.
- **Cada copia de un repo levanta su propio daemon de `lspd`, y ninguno se apaga solo.** `check`, `accept` y `apply` levantan uno por ruta —`base/`, cada `code-work/`, cada capa anidada— con sus language servers, y `lspd` no tiene apagado por inactividad. Medido el 2026-09-16: un `rust-analyzer` sobre bilinker ocupa 1,85 GB, y diecisiete vivos a la vez, de un día de trabajo en varias vistas, llevaron la sesión a 20,9 GB y el sistema la mató. Al terminar con una copia, `lspd stop` en ella, y `ps` antes de levantar otra.
- **La memoria es una sola para todas las sesiones abiertas.** Un daemon con sus language servers vivo en la máquina, no uno por sesión: `ps` antes de levantarlo mira también lo que dejaron las otras vistas, y una medición pesada —`jdtls` sobre un repo grande— espera a que no haya otro.
- **El binario instalado también es uno solo.** `~/.cargo/bin/<herramienta>` lo usan todas las sesiones: si una instala su rama a mitad del lote, las otras corren un comando con flags o estados que no conocen. Mientras haya otras sesiones abiertas nadie instala; cada una prueba con el binario de su worktree, `target/debug/<herramienta>`, y si algo externo lo llama por el PATH —el proveedor `bilink` de lattice llama a `bilinker`—, se antepone ese directorio sólo en ese comando. Se instala con las ramas ya en `main`, antes del lote siguiente (paso 1).
- **Encadenar con `&&`, nunca con `;`,** lo que verifica y lo que publica: un `verify-ref` que rechaza y un `push` que sale igual es el error más caro de esta lista.
- **Una capa que no hizo el corte** tiene `.bilink/` en el árbol de la rama y ninguna `refs/bilink/*`. La sesión que la toque hace el corte además de su trabajo: ver la skill `bilinker`, § "Dónde viven".
- **En el worktree del repo raíz, `bilinker check .` reporta `LAYER_UNREACHABLE`** en los bilinks que cruzan a capas que nadie trajo. Es esperado. Los que importan son los de la capa que se está tocando, y esos resuelven porque está traída en su lugar.
- **Con `auto_update` en `false`, cada escritura en el proveedor pide la frase en la terminal.** Una sesión de Claude no tiene terminal para eso: `transition` y `push` los corre la persona, con `!` desde el prompt. Cambiar el flag no es una opción de la sesión.
