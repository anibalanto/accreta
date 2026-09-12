---
name: bilink-anchors
description: "Qué se ata con un bilink y con qué grano: la regla contra la función que la cumple, nunca un archivo entero cuando adentro hay reglas sueltas, y qué no merece un bilink. La mecánica de los comandos es de la skill bilinker."
when_to_use: "Antes de crear una cadena, al elegir el fragmento de cada punta, al decidir si un cambio merece un bilink nuevo o entra en uno que ya existe, y al revisar un bilink que reporta drift que nadie pidió."
---

Elegir **qué** se ata es la decisión; cómo se escribe el capture es mecánica, y está en la skill [`bilinker`](../bilinker/SKILL.md). Un bilink mal elegido no falla: reporta drift que nadie va a mirar, y eso entrena a no mirar ninguno.

## Qué se ata con qué

**Una regla contra el fragmento de código que la cumple.** La cadena completa es `tarea ↔ decisión ↔ spec ↔ código`, y cada eslabón ata dos fragmentos del mismo tamaño conceptual:

| Punta | Qué es el fragmento |
|---|---|
| tarea | el ítem |
| decisión | la sección `### N.` entera, con su tabla de avance |
| spec | el h3 de la regla ([`spec-writing`](../spec-writing/SKILL.md)) |
| código | la función, el método o el tipo que la cumple |

**Un archivo entero se ata sólo cuando gobierna entero:** una decisión, un ADR, un README. Ahí no hay reglas sueltas que alguien toque por separado.

## El grano

### Nunca se ancla un h1

Un h1 es el archivo entero. Anclado ahí, **cualquier edición a cualquier regla del archivo marca el bilink**, y quien lo revisa no sabe si le tocaron la suya. Se ancla el h3 de la regla.

Medido el 2026-09-12: en accreta, 151 de 247 anclas de markdown son de archivo entero; en el impl de `muckpile`, que nació con la regla, 71 de 104 son de fragmento. En una spec de 156 líneas y doce reglas, el ancla de archivo entero marca el código cada vez que alguien toca cualquiera de las once que no son la suya.

### El grano fino cuesta, y se paga a propósito

Más anclas es más bilinks para aceptar en cada cambio: el impl de `muckpile` tiene 104 para un subsistema del tamaño del de lspd, que tiene 6. Se paga porque lo que se compra es que un `ALTERED` signifique algo.

### Un fragmento puede tener varios bilinks

La aridad de un bilink es siempre dos, pero un mismo h3 puede estar atado a la decisión que lo gobierna y a las dos funciones que lo cumplen. No hace falta elegir una sola punta por fragmento.

## Qué no merece un bilink

- **Un refactor sin contraparte en la spec.** Si lo que cambió no lo dice ninguna regla, lo que falta es la regla, no el bilink.
- **Un fragmento que nadie va a revisar.** Un bilink es una promesa de que alguien mira el `ALTERED`; si no hay quién, es ruido versionado.
- **Lo que ya está cubierto por el bilink de al lado.** Dos bilinks al mismo par de fragmentos son un solo aviso repetido.

## Antes de aceptar, mirar qué agarró

`chain new` escribe el capture, y un capture es opaco después. **`bilinker get <uuid>.<N>` muestra el fragmento**, y es el único momento barato para ver si el ancla es la que se quería.

Dos errores que se repiten:

- **En código, la columna es la del nombre.** `archivo.rs:10:1` cae en la indentación y captura el nodo que lo contiene —el `impl` entero—; `archivo.rs:10:5`, sobre `pub fn`, captura la función.
- **Un ancla que se repite no es ancla.** `capture` se niega antes de escribir cuando la query matchea dos veces: pasa entre el h1 y un `##` que se llaman igual, y en código entre dos `impl` del mismo tipo —`impl Foo` y `impl Drop for Foo`—. Se elige un nodo con nombre propio adentro.

## Cuándo se ata

Al terminar de escribir el código que cumple una regla que la spec ya dice, antes de dar la tarea por terminada: es la skill [`accreta-devs`](../accreta-devs/SKILL.md). Si el bilink ya existe, no se crea otro: el cambio aparece como no-OK en el que está.
