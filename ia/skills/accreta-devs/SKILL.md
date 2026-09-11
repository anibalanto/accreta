---
name: accreta-devs
description: "El disparador del método de bilinker: cargar después de escribir o modificar código que implementa un fragmento de spec ya existente en cualquier subsistema de accreta (una sección de una decisión, una fila de tabla de interfaz, un fragmento de docs/specs/ o de concepts/*.md) — antes de dar la tarea por terminada, aunque nadie haya mencionado bilinker. La mecánica en sí vive en la skill `bilinker`; esta sólo dice cuándo no alcanza con haber escrito el código."
---

Implementar algo que una spec de accreta ya describe no termina cuando el código compila y los tests pasan. Termina cuando queda un bilink conectando ese fragmento de spec con el fragmento de código, aceptado. Sin ese lazo, el próximo cambio a la spec no tiene cómo señalar que el código quedó atrás — es la misma clase de deuda silenciosa que el resto de accreta evita con esta herramienta en vez de a mano.

**No es de un subsistema en particular.** Vale para todo accreta, porque el `AGENTS.md` de la raíz dice que el método es el de bilinker.

## Cuándo

Después de escribir o modificar código que responde a un fragmento de spec que ya existía — una sección de una decisión, una fila de una tabla de interfaz, un fragmento de `docs/specs/` o de `concepts/*.md` — y antes de reportar la tarea como terminada. Aplica aunque el pedido original no haya mencionado bilinks ni bilinker: si el código tocado implementa algo que la spec ya dice, el disparador es ese, no la palabra.

**No aplica** cuando el código tocado no tiene contraparte de spec: un refactor interno, un fix de un bug que la spec no describe, un test que no corresponde a ninguna decisión puntual.

## Qué hacer

La mecánica completa —sintaxis de `--tip`, anclas estables, estados, comandos— está en la skill `bilinker`; acá sólo el orden de los pasos para este momento puntual:

1. **¿Ya hay un bilink que conecta ese fragmento de spec con ese fragmento de código?**
   - No: crearlo con `bilinker chain new`, un `--tip` a cada lado.
   - Sí: no crear uno nuevo — el cambio recién hecho va a aparecer como no-OK en el que ya existe al correr `check`.
2. `bilinker check .` en la capa del subsistema (el repo del impl, no el de accreta entero).
3. Cada no-OK que corresponda a este cambio: revisar con `bilinker get <uuid>.<N> --diff` si el estado lo pide, y `bilinker accept`.
4. Recién ahí la tarea está terminada — no antes, y no alcanza con haber commiteado el código.
