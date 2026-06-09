# Especificación: comando `bilinker impact`

## Propósito

Dado un punto de cambio, recorre el grafo de llamadas **hacia arriba** vía `incomingCalls` (LSP) hasta encontrar los bilinks que "poseen" las funciones afectadas, y reporta las especificaciones impactadas con el camino de propagación.

Sin argumentos, deriva el análisis directamente del estado ya calculado por `bilinker check`: para cada bilink con `state.N ≠ OK`, usa su `commit.N` como baseline y sus callers como punto de partida del traversal.

Complementa `get --diff --recursive` (que baja por callees): `impact` sube por callers hasta alcanzar el límite del subgrafo cubierto por bilinks.

## Firma

```
bilinker impact [<selector>] [--depth <N>] [--format <fmt>]
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `<selector>` | ver abajo | Punto de partida. Omitir para analizar todos los bilinks no-OK de la layer. |
| `--depth N` | int | Profundidad máxima de traversal hacia arriba. Default: ilimitada. |
| `--format fmt` | `tree` \| `flat` \| `json` | Formato de salida. Default: `tree`. |

### Formas de selector

| Selector | Descripción |
|---|---|
| *(ninguno)* | Todos los bilinks con `state.N ≠ OK` en la layer actual. El baseline de cada uno es su propio `commit.N`. |
| `file:line:col` | Función en esa posición. |
| `file` | Todas las funciones del archivo que tienen callers que alcanzan un bilink. |
| `UUID` / `UUID.N` | Partir desde el endpoint estructural de ese bilink. |

## Baseline

El baseline del análisis **siempre proviene del `commit.N` del bilink que se alcanza** — no de un argumento externo. Esto garantiza que la comparación refleja exactamente lo que estaba aceptado.

Cuando el traversal sube por callers y llega a un bilink, el diff de cada función en el camino se calcula respecto al `commit.N` de ese bilink, igual que en `get --diff --recursive`.

## Algoritmo

```
1. Resolver selector → lista de (file, line, col, commit_baseline?) de partida.
   Sin selector: iterar bilinks con state.N ≠ OK → (sref.file, range.N start, commit.N).

2. Para cada punto de partida:
   a. ¿El punto de partida mismo tiene un bilink?
      → incluirlo en el reporte con ruta vacía y su propio state.N.
   b. Llamar daemon incomingCalls(file, line, col) → callers directos.
   c. Para cada caller:
      - ¿Hay un bilink cuyo range.N cubre este caller? → BILINK ALCANZADO
        Agregar: (bilink, endpoint, ruta de propagación, diff del camino)
      - Si no → recursión: incomingCalls del caller (hasta --depth o sin callers).

3. Deduplicar por bilink UUID: múltiples rutas al mismo bilink → mostrar la más corta.

4. Emitir reporte.
```

### Detección de "bilink alcanzado"

Un caller en `(file, line)` tiene bilink si `find_by_file(file)` retorna algún endpoint cuyo `range.N` contiene `line`. Usa `.bilink/index/index` si está disponible (O(1)); si no, escaneo O(N).

## Salida — formato `tree`

### Sin selector (caso principal)

```
$ bilinker impact

2 bilinks con estado no-OK → 2 specs impactadas:

◆ fac79bf8.1  commands/check.md § "Algoritmo de detección"  [ALTERED]
  check_structural  src/check.rs:210  ALTERED  (desde commit ca76a590)

◆ 9662a432.1  commands/check.md § "Escritura de cache"  [OK]
  check_file  src/check.rs:53  OK
  └── check_structural  src/check.rs:210  ALTERED  (desde commit ca76a590)
      via incomingCalls
```

El estado `[ALTERED]` / `[OK]` junto al `◆` es el `state.N` cacheado del bilink.
El estado junto a cada función en el camino es el resultado de comparar contra el `commit.N` del bilink alcanzado.

### Con selector `file:line:col`

```
$ bilinker impact src/check.rs:210:5

check_structural  src/check.rs:210
│
├── check_endpoint  src/check.rs:174
│     └── ◆ fac79bf8.1  commands/check.md § "Algoritmo"  [ALTERED]
│
└── check_file  src/check.rs:53
      └── ◆ 9662a432.1  commands/check.md § "Escritura de cache"  [OK]
```

## Salida — formato `flat`

```
$ bilinker impact --format flat

fac79bf8.1  commands/check.md § "Algoritmo"  [ALTERED]
  via: check_structural → check_endpoint

9662a432.1  commands/check.md § "Escritura de cache"  [OK]
  via: check_structural → check_file
```

## Salida — formato `json`

```json
[
  {
    "bilink":   "fac79bf8-...",
    "endpoint": 1,
    "spec":     "commands/check.md",
    "state":    "ALTERED",
    "path":     ["check_structural", "check_endpoint"],
    "commit":   "ca76a590"
  }
]
```

## Traversal cross-layer

Si durante el traversal hacia arriba el caller pertenece a una layer adyacente (el archivo está fuera de la layer actual), `impact` sigue el grafo de bilinks de cadena hacia la layer superior usando los mismos mecanismos que `graph`. El traversal continúa hasta alcanzar un bilink o agotar `--depth`.

## Sin daemon LSP

Si el daemon no está corriendo, `impact` omite el traversal de `incomingCalls` y reporta únicamente los bilinks que apuntan directamente a los archivos afectados (equivalente a `find_by_file` sobre los archivos no-OK). Emite advertencia en stderr:

```
warn: daemon LSP no disponible — traversal de callers omitido, solo bilinks directos
```

## Propiedades garantizadas

- **Solo lectura**: `impact` no escribe ningún archivo.
- **Sin re-check**: usa `state.N` y `commit.N` ya cacheados. Para resultados frescos, correr `bilinker check` primero.
- **Baseline siempre desde el bilink**: el `commit.N` del bilink alcanzado es el único baseline — no se necesita pasar refs externos.
- **Cycle-safe**: visited-set en el traversal de callers para cortar recursión mutua.

## Código de salida

| Código | Condición |
|---|---|
| 0 | Al menos un bilink alcanzado. |
| 1 | Error: archivo no encontrado, selector inválido. |
| 2 | Sin bilinks alcanzados desde el punto de partida. |
