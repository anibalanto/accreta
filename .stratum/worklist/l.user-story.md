---
title: `retinar` detecta que `hsi` cambió lo que publica
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
parent: 1
---

# `retinar` detecta que `hsi` cambió lo que publica

**Lado:** adopción · **Depende de:** la US 07

Como quien mantiene `retinar`, quiero enterarme de que `hsi` movió o cambió lo que publica antes de que rompa en runtime — que es como nos enteramos hoy.

Es el criterio de terminado de la épica.

## Por qué es así

[ADR-0005](../../subsystems/bilinker/.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md) Decisión 2 — El endpoint repo.

## Cuándo está hecha

Sin escenarios propios: el mecanismo lo cubre la US 06. Está hecha cuando `retinar` reporta el drift de `USER_PERMISSIONS` contra `hsi`, y el reporte distingue *movió* de *cambió el contenido*.
