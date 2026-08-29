# Accreta

Specs de un ecosistema de herramientas: **bilinker** (referencias verificadas entre fragmentos), **stratum** (capas), **lattice** (grafo), **impact**, **worklist**. Cada subsistema tiene sus specs acá y su implementación Rust en `subsystems/<nombre>/.stratum/impl/`, que es un repo git independiente y gitignoreado por su padre.

## El trabajo en curso

**`.stratum/worklist/1.epic.md`** — la épica del MVP de bilinker para lamansys. Ahí están el orden de trabajo, las 8 user stories y las 13 tasks. Empezar por leerla.

Las decisiones que la épica ejecuta viven en `subsystems/bilinker/.stratum/impl/docs/adr/`:

| | |
|---|---|
| ADR-0003 | el formato — captures inmutables, YAML, aceptación explícita |
| ADR-0004 | los bilinks viven en `refs/bilink/<branch>` |
| ADR-0005 | la frontera entre proyectos |
| ADR-0006 | el formato como crate versionado |

Los cuatro están en **Propuesto**, y las specs de `subsystems/*/` **todavía no los reflejan** — reescribirlas es parte del trabajo, no un pendiente olvidado.

## Cómo se trabaja acá

1. Se toca **la spec**, nunca el código primero.
2. `bilinker check .` reporta los endpoints que quedaron no-OK.
3. Cada no-OK es un puntero al fragmento de código que implementaba esa spec. Se sigue con `bilinker get`.
4. Se cambia el código y se acepta.

**El inventario de trabajo de un cambio *es* la lista de no-OK.** Buscar el código a mano produce una lista que envejece el mismo día que se escribe; los bilinks están vivos.

## Dos trampas de bootstrap

**El skill de bilinker está desactualizado.** `ia/skills/bilinker/SKILL.md` documenta el formato anterior — `link.0`, `resolved_at`, `.stratum/impl` como path crudo. Como se carga solo al trabajar con bilinks, no es documentación vieja sino **una instrucción activa de escribir el formato anterior**. Reescribirlo es la primera task de la primera US, y hasta entonces conviene leerlo contra ADR-0003 y no al revés.

**La cobertura de bilinks tiene agujeros.** Tocar `subsystems/bilinker/concepts/capture.md` o `commands/migrate.md` hoy no rompe nada, porque ningún bilink apunta ahí; y `apply.rs` de la capa impl no es alcanzable desde ninguna spec. Cerrar eso es la otra task de la primera US, y hasta que esté, el método de arriba da silencio justo donde hay más trabajo.

## Paths

Usar el skill `stratum-paths`: los paths se escriben con tokens Stratum —`*` raíz, `<` subir, `>name` bajar— y se resuelven con `$(stratum '...')`. No hardcodear rutas absolutas.

## Commits

Mensaje de una línea, sin trailer de co-autoría.
