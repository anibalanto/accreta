# Comando: `worklist-server check-push`

El compare-and-swap de [`concepts/sync.md`](../concepts/sync.md#la-ventana-y-el-compare-and-swap), pensado para correr desde `hooks/pre-receive`. No mira el contenido que llega: compara el estado en vivo del proveedor contra lo que el tip actual de la rama —el del servidor, antes de este push— tiene escrito.

## Firma

```
worklist-server check-push (--provider-file <archivo> | --project <clave>) [--stdin]
```

| Argumento | Descripción |
|---|---|
| `--provider-file` | El proveedor **de prueba**: un archivo `clave -> status`. Ver [`worklist-server provider set-status`](provider-set-status.md). |
| `--project` | El proveedor **real**, por `acli`. |
| `--stdin` | Lee `<viejo> <nuevo> <ref>` por línea — el protocolo de un `pre-receive`. Sin esto, toma el rango de la rama actual. |

**Uno de los dos, y no los dos.** El de prueba sólo informa el `status`, así que con él **el título y el cuerpo no se comparan** — el paso 4 de abajo se achica a una tercera parte. Es lo que hoy tiene la instalación, porque el `status` del worklist y el de Jira no son el mismo campo y compararlos rechazaría todas las ventanas. La task `74` es ese mapeo.

## Qué hace

1. Para cada `ref` que llega: si es `refs/heads/insecure/**`, **rechaza el push** — una rama insegura no se puede verificar, así que no puede aceptar escrituras. Si no es `refs/heads/secure/**`, la ignora: este comando no opina sobre ramas que no son del worklist.
2. Lee, del árbol de `<viejo>`, todos los `*.md` cuyo nombre ya es una clave de proveedor (`is_unassigned` en falso), y su campo `status`.
3. Le pregunta al proveedor su estado ahora mismo para esas mismas claves.
4. Si algún valor difiere, rechaza — imprime cuál clave y los dos valores.
5. Y **prueba contra el panorama**: si lo que la ventana traería no aplica sobre `insecure/all`, rechaza. Ver [`concepts/propagation.md`](../concepts/propagation.md#y-el-panorama-nunca-guarda-un-conflicto).

Si las dos pruebas pasan, acepta.

## Son dos árbitros, y el segundo sólo ve lo que el primero no

El proveedor arbitra todo lo que tiene clave: dos ventanas que editan el mismo ítem chocan en el paso 4, un paso antes de que el panorama se entere. Lo que llega al paso 5 es lo que **no tiene contraparte allá** — en la práctica, un ancestro de sólo lectura que alguien editó adentro de su ventana.

**Se prueba en memoria, con `merge-tree`.** Un `pre-receive` que rechaza no puede dejar un worktree ni objetos atrás, y el paso 5 corre en todos los pushes, no sólo en los que fallan.

**Y no poder probar no es que entre.** Un repo sin panorama —hoy, el de la instalación— no puede correr el paso 5, y eso se **avisa**: dar por bueno lo que nadie miró es el mismo defecto de forma que confundir *"no había trabajo"* con *"el trabajo no se hizo"*.

```
aviso: refs/heads/secure/sprint/10 no se pudo probar contra el panorama — este repo no tiene refs/heads/insecure/all
```

No rechaza el push: informa. Que el panorama no esté del lado del servidor es un problema de la instalación, no de quien empuja.

**Y no prueba los renombres**, porque no se copian: [se rehacen](../concepts/propagation.md#el-renombre-es-el-único-que-no-se-copia-y-el-motivo-es-de-alcance) sobre el árbol del panorama, y ahí no hay parche que pueda no aplicar.

## Salida

```
$ worklist-server check-push --provider-file provider.json --stdin <<< "a1b2c3d e4f5g6h refs/heads/sprint/10"
reject: ACC-101 status era "open" en el tip, el proveedor dice "done"
```

Y el segundo árbitro:

```
reject: a2b035d edito ACC-14 no entra al panorama — choca en ACC-14.epic.md
        alguien mas escribio eso desde otra ventana. Regenera la tuya y volve a aplicarlo.
```

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | todas las ventanas del rango coinciden con el proveedor |
| `1` | al menos una clave difiere — push rechazado |
| `1` | lo que la ventana traería no aplica sobre el panorama — push rechazado |
