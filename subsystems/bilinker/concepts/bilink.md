# Especificación: Formato del archivo `.bilink`

## Ubicación y nomenclatura

Los archivos `.bilink` viven en carpetas `.bilink/` dentro de cada layer del proyecto.
El nombre del archivo es un **UUID v4** — es simultáneamente el identificador de la
cadena a la que pertenece el bilink y el mecanismo de localización entre layers.

```
bilinker/
  .bilink/
    7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink   ← tip (spec layer)
  .stratum/
    tech-decisions/
      .bilink/
        7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink   ← mid
    impl/
      .bilink/
        7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink   ← tip (impl layer)
```

El mismo UUID aparece en todas las layers que participan de una cadena.

## Tipos de endpoint

El tipo de un endpoint se infiere del valor de `link.N` — no hay prefijo de tipo:

| Tipo | Forma | Descripción |
|---|---|---|
| **Estructural** | `file [:: query [:: start~end]]` | Apunta a un archivo o fragmento concreto. |
| **Layer** | path Stratum | Apunta a un nodo adyacente en otra capa. |
| **Todo** | `todo` | Placeholder — el extremo aún no existe. |
| **Bilink** | `.bilink/<uuid>.bilink` | Apunta a otro bilink por UUID. |

Un endpoint estructural puede apuntar a un nodo AST específico (`:: query`) o al
archivo completo (sin query).

Los endpoints layer usan el lenguaje de paths Stratum (ver especificación en
[Stratum — lenguaje de paths](../../stratum/specific/paths.md)).

Un endpoint bilink referencia un archivo `.bilink` por su path relativo a la raíz
de la layer actual. El tipo se reconoce por el prefijo `.bilink/` en el valor.

### Resolución de un endpoint layer

Dado `link.N: <stratum-path>` en el archivo `<current-layer>/.bilink/<uuid>.bilink`:

1. Resolver el path Stratum tomando como base la raíz de la layer actual.
2. Usar el path resultante como `<layer-path>` en la fórmula:

```
resolved = ../<layer-path>/.bilink/<uuid>.bilink
```

El `../` sube del directorio `.bilink/` a la raíz de la layer actual. La carpeta
`.bilink/` nunca aparece en el valor de `link.N` — siempre es implícita.

## Topología

### Link directo (misma capa)

Un bilink puede conectar dos fragmentos dentro de la misma capa: ambos `link.N`
son endpoints estructurales. Hay un único archivo `.bilink` — no hay chain traversal.

```
[fragmento A] ←→ [fragmento B]
```

### Cadena entre capas

Una **cadena** es una secuencia lineal de bilinks con el mismo UUID que conecta
dos fragmentos a través de una o más capas. Los tipos de nodo en la cadena son:

- **tip**: un endpoint estructural + un endpoint layer. Son los extremos de la cadena.
  Siempre hay exactamente dos tips por cadena.
- **mid**: ambos endpoints son layer paths. Puede haber cero o más mids.

```
[fragmento] ←→ tip ←→ mid* ←→ tip ←→ [fragmento]
```

La topología es estrictamente lineal — sin ciclos ni bifurcaciones.

### Bilink abierto (intención pendiente)

Un bilink con un endpoint `todo` declara que un fragmento *debería* estar
conectado a algo que aún no existe. Es una intención pendiente, no un error.

```
[fragmento spec] ←→ todo
```

Creado por `worklist new ... capture`. Completado por `worklist done ... capture`,
que reemplaza `todo` con el endpoint real.

## Estructura del archivo

```
link.0: <endpoint>
link.1: <endpoint>

# Semántica (opcionales)
kind:   <tipo-de-relación>
name.0: <etiqueta-del-extremo-0>
name.1: <etiqueta-del-extremo-1>

# Cache
hash.0: <sha256>
commit.0: <sha1>
range.0: <start~end-bytes-absolutos>
hash.1: <sha256>
commit.1: <sha1>
range.1: <start~end-bytes-absolutos>
state.0: <estado>
state.1: <estado>
resolved_at: <iso8601-timestamp>
```

No existe campo `id`: el UUID del nombre de archivo es el identificador.
`range.N` es opcional.
`hash.N` y `commit.N` están ausentes hasta que el endpoint es aceptado por primera vez.
`kind`, `name.0` y `name.1` son opcionales. Presentes cuando el bilink tiene semántica
declarada más allá del vínculo estructural.

## Campos semánticos

### `kind`

Clasifica la relación que el bilink representa. Valor libre; valores definidos:

| Valor | Significado |
|-------|-------------|
| *(ausente)* | Vínculo estructural — relación de implementación entre fragmentos |
| `impact` | Decisión o documento que gobierna/afecta un vínculo entre capas |

Otros valores posibles en el futuro: `validates`, `documents`, `depends-on`, etc.

### `name.N`

Etiqueta opcional para el rol semántico del endpoint N en la relación declarada por
`kind`. Texto libre. Útil para herramientas de traversal y agentes de IA.

```
name.0: architecture-decision
name.1: spec-impl-bridge
```

## Campos de la cache

### `hash.N`

SHA-256 del contenido del endpoint N en el momento en que fue aceptado.

| Tipo de endpoint | `hash.N` |
|---|---|
| **Estructural** | SHA-256 del fragmento o archivo referenciado |
| **Layer** | SHA-256 del archivo `.bilink` adyacente completo |

Ausente cuando el endpoint nunca fue aceptado (estado `PENDING`).
Establecido por `bilinker accept`, sobrescrito en cada nueva aceptación.

### `commit.N`

SHA-1 del commit HEAD en el repo correspondiente al momento de la aceptación.

| Tipo de endpoint | repo |
|---|---|
| **Estructural** | repo donde vive el archivo referenciado |
| **Layer** | repo donde vive el `.bilink` adyacente |

Ausente cuando `hash.N` está ausente. Siempre presente junto a `hash.N`.

### `range.N`

Solo para endpoints estructurales. Byte range absoluto (`start~end`) del fragmento
en el archivo referenciado, correspondiente al último estado aceptado.

Permite que `bilinker get <file>:<line>:<col>` localice endpoints sin re-ejecutar
queries tree-sitter.

### `state.N`

Estado de consistencia del endpoint N. Calculado por `bilinker check`, persistido
en el archivo. Ver tablas completas de estados en la sección siguiente.

### `resolved_at`

Timestamp ISO 8601 UTC del último `check`.

## Estados y transiciones

### Endpoint estructural

| Estado | Significado | Cómo se llega | Cómo se sale |
|--------|-------------|---------------|--------------|
| `PENDING` | `hash.N` ausente | `chain new` / `capture` | `bilinker accept` |
| `OK` | Hash actual del fragmento == `hash.N` | `bilinker accept` | Cambio en el archivo |
| `ALTERED` | Archivo existe, query matchea, hash ≠ `hash.N` | `bilinker check` | `bilinker accept` |
| `DISPLACED` | Hash encontrado en posición diferente del mismo nodo AST | `bilinker check` | `bilinker accept` |
| `UNANCHORED` | Query ya no matchea ningún nodo | `bilinker check` | `bilinker accept` (o re-capture) |
| `DELETED` | Archivo no existe | `bilinker check` | Restaurar archivo + `accept` · o · `bilinker remove` |
| `BROKEN` | Error de lectura o parseo del archivo | `bilinker check` | Reparar archivo + `accept` · o · `bilinker remove` |

### Endpoint layer

| Estado | Significado | Cómo se llega | Cómo se sale |
|--------|-------------|---------------|--------------|
| `PENDING` | `hash.N` ausente | `chain new` | `bilinker accept` |
| `OK` | Hash actual del `.bilink` adyacente == `hash.N` | `bilinker accept` | Cambio en el nodo adyacente |
| `CHAIN_DIRTY` | `.bilink` adyacente existe pero su hash ≠ `hash.N` | `bilinker check` | `bilinker accept` |
| `BROKEN` | `.bilink` adyacente no existe | `bilinker check` | Crear nodo adyacente + `accept` · o · `bilinker remove` |

### Endpoint bilink

| Estado | Significado | Cómo se llega | Cómo se sale |
|--------|-------------|---------------|--------------|
| `PENDING` | `hash.N` ausente | creación | `bilinker accept` |
| `OK` | Hash actual del `.bilink` referenciado == `hash.N` | `bilinker accept` | Cambio en el bilink referenciado |
| `CHAIN_DIRTY` | `.bilink` referenciado existe pero su hash ≠ `hash.N` | `bilinker check` | `bilinker accept` |
| `UNREACHABLE` | `.bilink` referenciado no existe localmente | `bilinker check` | Obtener la layer + `accept` |

### Endpoint todo

| Estado | Significado | Cómo se llega | Cómo se sale |
|--------|-------------|---------------|--------------|
| `TODO` | Placeholder — el extremo aún no existe | `worklist new ... capture` | `worklist done ... capture` |

Un endpoint `todo` nunca tiene `hash.N`, `commit.N` ni `range.N`. `bilinker check`
reporta `TODO` sin considerarlo un error. `bilinker accept` no aplica sobre `todo`.

`bilinker remove` elimina el archivo `.bilink` del nodo actual. Los nodos
adyacentes detectarán `BROKEN` en el próximo `check` y deberán también decidir:
reparar o remover. La remoción se propaga hop a hop por la cadena.

## Propagación reactiva y hash chain

Toda la cadena forma un **hash chain distribuido**: cada nodo ancla
criptográficamente al estado aprobado de sus vecinos inmediatos.

Cuando se ejecuta `bilinker accept` en un endpoint:
1. `hash.N` y `commit.N` del archivo `.bilink` se actualizan.
2. El contenido del archivo cambia → su SHA-256 cambia.
3. El nodo adyacente, en el próximo `check`, detecta que el hash actual del
   `.bilink` vecino ≠ `hash.N` → `CHAIN_DIRTY`.
4. Para restaurar `OK`, el nodo adyacente debe ejecutar `bilinker accept`.

Esta propagación garantiza que **ningún nodo puede cambiar su estado aceptado sin
que todos los nodos adyacentes lo detecten**.

## Semántica de parseo

- Cada clave aparece como máximo una vez en el archivo.
- Las líneas de continuación (que no comienzan con clave reconocida) se concatenan
  al valor anterior con un espacio. Solo aplica a `link.N`.
- Claves reconocidas: `link.0:`, `link.1:`, `kind:`, `name.0:`, `name.1:`,
  `hash.0:`, `commit.0:`, `hash.1:`, `commit.1:`, `range.0:`, `range.1:`,
  `state.0:`, `state.1:`, `resolved_at:`.
- Líneas que comienzan con `#` son comentarios y se ignoran.
- El archivo usa codificación UTF-8 sin BOM.

## Ejemplo: bilink de impacto

Un documento de decisión que gobierna un vínculo entre capas:

```
# .bilink/f9a1b2c3-0000-0000-0000-000000000000.bilink
link.0: docs/adr/design-voting-machine.md
link.1: .bilink/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink

kind:   impact
name.0: architecture-decision
name.1: spec-impl-bridge

hash.0: b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2
commit.0: d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3
hash.1: e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6
commit.1: f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6
state.0: OK
state.1: OK
resolved_at: 2026-05-29T09:00:00Z
```

Esto crea una relación ternaria implícita:

```
docs/adr/design-voting-machine.md
        ↕ (f9a1b2c3 — impact)
specs/voting.yaml ↔ impl/Voting.java
        (7f3d8e9a)
```

## Ejemplo: bilink abierto (intención pendiente)

```
# .bilink/a3f9c821-4e5b-4c3d-9f2a-1b2c3d4e5f6a.bilink
link.0: specs/voting.yaml :: (block_mapping_pair
  key: (flow_node) @n0 (#eq? @n0 "impl")
  value: (_) @target)
link.1: todo

# Cache
hash.0: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
commit.0: d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3
range.0: 312~358
state.0: OK
state.1: TODO
resolved_at: 2026-05-29T09:00:00Z
```

Creado por `worklist new task "implementar vote" capture specs/voting.yaml:104:1`.
Cuando el trabajo termina, `worklist done 3 capture src/Persona.java:45:1` reemplaza
`link.1: todo` con el endpoint real y el bilink queda completo.

## Ejemplo completo: cadena de 3 nodos spec → tech-decisions → impl

```
# .bilink/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink   (tip — spec layer)
link.0: specs/voting.yaml :: (block_mapping_pair
  key: (flow_node) @n0 (#eq? @n0 "impl")
  value: (_) @target)
link.1: .stratum/tech-decisions

# Cache
hash.0: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
commit.0: d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3
range.0: 312~358
hash.1: e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6
commit.1: f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6
state.0: OK
state.1: OK
resolved_at: 2026-05-27T10:00:00Z
```

```
# .stratum/tech-decisions/.bilink/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink   (mid)
link.0: ../..
link.1: ../impl

# Cache
hash.0: c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0
commit.0: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
hash.1: a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4
commit.1: b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1
state.0: OK
state.1: OK
resolved_at: 2026-05-27T10:00:00Z
```

```
# .stratum/tech-decisions/.stratum/impl/.bilink/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink   (tip — impl layer)
link.0: ../tech-decisions
link.1: src/main/java/ar/example/demo/persona/Persona.java :: (class_declaration
  name: (identifier) @n0 (#eq? @n0 "Persona")
  body: (class_body
    (method_declaration
      name: (identifier) @n1 (#eq? @n1 "vote")) @target))

# Cache
hash.0: e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8
commit.0: b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1
hash.1: 479922a1ee55cc7f9f4f323bb002018e1b4e1cda65e069e0f6f4645926ce25ee
commit.1: c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2
range.1: 245~389
state.0: OK
state.1: OK
resolved_at: 2026-05-27T10:00:00Z
```

## Invariantes

1. El nombre del archivo es un UUID v4 válido con extensión `.bilink`.
2. Siempre existen `link.0` y `link.1`.
3. Un bilink de misma capa tiene dos endpoints estructurales. Una cadena entre capas tiene exactamente dos tips. Un bilink abierto tiene un endpoint estructural y uno `todo`.
4. `hash.N` y `commit.N` están siempre presentes juntos o ausentes juntos.
5. `hash.N` de un endpoint estructural: SHA-256 del fragmento referenciado.
6. `hash.N` de un endpoint layer: SHA-256 del archivo `.bilink` adyacente completo.
7. `hash.N` de un endpoint bilink: SHA-256 del archivo `.bilink` referenciado completo.
8. Un endpoint `todo` nunca tiene `hash.N`, `commit.N` ni `range.N`.
9. `state.N = OK` si y solo si el hash actual de lo apuntado == `hash.N`.
10. `state.N` siempre está presente en la cache una vez que el bilink fue verificado.
11. Si existe cualquier campo de cache, debe existir `resolved_at`.
12. `resolved_at` es siempre UTC (`YYYY-MM-DDTHH:MM:SSZ`).
13. La topología de la cadena es lineal — sin ciclos ni bifurcaciones.
14. Solo se puede crear un bilink sobre archivos con historial git.
15. `kind` y `name.N` son independientes de la cache — no afectan `hash.N` ni `state.N`.
