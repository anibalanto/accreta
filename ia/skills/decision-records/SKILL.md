---
name: decision-records
description: "Cómo se escribe y se lleva una decisión: `docs/decisions/<AAAA-MM-DD>.<slug>.md`, viva mientras se desarrolla y congelada al aceptarse, atada con bilinks a su tarea, a la spec y al código. Cargar al tomar o registrar una decisión de diseño, al abrir o resolver una sección abierta, al actualizar la tabla de avance de una decisión y al aceptarla."
---

Es cómo se escribe y se lleva una decisión hoy. El porqué está en la decisión `decisiones-vivas` (`docs/decisions/2026-09-10.decisiones-vivas.md`, en la raíz de accreta). Mientras esa decisión esté en borrador o en desarrollo, esta skill la sigue. Después, la regla vigente es la de la skill, y una decisión nueva que la cambie se refleja acá.

## Qué es una decisión

**Es el registro de por qué algo es así, y de qué opción se eligió.** No lleva apellido: no importa si es de arquitectura, de diseño o de convenciones. La spec dice qué hace el sistema hoy. La decisión dice por qué, y en qué momento se decidió.

**Tiene que ser corta:** el porqué y la elección. El detalle va a la spec: las tablas de interfaz, los formatos, los hechos medidos que fijan un comportamiento.

## El nombre

```
docs/decisions/<AAAA-MM-DD>.<slug>.md     la fecha es la de creación, y no cambia nunca
```

- **El id es el slug, sin la fecha y sin `@`.** El `@` es de los ítems de `muckpile`: `@algo` siempre es un ítem, y `algo` a secas, en un lugar donde va un id, es una decisión.
- **El archivo no se renombra nunca.** La fecha de cierre y el estado no van en el nombre, porque renombrar el archivo deja colgados los bilinks que apuntan a él.
- **El archivo se encuentra por `*.<slug>.md`.**
- **Si el mismo slug está en varios repos, son partes de una misma decisión.** Cada parte vive en el repo que gobierna, y su encabezado dice, con paths de Stratum, dónde caen las otras.
- **Los commits que la tocan llevan su id adelante:** `decisiones-vivas: …`.

## El encabezado y el ciclo

```markdown
# <slug> — <título>

**Estado:** Borrador · **Versión:** 0.3 · **Última actualización:** 2026-09-10
```

| Estado | Versión | Qué pasa |
|---|---|---|
| Borrador | 0.x | Se discute, y todavía no hay código. |
| En desarrollo | 1.x | Ya hay código, y el código la profundiza: los hechos medidos se escriben en la decisión, aparecen decisiones nuevas, y una pregunta de diseño para el trabajo hasta tener respuesta. |
| Aceptada | — | Todas sus dimensiones están cerradas. **Se congela.** |
| Descartada / Reemplazada por `<otra>` | — | — |

**Cada cambio sube la versión menor y deja una fila en `## Historial`** (`| Versión | Fecha | Qué cambió |`).

**Una decisión aceptada no se edita.** Si cambia el comportamiento, se toca la spec. Si ese cambio contradice lo que se decidió, se escribe otra decisión, que reemplaza a la anterior en esa parte.

## Las secciones

`## Contexto`, `## Decisión` con una sección `### N.` por cada cosa que se decide, `## Lo que esta decisión no decide` y `## Historial`.

**Una sección que todavía no decide se marca con `— abierta` en el título.** Dice qué está en juego y qué opciones se conocen, sin elegir ninguna.
- Se ata con un bilink a un ítem `question` de `muckpile`. La discusión vive en la question: los comentarios en `thread/`, y el borrador de la respuesta en `files/`.
- La question bloquea el `done` de la tarea de la sección, pero no su `in-progress`.
- **Resolverla es llenar la sección con una decisión concreta.** La question se cierra y su bilink queda como historia.
- **Una decisión no se acepta con secciones abiertas.**

**"Lo que esta decisión no decide"** es solo para lo que pertenece a otra parte o a otra decisión. Lo que falta decidir y es de esta decisión va en una sección abierta.

## La tabla de avance

Cada sección `### N.` termina con una tabla:

```markdown
| Dimensión | Estado | Tarea | Evidencia |
|---|---|---|---|
| Un solo binario, sin hooks | `cerrada` | `ACC-362` | `main`, en `main.rs` ↔ esta sección |
```

| Estado | Qué quiere decir |
|---|---|
| `cerrada` | La cadena hasta el código está aceptada. Es lo único que cuenta como hecho. |
| `sin bilink` | Existe y hace lo que se decidió, pero nada lo ata. |
| `diverge` | Existe, pero no hace lo que se decidió. |
| `pendiente` | Está decidido, y no existe. |
| `falta spec` | El código decide algo que ni la decisión ni la spec dicen. |
| `abierta` | Todavía no está decidido. La columna `Tarea` lleva la question. |
| `cumple` / `no cumple` | Para lo transversal, que no tiene un fragmento al que atar un bilink. |

- **Toda dimensión que no está cerrada tiene una tarea**, y el orden de las tareas en el board es el orden por riesgo.
- **Un hallazgo entra en el momento en que aparece:** una fila en la tabla y una tarea. Si no se sabe a qué sección pertenece, probablemente falta una.
- **La línea "Medido el … sobre `<commit>`"** se actualiza cada vez que se mide.
- **Editar una tabla deja `EXPANDED` a los bilinks que capturan la sección entera.** El diff lo da bilinker mismo (`bilinker get <uuid>.<N> --diff`): si lo único nuevo es la tabla, se re-aceptan.
- **Mientras `muckpile` no pueda dejar tareas sin subir al proveedor, la tabla es el avance.** No se crean tareas de seguimiento en Jira: pueden molestar al PM, al funcional o al Scrum Master.

## La cadena

```
tarea ↔ decisión ↔ spec ↔ código
```

**Cada eslabón es un bilink `governs`.** La spec vive en `docs/specs/` del repo del impl. Nada se muda: un cambio en la decisión deja la spec `ALTERED`, y corregir la spec deja el código `ALTERED`.

**Un bilink que ya no se va a re-aceptar no se borra.** Pasa, por ejemplo, cuando una decisión posterior reescribió el fragmento, o cuando la tarea se cerró. Su última aceptación fija qué decía cada punta en ese momento, y es el rastro que permite ir desde una tarea vieja hasta el código de entonces.

La mecánica de bilinker —capturas, `chain new`, `accept`— está en la skill `bilinker`. Cuándo hay que cerrar el ciclo está en `accreta-devs`.

## Qué va en el ítem y qué va en la decisión

| Va en el ítem | Va en la decisión |
|---|---|
| el diagnóstico con el que abre el cuerpo | por qué se decidió así |
| qué tiene que ser cierto cuando termine | qué alternativas se descartaron, y con qué argumento |
| contra qué se mide | qué premisa cambió, y cuándo |
| de qué depende | qué queda atado a la decisión |

**La deliberación no se borra: se muda a la decisión.** Un ítem que lleva adentro la historia de cómo se decidió cuesta releerlo, y el que no se relee conserva premisas muertas.

## Cómo se cita

- **La decisión no cita ítems en su texto.** El id de un ítem cambia cuando cruza al proveedor. La tarea se ata con un bilink, no con una mención.
- **Un comentario de código no cita decisiones, specs ni tareas.** Si hace falta explicar por qué el código es así, se explica en términos del código.
- **Una decisión puede citar otra decisión por su slug.** Lo que gobierna un fragmento se ata con un bilink, no con un link de markdown.
