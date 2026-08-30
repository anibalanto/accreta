# Especificación: comando `bilinker sync`

## Propósito

Alinea `refs/bilink/<branch>` con la rama del proyecto, absorbiéndola. Cubre el caso en que **el proyecto avanzó y nadie aceptó nada**.

**No verifica nada** — de ahí el nombre; `update` sugeriría que recalcula estados, que es lo que no hace.

## Firma

```
bilinker sync [--dry-run] [--push]
```

| Argumento | Descripción |
|---|---|
| `--dry-run` | Muestra qué haría sin escribir nada. |
| `--push` | Empuja `refs/bilink/<branch>` al remoto después de commitear. |

No toma rama: opera sobre la rama actual del proyecto y su ref. Cambiar de rama es un `git checkout`, y después de él [la materialización es automática](../concepts/ref.md#la-materialización-es-automática).

## Qué hace

Un solo commit sobre la ref, con dos padres: el tip de la ref y el tip de la rama del proyecto.

```
1. verificar disyunción sobre el árbol del tip de la rama
2. read-tree    <tip de la rama>
3. update-index únicamente .bilink/
4. commit-tree  -p <tip de la ref> -p <tip de la rama>
5. update-ref   refs/bilink/<branch>
6. escribir .bilink/head
```

El árbol de `.bilink/` no cambia: sale del índice propio, que ya lo tenía. Por eso **el diff de `sync` contra su primer padre es vacío** — es el único commit de la ref del que eso vale, y es lo que lo identifica como el acto que no registra ninguna decisión: alinea la foto y nada más.

Los pasos 1 a 3 son [la construcción del commit](../concepts/ref.md#cómo-se-arma-el-commit) que comparten todos los comandos que escriben sobre la ref. `sync` no agrega nada propio; lo que lo distingue es que no escribe ningún archivo antes.

## Cuándo no hay nada que hacer

Si el tip de la rama ya está absorbido, `sync` no escribe ningún commit. Un commit de merge con el mismo segundo padre que el anterior y el mismo `.bilink/` no dice nada que la ref no diga ya.

```
$ bilinker sync
refs/bilink/main ya absorbió main @ 4e77d20 — nada que hacer
```

No es un error, y es el caso más común: `accept` y `apply` absorben como parte de su acto, así que quien trabaja todos los días casi nunca necesita `sync`.

## No es lo mismo que `accept` ni que `apply`

Los tres absorben, porque [absorber es precondición de todo commit sobre la ref](../concepts/ref.md#la-invariante-de-fidelidad). La diferencia es qué escriben antes:

| | Escribe en `.bilink/` | Diff contra su 1er padre |
|---|---|---|
| [`accept`](accept.md) | `accepted` | los campos aceptados |
| [`apply`](apply.md) | `link` | los links repuntados |
| `sync` | nada | vacío |

Correr `sync` antes de un `accept` no cambia el resultado: el `accept` habría absorbido igual, en su propio commit. Lo que `sync` compra es poder alinear sin decidir nada — y que la ref del proveedor esté al día para el consumidor remoto, que es quien sí depende de la foto.

## No recalcula estados ni escribe la cache

`sync` no corre tree-sitter, no resuelve captures y no toca [`cache/state`](../concepts/cache.md).

Esto es deliberado y no una omisión: el árbol de código del árbol de trabajo no cambió —`sync` no hace checkout de nada— así que los estados que `check` calculó siguen siendo los mismos. Lo único que cambió es qué commit de la ref los avala, y eso lo anota la cache al ser escrita, no `sync`.

## La verificación de disyunción

Es la única que puede fallar acá, y falla ruidosamente:

```
$ bilinker sync
error: el árbol de main @ 8c31f0a contiene .bilink/

  Alguien mergeó refs/bilink/main en main, o commiteó bilinks a mano.
  Absorberlo haría que el árbol de la ref contenga dos .bilink/ que git
  fusionaría sin que nadie mire.

  No se escribió nada.
```

Va sobre el **árbol** del commit y no sobre su diff: el commit que *borra* `.bilink/` tiene un diff que lo toca y un árbol que no, y es exactamente el commit que hay que poder absorber — el `X` del corte es eso.

La reparación está diferida, en [`proposals/verificar-ref-ajena.md`](../proposals/verificar-ref-ajena.md).

## `--push`

Empuja `refs/bilink/<branch>` con el refspec que [`init`](init.md) dejó puesto. Es siempre fast-forward: la ref es append-only.

Separado del comando porque **sincronizar local y publicar son dos cosas**, y quien trabaja en una rama propia hace lo primero muchas veces antes de lo segundo. Un `sync` que empujara solo convertiría un comando local en uno que habla con la red.

## Salida

```
$ bilinker sync
absorbe:  main  4e77d20..b1e3f55  (3 commits)
commit:   refs/bilink/main  9c1f0ab → 7a2d4e8
diff:     vacío — ninguna decisión registrada
```

Con `--dry-run`:

```
$ bilinker sync --dry-run
absorbe:  main  4e77d20..b1e3f55  (3 commits)
disyunción: ok — ningún commit trae .bilink/ en su árbol

dry-run: no se escribió nada
```

## Códigos de salida

| Código | Condición |
|---|---|
| 0 | Absorbido, o ya estaba al día. |
| 1 | La verificación de disyunción falló; o `HEAD` está desacoplado; o la rama no tiene ref. |
| 1 | El `.bilink/` del árbol no corresponde al commit que `head` nombra. |

En `HEAD` desacoplado `sync` se niega: no hay rama actual contra la cual comparar, y [los comandos que commitean sobre la ref esperan a volver a una rama](../concepts/ref.md#en-head-desacoplado-no-se-materializa-nada).

Si la rama no tiene ref, el arreglo es [`track`](track.md), no `sync`: crear la ref de una rama es decidir de quién hereda los bilinks, y eso no se adivina.

## Propiedades garantizadas

- **No verifica**: no corre tree-sitter, no resuelve captures, no escribe la cache.
- **No escribe ningún archivo del árbol de trabajo** salvo `.bilink/head`.
- **Idempotente**: correrlo dos veces no escribe un segundo commit.
- **`--dry-run` no escribe** y no habla con la red.
- El commit que escribe tiene **diff vacío** contra su primer padre.
