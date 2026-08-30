# La aceptación

Aceptar es decir *"revisé esto y lo apruebo"*. Es el único acto del sistema que no se puede derivar de nada, y por eso es lo único —junto con la declaración— que queda versionado.

## Las dos dimensiones

Un endpoint puede desalinearse de dos formas distintas, y hay que poder aprobarlas por separado:

| Dimensión | Cambió | Se aprueba |
|---|---|---|
| **Ubicación** | el fragmento se movió: otro archivo, otro nodo | `accepted.link` |
| **Contenido** | el fragmento sigue donde estaba y dice otra cosa | `accepted.hash` |

Antes había una sola: `hash.N` respondía por el contenido, y la ubicación se corregía sola cuando `apply` reescribía el capture. Eso hacía que **mover un fragmento no requiriera aprobación de nadie**: `apply` decidía a qué otro nodo apuntaba el vínculo y lo dejaba en `OK`.

Con las dos dimensiones separadas, `apply` sigue proponiendo la ubicación nueva pero no la bendice. El endpoint queda en `RELOCATED` hasta que alguien acepte.

> **`apply` propone, `accept` dispone.** Es la misma división que ya había entre corregir y aprobar, extendida a la dimensión que se había quedado afuera.

## El bloque

```yaml
endpoint:
  0:
    link: capture 67ba7217e0334051becd4921b55a7872
    accepted:
      link: capture 67ba7217e0334051becd4921b55a7872
      hash: c00e07602bd560755096b57df1ddb9ed49d816fb8af58a4ec9cde82f21f38db3
      hash_ast: 1b9e44a2f0c8d3e7a5b1c9d4e2f6a8b0c3d5e7f9a1b3c5d7e9f1a3b5c7d9e1f3
  1:
    link: path >impl
```

| Campo | |
|---|---|
| `link` | La ubicación aprobada. Ausente en un endpoint `issue` o `abstract`, que no tienen capture. |
| `hash` | SHA-256 del fragmento aprobado. |
| `hash_ast` | SHA-256 de su s-expression. Opcional: ausente donde no hay gramática. |

**`accepted` está o no está.** Su ausencia *es* `PENDING`, literalmente — no hay que enunciar que los campos de aceptación están presentes juntos o ausentes juntos, porque el bloque no se puede escribir a medias. Lo verifica el tipo: `accepted` sin `hash` es rechazado, y un `hash` suelto afuera del bloque también.

`hash` y `hash_ast` van separados y no hasheados juntos porque `RESTYLED` necesita compararlos por separado. Donde `hash_ast` no está, `RESTYLED` no existe y todo cambio de texto es `ALTERED` — que en prosa es lo correcto.

## Aceptar exige el fragmento commiteado

`accept` **falla sobre un archivo sucio.**

No es una recomendación: es lo que hace posible calcular `commit`, que es el commit en el que el fragmento quedó con el contenido aprobado. Ese commit no existe si el fragmento no está commiteado.

Y es lo que garantiza que lo aprobado sea recuperable. Sin commit no hay `git show <commit>:<file>`, y sin eso `check` no puede recuperar el texto aceptado — que es lo que distingue `EXPANDED`, `DISPLACED` y `REANCHORED` de un `ALTERED` genérico.

## Aceptar es determinista

Dos personas que aceptan el mismo fragmento en el mismo estado escriben lo mismo. `accepted` no lleva quién aceptó, ni cuándo, ni desde qué HEAD: sólo qué se aprobó.

De ahí que `commit` sea el commit **del contenido** y no el HEAD de quien acepta, que es lo que hace hoy `accept`. Con el HEAD, el mismo acto daba distinto según quién y cuándo lo hiciera, y el valor no describía nada del fragmento.

## `--place` y `--content`

Por defecto `accept` aprueba las dos dimensiones. Los dos flags permiten aprobar una sola:

```sh
bilinker accept <uuid>.<N>              # ubicación y contenido
bilinker accept <uuid>.<N> --place      # sólo la ubicación
bilinker accept <uuid>.<N> --content    # sólo el contenido
```

Aceptar la ubicación de un fragmento cuyo contenido no se revisó es una situación real —el archivo se renombró y el código no cambió, o cambió y hay que mirarlo aparte— y sin los flags habría que aprobar de más para poder avanzar.

## Un endpoint layer copia las dos

El `accepted` de un endpoint layer o repo copia los **dos** valores del endpoint estructural del bilink adyacente: su `link` y su `hash`.

Cada uno cambia por una sola razón y por ninguna otra: `hash` cuando cambia el contenido publicado, `link` cuando cambia su ubicación aprobada. Los dos son inmunes a etiquetas, comentarios y reordenamientos del archivo vecino — que es la razón de copiar los valores y no hashear el archivo entero.

## Lo que no se acepta a ciegas

Un `accept .` sobre una capa recién cambiada fabrica aprobaciones que nadie miró. Cada estado no-OK es un puntero al fragmento que hay que revisar, y **el inventario de trabajo de un cambio es esa lista**; vaciarla para dejar el árbol verde es tirar justamente lo que hacía falta.

`accept .` existe para el caso en que ya se revisó todo, no para el caso en que no se revisó nada.

## Invariantes

1. `accepted` está completo o ausente. Su ausencia es `PENDING`.
2. `accept` es el único que escribe `accepted`.
3. `accept` falla si el fragmento no está commiteado.
4. Aceptar el mismo fragmento en el mismo estado produce siempre el mismo `accepted`.
5. `accepted.link` de un endpoint layer o repo es una copia opaca del id ajeno: se compara, no se resuelve.
6. Aprobar una ubicación y aprobar un contenido son actos separables.
