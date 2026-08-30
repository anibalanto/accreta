---
title: Ejecutar el corte a la ref (`005`) en los 4 repos
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
parent: c
---

# Ejecutar el corte a la ref (`005`) en los 4 repos

El segundo corte, y sólo después de que el `004` esté validado entero.

```
1. UN commit que saca .bilink/ del índice de la rama   → X   (pushear antes de seguir)
2. git update-ref refs/bilink/<branch> X
3. bilinker init  (exclude + refspec)
4. Commit sobre refs/bilink/<branch>: agrega .bilink/   → ●0, padre X
5. Ledger: 005
```

El paso 1 es `git rm --cached -r .bilink/` más commit: **los archivos se quedan en disco**, así que no hay que restaurarlos para commitearlos a la ref.

La regla que no se puede saltear: la ref nace de un commit donde `.bilink/` **no está en el árbol**. Es lo que hace disjuntos los dos lados de ahí en adelante; sin ese orden, el primer merge produce un modify/delete masivo — o peor, borra en silencio los bilinks que no cambiaron.

Cuatro veces, una por repo, y quedan cuatro `refs/bilink/main`.
