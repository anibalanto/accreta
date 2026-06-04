---
name: bilinker
description: "Crea, verifica y mantiene bilinks — referencias bidireccionales persistentes entre fragmentos de texto en distintas capas del proyecto (spec, impl, docs). Carga esta skill cuando necesites crear un .bilink, interpretar su estado, o ejecutar bilinker check/accept/apply."
---

Bilinker mantiene referencias bidireccionales entre fragmentos de texto a través de capas Stratum. La referencia apunta a un nodo AST (via tree-sitter), no a un número de línea, por lo que sobrevive reformateos y movimientos.

## Conceptos clave

- **Bilink**: un archivo `.bilink/<uuid>.bilink` con dos endpoints (`link.0`, `link.1`).
- **Cadena**: el mismo UUID en múltiples layers. Dos *tips* (endpoint estructural + layer) y cero o más *mids* (dos layer endpoints).
- **Endpoint estructural**: apunta a un fragmento en un archivo (`file :: query`).
- **Endpoint layer**: apunta a otra layer usando path Stratum (`.stratum/impl`, `../..`).
- No hay archivo de configuración. Bilinker resuelve la raíz buscando `.bilink/` o `.git/` hacia arriba desde cwd.

## Formato del archivo `.bilink`

```
link.0: <endpoint>
link.1: <endpoint>

# Semántica (opcionales)
kind:   impact
name.0: <etiqueta>
name.1: <etiqueta>

# Cache (ausente hasta el primer accept)
hash.0: <sha256>
commit.0: <sha1>
range.0: <start~end>
hash.1: <sha256>
commit.1: <sha1>
range.1: <start~end>
state.0: <estado>
state.1: <estado>
resolved_at: 2026-05-27T10:00:00Z
```

El nombre del archivo es el UUID v4 — es el ID de la cadena. No existe campo `id`.

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

Para cadenas entre layers: crear el mismo UUID en cada layer, con los endpoints cruzados como layer paths.

## Workflow: mantener bilinks

```bash
# Ver estado almacenado (no re-verifica)
bilinker status [<path>]

# Re-verificar y actualizar estados
bilinker check <path>          # path a .bilink/, layer, o .bilink individual

# Aplicar auto-fixes (MOVED / DISPLACED / REANCHORED / EXPANDED)
bilinker apply                 # pide confirmación
bilinker apply -y              # sin confirmación

# Aceptar cambios manuales (ALTERED / CHAIN_DIRTY / PENDING)
bilinker accept <uuid>.<N>     # un endpoint
bilinker accept .              # todos los que necesitan atención en la layer actual
```

## Estados de un endpoint

### Endpoint estructural

| Estado | Significado | Acción |
|--------|-------------|--------|
| `OK` | Hash coincide | — |
| `MOVED` | Archivo renombrado (≥50% similitud git) | `bilinker apply` |
| `DISPLACED` | Query matchea; hash en offset diferente | `bilinker apply` |
| `REANCHORED` | Anchor renombrado/movido, nueva posición en AST | `bilinker apply` |
| `EXPANDED` | Fragmento creció; AST interno sin cambio estructural | `bilinker apply` |
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

La propagación es siempre unidireccional desde el endpoint estructural hacia los layer endpoints. Nunca hay cascada circular.

## Comandos de referencia rápida

```bash
bilinker status                          # resumen de la layer actual
bilinker status $(stratum '*')           # resumen desde la raíz del proyecto
bilinker check .bilink/                  # verificar todos los bilinks de la layer
bilinker check .bilink/<uuid>.bilink     # verificar un bilink concreto
bilinker apply --dry-run                 # ver qué fixes se aplicarían
bilinker apply -y                        # aplicar todos los fixes sin confirmar
bilinker accept .                        # aceptar todo lo pendiente en la layer
bilinker accept <uuid>.0                 # aceptar endpoint 0 de un bilink
bilinker index --recursive               # reconstruir índice (acelera `get`)
```

## Invariantes a recordar

- `hash.N` y `commit.N` siempre presentes juntos o ausentes juntos.
- `hash.N` de un endpoint layer == `hash.N` del endpoint estructural del nodo adyacente (nunca el hash del `.bilink`).
- `kind` y `name.N` son independientes de la cache — no afectan `state.N`.
- Un bilink solo se puede crear sobre archivos con historial git.
- La topología de una cadena es lineal — sin ciclos ni bifurcaciones.
