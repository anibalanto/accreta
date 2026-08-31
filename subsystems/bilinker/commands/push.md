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

**El refspec lo arma bilinker**, y el del push va **con `+`** — a diferencia del de fetch, que [`init`](init.md) escribe sin él. No es opcional: un clon superficial no tiene la historia para probar que el tip nuevo desciende del viejo, y sin el `+` git rechaza como non-fast-forward un avance legítimo.

Y la diferencia entre los dos es **quién te protege**:

| | Qué saltea el `+` | Quién verifica igual |
|---|---|---|
| fetch | la verificación sobre **tu ref local** | nadie |
| push | la verificación **del cliente** | el servidor, con `denyNonFastForwards` y [`verify-ref`](verify-ref.md) |

Antes esto se justificaba diciendo que *"es seguro porque la ref es append-only por diseño"*, que era una **suposición**. Ahora es una condición chequeada del otro lado, así que el `+` del push no es un permiso: es sacarle al cliente una verificación que el servidor hace mejor.

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

**Un rechazo no se fuerza, y tiene dos causas.** Decir que un non-fast-forward significa que alguien reescribió la ref es falso en el caso más común: dos personas que aceptan en la misma rama los dos **agregaron**, y nadie reescribió nada.

| Causa | Cómo se distingue | Qué corresponde |
|---|---|---|
| las dos partes agregaron | los dos tips descienden de una base de merge común | [`bilinker pull`](pull.md) |
| alguien reescribió | no hay base de merge, o el tip viejo no es alcanzable | mirar, no forzar |

`git merge-base` las separa, así que `push` no adivina: **dice cuál de las dos fue**. Confundirlas manda a mirar un incidente donde no hubo ninguno, que es peor que no decir nada.

```
$ bilinker push
error: origin tiene refs/bilink/main adelantada, y las dos historias agregaron.
  Nadie reescribió nada: base de merge en 0af3c12.
  Unir con `bilinker pull`.
```

Ninguna de las dos se resuelve con `--force`, y por eso no existe.

## Propiedades garantizadas

- **Nadie tipea un refspec**: lo arma bilinker.
- **Publica la ref y nada más**: la rama del proyecto no se toca.
- **Idempotente**: publicar dos veces no es un error y la segunda no mueve nada.
- **No fuerza**: un rechazo se reporta, y se dice si fue divergencia o reescritura.
- **No sincroniza**: unir dos historias de aceptaciones puede tener conflicto, y esconder esa decisión adentro de un push sería tomarla por quien la tiene que tomar.
