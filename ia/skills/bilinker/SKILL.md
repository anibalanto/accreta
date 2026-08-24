---
name: bilinker
description: "Crea, verifica y mantiene bilinks — referencias bidireccionales persistentes entre fragmentos de texto en distintas capas del proyecto (spec, impl, docs). Opera solo con git y tree-sitter. Carga esta skill cuando necesites crear un .bilink, interpretar su estado, o ejecutar bilinker check/accept/apply/get."
---

Bilinker mantiene referencias bidireccionales entre fragmentos de texto a través de capas Stratum. La referencia apunta a un nodo AST (via tree-sitter), no a un número de línea, por lo que sobrevive reformateos y movimientos.

## Conceptos clave

- **Bilink**: un archivo `.bilink/<uuid>.bilink` con dos endpoints (`link.0`, `link.1`).
- **Cadena**: el mismo UUID en múltiples layers. Dos *tips* (endpoint estructural + layer) y cero o más *mids* (dos layer endpoints).
- **Endpoint estructural**: apunta a un fragmento en un archivo (`file :: query`).
- **Endpoint layer**: apunta a otra layer usando path Stratum (`.stratum/impl`, `../..`).
- Bilinker opera **solo con git y tree-sitter**. No consulta language servers ni indexers. El call graph y el análisis de alcance viven en lattice e impact.
- No hay archivo de configuración. Bilinker resuelve la raíz buscando `.bilink/` o `.git/` hacia arriba desde cwd.

## Estructura de `.bilink/`

```
.bilink/
  <uuid>.bilink          ← bilinks
  index/
    index                ← índice de lookup O(1) para bilinker get / graph
```

## Formato del archivo `.bilink`

```
link.0: <endpoint>
link.1: <endpoint>

# Semántica (opcionales)
kind:         governs        # opcional; ver concepts/bilink.md
name.0:       <etiqueta>
name.1:       <etiqueta>

# Cache (ausente hasta el primer accept)
hash.0: <sha256>
hash_ast.0: <sha256>
commit.0: <sha1>
range.0: <start~end>
hash.1: <sha256>
hash_ast.1: <sha256>
commit.1: <sha1>
range.1: <start~end>
state.0: <estado>
state.1: <estado>
resolved_at: 2026-06-04T10:00:00Z
```

- El nombre del archivo es el UUID v4. No existe campo `id`.
- `hash.N` es el SHA-256 del texto del fragmento; `hash_ast.N` el de su S-expression tree-sitter. Si `hash.N` difiere pero `hash_ast.N` coincide, el cambio fue solo de formato → `RESTYLED`.
- `hash.N` de un endpoint layer es una **copia** del `hash.N` del endpoint estructural del bilink adyacente, no el hash del archivo `.bilink`.

## Tipos de endpoint

| Tipo | Forma | Ejemplo |
|------|-------|---------|
| Estructural (archivo completo) | `file` | `docs/arch.md` |
| Estructural (con query AST) | `file :: (query @target)` | `src/lib.rs :: (function_item name: (identifier) @n0 (#eq? @n0 "foo") @target)` |
| Layer | path Stratum | `.stratum/impl` · `../..` |
| Task | `task <id>` | `task 3a` |
| Bilink | `.bilink/<uuid>.bilink` | `.bilink/7f3d8e9a-….bilink` |

## Workflow: crear un nuevo bilink

```bash
# 1. Generar UUID
uuid=$(uuidgen | tr '[:upper:]' '[:lower:]')

# 2. Capturar un endpoint estructural (selección por línea:col)
bilinker capture <archivo> <start_line>:<start_col> <end_line>:<end_col>
# stdout → valor listo para pegar en link.N
# stderr → hash del fragmento (informativo)

# 3. Crear el archivo .bilink
cat > .bilink/$uuid.bilink <<EOF
link.0: <endpoint-0>
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
```

### Qué hace `apply` y qué no

`apply` corrige **a dónde apunta** el endpoint (`file`, `start~end`, predicados de la query). Nunca escribe `hash.N` ni `commit.N` — eso es exclusivo de `accept`.

| Estado | Tras `apply` |
|--------|--------------|
| `MOVED`, `DISPLACED` | queda `OK` — el contenido no cambió |
| `EXPANDED`, `REANCHORED` | sigue no-OK — el contenido cambió, hace falta `bilinker accept` |

`apply` re-resuelve cada endpoint en el momento en vez de confiar en `range.N`. Si el estado re-derivado no coincide con `state.N` (la cache quedó vieja), descarta el fix y pide correr `check`. Correr `check` antes de `apply` es la secuencia normal.

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

### Endpoint estructural

| Estado | Significado | Acción |
|--------|-------------|--------|
| `OK` | Hash coincide | — |
| `MOVED` | Archivo renombrado (≥50% similitud git) | `bilinker apply` |
| `DISPLACED` | Query matchea; hash en offset diferente | `bilinker apply` |
| `REANCHORED` | Anchor renombrado/movido, nueva posición en AST | `bilinker apply` + `accept` |
| `EXPANDED` | Fragmento creció; AST interno sin cambio estructural | `bilinker apply` + `accept` |
| `RESTYLED` | `hash.N` difiere pero `hash_ast.N` coincide — solo formato | `bilinker accept` |
| `ALTERED` | Fragmento encontrado; AST interno cambió | revisar + `bilinker accept` |
| `UNANCHORED` | Query no matchea ningún nodo | re-capture o remove |
| `DELETED` | Archivo eliminado (rastreable en git) | restaurar o remove |
| `BROKEN` | Ninguna hipótesis aplica | intervención |

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
bilinker get <uuid>.<N>                       # ver fragmento del endpoint
bilinker get <uuid>.<N> --diff                # diff contra el estado aceptado
bilinker graph <file>                         # bilinks que referencian el archivo
bilinker graph . --recursive --format json    # aristas para lattice
bilinker chain new --tip <ref> --tip <ref>    # crear cadena entre layers
```

## Invariantes a recordar

- `hash.N` y `commit.N` siempre presentes juntos o ausentes juntos.
- `hash.N` de un endpoint layer == `hash.N` del endpoint estructural del nodo adyacente.
- Solo `accept` escribe `hash.N` / `commit.N`. `check` escribe `state.N` y `range.N`; `apply` escribe `link.N`, `range.N` y `state.N`.
- `kind` y `name.N` son independientes de la cache — no afectan `state.N`.
- Un bilink solo se puede crear sobre archivos con historial git.
- La topología de una cadena es lineal — sin ciclos ni bifurcaciones.
- Bilinker no persiste nada derivado del código más allá de hashes aceptados. No hay índice de símbolos ni call graph bajo control de versiones.
