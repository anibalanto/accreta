---
title: Ejecutar el corte a la ref (`005`) en los 4 repos
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T22:34:54Z
parent: c
---

# Ejecutar el corte a la ref (`005`) en los 4 repos

El segundo corte, y sólo después de que el `004` esté validado entero **y de que [`1a`](1a.task.md) pruebe la secuencia contra la forma real** — varios repos, no un directorio.

```
1. UN commit que saca .bilink/ del índice de la rama   → X   (pushear antes de seguir)
2. bilinker init                                      (exclude + refspec)
3. bilinker track <branch>                            → ●0, padre X
4. Ledger: 005
```

El paso 1 es `git rm --cached -r .bilink/` más commit: **los archivos se quedan en disco**, así que no hay que restaurarlos para commitearlos a la ref.

> Son tres pasos y no cinco. El ADR escribía un `git update-ref refs/bilink/<branch> X` y un commit aparte; al implementar la ref quedó claro que los dos son un solo `bilinker track`, que es exactamente el caso "ningún candidato califica". Y el `update-ref` estorbaba: dejaría la ref apuntando a un commit del proyecto, y `track` tendría que tratarlo como propio. Ver la enmienda en [ADR-0004](../subsystems/bilinker/.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md) § `005`.

**El paso 2 no materializa nada**, y es lo que hace que el corte no necesite setup propio: con `.bilink/` en el árbol y sin `head`, `init` no sabe de dónde salió, así que lo deja intacto y se limita al exclude y al refspec.

La regla que no se puede saltear: la ref nace de un commit donde `.bilink/` **no está en el árbol**. Es lo que hace disjuntos los dos lados de ahí en adelante; sin ese orden, el primer merge produce un modify/delete masivo — o peor, borra en silencio los bilinks que no cambiaron.

Cuatro veces, una por repo, y quedan cuatro `refs/bilink/main`.

## Corrido

Los cuatro, con [`scripts/corte-005.sh`](../subsystems/bilinker/.stratum/impl/scripts/corte-005.sh) de la task [`1a`](1a.task.md), en orden de menor a mayor riesgo:

| Repo | X | ●0 | bilinks en la ref |
|---|---|---|---|
| `stratum/.stratum/impl` | `f1fe0bf` | `10f6f3b` | 17 |
| `lattice/.stratum/impl` | `95290fd` | `fed60e9` | 61 |
| `bilinker/.stratum/impl` | `6402b77` | `caf0692` | 267 |
| `accreta` | `2898ec7` | `a536ca8` | 356 |

Accreta lleva sus **tres capas en una sola ref** —raíz, `subsystems/lattice` y `subsystems/stratum`— porque la ref es por repo. Los tres `.stratum/impl` tienen la suya, y ninguna se tragó nada del otro.

Después del corte: ninguna rama contiene `.bilink/`, `git branch -a` no lista ninguna ref, `git status` limpio en los cuatro, y las **seis capas** en `all clean`.

## Lo que falta

**Pushear.** Los cuatro repos tienen el corte local y nada salió a GitHub:

```
git push && git push origin 'refs/bilink/main:refs/bilink/main'
```

Hasta que eso pase, revertir es `git revert` del commit del corte más `git update-ref -d refs/bilink/main`. Después ya no.
