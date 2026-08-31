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
      agree:
        - pablo
      link: capture 67ba7217e0334051becd4921b55a7872
      hash: c00e07602bd560755096b57df1ddb9ed49d816fb8af58a4ec9cde82f21f38db3
      hash_ast: 1b9e44a2f0c8d3e7a5b1c9d4e2f6a8b0c3d5e7f9a1b3c5d7e9f1a3b5c7d9e1f3
  1:
    link: path >impl
```

| Campo | |
|---|---|
| `agree` | Quiénes aprobaron **estos** valores. Ver "Quiénes aprobaron". |
| `link` | La ubicación aprobada. Ausente en un endpoint `issue` o `abstract`, que no tienen capture. |
| `hash` | SHA-256 del fragmento aprobado. |
| `hash_ast` | SHA-256 de su s-expression. Opcional: ausente donde no hay gramática. |

**`accepted` está o no está.** Su ausencia *es* `PENDING`, literalmente — no hay que enunciar que los campos de aceptación están presentes juntos o ausentes juntos, porque el bloque no se puede escribir a medias. Lo verifica el tipo: `accepted` sin `hash` es rechazado, y un `hash` suelto afuera del bloque también.

`hash` y `hash_ast` van separados y no hasheados juntos porque `RESTYLED` necesita compararlos por separado. Donde `hash_ast` no está, `RESTYLED` no existe y todo cambio de texto es `ALTERED` — que en prosa es lo correcto.

## Aceptar exige el fragmento commiteado

`accept` **falla sobre un archivo sucio.**

No es una recomendación: es lo que hace posible calcular `commit`, que es el commit en el que el fragmento quedó con el contenido aprobado. Ese commit no existe si el fragmento no está commiteado.

Y es lo que garantiza que lo aprobado sea recuperable. Sin commit no hay `git show <commit>:<file>`, y sin eso `check` no puede recuperar el texto aceptado — que es lo que distingue `EXPANDED` y `REANCHORED` de un `ALTERED` genérico.

## Quiénes aprobaron

`agree` es el set de quienes aprobaron **exactamente estos valores**. Como los valores direccionan por contenido, *"estar de acuerdo"* no es ambiguo: es haber aprobado este hash y esta ubicación y no otros.

**Por endpoint y local, nunca copiado.** En un endpoint estructural están los que aprobaron ese fragmento; en un endpoint `path` o `repo`, los que aprobaron **esa copia**. Quién aprobó del otro lado de la cadena es un hecho de la otra capa, y traerlo acá sería atribuir mal. Los dos endpoints de un bilink pueden tener listas distintas, y es lo normal.

**Quien escribe el `accepted` entra solo.** Si no, el campo significaría "los que *además* aprobaron", que es otra cosa.

**Es un set: ordenado y único al serializar.** Un duplicado o un reordenamiento producirían un diff que no dice nada.

### Un nombre por línea, y por eso no guarda su commit

Se escribe en bloque, nunca en flow:

```yaml
agree:
  - ana
  - pablo
```

**Porque `git blame` sólo puede atribuir una línea a un commit.** En una sola línea, `- ana, - pablo` colapsa N actos distintos en un lugar, y blame devuelve el commit del último que la tocó: el primer aprobador se pierde. Con un nombre por línea, cada endoso queda atribuible por separado — autor, fecha y firma — y de ahí sale que **el campo no necesite guardar el commit de nadie**: git ya lo sabe, y con `blame` se llega en un salto.

Es la misma razón por la que `commit` no está en `accepted`: lo que git puede contestar no se duplica en el archivo.

**Y el orden es alfabético, no cronológico.** Cronológico sería tentador —diría quién aprobó primero— pero después de una unión el orden dependería del orden del merge, que no es un hecho sobre nada: dos repos con el mismo set escribirían archivos distintos. El alfabético es canónico, y no pierde nada: quién fue primero lo dice la fecha del commit, que blame entrega igual.

Insertar un nombre más arriba no rompe la atribución de los demás: blame sigue el contenido de la línea, no su número.

### Es lo que le da algo que escribir al segundo

Sin `agree`, aprobar algo que ya está `OK` no cambia ningún byte: no hay diff, no hay commit, no hay firma, no hay registro. **El endoso explícito era inexpresable.** Con la lista, es un diff de una línea sobre un commit firmado.

Por eso el campo no duplica la historia: **es lo que la crea.** Parece derivable —caminar el log del bilink juntando autores— y sin él no habría log del cual derivarlo.

### Y sale de la convergencia byte a byte

Dos personas que aceptan el mismo contenido escriben los mismos hashes y **listas distintas**, así que `agree` sale de la fila *"ya coincidía"* de la que depende [`adopt`](../commands/adopt.md).

La diferencia con un campo que no puede estar acá —`commit`, que el mismo contenido aceptado en dos ramas resuelve a dos valores sin forma de elegir— es que **acá la resolución es correcta y única: unión**. `adopt` es un merge campo por campo, así que une los dos sets sin preguntarle a nadie. Cuesta una fila en su tabla, no una suposición rota.

### Sin firma verificada es atribución, no atestación

`pablo` es texto, y **cualquiera puede escribir `agree: [ana]` sin que Ana se entere**. Un campo que parece atestación y es la afirmación de un tercero es peor que no tenerlo.

Vale lo que [`ref.md`](ref.md#autoría-atestación-y-autorización) ya dice: *"el autor de git es auto-declarado; lo que constituye atestación es la firma, no el campo."* Lo que convierte la lista en algo en que apoyarse es verificar que el commit que agregó `- ana` esté firmado por la clave de Ana — una allowlist en el `pre-receive`. Hasta entonces, `agree` se lee como lo que es: **una declaración local, sin más peso que quien la escribió.**

## Los valores son deterministas; la lista se une

Dos personas que aceptan el mismo fragmento en el mismo estado escriben **los mismos valores**. `accepted` no lleva cuándo se aceptó ni desde qué HEAD: eso es del commit de la ref.

De ahí que `commit` sea el commit **del contenido** y no el HEAD de quien acepta. Con el HEAD, el mismo acto daba distinto según quién y cuándo lo hiciera, y el valor no describía nada del fragmento.

Lo único que no converge es `agree`, y a propósito: **es el campo cuyo contenido *es* la diferencia entre las personas.** Reconcilia por unión, que es la única regla posible y no requiere que nadie decida.

## `--place` y `--content`

Por defecto `accept` aprueba las dos dimensiones. Los dos flags permiten aprobar una sola:

```sh
bilinker accept <uuid>.<N>              # ubicación y contenido
bilinker accept <uuid>.<N> --place      # sólo la ubicación
bilinker accept <uuid>.<N> --content    # sólo el contenido
```

Aceptar la ubicación de un fragmento cuyo contenido no se revisó es una situación real —el archivo se renombró y el código no cambió, o cambió y hay que mirarlo aparte— y sin los flags habría que aprobar de más para poder avanzar.

## Un endpoint layer copia las dos

El `accepted` de un endpoint layer o repo copia los **dos** valores del endpoint estructural del bilink adyacente: su `link` y su `hash`. **`agree` no se copia**: es de esta capa, no del vecino.

Cada uno cambia por una sola razón y por ninguna otra: `hash` cuando cambia el contenido publicado, `link` cuando cambia su ubicación aprobada. Los dos son inmunes a etiquetas, comentarios y reordenamientos del archivo vecino — que es la razón de copiar los valores y no hashear el archivo entero.

## Lo que no se acepta a ciegas

Un `accept .` sobre una capa recién cambiada fabrica aprobaciones que nadie miró. Cada estado no-OK es un puntero al fragmento que hay que revisar, y **el inventario de trabajo de un cambio es esa lista**; vaciarla para dejar el árbol verde es tirar justamente lo que hacía falta.

`accept .` existe para el caso en que ya se revisó todo, no para el caso en que no se revisó nada.

## Invariantes

1. `accepted` está completo o ausente. Su ausencia es `PENDING`.
2. `accept` es el único que escribe `accepted`.
3. `accept` falla si el fragmento no está commiteado.
4. Aceptar el mismo fragmento en el mismo estado produce siempre los mismos **valores** de `accepted`. `agree` es la excepción, y reconcilia por unión.
5. `accepted.link` de un endpoint layer o repo es una copia opaca del id ajeno: se compara, no se resuelve.
6. Aprobar una ubicación y aprobar un contenido son actos separables.
7. `accepted.agree` es un set, incluye a quien escribió el `accepted`, y es local: nunca se copia de un vecino.
8. `agree` no participa de ninguna comparación de estado ni de ningún hash. `OK` no depende de cuántos aprobaron.
