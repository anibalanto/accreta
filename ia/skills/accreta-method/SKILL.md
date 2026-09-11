---
name: accreta-method
description: "Cómo se trabaja en accreta y en sus subsistemas: la cadena tarea ↔ decisión ↔ spec ↔ código, TDD, las reglas del código, los commits y dónde está cada cosa. Cargar al empezar cualquier trabajo en accreta, en uno de sus subsistemas o en una vista de muckpile, y siempre antes de escribir una spec o código y antes de commitear."
---

Es cómo se trabaja hoy. El porqué está en las decisiones de `docs/decisions/`, en la raíz de accreta. Hoy hay una sola, `decisiones-vivas`, que todavía está en borrador: el método está en prueba, y esta skill la sigue mientras cambia. Cuando una decisión nueva cambia el método, el cambio se refleja acá.

## Qué es accreta

Especificaciones y herramientas de un ecosistema:
- **bilinker:** referencias verificadas entre fragmentos.
- **stratum:** las capas.
- **lattice:** el grafo del proyecto.
- **lspd:** un multiplexor de language servers.
- **impact**.
- **muckpile:** el trabajo pendiente, sincronizado con Jira.

Cada subsistema tiene su implementación en `subsystems/<nombre>/.stratum/impl/`, que es un repo git independiente. Las capas se traen con `stratum pull`, que lee la declaración de cada una en `.stratum/.<nombre>.toml`.

**`worklist` está deprecado.** `subsystems/worklist/` queda como historia: lo reemplaza `muckpile`.

## Las otras skills

| Skill | Cuándo |
|---|---|
| `bilinker` | siempre: las referencias verificadas son el método |
| `stratum-paths` | siempre: cualquier path se compone con ella |
| `decision-records` | al tomar, escribir o resolver una decisión, o al llevar su avance |
| `accreta-devs` | después de escribir código que implementa algo que una spec ya dice, antes de darlo por terminado |
| `item-writing` | al escribir o retitular un ítem |

## La cadena

```
tarea ↔ decisión ↔ spec ↔ código        cada eslabón, un bilink
```

1. **Todo cambio parte de algo escrito:** una sección de una decisión, o un ítem.
2. **La decisión dice el porqué** y la opción elegida, en pocas líneas.
3. **La spec dice qué hace el sistema hoy**, y se escribe antes que el código.
4. **El código se hace con TDD.**
5. **`bilinker check .` dice qué quedó sin atar**, y cada bilink se acepta. Recién ahí la dimensión está cerrada.

- **Una pregunta de diseño para el trabajo, y se hace en texto.** La respuesta se escribe primero en la decisión.
- **Se avanza de punta a punta, y solo se para ante un riesgo:** algo que la decisión no cerró, o una escritura contra un proveedor real.
- **El avance se lleva en las tablas de cada decisión.** Mientras `muckpile` no pueda dejar tareas sin subir al proveedor, no se crean tareas de seguimiento en Jira.

## Dónde está cada cosa

| | |
|---|---|
| Decisiones del método, o que cruzan subsistemas | `docs/decisions/` de la raíz de accreta |
| Decisiones de un subsistema | `docs/decisions/` de su impl. Los ADR numerados de `docs/adr/` son historia. |
| Spec de un subsistema nuevo | `docs/specs/` de su impl, al lado del código |
| Spec de un subsistema que ya existía | `subsystems/<nombre>/`, hasta que se decida otra cosa |
| El trabajo | `muckpile`: ver el README de su impl |

## Código

- **Un identificador traduce el término de la spec, nunca inventa uno propio.** Si la spec dice "vista", el código dice `view` en todos lados.
- **El idioma lo dice cada subsistema.** Por defecto, los identificadores van en inglés y los comentarios en castellano. En `muckpile` va todo en inglés. Un identificador en castellano se traduce cuando se lo toca por otra razón, nunca en una barrida: un renombre masivo deja `MOVED` todos los bilinks de la capa sin arreglar nada.
- **Un comentario de código no cita decisiones, specs ni ítems.** Si hace falta decir por qué el código es así, se dice en términos del código.
- **La suite de tests nunca le habla a un proveedor real.** Lo que se sabe de un proveedor real se mide a mano en un board propio, y esa medición alimenta el test. Solo se escribe en un proveedor real con permiso, y sobre ítems descartables.

## Paths

Los paths se escriben con tokens de Stratum —`*` para la raíz, `<` para subir, `>nombre` para bajar— y se resuelven con `$(stratum '...')`. No se hardcodean rutas absolutas.

## Commits

- **Un commit hace una sola cosa, y su mensaje ocupa una línea.**
- **Arranca con el id de lo que ejecuta:** el slug de la decisión (`decisiones-vivas: …`) o el id del ítem (`@arreglar-el-hook: …`, o `ACC-347: …` cuando ya tiene clave). El prefijo `@` envejece a propósito: el ítem cambia de nombre cuando cruza al proveedor, y la historia no se reescribe.
- **El orden es código, después spec y decisión, y después bilinks.** `accept` solo fija lo que ya está en la historia, así que nunca se acepta en una rama que se va a rebasar.
- **Cada repo commitea lo suyo:** accreta y cada impl son repos distintos.
- **accreta se empuja solo por decisión humana.** El impl de `muckpile` se empuja al cerrar cada ciclo.
