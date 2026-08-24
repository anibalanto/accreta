# Especificación: comando `bilinker migrate`

## Propósito

Migra los metadatos de bilinker de una capa al formato vigente. Es la implementación para bilinker del mecanismo general descripto en [Migración de metadatos](../../../concepts/migration.md).

## Firma

```
bilinker migrate [<path>] [--recursive] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `path` | Capa a migrar. Default: capa actual (cwd). |
| `--recursive` | Migra también todas las capas descendientes encontradas en `.stratum/`. |
| `--dry-run` | Muestra qué haría sin escribir nada. |

## Correr `--recursive` no es opcional cuando un repo tiene varias capas

El ledger es por repo y las migraciones corren por capa. Si un repo contiene varias capas —el repo de specs de un proyecto suele tener la suya y las de sus subproyectos— hay que alcanzarlas todas en la misma corrida.

Invocar `migrate` capa por capa registra la migración al terminar la primera, y las siguientes la ven registrada y se saltean, quedando sin migrar.

## Migraciones

| Id | Qué hace |
|---|---|
| `bilinker-001-capture-split` | Extrae la ubicación de cada endpoint estructural a un `.capture`, y reemplaza `link.N` por `capture <uuid>`. |

### `bilinker-001-capture-split`

Convierte los endpoints con la ubicación embebida —`file :: query :: offset`, más `range.N` en la cache— al formato con [capture](../concepts/capture.md) aparte.

**Deduplica.** Dos endpoints con `(file, query, offset)` idénticos comparten un capture, porque referencias idénticas describen la misma ubicación. Sin esto la duplicación existente quedaría congelada: nada la fusionaría después.

La deduplicación crea captures compartidos, y con ellos el fork de `bilinker apply` para los fixes que dependen de `hash.N`. Eso es el diseño funcionando, no un efecto colateral — ver [capture.md](../concepts/capture.md) § "Copy-on-write al aplicar un fix".

**No resuelve.** El `range` se copia tal cual estaba y el `state` del capture queda vacío. El `check` siguiente los recalcula.

**Descarta `subgraph.N`**, campo eliminado del formato, y lo reporta en el resumen para que la desaparición no sea silenciosa.

## Salida

```
$ bilinker migrate --recursive --dry-run

repo /home/anibal/Workspace/accreta
  bilinker-001-capture-split  [dry-run]
    /home/anibal/Workspace/accreta: 60 capture(s) creado(s), 13 endpoint(s) reusaron uno existente
    /home/anibal/Workspace/accreta/subsystems/stratum: 8 capture(s) creado(s), 1 endpoint(s) reusaron uno existente
    150 archivo(s) afectado(s)

repo /home/anibal/Workspace/accreta/subsystems/bilinker/.stratum/impl
  bilinker-001-capture-split  [dry-run]
    …: 56 capture(s) creado(s), 17 endpoint(s) reusaron uno existente
    …: 56 campo(s) subgraph.N descartado(s) — eliminados del formato
    129 archivo(s) afectado(s)

dry-run: no se escribió nada
```

El reporte se agrupa por repo porque cada uno tiene su propio ledger: una misma migración aparece una vez por repo alcanzado.

Cuando no hay nada pendiente:

```
$ bilinker migrate --recursive
ya aplicada: bilinker-001-capture-split
nada que migrar (4 capa(s) revisada(s))
```

## Código de salida

| Código | Condición |
|---|---|
| 0 | Migraciones aplicadas, o nada pendiente. |
| 1 | Error al aplicar una migración. Ninguna se registra en el ledger. |

## Propiedades garantizadas

- **Idempotente**: correrlo dos veces no hace nada la segunda.
- **`--dry-run` no escribe**: ni captures, ni bilinks, ni ledger.
- **No commitea**: revisar con `git diff` y commitear a mano.
- **No resuelve**: no corre tree-sitter ni git. Correr `bilinker check .` después.
