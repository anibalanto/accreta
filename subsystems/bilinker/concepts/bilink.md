# Especificación: Formato del archivo `.bilink`

## Ubicación y nomenclatura

Los archivos `.bilink` viven en carpetas `.bilink/` dentro de cada layer del proyecto. El nombre del archivo es un **UUID v4** — es simultáneamente el identificador de la cadena a la que pertenece el bilink y el mecanismo de localización entre layers.

```
bilinker/
  .bilink/
    7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink   ← tip (spec layer)
  .stratum/
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
| **Task** | `task <id>` | Apunta a un ítem del worklist en `<project-root>/.stratum/worklist/<id>.task`. Se hashea y verifica como un endpoint estructural. |
| **Bilink** | `.bilink/<uuid>.bilink` | Apunta a otro bilink por UUID. Se trata como un archivo estructural. |

Un endpoint estructural puede apuntar a un nodo AST específico (`:: query`) o al archivo completo (sin query).

Los endpoints layer usan el lenguaje de paths Stratum (ver especificación en [Stratum — lenguaje de paths](../../stratum/specific/paths.md)).

Un endpoint bilink referencia un archivo `.bilink` por su path relativo a la raíz de la layer actual. El tipo se reconoce por el prefijo `.bilink/` en el valor.

### Resolución de un endpoint layer

Dado `link.N: <stratum-path>` en el archivo `<current-layer>/.bilink/<uuid>.bilink`:

1. Resolver el path Stratum tomando como base la raíz de la layer actual.
2. Usar el path resultante como `<layer-path>` en la fórmula:

```
resolved = ../<layer-path>/.bilink/<uuid>.bilink
```

El `../` sube del directorio `.bilink/` a la raíz de la layer actual. La carpeta `.bilink/` nunca aparece en el valor de `link.N` — siempre es implícita.

## Topología

### Link directo (misma capa)

Un bilink puede conectar dos fragmentos dentro de la misma capa: ambos `link.N` son endpoints estructurales. Hay un único archivo `.bilink` — no hay chain traversal.

```
[fragmento A] ←→ [fragmento B]
```

### Cadena entre capas

Una **cadena** es una secuencia lineal de bilinks con el mismo UUID que conecta dos fragmentos a través de una o más capas. Los tipos de nodo en la cadena son:

- **tip**: un endpoint estructural + un endpoint layer. Son los extremos de la cadena.
  Siempre hay exactamente dos tips por cadena.
- **mid**: ambos endpoints son layer paths. Puede haber cero o más mids.

```
[fragmento] ←→ tip ←→ mid* ←→ tip ←→ [fragmento]
```

La topología es estrictamente lineal — sin ciclos ni bifurcaciones.

### Bilink con layer no creada todavía

Un bilink puede declarar un endpoint layer apuntando a una capa que aún no existe. El estado `TODO` indica que la conexión está planeada pero la capa destino no fue creada — no es un error.

```
link.0: spec/voting.yaml
link.1: .stratum/impl        ← layer que aún no existe
state.1: TODO                ← calculado por bilinker check
```

Una vez creada la capa y aceptado el endpoint, el estado pasa a `OK`.

### Bilink de tarea

Un bilink puede conectar un bilink estructural con un ítem del worklist. Vive en la capa donde se debe ejecutar la tarea.

```
link.0: .bilink/<uuid-estructural>.bilink   ← bilink estructural como archivo
link.1: task 3a                             ← ítem en worklist
```

`bilinker check` hashea el archivo de tarea (`<project-root>/.stratum/worklist/3a.task`) como cualquier endpoint estructural — detecta si el contenido de la tarea cambia.

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

No existe campo `id`: el UUID del nombre de archivo es el identificador. `range.N` es opcional. `hash.N` y `commit.N` están ausentes hasta que el endpoint es aceptado por primera vez. `kind`, `name.0` y `name.1` son opcionales. Presentes cuando el bilink tiene semántica declarada más allá del vínculo estructural.

## Campos semánticos

### `kind`

Clasifica la relación que el bilink representa. Valor libre; valores definidos:

| Valor | Significado |
|-------|-------------|
| *(ausente)* | Vínculo estructural — relación de implementación entre fragmentos |
| `impact` | Decisión o documento que gobierna/afecta un vínculo entre capas |

Otros valores posibles en el futuro: `validates`, `documents`, `depends-on`, etc.

### `name.N`

Etiqueta opcional para el rol semántico del endpoint N en la relación declarada por `kind`. Texto libre. Útil para herramientas de traversal y agentes de IA.

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
| **Layer** | Copia del `hash.N` del endpoint estructural del bilink adyacente |

Para un endpoint layer, `hash.N` es idéntico al `hash.N` del endpoint estructural en el nodo adyacente — no es el hash del archivo `.bilink` adyacente. Esto evita dependencia circular: aceptar un endpoint layer nunca modifica el archivo adyacente, por lo que no desencadena cascadas.

Ausente cuando el endpoint nunca fue aceptado (estado `PENDING`). Establecido por `bilinker accept`, sobrescrito en cada nueva aceptación.

### `commit.N`

SHA-1 del commit HEAD en el repo del contenido referenciado al momento de la aceptación.

| Tipo de endpoint | `commit.N` |
|---|---|
| **Estructural** | Commit HEAD del repo donde vive el archivo referenciado |
| **Layer** | Copia del `commit.N` del endpoint estructural del bilink adyacente |

Ausente cuando `hash.N` está ausente. Siempre presente junto a `hash.N`.

### `range.N`

Solo para endpoints estructurales. Byte range absoluto (`start~end`) del fragmento en el archivo referenciado, correspondiente al último estado aceptado.

Permite que `bilinker get <file>:<line>:<col>` localice endpoints sin re-ejecutar queries tree-sitter.

### `state.N`

Estado de consistencia del endpoint N. Calculado por `bilinker check`, persistido en el archivo. Ver tablas completas de estados en la sección siguiente.

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
| `TODO` | `hash.N` ausente **y** la layer apuntada no existe todavía | `chain new` / `bilinker check` | Crear la layer + `bilinker accept` |
| `PENDING` | `hash.N` ausente y la layer existe pero no fue aceptada | `bilinker check` | `bilinker accept` |
| `OK` | `hash.N` del endpoint estructural adyacente == `hash.N` almacenado | `bilinker accept` | El contenido del extremo estructural adyacente cambia y es re-aceptado |
| `CHAIN_DIRTY` | El endpoint estructural adyacente fue re-aceptado con contenido diferente | `bilinker check` | `bilinker accept` |
| `BROKEN` | `hash.N` presente pero la layer ya no existe, o el `.bilink` adyacente no tiene endpoint estructural aceptado | `bilinker check` | Restaurar layer + `accept` · o · `bilinker remove` |

### Endpoint bilink

| Estado | Significado | Cómo se llega | Cómo se sale |
|--------|-------------|---------------|--------------|
| `PENDING` | `hash.N` ausente | creación | `bilinker accept` |
| `OK` | Hash actual del `.bilink` referenciado == `hash.N` | `bilinker accept` | Cambio en el bilink referenciado |
| `CHAIN_DIRTY` | `.bilink` referenciado existe pero su hash ≠ `hash.N` | `bilinker check` | `bilinker accept` |
| `UNREACHABLE` | `.bilink` referenciado no existe localmente | `bilinker check` | Obtener la layer + `accept` |

`bilinker remove` elimina el archivo `.bilink` del nodo actual. Los nodos adyacentes detectarán `BROKEN` en el próximo `check` y deberán también decidir: reparar o remover. La remoción se propaga hop a hop por la cadena.

## Propagación reactiva y hash chain

Toda la cadena forma un **hash chain distribuido**: cada nodo ancla criptográficamente al contenido aprobado del extremo estructural de su vecino.

El mecanismo de propagación es:

1. El contenido del archivo referenciado por un endpoint estructural cambia.
2. `bilinker check` detecta el mismatch → `ALTERED`.
3. El usuario revisa y ejecuta `bilinker accept <uuid>.<N>` → el `hash.N` del endpoint estructural se actualiza al nuevo SHA-256 del contenido.
4. El nodo adyacente, en el próximo `check`, compara su `hash.N` almacenado con el `hash.N` actual del endpoint estructural adyacente → difieren → `CHAIN_DIRTY`.
5. El usuario acepta el endpoint layer adyacente → su `hash.N` se sincroniza con el nuevo `hash.N` del extremo estructural.

**Propiedad clave**: aceptar un endpoint layer solo actualiza el propio archivo `.bilink`; nunca modifica el archivo adyacente. Por lo tanto no existe cascada circular: la propagación es unidireccional desde el endpoint estructural que cambió hacia los nodos layer que lo referencian.

Esta propagación garantiza que **ningún cambio de contenido puede ser aprobado en un extremo sin que todos los nodos layer adyacentes lo detecten en el próximo `check`**.

## Semántica de parseo

- Cada clave aparece como máximo una vez en el archivo.
- Las líneas de continuación (que no comienzan con clave reconocida) se concatenan
  al valor anterior con un espacio. Solo aplica a `link.N`.
- Claves reconocidas: `link.0:`, `link.1:`, `kind:`, `name.0:`, `name.1:`,
  `hash.0:`, `commit.0:`, `hash.1:`, `commit.1:`, `range.0:`, `range.1:`, `state.0:`, `state.1:`, `resolved_at:`.
- El prefijo `task ` en el valor de `link.N` indica un endpoint task. El prefijo `task` sin espacio seguido es inválido.
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

## Ejemplo: bilink de tarea

Bilink que conecta un bilink estructural con un ítem del worklist:

```
# .bilink/a3f9c821-4e5b-4c3d-9f2a-1b2c3d4e5f6a.bilink
link.0: .bilink/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink
link.1: task 3a

# Cache
hash.0: e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6
commit.0: d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3
hash.1: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
commit.1: c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2
state.0: OK
state.1: OK
resolved_at: 2026-05-29T09:00:00Z
```

`link.0` apunta al archivo del bilink estructural (tratado como structural). `link.1` apunta al ítem `3a` en `<project-root>/.stratum/worklist/3a.task`. Si el contenido de la tarea cambia, `bilinker check` reporta `ALTERED` en `state.1`.

## Ejemplo completo: cadena de 2 nodos spec → impl

```
# .bilink/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink   (tip — spec layer)
link.0: specs/voting.yaml :: (block_mapping_pair
  key: (flow_node) @n0 (#eq? @n0 "impl")
  value: (_) @target)
link.1: .stratum/impl

# Cache
hash.0: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2   ← SHA-256 del fragmento spec
commit.0: d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3
range.0: 312~358
hash.1: 479922a1ee55cc7f9f4f323bb002018e1b4e1cda65e069e0f6f4645926ce25ee   ← copia de impl.hash.1
commit.1: c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2                          ← copia de impl.commit.1
state.0: OK
state.1: OK
resolved_at: 2026-05-27T10:00:00Z
```

```
# .stratum/impl/.bilink/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink   (tip — impl layer)
link.0: ../..
link.1: src/main/java/ar/example/demo/persona/Persona.java :: (class_declaration
  name: (identifier) @n0 (#eq? @n0 "Persona")
  body: (class_body
    (method_declaration
      name: (identifier) @n1 (#eq? @n1 "vote")) @target))

# Cache
hash.0: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2   ← copia de spec.hash.0
commit.0: d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3                          ← copia de spec.commit.0
hash.1: 479922a1ee55cc7f9f4f323bb002018e1b4e1cda65e069e0f6f4645926ce25ee   ← SHA-256 del método en Persona.java
commit.1: c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2
range.1: 245~389
state.0: OK
state.1: OK
resolved_at: 2026-05-27T10:00:00Z
```

Notar que `spec.hash.1 == impl.hash.1` y `impl.hash.0 == spec.hash.0`: los layer endpoints almacenan exactamente el hash del contenido estructural del nodo adyacente.

## Invariantes

1. El nombre del archivo es un UUID v4 válido con extensión `.bilink`.
2. Siempre existen `link.0` y `link.1`.
3. Un bilink de misma capa tiene dos endpoints estructurales. Una cadena entre capas tiene exactamente dos tips. Un bilink de tarea conecta un endpoint bilink con un endpoint `task <id>`.
4. `hash.N` y `commit.N` están siempre presentes juntos o ausentes juntos.
5. `hash.N` de un endpoint estructural: SHA-256 del fragmento referenciado.
6. `hash.N` de un endpoint layer: idéntico al `hash.N` del endpoint estructural del bilink adyacente. Nunca es el hash del archivo `.bilink` adyacente.
7. `hash.N` de un endpoint bilink: SHA-256 del archivo `.bilink` referenciado completo.
8. Un endpoint `task <id>` se hashea como el contenido del archivo de tarea. `range.N` no aplica (no hay query tree-sitter sobre tareas).
9. `state.N = OK` si y solo si el hash actual de lo apuntado == `hash.N`.
10. `state.N` siempre está presente en la cache una vez que el bilink fue verificado.
11. Si existe cualquier campo de cache, debe existir `resolved_at`.
12. `resolved_at` es siempre UTC (`YYYY-MM-DDTHH:MM:SSZ`).
13. La topología de la cadena es lineal — sin ciclos ni bifurcaciones.
14. Solo se puede crear un bilink sobre archivos con historial git.
15. `kind` y `name.N` son independientes de la cache — no afectan `hash.N` ni `state.N`.
