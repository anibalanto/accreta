---
title: El trabajo diario sobre la ref no requiere pensar en ella
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T22:35:03Z
parent: 1
---

# El trabajo diario sobre la ref no requiere pensar en ella

**Lado:** desarrollo · **Decisión:** [ADR-0004](../../subsystems/bilinker/.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md)

Como quien trabaja todos los días en el repo, quiero cambiar de rama, rebasear y aceptar sin acordarme de que los bilinks viven aparte, para que la ref no sea un impuesto cognitivo.

## Por qué es así

No se repite acá. Vive en [0004-bilinks-en-ref-paralela.md](../../subsystems/bilinker/.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md):

- Decisión 1 § `bilinker init`
- Decisión 1 § Cambiar de rama
- Decisión 1 § `bilinker track`
- Decisión 1 § Cuando la rama se rebasea

## Cuándo está hecha

Los escenarios de `0004-bilinks-en-ref-paralela-scenarios.yaml`:

- `init-is-required-and-explicit`
- `fetch-brings-both-after-init`
- `branch-switch-rematerializes`
- `branch-switch-refuses-on-dirty-bilinks`
- `cache-invalidates-on-branch-change`
- `track-picks-newest-reachable`
- `track-refuses-when-ambiguous`
- `adopt-brings-neighbour-decisions`
- `adopt-dry-run-is-inert-and-exact`
- `adopt-converges-without-conflict`
- `bilinker-has-its-own-status`
- `check-runs-hot`
