---
title: Los captures inmutables entran en la migración 002
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T04:31:51Z
parent: 9
---

# Los captures inmutables entran en la migración 002

Esta task existía para escribir `bilinker-003-immutable-captures`, la segunda de dos migraciones. **No hay segunda migración.**

## Por qué el ADR pedía dos

[ADR-0003](../../subsystems/bilinker/.stratum/impl/docs/adr/0003-formato-captures-y-aceptacion.md) daba el motivo del orden:

> La partición va primera: mientras `range`, `state` y `resolved_at` sigan dentro del `.capture`, no se le puede calcular un id estable.

Eso es cierto **si el id sale de hashear el archivo**. Sale de `H(file, query, offset)` —los tres campos— y una migración que escribe a una carpeta nueva simplemente no copia lo demás. Nunca hay un momento en que el capture tenga basura adentro, así que no hay orden que respetar.

## Lo que costaba separarlas

`002` iría de formato 1 a un intermedio: captures ya en YAML, todavía nombrados por su uuid viejo. Eso viola la invariante 1 de [`capture.md`](../../subsystems/bilinker/concepts/capture.md) —"el nombre de un capture es `H(file, query, offset)`"— así que **no es formato 2: es un formato propio**.

Y por la Decisión 2 de [ADR-0006](../../subsystems/bilinker/.stratum/impl/docs/adr/0006-formato-como-crate-versionado.md), un formato propio pide su crate, su versión registrada y su hash de esquema **para siempre**, porque el conjunto de migraciones es de sólo-agregar. Todo para un estado en el que nadie iba a estar: las dos correrían en la misma pasada, antes del corte.

## Lo que se conserva

La inspeccionabilidad que el encadenamiento buscaba está, por otro lado y mejor: la migración está partida en `plan()`, que calcula sin escribir nada, y `write()`. `--dry-run` no puede divergir de la corrida real porque es el mismo plan.

Y la verificación la hace la migración, que es lo que ADR-0006 pide: `verify()` compara hash por hash entre los dos formatos de punta a punta. Un comando que mirara dos árboles no podría — linkea un solo parser.

## Qué se tocó

La enmienda en ADR-0003, y las dos specs que nombraban la `003`: `commands/migrate.md` y `concepts/capture.md`.
