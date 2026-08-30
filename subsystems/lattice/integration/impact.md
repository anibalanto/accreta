# Integración con impact

Impact es consumidor de lattice. No lee bilinks ni habla con language servers: consulta el grafo y evalúa.

## Reparto

```
bilinker   detecta drift          → estado por endpoint
lattice    agrega y recorre       → qué está conectado con qué, y con qué garantía
impact     evalúa coherencia      → si el cambio contradice una decisión declarada
```

Lattice responde preguntas estructurales — *qué está conectado, por dónde, con qué garantía*. Impact responde la semántica — *si ese cambio es aceptable dado lo que se había decidido*. Lattice no tiene opinión sobre el contenido; impact no recorre grafos.

## Lo que impact deja de implementar

De los siete componentes de [su arquitectura](../../impact/architecture.md), dos desaparecen:

| Componente | Antes | Ahora |
|---|---|---|
| **Chain Resolver** | Escanea los bilinks de la capa buscando los que referencian un archivo. | `lattice graph <archivo> --via bilink` |
| **Impact Element Finder** | Consulta un índice de backlinks en `.bilink/.index`. | `lattice graph <uuid> --via governs` |

El resto —Event Collector, Git Analyzer, Skill Runner, Report Builder, Thread Manager— es trabajo propio de impact y no cambia.

El caso del Impact Element Finder vale la pena: dependía de `.bilink/.index`, un índice de backlinks que **nunca existió** (el índice real de bilinker es `.bilink/index/index`, archivo→endpoint, sin backlinks). Con lattice el problema se disuelve sin construir nada: las aristas `governs` son no dirigidas, así que un backlink es recorrerla en cualquier sentido.

## Blast radius como consulta

El [blast radius](../../impact/concepts/blast-radius.md) —"el conjunto de cadenas afectadas directa o indirectamente"— es un traversal del grafo agregado:

```bash
lattice graph <archivo> --both --guarantee accepted --format json
```

La definición no cambia; cambia quién la calcula. Y gana un nivel que antes no tenía: con el proveedor LSP disponible,

```bash
lattice graph <archivo> --up --via bilink,governs,call --format json   # governs: pendiente del proveedor
```

alcanza también las specs que gobiernan funciones que *llaman* al código modificado, no solo las que lo referencian directamente. Esas aristas llegan marcadas `derived`, y el reporte debe distinguirlas.

## Garantía y peso en el reporte

Un Impact Report que trate por igual una arista verificada y una inferencia de un language server pierde credibilidad. La correspondencia es directa:

| `guarantee` | Qué puede afirmar el reporte |
|---|---|
| `accepted` | *"Esto está roto"* — hay un estado anterior aceptado contra el cual comparar. |
| `derived` | *"Esto podría estar afectado"* — inferencia de una herramienta que falla con dispatch dinámico, macros y callbacks. |
| `asserted` | *"Alguien escribió que esto se relaciona"* — sin verificación. |

Solo las `accepted` llevan `state` y `commit`, y por eso solo ellas sostienen una afirmación de drift.

## Baseline

El baseline de todo diff sale del campo `commit` de la arista `accepted` alcanzada — nunca de un ref externo ni de `HEAD~n`:

```
git log <edge.commit[n]>..HEAD -- <archivo>
```

Es lo que hace hoy el Git Analyzer, leyendo el commit aceptado por su cuenta. Cambia de dónde viene el dato, no el cálculo.

El dato es el commit en que el contenido aceptado quedó establecido; lattice lo transporta como `commit` en la arista, así que impact ya no lo busca por su cuenta.

## Degradación

Todo resultado de lattice lleva el estado de cada proveedor. Impact debe propagarlo al Impact Report: un reporte generado sin el proveedor LSP vio menos grafo, y quien lo lea tiene que poder saberlo.

Un reporte que no distingue "no encontré impacto" de "no pude buscarlo" es peor que no tener reporte.

## Escritura

Ninguna. Lattice es solo lectura y no dispara `check`; impact escribe únicamente bajo `.impact/`. La aceptación de un cambio sigue siendo `bilinker accept`, invocado por una persona.
