---
title: El terreno está preparado para trabajar spec-first
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
---

# El terreno está preparado para trabajar spec-first

**Lado:** desarrollo · **Decisión:** ninguna — es la precondición del método de trabajo de la épica.

Como quien va a ejecutar esta épica, quiero que cargar el skill y correr `bilinker check` alcance para encontrar el código a tocar, para no volver a buscarlo a mano.

## Por qué va primera

Dos razones, y las dos son de método:

**El skill se carga en cada tarea.** Hoy documenta el formato viejo, y ADR-0003 lo dice sin vueltas: un skill desactualizado *"no es documentación desactualizada sino una instrucción activa de crear bilinks en el formato anterior"*. Si el prerequisito de todo es cargarlo, cargarlo viejo envenena todo lo que sigue.

**Los bilinks son el índice del trabajo, y hoy tiene agujeros.** El método se apoya en que tocar una spec rompa bilinks que apunten al código. Medido contra lo que los ADRs mandan tocar, eso hoy no se cumple: `concepts/capture.md` —la spec que ADR-0003 más reescribe— no tiene **ningún** bilink, y `apply.rs` —que cambia de raíz— tampoco. Reescribirlas no rompería nada y el método daría silencio justo donde hay más trabajo.

## Cuándo está hecha

Correr `bilinker check .` después de tocar cualquiera de las specs que los tres ADRs listan reporta al menos un endpoint no-OK que lleva al código correspondiente. Ninguna de esas specs queda muda.
