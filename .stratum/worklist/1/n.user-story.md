---
title: El terreno está preparado para trabajar spec-first
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T00:37:31Z
---

# El terreno está preparado para trabajar spec-first

**Lado:** desarrollo · **Decisión:** ninguna — es la precondición del método de trabajo de la épica.

Como quien va a ejecutar esta épica, quiero que cargar el skill y correr `bilinker check` alcance para encontrar el código a tocar, para no volver a buscarlo a mano.

## Por qué va primera

**Los bilinks son el índice del trabajo, y hoy tiene agujeros.** El método se apoya en que tocar una spec rompa bilinks que apunten al código. Medido contra lo que los ADRs mandan tocar, eso hoy no se cumple: `concepts/capture.md` —la spec que ADR-0003 más reescribe— no tiene **ningún** bilink, y `apply.rs` —que cambia de raíz— tampoco. Reescribirlas no rompería nada y el método daría silencio justo donde hay más trabajo.

Y va **antes** de tocar cualquier spec, no después: cerrar la cobertura sobre archivos ya reescritos sería vincular a ciegas.

## Cuándo está hecha

Correr `bilinker check .` después de tocar cualquiera de las specs que los tres ADRs listan reporta al menos un endpoint no-OK que lleva al código correspondiente. Ninguna de esas specs queda muda.

## Cómo se verificó

No por lectura: se editó cada una de las quince specs que los tres ADRs listan —agregando una sección nueva al final, que es la edición que menos rompe— y se corrió `check`. Las quince reportan al menos un endpoint no-OK que lleva al código. Las cinco que quedaban mudas se cerraron en la task `o`.

Los dos defectos que la cobertura destapó están arreglados: la salida de `check` volvió a ser fiel a lo que `check` sabe (task `v`) y `capture` ya no escribe un ancla que apunte al nodo equivocado (task `w`). Los dos con test de regresión verificado contra el código viejo.

Queda abierta la task `u` —el endpoint `task` no resuelve a ningún archivo—, que deja cinco endpoints no-OK. Están en la US `5`, y ese ruido es la task, no un pendiente de ésta.
