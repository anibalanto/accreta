# Comando: `worklist window open`

Recorta una ventana: produce la rama `secure/sprint/<id>` con **sólo los ítems de ese sprint y sus ancestros**, cortada desde el panorama.

Sin esto, una rama segura no puede prometer lo que su nombre dice. Nacida de un `git checkout -b` sobre `insecure/all`, se trae todos los ítems, y *"me hago responsable de verificarme entera"* deja de ser sostenible — ver [`concepts/sync.md`](../concepts/sync.md#dos-clases-de-rama-y-el-nombre-dice-qué-se-puede-hacer).

## Firma

```
worklist window open <sprint-id> [--from <rama>] [--dry-run] [--force]
```

| Argumento | Descripción |
|---|---|
| `<sprint-id>` | El id del sprint: recorta `_sprints/<id>.sprint.md` y lo que declara. |
| `--from` | De dónde cortar. Por defecto `insecure/all`, el panorama. |
| `--dry-run` | Lista qué entraría, sin escribir la rama. |
| `--force` | Recorta aunque la rama ya exista con commits propios, **descartándolos**. |

## Qué entra

1. **El `.sprint.md`**, que vive en su propia ventana: es el plan de esa iteración.
2. **Los ítems que su `items` declara**, y para cada uno **todo su subárbol** — los hijos van con el padre, porque [lo que entra a un sprint es un subárbol entero](../concepts/hierarchy.md#la-regla-del-ancestro).
3. **Los ancestros de cada uno, de sólo lectura** — en la práctica la épica, que [no entra a un sprint](../concepts/hierarchy.md#épicas) y viaja para que la cadena `parent` cierre adentro de la ventana.

Y nada más. **Lo que no es de la ventana no está**, que es el punto: parado adentro, un ítem ajeno no puede confundirse con uno tuyo, porque no está.

## Para leer algo que no es de la ventana

El panorama lo tiene todo:

```
git show insecure/all:ACC-3.task.md
```

O su worktree, si está materializado. Traerlo a la ventana sería ensanchar el conjunto que la ventana promete verificar — exactamente lo que el recorte evita.

## Salida

```
$ worklist window open 10
secure/sprint/10: 5 archivo(s)
  _sprints/10.sprint.md
  ACC-2.task.md        ← de items
  ACC-3.task.md        ← subárbol de ACC-2
  ACC-9.task.md        ← de items
  ACC-1.epic.md        ← ancestro, sólo lectura
recortado desde insecure/all (a1b2c3d) -> 9z8y7x6
```

## Abrir es una vez. Traerla al día es un `pull`

> **`window open` no se corre dos veces sobre la misma ventana.**

Recortar produce una rama derivada del panorama. Una ventana que ya vivió tiene encima lo que el servidor escribió —los `rename <slug> -> <clave>`, los `normalize:`— y eso **no está en el panorama**, porque las claves quedan en la rama de cada ventana y la propagación todavía no existe.

Así que volver a cortar no actualiza: **reemplaza**, y se lleva puesto todo eso. Y si alguien editó un ítem adentro de su ventana, se lleva también su trabajo.

**Por eso el comando se niega** cuando la rama ya existe y su punta no es ancestro del corte nuevo:

```
$ worklist window open 7
error: secure/sprint/7 ya existe y tiene 19 commit(s) que este corte no contiene
  a2b035d normalize: ACC-93
  27faaeb rename m -> ACC-93 (2 refs)
  …
  recortar de nuevo los descarta. Para traer lo del servidor:
      git fetch srv && git merge --ff-only srv/secure/sprint/7
  Para descartarlos igual: --force
```

No es una advertencia: **es un error y no escribe nada.** Perder lo que el servidor escribió deja de ser algo que pasa en silencio.

### Traer es `--ff-only`, y que no lo sea es información

El servidor **commitea encima** de lo que empujaste: no reescribe tu commit, le agrega los suyos. Así que el caso sano es siempre un fast-forward, y **que no lo sea quiere decir que la rama divergió** — alguien más la empujó, o se re-cortó con `--force`. Un merge automático ahí escondería justo lo que hay que mirar.

### Y `--force` es para cuando el `items` cambió

Es el caso legítimo de re-cortar: el sprint tomó o soltó un ítem, y la ventana tiene que reflejarlo. Hacerlo bien **depende de la propagación** —sin ella, cortar desde el panorama pierde las claves que sólo la ventana tiene— así que hoy `--force` es una salida de emergencia y no el flujo de ese caso.

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | la ventana quedó recortada |
| `1` | el sprint no existe en `--from`, o su `items` nombra algo que no está |
| `1` | la rama ya existe y el corte descartaría commits — salvo `--force` |
