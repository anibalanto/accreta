---
title: Cerrar la cobertura de bilinks antes de tocar nada
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T00:37:31Z
---

# Cerrar la cobertura de bilinks antes de tocar nada

Crear los bilinks que faltan para que reescribir una spec rompa algo que lleve al código. **Es la primera tarea de la épica**, y la única donde hay que buscar código a mano — se hace una vez, precisamente para no volver a hacerlo.

Opera sobre el **formato de hoy**, con el binario de hoy. No depende de ninguno de los cuatro ADRs.

## Antes de empezar

Cargar dos skills:

- **`bilinker`** — es el prerequisito del método de la épica, y acá además se crean bilinks a mano con `capture` y `chain new`. Su aviso dice que describe el formato anterior: para esta tarea eso **no molesta**, porque el formato anterior es el vigente hasta el corte `004`.
- **`stratum-paths`** — el cruce va de la capa de spec a la de impl, que son repos distintos. Los paths se componen con `$(stratum '...')`, no a mano.

## El criterio

> **Si una especificación tiene implementación, está bilinkeada.**

No es "todo bilinkeado". Buena parte de una spec —el porqué de una decisión, una alternativa descartada, un ejemplo— no tiene código que la implemente, y vincularla no detecta nada porque no hay contra qué derivar. Y sobre-cubrir tiene un costo propio: si todo estuviera vinculado, editar una spec produciría cientos de no-OK y **el inventario dejaría de servir como inventario** — la respuesta racional pasaría a ser `accept .` a ciegas, que es justo lo que ADR-0003 prohíbe.

El cruce **parte del código**: para cada cosa que la implementación hace, la spec que la describe tiene que llegar hasta ahí por un bilink. Un archivo con siete afirmaciones verificables lleva siete — `commands/check.md` ya tiene 7, y está bien así.

## Aceptar no es automático

Crear un bilink es decir *"este fragmento de spec y este de código se corresponden"*. Aceptarlo es además decir *"y apruebo el estado en que están"*. Encadenar `capture`, `chain new` y `accept .` sobre cincuenta bilinks nuevos **fabrica cincuenta aprobaciones que nadie miró** — y a partir de ahí el `check` reporta verde sobre correspondencias que quizá no existen.

Cada bilink nuevo se verifica antes de aceptarlo: que el fragmento de código sea efectivamente lo que la spec describe, y que lo que dice hoy sea cierto. Si no lo es, el hallazgo es la tarea — no se acepta para dejar el árbol limpio.

## El punto de partida medido

De las specs de bilinker, 22 de 27 archivos ya tienen al menos un bilink. Los que el cambio va a tocar y hoy no señalan nada:

| | Bilinks |
|---|---|
| `concepts/capture.md` | **0** — y es la que más se reescribe |
| `concepts/configuration.md` | **0** |
| `concepts/consistency.md` | **0** |
| `commands/migrate.md` | **0** |
| `commands/status.md` | **0** |
| `architecture.md` | **0** |

Y del lado del código, sobre los 47 captures de la capa impl: `check.rs` tiene 13, `accept.rs` 3, `bilink.rs` 2, `capture.rs` 1 — y **`apply.rs`, `task.rs`, `migrations.rs`, `hash.rs`, `git.rs` y `query.rs` tienen 0**.

Es el punto de partida, no el alcance. El cruce completo —los 17 comandos del CLI y los módulos del core contra las specs que los describen— es parte de la tarea.

## Resultado

55 cadenas nuevas: la capa raíz pasó de 63 a 111 bilinks y la de lattice de 7 a 16.

Los archivos que el enunciado listaba en cero quedaron cubiertos —`concepts/capture.md` con 8, `commands/migrate.md` con 5, `architecture.md` con 4, `commands/recapture.md` con 2— y también `concepts/migration.md` de la capa raíz, que tiene su implementación en el crate `accreta-migrate` y tampoco señalaba nada. Del lado de lattice, `concepts/node.md` e `integration/bilinker.md`, que ADR-0003 reescribe.

Del lado del código quedan sin captures tres archivos, los tres a propósito: `lib.rs` (manifiesto de módulos), `bin/debug_ast.rs` (herramienta de desarrollo) y `crates/bilinker-lsp/` (no hay spec de la cual salir — ver task `x`).

## Hallazgos

Cinco, y ninguno se aceptó para dejar el árbol limpio:

| | Dónde quedó |
|---|---|
| El endpoint `task` no resuelve a ningún archivo | task `u` — deja 5 endpoints no-OK |
| La salida de `check` no es fiel a lo que `check` sabe | task `v` |
| `capture` ancla al primer nodo del tipo si no hay campo `name` | task `w` |
| `chain new` no alcanza la capa de un subsistema hermano | task `y` |
| Cobertura pendiente fuera del alcance de los ADRs | task `x` |

La contradicción sobre `hash.N` de un endpoint layer apareció por el mismo camino y se cerró en el momento: es la task `6`, adelantada del sprint 3.

## Que ninguna quede muda, medido y no supuesto

Cubrir por secciones deja un hueco: agregar una sección **nueva** al final de una spec no toca ninguna de las existentes, así que no rompe nada. Se midió archivo por archivo —agregar `## Sección de prueba` al final y correr `check`— y aparecieron cinco mudas: `commands/migrate.md`, `commands/recapture.md`, `concepts/migration.md`, `lattice/concepts/node.md` y `lattice/integration/bilinker.md`.

Se cerraron con una cadena archivo-entero ↔ archivo-entero, que es la forma que el resto de las specs ya tenía. `architecture.md` y `concepts/capture.md` llevan una también, aunque pasaban la prueba por casualidad —su última sección estaba bilinkeada—. La de `architecture.md` apunta a `lib.rs`, que así deja de ser una exclusión: el manifiesto de módulos *es* lo que "Componentes internos" describe.

Los `.capture` de archivo entero se escribieron a mano: el CLI no puede crearlos (task `y`).

Hoy las quince specs que los ADRs tocan rompen algo al editarlas.

