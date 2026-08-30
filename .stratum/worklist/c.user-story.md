---
title: Los bilinks salen de las ramas del proyecto
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
parent: 1
---

# Los bilinks salen de las ramas del proyecto

**Lado:** desarrollo · **Depende de:** el ítem `9` validado entero · **Decisión:** [ADR-0004](../../subsystems/bilinker/.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md)

Como responsable de un repo que no usa bilinker, quiero que no aparezca ninguna carpeta nueva en mi rama principal, para que adoptar la herramienta no sea una negociación con todo el equipo.

Es el **segundo corte** (`005`). Va sólo cuando el `004` está validado: mover los bilinks fuera de la vista de git es lo que más cuesta depurar, así que no se hace junto con un cambio de formato.

## Por qué es así

No se repite acá. Vive en [0004-bilinks-en-ref-paralela.md](../../subsystems/bilinker/.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md):

- Decisión 1 — Los bilinks viven en una ref paralela
- Decisión 2 — El corte

## Cuándo está hecha

Los escenarios de `0004-bilinks-en-ref-paralela-scenarios.yaml`:

- `project-branch-has-no-bilinks`
- `bilink-ref-not-listed-as-branch`
- `ref-snapshot-is-faithful`
- `ref-may-carry-drift`
- `act-without-new-code-has-one-parent`
- `merge-absorbs-project-without-conflict`
- `merge-back-is-detected`
- `cutover-first-absorb-is-clean`
- `ledger-written-at-cutover`
