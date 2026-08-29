---
title: Cerrar la cobertura de bilinks antes de tocar nada
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
---

# Cerrar la cobertura de bilinks antes de tocar nada

Crear los bilinks que faltan para que reescribir una spec rompa algo que lleve al código. **Es la única tarea de la épica donde hay que buscar código a mano** — y se hace una vez, precisamente para no volver a hacerlo.

## Los agujeros medidos

Del lado de las specs, sobre los 63 bilinks de la raíz:

| Spec que los ADRs reescriben | Bilinks hoy |
|---|---|
| `concepts/capture.md` | **0** — es la que ADR-0003 más reescribe |
| `commands/migrate.md` | **0** |
| `architecture.md` | **0** |
| `concepts/index.md` | **0** |
| `concepts/bilink.md` | 1, en un archivo de 300+ líneas |
| `commands/check.md` | 7 — bien cubierta |
| `commands/accept.md` · `capture.md` · `chain.md` · `get.md` | 3-4 — aceptable |

Del lado del código, sobre los 47 captures de la capa impl:

| Archivo | Captures |
|---|---|
| `check.rs` | 13 |
| `accept.rs` | 3 · `bilinker-cli/src/main.rs` 4 · `index.rs` 3 |
| `bilink.rs` | 2 · `get.rs` 2 · `chain.rs` 2 |
| `capture.rs` · `link.rs` · `grammar.rs` · `config.rs` | 1 |
| **`apply.rs`** | **0** |
| `task.rs` · `migrations.rs` · `hash.rs` · `git.rs` · `query.rs` | **0** |

## Dónde apuntar — pista, no instrucción

Esta lista salió de grepear el código a mano, y **sirve sólo para saber dónde falta un bilink**. No es la lista de cambios a hacer: eso lo dicta lo que se rompa. Si al terminar esta tarea un archivo de acá no aparece por ningún bilink, es que falta uno; si aparece algo que no está acá, mejor.

Los ADRs esperan tocar: `bilink.rs` (las claves del formato y su escritura), `capture.rs` (el id y el escaneo de equivalentes), `accept.rs` (qué escribe y contra qué falla), `apply.rs` (el fork por tipo de fix), `check.rs` (las comparaciones y los estados), `task.rs` (la extensión del ítem de worklist), y `migrations.rs` (las dos migraciones nuevas).

## Cuándo está hecha

> **Si una especificación tiene implementación, está bilinkeada.**

No es "todo bilinkeado". Buena parte de una spec —el porqué de una decisión, una alternativa descartada, un ejemplo— no tiene código que la implemente, y vincularla no detecta nada porque no hay contra qué derivar. Y sobre-cubrir tiene un costo propio: si todo estuviera vinculado, editar una spec produciría cientos de no-OK y **el inventario dejaría de servir como inventario** — la respuesta racional pasaría a ser `accept .` a ciegas, que es justo lo que ADR-0003 prohíbe.

El criterio es al revés: **partir del código**. Para cada cosa que la implementación hace, la spec que la describe tiene que llegar hasta ahí por un bilink. Un archivo con siete afirmaciones verificables lleva siete — `commands/check.md` ya tiene 7, y está bien así.

**El cruce hay que hacerlo**: recorrer los comandos del CLI y los módulos del core contra las specs que los describen, y anotar dónde no hay bilink. La lista de arriba es el punto de partida medido, no el alcance.
