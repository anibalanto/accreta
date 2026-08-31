# Especificación: comando `bilinker pull`

## Propósito

Trae las decisiones que otro aceptó **en la misma rama** y las une con las propias.

Es el [caso 3.b](../concepts/ref.md#el-tipo-3-tiene-dos-casos-y-también-se-distinguen-solos) de la taxonomía: sincronización de decisiones donde los dos lados cuelgan de la misma absorción. [`adopt`](adopt.md) cubre el 3.a —otra rama, otra absorción— y no aplica acá, porque no hay otra rama que nombrar.

## Firma

```
bilinker pull [<remoto>] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `<remoto>` | De cuál traer. Default: el único que haya, o `origin`. |
| `--dry-run` | Muestra qué entraría sin escribir nada. |

## Por qué es un verbo propio y no `adopt` sin argumento

`adopt <rama>` nombra **la rama de la que se trae**, y acá no hay ninguna: la fuente es la copia que el remoto tiene de **esta misma** ref. Un `adopt` sin argumento sería el mismo comando significando algo distinto según le pasen o no un nombre, y su primera verificación —*"no hay nada que adoptar de uno mismo"*— tendría que aprender una excepción.

**Y `pull` es el nombre que ya tiene el acto.** Es lo que git llama traer del remoto y unir, y es el contrapié exacto del [`push`](push.md) que fue rechazado: el rechazo puede decir *"correr `bilinker pull`"*, que es la frase más corta que resuelve el problema de quien lo lee.

## `push` no lo hace solo

Un `push` que sincroniza al ser rechazado es cómodo y **esconde una decisión**: unir dos historias de aceptaciones puede tener conflicto, y resolverlo es mirar dos decisiones humanas incompatibles.

Así que `push` reporta y **nombra el comando que corresponde**, igual que hoy nombra `--remote` cuando hay varios.

## Un non-fast-forward tiene dos causas, y sólo una es un problema

[`push.md`](push.md) decía que un rechazo significa que alguien reescribió la ref. **Es falso en el caso más común**: Ana y Luis los dos *agregaron*, y nadie reescribió nada.

| Causa | Cómo se distingue | Qué corresponde |
|---|---|---|
| las dos partes agregaron | los dos tips descienden de una base de merge común | **`bilinker pull`** |
| alguien reescribió | no hay base de merge, o el tip viejo no es alcanzable | mirar, no forzar |

`git merge-base` las separa, así que no hace falta adivinar — y confundirlas es lo peor que puede hacer el mensaje, porque manda a mirar un incidente donde no hubo ninguno.

## El commit

Un solo commit sobre la ref, con **dos padres, los dos de la ref**: el tip propio y el que trajo el fetch.

```
refs/bilink/main (remoto):  ●1 ─── ●2 ──────╮   2º padre: lo que aceptó el otro
                            │               │
refs/bilink/main (local):   ●1 ─── ●3 ───── ●4
                            └ la misma absorción arriba de los dos
```

**El árbol de código no se elige.** Los dos lados cuelgan de la misma absorción, así que describen el mismo código: sale del primer padre y no se fusiona. Es más simple que el `●c` de `adopt`, donde cada rama absorbió lo suyo.

Y como la fusión queda confinada a `.bilink/`, el commit no toca ningún archivo del proyecto.

## Qué es conflicto y qué no

El merge es **a tres puntas y campo por campo**, el mismo de [`adopt`](adopt.md#qué-se-compara):

| | Qué pasa |
|---|---|
| endpoints distintos | **unión**, sin conflicto: cada uno decidió sobre algo que el otro no tocó |
| el mismo endpoint, `accepted` idéntico | ya coincidía — los valores direccionan por contenido |
| el mismo endpoint, `accepted` distinto | **conflicto**: dos decisiones humanas incompatibles |
| el mismo endpoint, mismos valores y `agree` distinto | **unión**, y el resultado dice algo verdadero que antes no se podía decir: **los dos aprobaron** |

La última fila es la que hace que el caso más frecuente se cierre **sin criterio humano**. Si Ana y Luis aceptaron lo mismo, los valores coinciden byte a byte —`link`, `hash` y `hash_ast` direccionan por contenido, y `commit` no está en `accepted` justamente porque nunca converge— y lo único que difiere es [`agree`](../concepts/accept.md#quiénes-aprobaron), cuya reconciliación es una **regla**, no una decisión.

**Con un conflicto no se escribe nada**, ni siquiera el fetch se deshace: resolver un `accepted` en conflicto es elegir una de dos decisiones, y eso es [`accept`](accept.md), con una persona mirando.

## El fetch va a un namespace aparte

```
refs/bilink-remote/<remoto>/<branch>
```

No a `refs/bilink/<branch>`, que es la ref local y es justo la que no se quiere pisar — el [refspec sin `+`](init.md) existe para que ese fetch falle en vez de pisarla.

Ahí sí se trae forzando, y no hay nada que proteger: es una copia de lectura del remoto, se descarta y se vuelve a traer.

**Y el fetch va con `--refmap=`**, que no es un detalle: sin él, `git fetch <remoto> <refspec>` aplica *además* los refspecs configurados, y el de [`init`](init.md) va sin `+` justo para fallar cuando el remoto divergió. O sea que el fetch de `pull` fallaría exactamente en el único caso en que `pull` existe. El refmap vacío apaga esa parte y deja sólo el refspec que se pidió. **Es la misma distinción que git hace con `refs/remotes/`**, y la razón por la que la ref propia *no* se mapea ahí sigue valiendo: no hay flujo donde alguien quiera comparar sin traer. Acá se trae **para** unir, que es otra cosa.

## Salida

```
$ bilinker pull
base ●1 · 3 aceptaciones de origin en ●1..●2

  entra limpio  7f3d8e9a.0  contenido   ← sólo el otro lo tocó
  entra limpio  3a4b5c6d.1  ubicación
  ya coincidía  8e9f0a1b.0  contenido   ← los dos aceptaron lo mismo
                8e9f0a1b.0  agree: ana, luis

commit:  refs/bilink/main  ●3 → ●4   (2 endpoints)
```

Con conflicto:

```
$ bilinker pull
base ●1 · 3 aceptaciones de origin en ●1..●2

  conflicto     7f3d8e9a.0  contenido
                  acá:   c00e0760…
                  allá:  40acd80f…

no se escribió nada. Resolver aceptando uno de los dos: `bilinker accept 7f3d8e9a.0`
```

## Códigos de salida

| Código | Condición |
|---|---|
| 0 | Unido, o no había nada que traer. |
| 1 | Hubo conflicto, y no se escribió nada. |
| 1 | No hay remoto, o hay varios y ninguno es `origin`. |

## Invariantes

- **Ninguna aceptación se pierde**: los dos commits siguen alcanzables desde el resultado, porque son sus dos padres.
- **El árbol de código no cambia**: los dos lados cuelgan de la misma absorción.
- **Todo o nada**: con un conflicto no se escribe ningún commit.
- **No decide**: un `accepted` en conflicto lo resuelve una persona con `accept`.
- La ref del remoto se trae a `refs/bilink-remote/`, nunca sobre la local.
