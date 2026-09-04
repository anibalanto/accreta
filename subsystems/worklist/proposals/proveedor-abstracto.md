# Propuesta: el proveedor como implementación, no como la forma del puerto

**Estado:** en discusión. Sin diseño. La pide una mejora enunciada así: *"que todas las operaciones sobre Jira estén en una API Rust estable usada por worklist, y que sea una implementación abstracta — mañana se puede usar GitHub, GitLab o Taiga"*.

## El eje que falta

[`sync.md`](../concepts/sync.md) § "El puerto son las operaciones, no los comandos" abstrajo **el transporte**: con qué se le habla a Jira. Jira quedó adentro.

```
hoy    worklist → puerto → transporte (acli | jira-cli) → Jira
esto   worklist → puerto → proveedor (Jira | GitHub | GitLab | Taiga)
                              └→ sus transportes, que dejan de ser del puerto
```

**Y el transporte pasa a ser un asunto de cada proveedor.** Que Jira necesite dos binarios porque uno no alcanza es un problema de Jira; que GitHub tenga un solo cliente HTTP es un problema de GitHub. El puerto no debería tener una enumeración de transportes: hoy la tiene, y es lo primero que se cae hacia adentro del proveedor.

## Ya hay media prueba de que se puede

`FileProvider` existe, no es Jira, y sostiene el compare-and-swap entero en los tests. **La lectura ya tiene dos implementaciones.** La que tiene una sola es la escritura, y es donde está todo lo difícil.

## Dónde está Jira hoy, medido

| archivo | líneas con Jira / totales | qué es |
|---|---|---|
| `board.rs` | 70 / 506 | invocaciones de `acli` y `jira-cli`, la JQL, el mapeo de tipos |
| `port.rs` | 33 / 196 | `Transport::{Acli, JiraCli}`, y nombres de operación que dicen "épica" y "sprint" |
| `assign.rs` | 23 / 390 | **casi todo comentarios** que explican una limitación de `acli` |
| `body.rs` | 16 / 118 | la conversión a ADF, que es el formato de documento de Jira |
| `provider.rs` | 11 / 155 | la JQL de lectura |
| `main.rs` | 12 / 303 | `--project`, que es vocabulario de Jira |

`window.rs` menciona *sprint* veintiún veces y **ninguna es de Jira**: el sprint es del worklist. Vale distinguirlo, porque la confusión entre "concepto del worklist" y "concepto del proveedor" es exactamente lo que esta propuesta tiene que resolver.

## El problema difícil: la intersección pierde lo que hoy funciona

Los proveedores no son variaciones de lo mismo.

| | jerarquía | iteración | cuerpo | vínculos |
|---|---|---|---|---|
| Jira | épica → historia → tarea | sprints | ADF | tipados (`Blocks`, `Relates`) |
| GitHub | sub-issues | milestones | Markdown | referencias sin tipo |
| GitLab | épicas, y sólo en las pagas | milestones, iterations | Markdown | tipados |
| Taiga | user stories → tasks | sprints | Markdown | sin tipo |

Si el puerto es la **intersección**, se pierde la jerarquía que hoy anda y el worklist queda peor por haberse abstraído.

**Y la spec ya tiene la forma de la respuesta**, escrita para otra cosa: § "La jerarquía entra hasta donde el proveedor la tiene". Ese es el patrón — el proveedor **declara qué puede**, y el worklist degrada **a sabiendas**: el escalón que Jira no tiene ya viaja como link, y eso se decidió y se escribió en vez de perderse en silencio.

> **La abstracción no es el mínimo común: es un conjunto de capacidades declaradas y una degradación que se reporta.**

## Tres cosas que hoy parecen del modelo y son del proveedor

Es lo que una abstracción mal cortada dejaría del lado equivocado, y ninguna es obvia.

**El commit `normalize:`.** Existe porque el schema de Jira **poda** lo que no admite, y lo que se guarda es la vuelta. Un proveedor con Markdown nativo no produciría ninguno. ¿Es parte del contrato del puerto —*"el cuerpo vuelve convertido"*— o una rareza de Jira que el worklist absorbió como si fuera la regla?

**La traducción de tipos.** `jira_type` mapea `task → "Tarea"`, `user-story → "Historia"`, `epic → "Epic"`. No son nombres de Jira: son los de **este proyecto** de Jira, en el idioma en que se creó. Eso no es del proveedor siquiera — es de la instancia.

**La búsqueda por título.** `create_or_find` es lo que hace que un reintento no duplique, y descansa en que el proveedor sepa encontrar por título exacto. La JQL costó dos duplicados —[`5l`](../../../.worklist/insecure/all/5l.task.md) y [`65`](../../../.worklist/insecure/all/65.task.md)— y las reglas que sobrevivieron son de la tokenización de Jira. Cada proveedor busca distinto, y **la idempotencia entera cuelga de ahí.**

## Qué querría decir "estable"

Hoy el puerto cambia cuando cambia lo que `acli` puede hacer. Estable quiere decir lo contrario: **cambia cuando cambia lo que el worklist necesita.** Es una superficie con versión, y lo que la mueve es el modelo de arriba, no la herramienta de abajo.

## El riesgo, y es el mismo de antes

> **Una abstracción escrita con una sola implementación se dibuja con la forma de esa implementación.**

Es literalmente por qué existió el puerto de `66`: *"quedó dibujado por lo que `acli` sabe hacer, así que cuando apareció algo que `acli` no puede no hubo dónde ponerlo"*. Hacer lo mismo un nivel más arriba —un puerto de proveedor con Jira de único proveedor— repite el defecto con más ceremonia.

Las salidas, y ninguna es gratis:

**Escribir un segundo proveedor de verdad**, aunque sea chico. Es lo único que contesta si el corte sirve. Un `FileProvider` de escritura no alcanza: no tiene un modelo ajeno que resista.

**O cortar por lo que el worklist necesita y no por lo que Jira ofrece.** Las operaciones salen del modelo —clave, título, cuerpo, jerarquía, iteración, vínculo, estado— y no del inventario de lo que `acli` sabe. Es más barato y es lo que ya se hizo una vez a nivel transporte.

**O no hacerlo todavía**, y dejar escrito el corte para cuando aparezca el segundo. La opción honesta si nadie va a usar GitHub este año: el trabajo no se pierde, se posterga con el argumento a la vista.

## Lo que habría que decidir

- **Cómo se declaran las capacidades**, y qué hace el worklist cuando una falta: degradar y reportar, o rechazar.
- **Dónde vive el mapeo de tipos**, sabiendo que es de la instancia y no del proveedor: ¿configuración?
- **Qué es una clave.** `ACC-123`, `#42` y un uuid no se parecen, y la clave hoy es el **nombre del archivo**.
- **Qué formato tiene el cuerpo en el contrato**, y si el `normalize:` sobrevive a un proveedor que no poda.
- **Cómo se busca por título** en un proveedor sin JQL, y si la idempotencia puede dejar de depender de eso.
- **Qué es una iteración** donde no hay sprints, y si `_sprints/` mapea a milestones sin mentir.
- **Y si el `Provider` de lectura y el `Board` de escritura son un puerto o dos.** Hoy son dos porque el de prueba sólo lee.
