# Propuesta: endpoint de tipo bilink

**Estado:** especificado, no implementado. Vive acá hasta que alguien lo implemente.

Un cuarto tipo de endpoint, que apunta a **otro bilink** en vez de a un fragmento, una capa o una tarea.

```
| Tipo | Forma | Descripción |
| **Bilink** | `.bilink/<uuid>.bilink` | Apunta a otro bilink por UUID. Se trata como un archivo estructural. |
```

Referencia un archivo `.bilink` por su path relativo a la raíz de la capa actual. El tipo se reconoce por el prefijo `.bilink/` en el valor.

## Por qué existe

**Porque un bilink no puede hablar de una relación, sólo de sus extremos.** El fan-out del capture da una estrella: *"D se relaciona con A"* y *"D se relaciona con B"*, por separado. No dice *"D gobierna el vínculo entre A y B"*. Para eso hace falta que un endpoint apunte al bilink mismo, y no a ninguna de sus dos puntas.

Ver [`concepts/capture.md`](../concepts/capture.md) § "El fan-out vive del lado del capture", que es donde el límite queda enunciado.

## Los dos casos de uso

### Gobernanza — `kind: governs`

Un documento de decisión que gobierna un vínculo entre capas.

```
# .bilink/f9a1b2c3-0000-0000-0000-000000000000.bilink
link.0: docs/adr/design-voting-machine.md
link.1: .bilink/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink

kind:   governs
name.0: architecture-decision
name.1: spec-impl-bridge
```

Crea una relación ternaria implícita:

```
docs/adr/design-voting-machine.md
        ↕ (f9a1b2c3 — governs)
specs/voting.yaml ↔ impl/Voting.java
        (7f3d8e9a)
```

`kind` y `name.N` **no** son parte de esta propuesta: viven en [`concepts/bilink.md`](../concepts/bilink.md) § "Campos semánticos". Lo que falta para que `governs` funcione es sólo el endpoint: sin él, `link.1` no puede apuntar a un vínculo.

### Bilink de tarea

Un bilink que conecta un bilink estructural con un ítem del worklist. Vive en la capa donde se debe ejecutar la tarea.

```
# .bilink/a3f9c821-4e5b-4c3d-9f2a-1b2c3d4e5f6a.bilink
link.0: .bilink/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink   ← bilink estructural como archivo
link.1: issue 3a                                               ← ítem en worklist
```

`bilinker check` hashea el archivo de tarea (`<project-root>/.worklist/insecure/all/3a.task.md`) como cualquier endpoint estructural — detecta si el contenido de la tarea cambia. Si el fragmento cambia, `state.0` reporta el cambio del bilink estructural referenciado.

**El endpoint `issue <id>` sí está implementado**; lo que falta es el otro extremo. La motivación de colgar la tarea de *la relación* y no de uno de sus extremos: una tarea que nace de "esta spec no coincide con esta implementación" no es sobre la spec ni sobre el código, es sobre el desacuerdo entre los dos.

## Estados

| Estado | Significado | Cómo se llega | Cómo se sale |
|--------|-------------|---------------|--------------|
| `PENDING` | sin estado aceptado | creación | `bilinker accept` |
| `OK` | hash actual del `.bilink` referenciado == el aceptado | `bilinker accept` | cambio en el bilink referenciado |
| `CHAIN_DIRTY` | el `.bilink` referenciado existe pero su hash difiere | `bilinker check` | `bilinker accept` |
| ausente | el `.bilink` referenciado no existe localmente | `bilinker check` | obtener la capa + `accept` |

La última fila tenía el nombre `UNREACHABLE`, que ADR-0003 elimina del vocabulario por estar sobrecargado. Cuando esta propuesta vuelva, hay que darle un nombre que no colisione con `LAYER_UNREACHABLE`, `REMOTE_UNREACHABLE` ni `BROKEN`.

## Invariantes propias

1. El hash aceptado de un endpoint bilink es el SHA-256 del archivo `.bilink` referenciado completo. **Ojo:** ADR-0003 descarta esa lectura para los endpoints layer y repo, que copian el valor estructural del vecino. Al volver, hay que decidir si el endpoint bilink es la excepción o se alinea.
2. Un bilink de tarea conecta un endpoint bilink con un endpoint `issue <id>`.

## Qué depende de esto

No es una idea suelta: hay specs escritas encima.

- **impact** — `overview.md` y `concepts/impact-element.md` definen un elemento de impacto como *"un bilink con `kind: governs`"*, cuyo `link.1` es siempre un path `.bilink/<uuid>.bilink`. `concepts/skills.md` activa skills por ese `kind`, y `architecture.md` tiene el Impact Element Finder.
- **worklist** — `concepts/bilink-tasks.md` e `integration/bilinker.md` describen el bilink de tarea entero.
- **lattice** — `concepts/edge.md`, `concepts/provider.md`, `commands/graph.md` e `integration/bilinker.md` cuentan con las aristas `governs` y `bilink`.

Todas quedan describiendo algo que todavía no existe, que es exactamente lo que este directorio significa. Ninguna es una referencia rota: es una referencia correcta a una propuesta.
