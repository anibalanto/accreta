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

| Id | Estado | Qué hace |
|---|---|---|
| `bilinker-001-capture-split` | retirada | Extrajo la ubicación de cada endpoint estructural a un `.capture` y reemplazó `link.N` por `capture <uuid>`. |
| `bilinker-002-file-partition` | vigente | Reescribe cada bilink a YAML, con los endpoints bajo `endpoint.0`/`endpoint.1` y el tipo de cada `link` explícito. Lo derivable sale a la cache. |

**Una migración retirada no se borra del ledger; se borra del registro.** `001` corrió en todos los repos que existían y su id sigue en cada ledger —quitarlo sería reescribir lo que pasó—, pero su código ya no está: `002` lee la forma embebida directamente, así que un repo que nunca corrió `001` se migra igual, en un paso.

Es la asimetría que hace útil al ledger: registra qué le pasó a este repo, no qué sabe hacer este binario.


### `bilinker-001-capture-split` (retirada)

Convertía los endpoints con la ubicación embebida —`file :: query :: offset`— al formato con [capture](../concepts/capture.md) aparte, deduplicando: dos endpoints con `(file, query, offset)` idénticos compartían un capture, porque referencias idénticas describen la misma ubicación.

**Su trabajo lo hace `002` en una sola pasada.** El id de un capture sale de la ubicación, así que la dedup es por construcción y no hace falta un paso que la produzca; y `002` lee la forma embebida además de la de `001`, así que no hay orden que respetar entre las dos. Lo que `001` hacía en dos pasos, `002` lo hace en uno.

Descartaba también `subgraph.N`, campo eliminado del formato.

### `bilinker-002-file-partition`

De `clave: valor` plano a YAML. `hash.N` → `accepted.hash`, `hash_ast.N` → `accepted.hash_ast`; `commit.N`, `state.N`, y el `range` y el `state` que salen del `.capture` van a [`cache/state`](../concepts/cache.md). Se escribe `.bilink/version`.

**`resolved_at` se descarta** en los dos archivos: no se muda a la cache, desaparece del formato.

**`kind` y `name.N` se preservan** —es el momento en que dejan de perderse— y `name.N` pasa a ser `name` adentro de su endpoint.

**`accepted.link` se siembra copiando `link.N`** donde había `hash.N`. Es exacto donde el endpoint estaba `OK`: en el formato viejo un endpoint `OK` es uno cuyo contenido actual coincide con el aceptado en la ubicación que `link.N` describe, así que esa ubicación *es* la bendecida. Donde estaba no-OK es la única lectura disponible —el formato viejo no distingue drift de ubicación de drift de contenido— y es la que preserva la invariante de aceptación sin poner todos los bilinks en `RELOCATED` de golpe ni degradarlos a `PENDING`, que borraría el inventario de trabajo. En un endpoint `PENDING`, `accepted` queda ausente y sólo sobrevive `link`.

#### Los captures, en la misma pasada

Cada capture se acuña bajo `H(file, query, offset)` y se repuntan las dos clases de referencia: `link` y `accepted.link`.

**No tiene fan-out.** Como el id no depende del hash del contenido, dos bilinks que aceptaron contenidos distintos del mismo fragmento siguen compartiendo capture, y la divergencia queda en sus `accepted`. Dos captures con la misma ubicación colapsan en uno: es la dedup por construcción, aplicada de una vez a lo que ya existía.

> Esto llegó a estar planeado como una migración aparte, `003`. No lo es: el id sale de los tres campos y no de hashear el archivo, así que no hay que sacarle nada antes para poder calcularlo. Separarlas habría creado un formato intermedio —captures en YAML con su uuid viejo— que exige un crate propio para siempre y en el que nadie iba a estar. Ver la enmienda en [ADR-0003](../.stratum/impl/docs/adr/0003-formato-captures-y-aceptacion.md).

## El problema de bootstrap

La herramienta que cambia de formato es la que se usa para cambiarlo, y las specs que describen el formato están bilinkeadas al código que lo implementa. Durante la transición conviven specs viejas y nuevas, binario viejo y nuevo, y bilinks en los dos formatos.

**Coexistencia por path.** Los dos formatos no pueden ocupar `.bilink/` a la vez, así que la migración escribe en un path transitorio y deja `.bilink/` intacto:

```
.bilink/  →  .bilink-migrate-002-file-partition/
```

El binario viejo sigue trabajando contra `.bilink/` sin enterarse; el nuevo se ejerce contra el path nuevo con datos reales antes de que nada sea irreversible. Los dos corriendo en el mismo instante sobre el mismo repo, cada uno contra la carpeta que entiende.

**El path lleva el id de la migración.** Con una sola el nombre parece de más, y no lo es: distingue una carpeta en curso de una abandonada de un intento anterior, y deja la puerta abierta a encadenar si alguna vez hay dos. El prefijo `bilinker-` se omite: dentro del directorio de bilinker es redundante.

**Es un derivado, no un espacio de trabajo.** No se edita a mano: si se lo edita, deja de poder regenerarse, que es lo único que lo vuelve seguro. Y tiene que poder regenerarse, porque si entre la generación y el corte alguien acepta algo con el binario viejo, la copia migrada queda vieja y el corte se comería esa aceptación.

**La idempotencia tiene dos regímenes.** Antes del corte, `migrate` **siempre regenera** — la regla operativa es regenerar justo antes de cortar. Después del corte el ledger la vuelve no-op, que es lo que [`migration.md`](../../../concepts/migration.md) exige.

**La entrada en el ledger va en el corte, no en la generación.** Si se escribiera al generar, el repo quedaría marcado como migrado mientras sigue corriendo el formato viejo. Se registra cuando el estado es verdadero, no cuando el trabajo empezó.

`.git/info/exclude` recibe `.bilink-migrate-*` al empezar: esas carpetas son temporales y nunca se commitean.

## Lo que no necesita migración

**Los endpoints `repo` y `abstract`** de [ADR-0005](../.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md) son **aditivos**: ningún archivo existente los usa y todos siguen siendo válidos. La frontera se adopta bilink por bilink.

Que un cambio aditivo no lleve migración es justamente por qué `.bilink/version` hace falta además del ledger: un parser viejo leería `abstract` como un path y no fallaría, y el ledger no puede expresar eso porque no hubo migración que registrar. Ver [versión del formato](../concepts/format-version.md).

**Mover los bilinks a una ref** ([ADR-0004](../.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md)) tampoco es una migración, y no puede serlo: no transforma ningún archivo —los deja idénticos y cambia dónde viven— y `migration.md` prohíbe que una migración consulte git, que es todo lo que esa operación hace.

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
