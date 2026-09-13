---
name: spec-writing
description: "Cómo se escribe una spec: un archivo por concepto, cada regla un h3 que la nombra, en prosa y en presente, con los hechos medidos fechados. Y qué no es de la spec: la historia, que es de la decisión."
when_to_use: "Al escribir o corregir una spec, al agregarle una regla, al mudar una spec de lugar o reordenarla, y antes de atar un bilink a un fragmento de spec."
---

Es cómo se escribe una spec hoy. El porqué está en la decisión `decisiones-vivas` (`docs/decisions/2026-09-10.decisiones-vivas.md`, en la raíz de accreta), decisión 14. Qué fragmento se ata con un bilink es de la skill `bilink-anchors`, y las dos se leen juntas: **la forma de la spec es lo que decide qué tan fino queda cada bilink.**

## Dónde vive

`docs/specs/` del repo del impl, al lado del código. No se congela nunca: se edita cada vez que cambia el comportamiento.

| | |
|---|---|
| `docs/specs/concepts/<concepto>.md` | un archivo por concepto |
| `docs/specs/commands.md` | la tabla de interfaz, una fila por comando |

**Un archivo por comando no existe.** La fila de la tabla ya es un ancla estable, y el detalle de cada comando repetiría lo que dicen los conceptos: la fila remite al concepto.

## La forma

### Cada regla es un h3, y su título la nombra

El h1 es el concepto, los h2 agrupan, y **cada regla es un h3 cuyo título dice la regla**, no el tema: "El token sólo se lee del entorno", no "El token".

Un título que es una pregunta retórica —"Por qué no TCP en loopback"— no nombra una regla: nombra una discusión. Si la discusión justifica la regla, el título dice la regla y la discusión queda en el cuerpo.

### El título es un ancla, y se elige una vez

Un bilink ancla en el título del h3 ([`bilink-anchors`](../bilink-anchors/SKILL.md)). Renombrarlo deja el capture `UNANCHORED`, y hay que recapturar.

**Por eso, al mudar o reordenar una spec, cambia la estructura y no las palabras:** un `##` que es una regla pasa a `###`, un `####` sube, y los grupos se arman alrededor. Las palabras de un título se tocan sólo si no nombran su regla, o si chocan con otro título del mismo archivo —dos títulos iguales no son ancla—.

### Se escribe en prosa y en presente

Dice lo que el sistema hace hoy. Sin notación de requisitos: ordena lo que dice una regla y no el porqué, que ya vive en la decisión.

### Un hecho medido va con su fecha

Lo que se sabe de un proveedor, de un sistema operativo o de un servidor ajeno se mide a mano, y la spec lo dice con la fecha: "Medido el 2026-09-10: `/rest/api/3/search` responde 410". Es lo que permite volver a medirlo cuando algo cambie del otro lado.

### Lo que no es de la spec

- **La historia.** "Por qué se hizo ahora y no antes" es de la decisión. La spec dice la regla que quedó.
- **Los ítems.** Un id de ítem envejece; la spec no lo cita.
- **Los links a otra capa.** Una spec que vive en su impl no sabe dónde está clonada la de al lado: lo que es de otro subsistema se nombra en prosa, y lo que lo ata es un bilink.

## Mudar una spec a su impl

Cuando la spec de un subsistema se muda de accreta a `docs/specs/` de su impl (`decisiones-vivas`, decisión 12), en este orden:

1. **Los archivos se mudan con su contenido**, y se reordenan a la forma de arriba: cada regla en un h3, agrupadas en h2. Las palabras de los títulos se conservan.
2. **Lo que era historia sale de la spec.** Si vale la pena conservarlo, va a una decisión del impl, no a la basura.
3. **Los links que cruzaban de capa se vuelven prosa.**
4. **Se crean los bilinks nuevos, anclados en la regla** —no en el archivo entero— y se aceptan.
5. **La spec vieja se borra en accreta, y sus bilinks se borran con ella.** No es la decisión 6: ahí el fragmento sigue existiendo y el bilink queda `ALTERED` como rastro. Acá el fragmento ya no está en esa capa, el bilink queda `UNRESOLVED` para siempre y no puede decir nada de hoy. El rastro no se pierde: la última aceptación queda en la historia de `refs/bilink/<branch>`, que es donde vive toda decisión.
