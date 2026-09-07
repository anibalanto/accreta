# Comando: `worklist status`

Dice en qué estado está la vista donde uno está parado, **con una línea por pregunta y sin mezclarlas**.

Trabajar sobre una vista desactualizada produce trabajo que después hay que reconciliar, y **un agente no tiene cómo notarlo**: ve archivos, no ve que el servidor se movió. Hoy la única señal es acordarse de mirar.

## Firma

```
worklist status [<vista>] [--all] [--verify] [--exit-code]
```

| Argumento | Descripción |
|---|---|
| `<vista>` | Cuál mirar. Por defecto, aquella en la que se está parado. |
| `--all` | Todas las vistas del clon. |
| `--verify` | Además, le pregunta al proveedor. **Es la cara**, y por eso se pide. |
| `--exit-code` | Sale con 1 si algo necesita atención, para poder encadenarlo. |

## Son cuatro preguntas, y confundirlas es el error

|  | Contra qué | Qué cuesta |
|---|---|---|
| **al día con el servidor** | git — ¿tengo lo que el servidor resolvió? | un `fetch`, **cero** llamadas al proveedor |
| **sin empujar** | git — ¿hay trabajo que no salió de esta máquina? | nada, es un rango |
| **local** | el worktree — ¿hay algo sin commitear? | nada |
| **coincide con el proveedor** | Jira — ¿alguien movió algo del otro lado? | **una consulta por ítem** |

**No son la misma pregunta y no cuestan lo mismo.** Un chequeo que las mezcle es caro siempre o mentiroso siempre.

### La segunda es la que no tenía quien la contestara

> **Una vista con trabajo sin empujar se ve limpia.** Y es peor que la primera: lo que no se nota no es que falte bajar algo, es que hay trabajo hecho que nadie más tiene.

Medido el 2026-09-07, parado en una ventana con un commit que nunca salió:

```
$ git status
En la rama secure/sprint/21
nada para hacer commit, el árbol de trabajo está limpio

$ git rev-parse --abbrev-ref @{u}
fatal: no se ha configurado upstream para la rama 'secure/sprint/21'
```

Sin upstream, `git status` **no tiene contra qué compararse**, así que nunca dice *"ahead by 1"*. Y las vistas nacen sin upstream, porque `git worktree add` no lo configura: las veinte del clon estaban así.

**Que la vista lo tenga o no es de [`worklist push`](.)**, que es quien decide si configurarlo. `status` no depende de eso — pregunta contra `srv/<rama>`, que existe igual.

## Lo barato corre siempre; lo caro se pide

```
$ worklist status
  vista        secure/sprint/21       ventana
  servidor     al día
  sin empujar  1 commit               → worklist push
  local        limpio
  proveedor    sin verificar          → worklist status --verify
```

**`sin verificar` no es `coincide`**, y por eso ocupa un renglón en vez de callarse. Es la misma regla que [`check-push`](check-push.md#son-dos-árbitros-y-el-segundo-sólo-ve-lo-que-el-primero-no) aplica cuando no puede probar contra el panorama: dar por bueno lo que nadie miró es el mismo defecto de forma que confundir *"no había trabajo"* con *"el trabajo no se hizo"*.

El costo de la pregunta tiene que ser proporcional a lo que se va a hacer — el mismo reparto que el compare-and-swap hace en el push.

### Y la cara no la puede hacer el cliente

`--verify` es una consulta al proveedor, y [el cliente no tiene credenciales](../concepts/distribution.md). Se la pide al servidor, que ya tiene el comando: [`check-push --dry-run --ref <rama>`](check-push.md#comparar-sin-rechazar), que compara y no rechaza.

Es el mismo apoyo que usa el paso 1 de [`pull`](pull.md#y-el-paso-1-supone-que-el-servidor-está-a-mano): hoy el bare está en esta máquina. El día que sea un GitLab, esto necesita el canal cliente→servidor que todavía no existe.

#### Y los argumentos no se inventan: salen del hook

Contra qué proveedor pregunta esta instalación, con qué mapeo de estados y con qué cuenta es **configuración**, y hoy no tiene casa propia: vive en el `hooks/pre-receive` que [`install-hooks`](install-hooks.md) genera. `status --verify` lo lee de ahí y le agrega `--dry-run --ref`.

> **Si `--verify` armara sus propios argumentos, mediría contra un proveedor que puede no ser el que rechaza los pushes.** Sería una segunda fuente, y las dos fuentes difieren el día que alguien reinstala los hooks con otra cosa.

Es la misma razón por la que [`--dry-run` no es un comando aparte](check-push.md#y-no-es-un-comando-aparte-a-propósito): lo que hay que medir es exactamente lo que va a rechazar.

**Es una costura y se sabe.** Leer argumentos de un `sh` generado no es una interfaz; es el único lugar donde esa configuración es autoritativa hoy. El día que tenga casa propia, esto la lee de ahí y esta sección se borra.

#### Y con el proveedor de prueba, `coincide` sería afirmar de más

El de prueba **sólo informa el `status`**, así que el título y el cuerpo no se comparan. La salida lo dice:

```
  proveedor    coincide el status     → el titulo y el cuerpo no se comparan con el proveedor de prueba
```

Que la instalación siga apuntando ahí es una decisión escrita, con su número al lado: cruzar hoy encendería un rechazo que en su mayoría no informa nada. Pero **eso no autoriza a que `status` redondee para arriba** — es el mismo *"`sin verificar` no es `coincide`"* un nivel más adentro.

## El código de retorno informa si se pudo contestar, no la respuesta

**`status` sale con cero aunque las cuatro líneas estén en rojo.** Es la misma regla que [`concepts/sync.md`](../concepts/sync.md#el-éxito-se-lee-de-la-salida-nunca-del-código-de-retorno) le pide al proveedor y que [`check-push --dry-run`](check-push.md#códigos-de-salida) ya aplica: lo que el retorno dice es si la medición se pudo hacer.

**Y un agente necesita lo otro**, porque encadenar es lo único que vuelve verificable una regla. De ahí `--exit-code`, que es explícito y no el default:

```
worklist status --exit-code && …
```

Se decidió así y no al revés porque **el default lo lee una persona**, que quiere mirar y no quiere que su shell se ponga roja por un renglón informativo. Quien encadena lo pide.

| Código | Condición |
|--------|-----------|
| `0` | se pudo contestar, diga lo que diga |
| `1` | no se pudo contestar — no es una vista del worklist, o el servidor no está a mano |
| `1` | con `--exit-code`, además: la vista está atrás, tiene trabajo sin empujar, o el worktree está sucio |

## Lo que todavía no contesta, dicho

**Qué significa *"al día"* para una vista dinámica.** Una vista dinámica [no se empuja](../concepts/sync.md#el-panorama-vive-en-un-solo-lado-y-la-ventana-en-los-dos), así que *"atrás del servidor"* no aplica igual: lo que puede estar viejo es **el recorte** — el panorama tiene ítems que la vista no muestra, o al revés. Es una quinta pregunta y hoy no hay ninguna vista dinámica sobre la cual hacerla.

Cuando la haya, la línea `vista` es la que lo dice: hoy imprime `ventana`, y ahí diría otra cosa.
