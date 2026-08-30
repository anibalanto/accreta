# Especificación: comando `bilinker adopt`

## Propósito

Trae a la ref de esta rama las **decisiones** que otra rama aceptó, sin llevarse ninguna de las mías para allá.

Es lo que hace falta después de un rebase: rebasear sobre `main` metió el código de `main` en la rama, y si `main` aceptó algo sobre ese código, los bilinks heredados no lo tienen y van a reportar drift que `main` ya resolvió.

## Firma

```
bilinker adopt <rama> [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `<rama>` | La rama del proyecto de la que traer decisiones. `origin/main` y `main` son lo mismo. |
| `--dry-run` | Calcula y reporta exactamente lo mismo, sin escribir un solo archivo. |

Se nombra la **rama del proyecto**, no su ref de bilinks: `bilinker adopt origin/main`, y la traducción a `refs/bilink/main` la hace la herramienta. Una sola fuente de verdad, y nadie tipeando namespaces de refs.

## No se llama `merge` a propósito

En este diseño *merge* ya nombra una cosa muy precisa y estructural: un commit de la ref que absorbe un commit del proyecto como segundo padre. Es la palabra con la que están escritas [la invariante de fidelidad](../concepts/ref.md#la-invariante-de-fidelidad) y la evolución de la ref.

`adopt` dice lo que pasa y es asimétrico, que es la verdad: las decisiones del vecino entran acá, y ninguna de las mías va para allá. Que su implementación produzca un commit de merge se sigue pudiendo decir sin ambigüedad, porque el verbo y el mecanismo dejaron de compartir nombre.

## Son dos commits, y por qué

```
refs/bilink/main:           ●2 ── ●3 ─────────────────────╮
                                                          │    2º padre de ●c: trae ●2..●3,
feature/x (rebaseada):            E ─── F' ─── G'         │    las decisiones de main
                                                 ╲        │
refs/bilink/feature/x:      ●a ────────────────── ●b ──── ●c
```

**`●b` — absorber `G'`.** No tiene nada de especial: es la absorción de siempre, y no hay que pedirla. `adopt` escribe un commit sobre la ref, así que [la precondición de fidelidad](../concepts/ref.md#la-invariante-de-fidelidad) lo obliga a absorber `G'` antes. Es obligatorio en cualquier caso, porque si no la ref sigue afirmando que su código es `G`, un commit que el rebase abandonó.

**`●c` — traer `●2..●3`.** Las decisiones del vecino, como segundo padre.

**Dos commits y no uno de tres padres.** Con `(●a, G', ●3)` en un solo commit, `--first-parent` mostraría una línea para dos cosas distintas, y la invariante de fidelidad necesitaría una regla acompañante para el tercer padre. Dos commits dejan las dos reglas intactas y no cuestan nada.

Quien sólo quiere ponerse al día sin traer nada del vecino corre [`sync`](sync.md), que escribe `●b` y para ahí.

## La base del merge a tres puntas sale gratis

Es `●2`, sin que nadie la calcule: es la base de merge real entre `●a` y `●3`, porque [`track`](track.md) puso `●2` como **primer padre** de `●a` en vez de copiar archivos.

Ésa es la razón de fondo de la forma que `track` tiene, y recién acá se cobra.

## Qué se compara

`accepted` son campos con nombre, por endpoint, así que el merge a tres puntas los compara **de a uno**. Las cuatro filas son las únicas posibles, y salen del formato sin nada agregado:

```
$ bilinker adopt origin/main --dry-run
base ●2 · 4 aceptaciones de main en ●2..●3

entra limpio     7f3d8e9a.0   contenido    Luis
                 a3f9c821.1   ubicación    Ana
ya coincidía     c1a2b3c4.0   contenido    — mismo valor de los dos lados
conflicto        d5e6f7a8.0   contenido    main 838ea0a4…  ·  acá 9211a4f3…
```

| Fila | Qué pasó | Qué se escribe |
|---|---|---|
| **entra limpio** | el vecino cambió el campo y acá nadie lo tocó desde la base | el valor del vecino |
| **ya coincidía** | los dos lados escribieron el mismo valor | nada — ya está |
| **conflicto** | los dos lados lo cambiaron, a valores distintos | nada; se enumera y se para |
| *(sin fila)* | acá se cambió y el vecino no | nada — mis decisiones no se pisan |

**Que la fila "ya coincidía" exista es la convergencia** que el direccionamiento por contenido hace posible: dos personas que aceptan el mismo contenido en HEADs distintos escriben los mismos valores. Por eso el caso común de un rebase no conflictúa nada.

Y la fila que no existe es la que dice que `adopt` es asimétrico: un campo que sólo yo cambié se queda como está, y no viaja para el otro lado.

## Los conflictos paran el comando

No se escribe un commit a medias con marcadores de conflicto adentro de un YAML. Un `accepted` en conflicto son dos decisiones humanas incompatibles sobre el mismo fragmento, y resolverlo es aceptar una de las dos — que es [`accept`](accept.md), con una persona mirando.

```
$ bilinker adopt origin/main
base ●2 · 4 aceptaciones de main en ●2..●3

conflicto        d5e6f7a8.0   contenido    main 838ea0a4…  ·  acá 9211a4f3…

1 conflicto: no se escribió nada.
  Revisar con `bilinker get d5e6f7a8.0 --diff` y decidir con `bilinker accept`.
```

Absorber `G'` tampoco ocurre: si el comando no puede terminar, no deja la ref a mitad de camino. Para ponerse al día sin traer nada, `sync`.

## `--dry-run` no toca la red

Es lo que hace que reporte **exactamente lo mismo** que el comando real: si `adopt` fetcheara, el dry-run reportaría sobre otros datos que la corrida de verdad. La ref del vecino se trae con `git fetch`, que [`init`](init.md) dejó configurado, y `adopt` opera sobre lo que ya está.

Mismo contrato que el `--dry-run` de [`migrate`](migrate.md): calcula y reporta lo mismo, sin escribir un solo archivo.

## Ver qué decidió el vecino antes de traerlo

No necesita un verbo propio. Son dos preguntas y las dos ya tienen respuesta:

- *¿Qué actos hubo del otro lado?* — `bilinker log --first-parent <rama> ^<mi-rama>`, que es [el registro de decisiones](../concepts/ref.md#la-correspondencia-con-el-proyecto-es-el-segundo-padre) acotado al rango que falta.
- *¿Qué le harían a mis bilinks?* — `bilinker adopt <rama> --dry-run`.

## Cuándo no hay nada que adoptar

Si la ref del vecino no avanzó desde la base, `adopt` no escribe ningún commit y lo dice. [`status`](status.md) es lo que avisa cuando sí avanzó.

```
$ bilinker adopt origin/main
refs/bilink/main no avanzó desde ●2 — nada que adoptar
```

## Salida

```
$ bilinker adopt origin/main
base ●2 · 4 aceptaciones de main en ●2..●3

entra limpio     7f3d8e9a.0   contenido    Luis
                 a3f9c821.1   ubicación    Ana
ya coincidía     c1a2b3c4.0   contenido

absorbe:  feature/x → 8f1a2b3   (●b)
commit:   refs/bilink/feature/x  ●b → ●c   (2 endpoints)
```

## Códigos de salida

| Código | Condición |
|---|---|
| 0 | Adoptado, o no había nada que adoptar. |
| 1 | Hay conflictos. No se escribió nada. |
| 1 | La rama nombrada no tiene ref; o `HEAD` está desacoplado; o la disyunción falló. |

## Propiedades garantizadas

- **Asimétrico**: ninguna decisión de esta rama viaja a la del vecino.
- **No pisa decisiones propias**: un campo que sólo se cambió acá se queda como está.
- **Todo o nada**: con un conflicto no se escribe ningún commit, ni siquiera el de absorción.
- **`--dry-run` no escribe y no habla con la red**, y por eso reporta lo mismo que la corrida real.
- Los conflictos se enumeran por endpoint y por dimensión, nunca se fusionan a mano dentro de un archivo.
