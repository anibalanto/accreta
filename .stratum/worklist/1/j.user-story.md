---
title: `hsi` publica una abstracción sin cambiar su `main`
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
---

# `hsi` publica una abstracción sin cambiar su `main`

**Lado:** adopción · **Depende de:** la US 06

Como equipo de `hsi`, quiero declarar que un fragmento de mi API pública es un punto de anclaje, sin que eso cambie un byte de mi rama principal ni me obligue a saber quién lo consume.

**Depende de lamansys, no de nosotros.** Es el primer bloqueo externo de la épica, y por eso está aislado en su propia US en vez de escondido adentro de una task.

## Por qué es así

[ADR-0005](../../../subsystems/bilinker/.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md) Decisión 1 (el endpoint `abstract`) y [ADR-0004](../../../subsystems/bilinker/.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md) (que es lo que hace que `main` no cambie).

## Cuándo está hecha

No tiene escenarios propios: el comportamiento ya está cubierto por la US 06. Está hecha cuando `hsi` tiene su `refs/bilink/rc-2.32` con un bilink abstracto publicado, y `git diff` sobre su `main` antes y después no muestra nada.
