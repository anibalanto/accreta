# Comando: `worklist push`

Empuja la rama actual a la ventana correspondiente en el remoto del worklist, y reporta si el proveedor la aceptó tal cual o la rechazó por haber cambiado del otro lado.

Ejecuta el flujo que [`proposals/ventanas-por-rama.md`](../proposals/ventanas-por-rama.md) describe: el servidor —hoy `pre-receive`— hace un compare-and-swap contra el proveedor antes de aceptar, y un paso posterior —`post-receive`— renombra los ítems sin clave y reescribe lo que los nombraba.

## Uso

```
worklist push [<rama>] [--remote <nombre>]
```

Sin `<rama>`, usa la rama actual. Sin `--remote`, usa `origin`.

## Comportamiento

1. `git push <remote> <rama>`.
2. Si el remoto rechaza el push —el proveedor se movió del lado de algún ítem de la ventana—, lo reporta tal cual lo dice git y no reintenta solo: hay que `fetch` + rebase y volver a empujar. Es el mismo flujo de un push normal rechazado por no ser fast-forward.
3. Si el remoto acepta, hace `fetch` de la rama y compara el commit alcanzado con el que se empujó:
   - **Iguales** — nada que renombrar. Termina.
   - **El remoto avanzó** — el paso posterior corrió: renombró ítems sin clave, reescribió referencias, y agregó un commit. Reporta qué archivos cambiaron de nombre y deja la rama local desactualizada — el llamador decide si hace `pull --rebase`.

## Qué no hace

No renombra nada localmente. No le pregunta nada al proveedor por su cuenta — eso es del lado del servidor. `worklist push` sólo empuja y reporta lo que volvió.

## Salida

```
$ worklist push sprint/10
pushed: sprint/10 -> origin/sprint/10  (a1b2c3d)
server: 2 items renamed
  agregar-funcionalidad-1.task.md -> ACC-101.task.md
  agregar-funcionalidad-2.task.md -> ACC-102.task.md
local branch is behind — run `git pull --rebase` to see the new ids
```

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | push aceptado, con o sin renombres del servidor |
| `1` | push rechazado por el proveedor (ventana desactualizada) |
| `2` | error de red o de git ajeno al proveedor |
