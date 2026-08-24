# Integración con lattice

Impact consulta el grafo del proyecto a través de lattice. No lee archivos `.bilink`, no resuelve paths Stratum y no habla con language servers.

## Por qué

La versión anterior de impact escaneaba los `.bilink` por su cuenta para resolver cadenas ([blast-radius.md](../concepts/blast-radius.md) § "Cálculo"). Eso metía el formato de bilinker dentro de impact: cualquier cambio en la topología de cadena o en la resolución de paths había que arreglarlo en dos lugares.

Con lattice, impact recibe aristas ya resueltas y se queda con lo suyo: decidir si un cambio contradice una decisión declarada.

## Qué consulta

| Pregunta de impact | Consulta |
|---|---|
| ¿Qué cadenas referencian este archivo? | `lattice graph <archivo> --via bilink` |
| ¿Qué documentos gobiernan este vínculo? | `lattice graph <uuid> --via governs` |
| ¿Cuál es el blast radius de este cambio? | `lattice graph <archivo> --both --guarantee accepted` |
| ¿Qué specs alcanza el cambio subiendo por llamadas? | `lattice graph <archivo> --up --via bilink,governs,call` |
| ¿Cuál es el baseline para el diff? | el campo `commit` de la arista alcanzada |

Todas devuelven aristas con `kind`, `guarantee`, `state` y `commit`. Ver [concepts/edge.md](../../lattice/concepts/edge.md).

## Qué sigue siendo de impact

Event Collector, Git Analyzer, Skill Runner, Report Builder y Thread Manager no cambian. El Chain Resolver y el Impact Element Finder se reemplazan por las dos primeras consultas de la tabla.

El Impact Element Finder dependía de un índice de backlinks en `.bilink/.index` que nunca existió. Las aristas `governs` de lattice son no dirigidas, así que el backlink es recorrerlas en cualquier sentido — no hay índice que construir.

## Garantía

Un Impact Report no debe presentar una inferencia del LSP como si fuera un vínculo verificado. La regla:

- `accepted` → sostiene una afirmación de drift; hay estado anterior aceptado.
- `derived` → sugiere revisión; el call graph falla con dispatch dinámico, macros y callbacks.
- `asserted` → contexto, sin verificación.

## Degradación

Todo resultado de lattice lleva el estado de cada proveedor. Impact lo propaga al Impact Report: uno generado sin el proveedor LSP vio menos grafo, y el lector tiene que poder distinguirlo de uno completo.

Un reporte que no distingue "no encontré impacto" de "no pude buscarlo" es peor que no tener reporte.
