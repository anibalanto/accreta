# Especificación: comando `bilinker track`

## Propósito

Crea `refs/bilink/<branch>` para una rama que no la tiene. Sin él, empezar a seguir una rama nueva deja todos los endpoints en `PENDING`.

Compone directo con [`architecture.md`](../architecture.md) § "Implementaciones alternativas por branch": la variante `feature/X` de las specs tiene sus bilinks en `refs/bilink/feature/X`, y `track` es lo que contesta qué hereda al abrirse.

## Firma

```
bilinker track <branch> [--from <rama>]
```

| Argumento | Descripción |
|---|---|
| `<branch>` | La rama del proyecto a trackear. |
| `--from <rama>` | Heredar de la ref de esa rama, en vez de buscarla. |

`--from` nombra la **rama del proyecto**, no su ref de bilinks: `bilinker track feature/x --from main`. Una sola fuente de verdad, y nadie tipeando namespaces de refs.

## Qué hace

```
1. Con --from explícito: se usa y listo.
2. Sin él, buscar entre refs/bilink/* el commit M tal que:
     · M es un commit propio de alguna refs/bilink/<X>, en su cadena de primeros padres
     · su segundo padre P cumple  git merge-base --is-ancestor P <branch>
     · P es el más nuevo de los que cumplen
3. Crear refs/bilink/<branch>:  primer padre M   (hereda los bilinks)
                                segundo padre    tip de <branch>  (absorbe el código)
```

`feature/x` sale de `D`, y la ref de `main` ya absorbió `E`:

```
main:                  X ─── B ─── C ─── D ─── E
                       │           ╲     ╲     ╲
refs/bilink/main:      ●0 ───────── ●1 ── ●2 ── ●3

feature/x:                               D ─── F ─── G
                                                     ╲
refs/bilink/feature/x:                    ●2 ──────── ●a
```

`●a` es un commit de la misma forma que cualquier otro de una ref, sólo que sus dos padres vienen de lugares distintos: el **primero** es `●2`, de donde hereda los bilinks; el **segundo** es `G`, de donde saca el código. Su árbol de código es el de `G` y su `.bilink/` es el de `●2`, así que **su diff contra el primer padre es vacío** — `track` no decide nada, igual que [`sync`](sync.md).

Y el `.bilink/` heredado sale del árbol de `M`, no del árbol de trabajo: ir por el árbol de trabajo obligaría a materializar antes de saber si el commit se puede escribir.

## `●2` y no `●3`

Los candidatos se miran por su segundo padre: el de `●1` es `C`, el de `●2` es `D`, el de `●3` es `E`. `C` y `D` siguen siendo ancestros de `feature/x`; `E` no, porque la rama se bifurcó antes. `D` es el más nuevo de los que califican, y por eso gana `●2`.

Heredar de `●3` traería bilinks que describen código que `feature/x` no tiene.

El test `--is-ancestor` es lo que traduce *"la última versión de los bilinks accesible"* a algo exacto. En el caso común —la rama sale del tip de una rama trackeada y la ref está al día— el `P` más nuevo es el commit base de la rama nueva y la búsqueda termina en la primera iteración. Los tres casos donde el filtro deja de ser cosmético:

| Situación | Qué pasa sin el test | Qué hace `track` |
|---|---|---|
| La ref va **adelante** del punto de fork | hereda bilinks que apuntan a código que la rama no tiene → `UNRESOLVED` y `ALTERED` falsos desde el minuto cero | toma el `M` cuyo `P` es `D` |
| **Ningún `P` califica** | elegiría algo parecido | lo dice y crea la ref desde cero |
| **Califican varias refs** | adivina | exige `--from` |

## La búsqueda va de la ref hacia el proyecto, nunca al revés

Ningún commit del proyecto tiene un merge a `refs/bilink/*` — la relación es exactamente la inversa. Buscar en los ancestros de la rama nueva un merge hacia la ref sólo puede encontrar el bug que la verificación de disyunción detecta.

Y la cadena de candidatos son los commits **propios** de la ref, no todo lo que `git rev-list --first-parent` devuelve: al llegar al corte ese recorrido sigue por la historia del proyecto. Ver [la ref](../concepts/ref.md#la-correspondencia-con-el-proyecto-es-el-segundo-padre) — sin el freno, el commit más viejo del proyecto es ancestro de cualquier rama, así que siempre califica y siempre gana.

## Sin candidato, la ref nace desde cero — y eso es el corte

Cuando ningún `P` califica —la rama sale de antes del corte, o de una línea nunca trackeada— `track` lo dice y crea la ref con **el `.bilink/` del árbol de trabajo** y el tip de la rama como padre único.

Ése es exactamente el paso 4 del corte `005`:

```
1. UN commit que saca .bilink/ del índice de la rama   → X   (pushear antes de seguir)
2. bilinker init  (exclude + refspec)
3. bilinker track <branch>                             → ●0, padre X
4. Ledger: 005
```

El corte no necesita un comando propio ni un `git update-ref` a mano: **es el caso "no hay de quién heredar" de `track`**, que es lo que el corte literalmente es. El paso 2 del ADR —`git update-ref refs/bilink/<branch> X`— se cae: `track` crea la ref, y crearla apuntando a `X` antes obligaría a `track` a tratar un commit del proyecto como si fuera suyo.

Un camino menos que mantener, y uno que se ejercita cada vez que alguien abre una rama.

**El corte es el único commit de la ref sin ningún commit del proyecto absorbido por debajo**: nace de `X` como padre único, y ahí la fidelidad se lee contra `X` mismo.

## No es `sync`

[`sync`](sync.md) no puede escribir el corte, y es correcto que no pueda: cuando no hay nada que absorber `sync` no escribe ningún commit, porque un merge con el mismo segundo padre y el mismo `.bilink/` no dice nada nuevo. En el corte no hay nada que absorber —`X` ya es el tip— y sin embargo hay un commit que escribir, porque el `.bilink/` del árbol todavía no está en ninguna parte.

Son dos actos distintos: `sync` alinea una ref que existe, `track` crea la que no.

## Una rama que ya tiene ref es un error

No un no-op silencioso: quien tipea `track` sobre una rama trackeada o se equivocó de rama, o quería `sync`.

```
$ bilinker track main
error: refs/bilink/main ya existe.
  Para ponerla al día con la rama, `bilinker sync`.
```

## El mensaje

Los dos casos tienen verbo propio en [la gramática de la ref](../concepts/ref.md#el-mensaje-es-el-comando), y los dos nombran la rama que nace:

```
track feature/x: hereda de 9c1f0ab sobre 4e77d20
corte main: los bilinks pasan a refs/bilink/main
```

Que el corte sea un verbo aparte y no `track` sin candidato es lo mismo que la [tabla de tipos](../concepts/ref.md#un-commit-hace-una-cosa) ya dice: `track` con herencia es un commit de dos padres que trae código, y el corte es de padre único. Distinguirlos en el mensaje evita que un verificador tenga que mirar los padres para saber cuál esperaba.

## Salida

```
$ bilinker track feature/x
hereda:  9c1f0ab sobre 4e77d20
commit:  refs/bilink/feature/x @ 7a2d4e8
árbol:   64 archivo(s)
```

Sin candidato:

```
$ bilinker track main
ningún commit de refs/bilink/* califica: la ref nace desde cero.
commit:  refs/bilink/main @ 0af3c12
árbol:   63 archivo(s)
```

Con varios candidatos:

```
$ bilinker track feature/x
error: el punto de fork de feature/x es ancestro de main y rc-2.35.
  Elegir con `bilinker track feature/x --from <rama>`.
```

## Códigos de salida

| Código | Condición |
|---|---|
| 0 | Ref creada. |
| 1 | La rama ya tiene ref; o califican varias y falta `--from`; o la rama no existe. |
| 1 | La verificación de disyunción falló sobre el tip de la rama. |

## Propiedades garantizadas

- **No decide nada**: el diff del commit contra su primer padre es vacío.
- **No absorbe de más**: el segundo padre es el tip de la rama nombrada, y nada más.
- **No pisa trabajo sin commitear**: materializar pasa por la misma guarda que el resto.
- Si el commit no se puede escribir, el `.bilink/` del árbol queda como estaba.
