---
name: bilinker
description: "Crea, verifica y mantiene bilinks — referencias bidireccionales persistentes entre fragmentos de texto en distintas capas del proyecto (spec, impl, docs). Opera solo con git y tree-sitter. Carga esta skill cuando necesites crear un .bilink, interpretar su estado, o ejecutar bilinker check/accept/apply/get."
---

Bilinker mantiene referencias bidireccionales entre fragmentos de texto a través de capas Stratum. La referencia apunta a un nodo AST (via tree-sitter), no a un número de línea, por lo que sobrevive reformateos y movimientos.

## Conceptos clave

- **Capture**: un archivo `.bilink/capture/<uuid>.capture` que describe **dónde** está un fragmento (`file`, `query`, `offset`). No guarda hashes. Varios bilinks pueden referenciar el mismo.
- **Bilink**: un archivo `.bilink/<uuid>.bilink` con dos endpoints (`link.0`, `link.1`). Guarda **qué se aceptó** (`hash.N`, `commit.N`), no dónde está.
- **Cadena**: el mismo UUID en múltiples layers. Dos *tips* (endpoint estructural + layer) y cero o más *mids* (dos layer endpoints).
- **Endpoint estructural**: `capture <uuid>` — referencia a un capture de la misma capa.
- **Endpoint layer**: apunta a otra layer usando path Stratum (`.stratum/impl`, `../..`).
- Bilinker opera **solo con git y tree-sitter**. No consulta language servers ni indexers. El call graph y el análisis de alcance viven en lattice e impact.
- No hay archivo de configuración. Bilinker resuelve la raíz buscando `.bilink/` o `.git/` hacia arriba desde cwd.

## Estructura de `.bilink/`

```
.bilink/
  <uuid>.bilink          ← relaciones (qué se aceptó)
  capture/
    <uuid>.capture       ← ubicaciones (dónde está el fragmento)
  index/
    index                ← índice de lookup O(1) para bilinker get / graph
```

## Formato del archivo `.bilink`

```
# .bilink/<uuid>.bilink
link.0: capture c1a2b3c4-…       # endpoint estructural
link.1: .stratum/impl            # endpoint layer

# Semántica (opcionales)
kind:         governs        # opcional; ver concepts/bilink.md
name.0:       <etiqueta>
name.1:       <etiqueta>

# Cache (ausente hasta el primer accept)
hash.0: <sha256>
hash_ast.0: <sha256>
commit.0: <sha1>
hash.1: <sha256>
hash_ast.1: <sha256>
commit.1: <sha1>
state.0: <estado>
state.1: <estado>
resolved_at: 2026-06-04T10:00:00Z
```

```
# .bilink/capture/c1a2b3c4-….capture
file:   src/lib.rs
query:  (function_item name: (identifier) @n0 (#eq? @n0 "foo")) @target
offset: 42~118                   # opcional: sub-rango relativo al nodo

range:       5100~7300           # cache: absoluto en el archivo
state:       RESOLVED
resolved_at: 2026-06-04T10:00:00Z
```

- El nombre de cada archivo es su UUID v4. El del capture es independiente del de la cadena.
- **El bilink no sabe dónde está su fragmento** — sabe qué capture preguntarle. No tiene `file`, `query` ni `range`.
- `hash.N` es el SHA-256 del texto del fragmento; `hash_ast.N` el de su S-expression tree-sitter. Si `hash.N` difiere pero `hash_ast.N` coincide, el cambio fue solo de formato → `RESTYLED`.
- `hash.N` de un endpoint layer es una **copia** del `hash.N` del endpoint estructural del bilink adyacente, no el hash del archivo `.bilink`.
- Los hashes viven en el bilink y no en el capture **a propósito**: dos bilinks sobre el mismo fragmento pueden haber aceptado versiones distintas y deben poder reportar estados distintos.

## Tipos de endpoint

| Tipo | Forma | Ejemplo |
|------|-------|---------|
| Estructural | `capture <uuid>` | `capture c1a2b3c4-2e3f-4a5b-9c6d-7e8f9a0b1c2d` |
| Layer | path Stratum | `.stratum/impl` · `../..` |
| Task | `task <id>` | `task 3a` |
| Bilink | `.bilink/<uuid>.bilink` | `.bilink/7f3d8e9a-….bilink` |

## Workflow: crear un nuevo bilink

```bash
# 1. Generar UUID
uuid=$(uuidgen | tr '[:upper:]' '[:lower:]')

# 2. Capturar cada endpoint estructural (selección por línea:col)
c0=$(bilinker capture <archivo> <start_line>:<start_col> <end_line>:<end_col>)
# stdout → UUID del capture creado
# stderr → metadata (path, anchor, range)

# 3. Crear el archivo .bilink referenciando el capture
cat > .bilink/$uuid.bilink <<EOF
link.0: capture $c0
link.1: <endpoint-1>
EOF

# 4. Aceptar el estado inicial
bilinker accept .
```

Para cadenas entre layers: usar `bilinker chain new --tip <layer>:<ref> --tip <layer>:<ref>`.

## Workflow: mantener bilinks

```bash
# Ver estado almacenado (no re-verifica)
bilinker status [<path>]

# Re-verificar y actualizar estados — solo git + tree-sitter, offline
bilinker check [<path>]

# Aplicar auto-fixes (MOVED / DISPLACED / REANCHORED / EXPANDED)
bilinker apply --dry-run       # ver qué haría
bilinker apply                 # pide confirmación
bilinker apply -y              # sin confirmación

# Aceptar cambios de contenido (ALTERED / RESTYLED / CHAIN_DIRTY / PENDING)
bilinker accept <uuid>.<N>     # un endpoint
bilinker accept .              # todos los que necesitan atención en la layer actual

# Repuntar un endpoint que ya no ancla (UNANCHORED, o fragmento movido a otro repo)
bilinker recapture <uuid>.<N> <file> <línea>:<col>
# crea el capture, reescribe link.N y limpia state.N — no acepta
```

### Qué hace `apply` y qué no

`apply` corrige **dónde está el fragmento**, escribiendo en el **capture** (`file`, `query`, `offset`). Nunca escribe `hash.N`, `hash_ast.N` ni `commit.N` — eso es exclusivo de `accept`.

Si el capture está compartido por varios bilinks, `apply` **forkea** en vez de corregir en el lugar para DISPLACED y REANCHORED (dependen de `hash.N` o de una inferencia ambigua). MOVED y EXPANDED sí se corrigen en el lugar: son hechos objetivos sobre el archivo o el nodo.

| Estado | Tras `apply` |
|--------|--------------|
| `MOVED`, `DISPLACED` | queda `OK` — el contenido no cambió |
| `EXPANDED`, `REANCHORED` | sigue no-OK — el contenido cambió, hace falta `bilinker accept` |

`apply` re-resuelve cada endpoint en el momento en vez de confiar en el `range` cacheado del capture. Si el estado re-derivado no coincide con `state.N` (la cache quedó vieja), descarta el fix y pide correr `check`. Correr `check` antes de `apply` es la secuencia normal.

## Navegar el código referenciado

```bash
# Ver el fragmento de un endpoint
bilinker get <uuid>.<N>

# Contexto adicional
bilinker get <uuid>.<N> -B 3 -A 3

# Diff entre el fragmento aceptado (commit.N) y el actual
bilinker get <uuid>.<N> --diff

# Recorrer el grafo de bilinks (cruza capas)
bilinker graph <file>                 # bilinks que referencian el archivo
bilinker graph . --recursive          # todas las capas bajo la raíz
bilinker graph . --format json        # aristas para lattice
```

El baseline de `--diff` es siempre el `commit.N` del endpoint, nunca un ref externo.

El traversal del call graph (qué llama a qué) **no es de bilinker** — vive en lattice, que agrega las aristas de bilinker con las derivadas del LSP y las de links en documentos.

## Estados de un endpoint

Los estados están partidos en dos: **dónde está** (capture) y **coincide con lo aceptado** (bilink).

### Resolución — en el capture

| Estado | Significado | Acción |
|--------|-------------|--------|
| `RESOLVED` | La query matchea; `range` actualizado | — |
| `MOVED` | Archivo renombrado (≥50% similitud git) | `bilinker apply` |
| `REANCHORED` | Anchor renombrado/movido, nueva posición en AST | `bilinker apply` + `accept` |
| `UNANCHORED` | Query no matchea ningún nodo | `bilinker recapture` o `remove` |
| `DELETED` | Archivo eliminado (rastreable en git) | restaurar o remove |
| `BROKEN` | Ninguna hipótesis aplica | intervención |

### Aceptación — en el bilink (endpoint estructural)

| Estado | Significado | Acción |
|--------|-------------|--------|
| `OK` | Hash coincide | — |
| `DISPLACED` | Hash en otro offset del nodo | `bilinker apply` |
| `EXPANDED` | Fragmento creció; AST interno sin cambio estructural | `bilinker apply` + `accept` |
| `RESTYLED` | `hash.N` difiere pero `hash_ast.N` coincide — solo formato | `bilinker accept` |
| `ALTERED` | Fragmento encontrado; AST interno cambió | revisar + `bilinker accept` |
| `UNRESOLVED` | El capture referenciado no resuelve | resolver el capture |

### Endpoint layer

| Estado | Significado | Acción |
|--------|-------------|--------|
| `PENDING` | Sin hash aceptado | `bilinker accept` |
| `TODO` | Layer destino no existe todavía | crear layer + `bilinker accept` |
| `OK` | Hash sincronizado con el nodo adyacente | — |
| `CHAIN_DIRTY` | El extremo estructural adyacente cambió y fue re-aceptado | `bilinker accept` |
| `BROKEN` | Layer o bilink adyacente no existe | restaurar o remove |

## Propagación de cambios

Cuando el contenido de un endpoint estructural cambia:
1. `check` → `ALTERED` en el tip estructural.
2. `accept` → actualiza `hash.N` del tip → el `.bilink` cambia.
3. En el próximo `check` del nodo layer adyacente → `CHAIN_DIRTY`.
4. `accept` en el layer → sincroniza su `hash.N`.

La propagación es siempre unidireccional desde el endpoint estructural hacia los layer endpoints.

## Comandos de referencia rápida

```bash
bilinker status                               # resumen de la layer actual
bilinker status $(stratum '*')                # resumen desde la raíz del proyecto
bilinker check .                              # verificar todos los bilinks de la layer
bilinker check .bilink/<uuid>.bilink          # verificar un bilink concreto
bilinker apply --dry-run                      # ver qué fixes se aplicarían
bilinker apply -y                             # aplicar todos los fixes sin confirmar
bilinker accept .                             # aceptar todo lo pendiente en la layer
bilinker accept <uuid>.0                      # aceptar endpoint 0 de un bilink
bilinker index --recursive                    # reconstruir índice (acelera get / graph)
bilinker capture <file> <l>:<c> <l>:<c>       # crear un capture, devuelve su UUID
bilinker capture prune                        # borrar captures sin referentes
bilinker capture remove <uuid>                # borrar un capture puntual
bilinker recapture <uuid>.<N> <file> <l>:<c>  # repuntar un endpoint que ya no ancla
bilinker get <uuid>.<N>                       # ver fragmento del endpoint
bilinker get <uuid>.<N> --diff                # diff contra el estado aceptado
bilinker graph <file>                         # bilinks que referencian el archivo
bilinker graph . --recursive --format json    # aristas para lattice
bilinker chain new --tip <ref> --tip <ref>    # crear cadena entre layers
```

## Invariantes a recordar

- `hash.N` y `commit.N` siempre presentes juntos o ausentes juntos.
- `hash.N` de un endpoint layer == `hash.N` del endpoint estructural del nodo adyacente.
- Solo `accept` escribe `hash.N` / `hash_ast.N` / `commit.N`, y solo en el bilink.
- `check` escribe `range` y `state` en el capture, y `state.N` en el bilink.
- `apply` escribe el capture; su único efecto sobre el bilink es `state.N` y repuntar `link.N` al forkear.
- Un endpoint estructural referencia exactamente un capture, **de su misma capa**.
- Borrar un bilink nunca borra un capture.
- `kind` y `name.N` son independientes de la cache — no afectan `state.N`.
- Un bilink solo se puede crear sobre archivos con historial git.
- La topología de una cadena es lineal — sin ciclos ni bifurcaciones.
- Bilinker no persiste nada derivado del código más allá de hashes aceptados. No hay índice de símbolos ni call graph bajo control de versiones.
