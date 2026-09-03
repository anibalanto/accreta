# Comando: `worklist window open`

Recorta una ventana: produce la rama `secure/sprint/<id>` con **sólo los ítems de ese sprint y sus ancestros**, cortada desde el panorama.

Sin esto, una rama segura no puede prometer lo que su nombre dice. Nacida de un `git checkout -b` sobre `insecure/all`, se trae todos los ítems, y *"me hago responsable de verificarme entera"* deja de ser sostenible — ver [`concepts/sync.md`](../concepts/sync.md#dos-clases-de-rama-y-el-nombre-dice-qué-se-puede-hacer).

## Firma

```
worklist window open <sprint-id> [--from <rama>] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `<sprint-id>` | El id del sprint: recorta `_sprints/<id>.sprint.md` y lo que declara. |
| `--from` | De dónde cortar. Por defecto `insecure/all`, el panorama. |
| `--dry-run` | Lista qué entraría, sin escribir la rama. |

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

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | la ventana quedó recortada |
| `1` | el sprint no existe en `--from`, o su `items` nombra algo que no está |
