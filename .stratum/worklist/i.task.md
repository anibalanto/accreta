---
title: Especificar e implementar el endpoint `abstract` y el endpoint repo
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
parent: h
---

# Especificar e implementar el endpoint `abstract` y el endpoint repo

Spec primero: `concepts/configuration.md` (deja de ser cierto que no hay ningún archivo de configuración), `concepts/bilink.md` y `reference.md` (los dos tipos nuevos), los comandos `get`, `graph` y `chain`, y `scenarios/frontier.yaml`. En Stratum, `sublayer-config.md`.

Después el código, por los bilinks que se rompan. No requiere ningún repo ajeno: se ejercita entre dos repos locales.

**La verificación de versión en la frontera.** El consumidor lee el `.bilink/version` que viene adentro de la ref del proveedor y se niega si no lo entiende, en vez de malinterpretar. Es la razón de fondo del campo: son repos con ciclos de release independientes, así que la divergencia de versiones no es un accidente sino lo normal. Y arrastra algo contraintuitivo que hay que dejar escrito: **un cambio aditivo también sube la versión** — `link.1: abstract` no rompe al que escribe, pero un parser viejo lo lee como un path de capa, en silencio.
