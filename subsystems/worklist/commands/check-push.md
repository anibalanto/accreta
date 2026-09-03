# Comando: `worklist check-push`

El compare-and-swap de [`concepts/sync.md`](../concepts/sync.md#la-ventana-y-el-compare-and-swap), pensado para correr desde `hooks/pre-receive`. No mira el contenido que llega: compara el estado en vivo del proveedor contra lo que el tip actual de la rama —el del servidor, antes de este push— tiene escrito.

## Firma

```
worklist check-push --provider-file <archivo> [--stdin]
```

| Argumento | Descripción |
|---|---|
| `--provider-file` | Dónde vive el estado del proveedor de prueba — ver [`worklist provider set-status`](provider-set-status.md). |
| `--stdin` | Lee `<viejo> <nuevo> <ref>` por línea — el protocolo de un `pre-receive`. Sin esto, toma el rango de la rama actual. |

## Qué hace

1. Para cada `ref` que llega: si es `refs/heads/insecure/**`, **rechaza el push** — una rama insegura no se puede verificar, así que no puede aceptar escrituras. Si no es `refs/heads/secure/**`, la ignora: este comando no opina sobre ramas que no son del worklist.
2. Lee, del árbol de `<viejo>`, todos los `*.md` cuyo nombre ya es una clave de proveedor (`is_unassigned` en falso), y su campo `status`.
3. Le pregunta al proveedor su estado ahora mismo para esas mismas claves.
4. Si algún valor difiere, rechaza — imprime cuál clave y los dos valores. Si todos coinciden, acepta.

## Salida

```
$ worklist check-push --provider-file provider.json --stdin <<< "a1b2c3d e4f5g6h refs/heads/sprint/10"
reject: ACC-101 status era "open" en el tip, el proveedor dice "done"
```

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | todas las ventanas del rango coinciden con el proveedor |
| `1` | al menos una clave difiere — push rechazado |
