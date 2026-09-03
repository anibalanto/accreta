# Comando: `worklist assign-keys`

Resuelve los pedidos que un push dejó en una ventana: les pide una clave al proveedor, los renombra y reescribe lo que los nombraba. Pensado para correr desde `hooks/post-receive`.

**No puede vivir en `pre-receive`.** Ese hook sólo acepta o rechaza lo que llega; no puede reescribirlo. Ver [`concepts/sync.md`](../concepts/sync.md#dos-pasos-no-uno).

## Firma

```
worklist assign-keys --project <clave> [--stdin] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--project` | La clave del proyecto en el proveedor. |
| `--stdin` | Lee `<viejo> <nuevo> <ref>` por línea — el protocolo de un hook de recepción. |
| `--dry-run` | No llama al proveedor ni escribe nada. Imprime qué asignaría, en qué orden. |

## Comportamiento

Para cada `ref` que sea una ventana —`refs/heads/sprint/*` o `refs/heads/backlog`—:

1. Lista los `*.md` del árbol de `<nuevo>` cuyo nombre **no** es una clave de proveedor: ésos son los pedidos.
2. Los ordena topológicamente por `parent` y `relation.*`, sobre el subgrafo de pedidos. Un ciclo aborta sin escribir nada.
3. Para cada uno, en ese orden, pide su clave con [`create-or-find`](create-or-find.md).
4. Renombra y reescribe sus referencias, un commit por ítem.
5. Mueve la ref al commit resultante.

## Corre sobre un repo bare, sin working tree

Un hook de recepción no tiene árbol de trabajo, y los pasos 4 y 5 necesitan uno. La salida es un **worktree temporal en `--detach`** sobre `<nuevo>`: ahí se hacen los renombres, y al terminar se mueve la ref al `HEAD` resultante y se descarta el worktree.

**En `--detach` y no sobre la rama**, porque una rama con worktree asignado no acepta pushes — y el hook no puede dejar el repo en un estado donde el próximo push falle.

## Salida

```
$ worklist assign-keys --project ACC --stdin <<< "a1b2c3d e4f5g6h refs/heads/sprint/10"
refs/heads/sprint/10: 2 pedido(s)
  orden: agregar-b, agregar-a
  agregar-b -> ACC-101
  agregar-a -> ACC-102  (1 refs reescritas)
refs/heads/sprint/10: e4f5g6h -> 9z8y7x6
```

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | todos los pedidos resueltos, o no había ninguno |
| `1` | un ciclo entre pedidos, o el proveedor falló — la ref queda como estaba |
