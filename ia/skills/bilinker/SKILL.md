---
name: bilinker
description: Referencias verificadas entre fragmentos de texto a través de capas Stratum. Cargar antes de cualquier tarea que toque bilinks — crear, revisar, aceptar, repuntar o seguir el inventario de un cambio.
---

Bilinker mantiene referencias bidireccionales entre fragmentos a través de capas Stratum. La referencia apunta a un nodo del AST vía tree-sitter, no a un número de línea, así que sobrevive reformateos y movimientos.

Opera **solo con git y tree-sitter**. No consulta language servers ni indexers: el call graph vive en lattice, el alcance de un cambio en impact.

No hay archivo de configuración. La raíz se resuelve caminando hacia arriba desde cwd, buscando `.bilink/` o `.git/`.

## Las tres cosas y qué guarda cada una

```
.bilink/
  <uuid>.yaml            ← el bilink: qué se relaciona con qué, y qué se aprobó
  capture/
    <id>.yaml            ← una ubicación, inmutable. El id es su hash.
  cache/state            ← lo derivable · no versionado
  version                ← la versión de formato
  .gitignore             ← cache/ e index/
  index/index            ← lookup O(1) · no versionado
```

**Un capture es una ubicación y nada más** — `file` y `query`. Su nombre es el hash de esos campos, así que es inmutable por construcción: cambiarle la ubicación le cambiaría el nombre. Dos referencias a la misma ubicación son el mismo archivo, sin buscar duplicados.

**Nombra un nodo entero.** No hay sub-rango: la selección con la que se crea sirve para *encontrar* el nodo y después se descarta. Seleccionar media función captura la función. Para algo más chico hace falta una query que lo nombre.

**Un bilink referencia captures y guarda decisiones.** No sabe dónde está su fragmento: sabe a qué capture preguntarle.

**La cache no está en git.** Estar fría es normal — un clon fresco, otra rama, otra máquina.

## El archivo de bilink

```yaml
kind: governs                       # opcional, inerte
endpoint:
  0:
    link: capture 67ba7217e0334051becd4921b55a7872
    name: la-decision               # opcional, inerte
    accepted:
      link: capture 67ba7217e0334051becd4921b55a7872
      hash: c00e07602bd5…
      hash_ast: 1b9e44a2f0c8…       # sólo donde el AST discrimina contenido
  1:
    link: path subsystems/bilinker>impl
    accepted:
      link: capture f5e8a7d7164c58cc32e8a5f035c54bcb
      hash: 30985a8c8437…
```

**La ausencia de `accepted` *es* el estado PENDING.** No hay campo que lo diga.

`hash_ast` cubre la forma del árbol más el texto de cada token. No se escribe sobre prosa: en markdown el AST no lleva el texto, así que compararlo llamaría "sólo formato" a una reescritura entera.

## Tipos de endpoint

El tipo va **adelante**, en un prefijo. Partir en el primer espacio y matchear. **Un prefijo desconocido es un error, no un fallback.**

| Prefijo | El resto es |
|---|---|
| `capture <id>` | un capture de esta capa |
| `path <stratum-path>` | una capa vecina — `path <`, `path >impl`, `path subsystems/bilinker>impl` |
| `issue <id>` | un ítem del worklist |

`repo <alias>` y `abstract` están especificados y no implementados (ADR-0005); `bilink <uuid>` también (`proposals/bilink-endpoint.md`).

## Las dos dimensiones

Un endpoint puede derivar de dos maneras independientes, y se aprueban por separado:

| Dimensión | Se compara | Deriva a |
|---|---|---|
| **ubicación** | `link` contra `accepted.link` | `RELOCATED` |
| **contenido** | el fragmento contra `accepted.hash` | `ALTERED`, `EXPANDED`, `RESTYLED` |

La de ubicación son dos ids: no abre ningún archivo, y por eso se decide siempre — incluso donde la otra degrada.

### Resolución del capture

| Estado | Significado | Fix |
|---|---|---|
| `RESOLVED` | La query matchea. | — |
| `MOVED` | El archivo cambió de path (git rename ≥ 50%). | `apply` |
| `REANCHORED` | El anchor se renombró; se localizó por similitud. | `apply` |
| `UNANCHORED` | La query no matchea y el anchor no aparece. | `recapture` |

### Aceptación

| Estado | Significado | Fix |
|---|---|---|
| `PENDING` | `accepted` ausente. | `accept` |
| `OK` | Ubicación y contenido coinciden. | — |
| `RELOCATED` | `link` ≠ `accepted.link`. | `accept --place` |
| `EXPANDED` | El fragmento contiene lo aceptado y algo más. | revisar + `accept` |
| `RESTYLED` | El texto difiere y los tokens no — sólo espaciado. | `accept` |
| `ALTERED` | El fragmento cambió. | revisar + `accept` |
| `UNRESOLVED` | El capture no resolvió. | resolver el capture |

Propios de un endpoint `path`: `TODO` (la capa todavía no existe), `CHAIN_DIRTY` (el vecino re-aceptó), `BROKEN` (la capa o el bilink vecino desaparecieron).

## Quién escribe qué

| Comando | Escribe |
|---|---|
| `capture` | un capture nuevo, si no existía |
| `check` | sólo `cache/state`. **Ni el bilink ni el capture se tocan.** |
| `accept` | `accepted` en el bilink. Lo único que escribe una decisión. |
| `apply` | acuña captures y repunta un `link`. Nunca escribe `accepted`. |
| `recapture` | repunta un `link` a mano. Tampoco acepta. |

**`apply` propone, `accept` dispone.** Un fix nunca cierra el ciclo solo: `apply` repunta y deja el endpoint en `RELOCATED`, porque mover un vínculo a otro fragmento es una decisión igual que aprobar un contenido.

Y sólo tienen fix los estados del **capture** —`MOVED`, `REANCHORED`—, que son sobre dónde está el fragmento. Ningún estado de aceptación se arregla solo: aprobar un contenido es una decisión.

Ningún comando modifica un capture existente. La única operación sobre el conjunto es agregar, y `prune` sacar los que no referencia nadie.

## El método

1. Se toca **la spec**, nunca el código primero.
2. `bilinker check .` reporta los endpoints no-OK.
3. Cada no-OK es un puntero al fragmento de código que implementaba esa spec. Se sigue con `bilinker get`.
4. Se cambia el código y se acepta.

**El inventario de trabajo de un cambio *es* la lista de no-OK.** Buscar el código a mano produce una lista que envejece el mismo día que se escribe.

## Comandos

```bash
bilinker check .                              # verificar la capa
bilinker check <path>                         # verificar lo que caiga bajo ese path
bilinker status                               # resumen sin re-verificar

bilinker get <uuid>.<N>                       # ver el fragmento
bilinker get <uuid>.<N> -B 3 -A 3             # con contexto
bilinker get <uuid>.<N> --diff                # contra el contenido aceptado
bilinker get <file>                           # endpoints que referencian el archivo

bilinker apply --dry-run                      # qué repuntaría
bilinker apply -y                             # repuntar sin confirmar

bilinker accept <uuid>.<N>                    # un endpoint
bilinker accept <uuid>                        # los dos
bilinker accept .                             # todo lo pendiente de la capa
bilinker accept --place <uuid>.<N>            # sólo la ubicación
bilinker accept --content <uuid>.<N>          # sólo el contenido

bilinker recapture <uuid>.<N> <file> [<l>:<c> <l>:<c>]   # repuntar a mano
bilinker capture <file> [<l>:<c> <l>:<c>]     # un capture suelto; sin posición, el archivo entero
bilinker capture prune                        # borrar captures sin referentes

bilinker chain new --tip <REF> --tip <REF>    # crear una cadena
bilinker chain status <uuid>                  # todos los nodos de una cadena
bilinker chain list

bilinker index --recursive                    # reconstruir el índice
bilinker migrate --recursive                  # migrar el formato
bilinker remove <uuid>                        # borrar un bilink de esta capa
```

Cada `--tip` es un path Stratum con `:LINE:COL` opcional. Sin posición captura el archivo entero. El path puede atravesar directorios comunes antes de bajar a una capa:

```bash
bilinker chain new \
  --tip 'subsystems/bilinker/concepts/capture.md:29:1' \
  --tip 'subsystems/bilinker>impl/crates/bilinker/src/capture.rs:523:1'
```

Para poblar `kind` y `name`: `--kind governs --name.0 <etiqueta> --name.1 <etiqueta>`.

## Crear una cadena

Siempre con `chain new`. Escribir los archivos a mano es lo que el formato no le pide a nadie — y `capture` verifica que la query identifique el fragmento unívocamente antes de escribir, cosa que a mano no pasa.

El fragmento tiene que estar commiteado: aceptar fija un contenido, y ese contenido tiene que existir en la historia.

**Elegir un ancla que se nombre a sí misma.** Un nodo sin nombre propio produce una query que matchea el primero de su tipo en el archivo. `capture` falla antes de escribir si no puede identificarlo, y el error dice qué seleccionar en su lugar.

| Tipo de documento | Ánclas estables | Frágil |
|---|---|---|
| Código | función, método, clase, declaración con nombre | comentario, `use`/`import` |
| Markdown | heading h1–h4, fila de tabla, bloque de código | párrafo libre |
| YAML / TOML | clave de mapping, item con `id:` | valor string libre |

## Propagación por la cadena

Un endpoint `path` **no** hashea el archivo vecino: copia el `accepted` del endpoint **estructural** de ese bilink. Es lo que evita la cascada circular — si hasheara el archivo entero, aceptar reescribiría su propio archivo y esa escritura volvería al vecino como un cambio.

```
el fragmento de B cambia   → tip-B: ALTERED
accept en tip-B            → su accepted cambia
                           → tip-A: CHAIN_DIRTY
accept en tip-A            → sincronizado
```

Siempre unidireccional, desde el endpoint estructural hacia los `path`.

## Invariantes

- Sólo `accept` escribe `accepted`.
- `check` no escribe nada versionado.
- Un `link` referencia captures de **su misma capa**. Un `accepted.link` de un endpoint `path` lleva una copia opaca del id ajeno: se compara, no se resuelve.
- Borrar un bilink nunca borra un capture.
- Un bilink tiene siempre exactamente dos endpoints. La multiplicidad la aporta el capture: un fragmento puede tener N bilinks.
- La topología de una cadena es lineal — sin ciclos ni bifurcaciones.
- `kind` y `name` son inertes: no entran en ningún hash ni en ningún estado.

## Defectos conocidos

Ninguno abierto sobre el formato. `subsystems/bilinker/proposals/` lleva lo especificado y no implementado: el endpoint de tipo `bilink`, y detectar el corrimiento con los hunks de git en vez de un escaneo.
