---
title: El formato vive en un crate y su versión se verifica sola
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T02:23:45Z
parent: 1
---

# El formato vive en un crate y su versión se verifica sola

**Lado:** desarrollo · **Decisión:** [ADR-0006](../../subsystems/bilinker/.stratum/impl/docs/adr/0006-formato-como-crate-versionado.md)

Como quien mantiene bilinker, quiero que cambiar el formato sea imposible sin subir su versión, para que el guard que protege a los binarios viejos y a los consumidores de la frontera no dependa de que alguien se acuerde.

## Por qué es así

Vive en [ADR-0006](../../subsystems/bilinker/.stratum/impl/docs/adr/0006-formato-como-crate-versionado.md):

- Decisión 1 — El formato es un crate, y su versión es la del formato
- Decisión 2 — Las migraciones viven al lado, y pinean dos versiones
- Decisión 3 — El esquema se publica, y es lo que la frontera consume

## Cuándo está hecha

Cambiar los tipos sin subir la versión **falla un test**. El esquema JSON se genera y se publica como artefacto de la release. Y una migración puede leer los dos formatos que puentea, que es lo que hace posible verificar que no perdió nada.

## Tasks

- `s` — extraer el crate de formato (parte `capture.rs`, que hoy mezcla formato y captura)
- `t` — el test de hash del esquema contra la versión registrada

## Cómo se verificó

Cambiar los tipos sin subir la versión **falla un test**, probado de dos formas: renombrando un valor serializado y agregando un tipo de endpoint. La segunda es la que importa, porque es el cambio aditivo que ADR-0006 pone como su motivo, y la primera versión del guard **no la detectaba** — ver la task `t`.

El esquema se genera con `cargo run -p bilink-format --bin schema`. Publicarlo como artefacto de release es trabajo de CI, que todavía no existe.

Queda una parte del enunciado sin hacer, y está dicho en la task `s`: **una migración todavía no depende de los dos formatos que puentea**, porque hoy hay un solo crate de formato. Se resuelve en el sprint 3, con el primer bump real.
