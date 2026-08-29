---
title: Cerrar la cobertura de bilinks antes de tocar nada
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
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
