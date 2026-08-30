---
title: El endpoint `abstract` y el endpoint repo funcionan entre dos repos
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
parent: 1
---

# El endpoint `abstract` y el endpoint repo funcionan entre dos repos

**Lado:** desarrollo · **Decisión:** [ADR-0005](../../subsystems/bilinker/.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md)

Como quien desarrolla bilinker, quiero que un bilink cruce de un repo a otro —con una punta `abstract` de un lado y un endpoint repo del otro— y reporte drift, para tener la funcionalidad terminada y probada antes de que exista un proveedor real.

**No depende de nadie afuera.** Se ejercita entre dos repos locales cualesquiera; que el primero de verdad sea `hsi` es una circunstancia, no un requisito.

## Por qué es así

No se repite acá. Vive en [0005-frontera-entre-proyectos.md](../../subsystems/bilinker/.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md):

- Decisión 1 — El endpoint `abstract`
- Decisión 2 — El endpoint repo

## Cuándo está hecha

Los 16 escenarios de `0005-frontera-entre-proyectos-scenarios.yaml`, todos verificables con dos repos locales:

- `abstract-endpoint-is-open`
- `accept-bulk-skips-open`
- `provider-detects-own-drift`
- `remote-ok-when-accepted-pair-unchanged`
- `remote-drift-after-provider-accepts`
- `remote-ignores-cosmetic-changes`
- `remote-fan-out-is-independent`
- `remote-unreachable-is-not-an-error`
- `broken-when-remote-bilink-gone`
- `absence-taxonomy-is-five-rows`
- `rejected-when-remote-stops-being-abstract`
- `sparse-set-is-derived-and-incremental`
- `diff-deepens-on-demand`
- `consumer-stores-nothing-about-provider`
- `remote-fetch-is-one-ref`
- `frontier-needs-no-migration`
