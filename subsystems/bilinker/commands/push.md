# Especificación: comando `bilinker push`

## Propósito

Publica `refs/bilink/<branch>` en el remoto.

## Firma

```
bilinker push [<branch>] [--remote <nombre>]
```

| Argumento | Descripción |
|---|---|
| `<branch>` | Rama cuya ref publicar. Default: la rama actual. |
| `--remote` | A cuál publicar. Default: el único que haya, o `origin`. |

## Ninguna interacción con `refs/bilink/*` se hace tipeando git

La ref vive fuera de `refs/heads/`, así que `git push` a secas **no la empuja**: hay que nombrarla con un refspec.

Y hacer que alguien tipee un refspec es exactamente lo que [la ref](../concepts/ref.md#fuera-de-refsheads) evita al decir que *los refspecs los pone bilinker; nadie los tipea*. Escribir `refs/bilink/main:refs/bilink/main` una sola vez ya es una fuga del namespace hacia afuera; a la segunda ya es una convención que alguien copia mal, con la rama de otro adentro.

**El refspec lo arma bilinker**, con el mismo `+` que [`init`](init.md) deja en el config. No es opcional: un clon superficial no tiene la historia para probar que el tip nuevo desciende del viejo, y sin el `+` git rechaza como non-fast-forward un avance legítimo. Es seguro porque **la ref es append-only por diseño** — nunca se rebasea ni se cherry-pickea, así que un avance siempre lo es de verdad.

## Publica la ref, no la rama

Qué commits de tu proyecto salen a la luz es una decisión tuya, y la tomás con `git push` cuando quieras. `bilinker push` no la toma por vos.

Son dos cosas que a veces van juntas y no siempre: se puede publicar la ref de una rama que ya está pusheada, y se puede pushear la rama sin publicar todavía las decisiones.

## No es una flag de `sync`

Alinear la ref con la rama y publicarla son **dos actos**, y quien trabaja en una rama propia hace el primero muchas veces antes del segundo.

Un `sync --push` convertiría un comando local en uno que *a veces* habla con la red, y "a veces" es lo peor que puede ser una operación de red: deja de poder correrse sin pensar. Ver [`sync`](sync.md) § "No publica".

## Con varios remotos

Con uno solo, se usa ése. Con varios, gana `origin` si está; si no, se pide elegir:

```
$ bilinker push
error: hay más de un remoto (upstream, fork) y ninguno es `origin`.
  Elegir con `bilinker push --remote <nombre>`.
```

Adivinar sería adivinar **a quién le publicás**, que es la clase de cosa que no se adivina.

## Salida

```
$ bilinker push
publicado: refs/bilink/main @ b1e3f55 → origin
```

Cuando el remoto ya lo tenía:

```
$ bilinker push
refs/bilink/main ya estaba en origin @ b1e3f55
```

No es un error: publicar dos veces es normal, y decir que no se movió nada es más útil que no decir nada.

## Códigos de salida

| Código | Condición |
|---|---|
| 0 | Publicado, o el remoto ya lo tenía. |
| 1 | La rama no tiene ref; o no hay remoto; o hay varios y ninguno es `origin`. |
| 1 | El remoto rechazó el push. |

**Un rechazo no se fuerza.** La ref es append-only, así que un non-fast-forward significa que alguien reescribió una historia que no se reescribe — es algo que mirar, no un caso a resolver con `--force`.

## Propiedades garantizadas

- **Nadie tipea un refspec**: lo arma bilinker.
- **Publica la ref y nada más**: la rama del proyecto no se toca.
- **Idempotente**: publicar dos veces no es un error y la segunda no mueve nada.
- **No fuerza**: un rechazo se reporta.
