# Especificación: Formato del archivo `.bilink`

Un bilink es **una declaración y dos decisiones**: qué dos cosas están vinculadas, y qué versión de cada una alguien aprobó. Nada más entra al archivo — todo lo demás se puede reconstruir y vive en [la cache](cache.md).

## Ubicación y nomenclatura

Los bilinks viven en carpetas `.bilink/` dentro de cada capa del proyecto. El nombre del archivo es un **UUID v4**: es a la vez el identificador de la cadena y el mecanismo de localización entre capas.

```
bilinker/
  .bilink/
    7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.yaml   ← tip (capa spec)
  .stratum/
    impl/
      .bilink/
        7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.yaml   ← tip (capa impl)
```

El mismo UUID aparece en todas las capas que participan de una cadena.

**La extensión es `.yaml`.** El tipo lo dice la carpeta que lo contiene; repetirlo en el nombre sería redundante.

## Estructura del archivo

```yaml
endpoint:
  0:
    link: capture 67ba7217e0334051becd4921b55a7872
    accepted:
      agree:
      - pablo
      link: capture 67ba7217e0334051becd4921b55a7872
      hash: c00e07602bd560755096b57df1ddb9ed49d816fb8af58a4ec9cde82f21f38db3
      hash_ast: 1b9e44a2f0c8d3e7a5b1c9d4e2f6a8b0c3d5e7f9a1b3c5d7e9f1a3b5c7d9e1f3
  1:
    link: path >impl
```

Dos campos por endpoint, y cada uno tiene un escritor y uno solo:

> **`apply` escribe `link`. `accept` escribe `accepted`. `check` no escribe nada en el bilink.**

La frontera deja de ser una convención de nombres y pasa a ser estructura. Ver [aceptación](accept.md).

No existe campo `id`: el UUID del nombre es el identificador. No existe `range`: la ubicación vive en el [capture](capture.md) que el `link` referencia. No existe `resolved_at`, ni `state`, ni `commit`: son derivados y viven en [la cache](cache.md).

## Tipos de endpoint

**El tipo es explícito, en un prefijo.** El prefijo no nombra el destino sino en qué lenguaje está el resto, que es lo que el parser necesita saber:

| Prefijo | El resto es |
|---|---|
| `capture <id>` | un id de [capture](capture.md) de esta capa |
| `path <stratum-path>` | un [path Stratum](../../stratum/concepts/paths.md) hacia una capa vecina |
| `issue <id>` | un id de ítem del worklist |
| `repo <alias>` | un alias de repo ajeno, declarado en `.bilink/.{alias}.toml` |
| `abstract` | nada — la punta abierta de un bilink que otro proyecto consume |

`path` y no `layer` porque un stratum-path también cruza a sub-proyectos —`*/subsystems/lattice`— que el [modelo de capas](../../stratum/concepts/layer-model.md) distingue de las capas internas: `layer` afirmaría de más.

Sin fallback no hay desempate. Antes el endpoint layer era lo que quedaba cuando ninguna otra forma matcheaba, y eso obligaba a una regla de precedencia entre prefijos, palabras reservadas y paths. Con el tipo adelante, esa regla no hace falta.

Los dos últimos son [la frontera entre proyectos](frontier.md), y son aditivos: ningún archivo existente los usa, y todos siguen siendo válidos. El endpoint de tipo **bilink** está especificado y no implementado, en [`proposals/bilink-endpoint.md`](../proposals/bilink-endpoint.md), con sus dos casos de uso: la gobernanza y el bilink de tarea.

Un endpoint estructural **no describe el fragmento**: referencia un capture. Lo que sí es propio de cada endpoint es **qué se aceptó**: dos bilinks sobre el mismo capture pueden haber aprobado contenidos distintos y reportar estados distintos.

### Resolución de un endpoint `path`

Dado `link: path <stratum-path>` en `<capa-actual>/.bilink/<uuid>.yaml`:

1. Resolver el path Stratum tomando como base la raíz de la capa actual.
2. Usar el resultado como `<layer-path>`:

```
resolved = ../<layer-path>/.bilink/<uuid>.yaml
```

El `../` sube del directorio `.bilink/` a la raíz de la capa. La carpeta `.bilink/` nunca aparece en el valor — siempre es implícita.

## Topología

### Link directo (misma capa)

Un bilink puede conectar dos fragmentos dentro de la misma capa: los dos `link` son endpoints estructurales. Hay un único archivo — no hay traversal.

```
[fragmento A] ←→ [fragmento B]
```

### Cadena entre capas

Una **cadena** es una secuencia lineal de bilinks con el mismo UUID que conecta dos fragmentos a través de una o más capas:

- **tip**: un endpoint estructural + un endpoint `path`. Son los extremos, y siempre hay exactamente dos.
- **mid**: los dos endpoints son `path`. Puede haber cero o más.

```
[fragmento] ←→ tip ←→ mid* ←→ tip ←→ [fragmento]
```

La topología es estrictamente lineal — sin ciclos ni bifurcaciones.

### Bilink con capa no creada todavía

Un endpoint `path` puede apuntar a una capa que aún no existe. El estado `TODO` dice que la conexión está planeada, no que haya un error. Una vez creada la capa y aceptado el endpoint, pasa a `OK`.

## Campos semánticos

Opcionales, y **inertes**: no afectan ningún hash ni ningún estado. Son declaración, así que van al lado de `link` y los escribe quien escribe `link`.

```yaml
kind: governs
endpoint:
  0:
    link: capture 67ba7217e0334051becd4921b55a7872
    name: architecture-decision
  1:
    link: path >impl
    name: spec-impl-bridge
```

### `kind`

Clasifica la relación. Valor libre; valores definidos:

| Valor | Significado |
|-------|-------------|
| *(ausente)* | Vínculo estructural — relación de implementación entre fragmentos |
| `governs` | Decisión o documento que gobierna un vínculo entre capas |

`governs` es el único valor definido y **todavía no se puede expresar**: exige que un `link` apunte a otro bilink, y ese tipo de endpoint está en [`proposals/bilink-endpoint.md`](../proposals/bilink-endpoint.md). El campo existe y se preserva; su valor documentado espera a que vuelva el endpoint.

No existe un `kind` para relaciones de llamada. Un bilink declara una referencia que un humano aceptó; las aristas de llamada las deriva una herramienta del código actual y las agrega [lattice](../../lattice/overview.md) como aristas `derived`. Declararlas a mano crearía un duplicado permanente que hay que mantener sincronizado.

### `name`

Etiqueta del rol semántico del endpoint en la relación que `kind` declara. Texto libre. Va **adentro** del endpoint, no como `name.N` suelto: es un dato de una punta y ahora hay dónde ponerlo.

## Estados

Ningún estado vive en el archivo: `check` los calcula y los escribe en [la cache](cache.md).

### Endpoint estructural

Un endpoint puede desalinearse en dos dimensiones —dónde está y qué dice— y los estados las distinguen. Si el capture no resuelve, eso es estado del capture y el bilink sólo registra que no puede evaluarse.

| Estado | Significado | Cómo se sale |
|--------|-------------|--------------|
| `PENDING` | `accepted` ausente | `bilinker accept` |
| `OK` | La ubicación y el contenido coinciden con lo aceptado | — |
| `RELOCATED` | `link` ≠ `accepted.link` — la ubicación cambió y nadie la aprobó | `bilinker accept --place` |
| `RESTYLED` | El texto difiere pero el AST coincide — sólo formato | `bilinker accept` |
| `ALTERED` | El contenido cambió | revisar + `bilinker accept` |
| `EXPANDED` | El fragmento creció alrededor de lo aceptado | `bilinker apply` + `accept` |
| `UNRESOLVED` | El capture referenciado no resuelve | `bilinker apply` o `recapture` |
| `CONTRACT_RESTYLED` | El vecindario se reformateó y su AST no cambió | `bilinker accept` |
| `CONTRACT_ALTERED` | Un vecino cambió: **el contrato se movió** | revisar + `bilinker accept` |
| `CONTRACT_UNVERIFIED` | Hay `n1` adquirido y **nadie pudo resolver el vecindario** | levantar el proveedor · o nada |

**Los tres últimos son de un eje aparte**: no hablan del fragmento sino de [los tipos que su firma menciona](accept.md#el-cierre-de-firma). Llevan prefijo por eso — `ALTERED` y `CONTRACT_ALTERED` no son grados de lo mismo, son dos preguntas.

**Y sólo aparecen cuando el eje del contenido dice `OK`.** Un endpoint tiene un estado y no dos, así que hay que elegir cuál nombrar: si el fragmento mismo cambió, eso se reporta y alguien va a mirar igual. Lo que el eje del vecindario aporta es justamente el caso donde **el fragmento no cambió** y aun así el contrato se movió.

**`CONTRACT_UNVERIFIED` no sale con 1.** Es de la familia de `LAYER_UNREACHABLE` y `REMOTE_UNREACHABLE` — *no pude ver el otro lado*—, y no es trabajo pendiente de nadie: un `check` sin daemon es un modo de operación normal, no un repo en mal estado.

**`RELOCATED` sale con 1.** Es la contrapartida de que `apply` ya no devuelve un endpoint a `OK`: mover un vínculo a otro fragmento es una decisión, y una decisión sin aprobar es trabajo pendiente.

`UNRESOLVED` absorbe lo que antes eran `UNANCHORED`, `DELETED`, `BROKEN` y `MOVED` del lado del bilink: el problema no es el vínculo sino la ubicación, y quien lo detalla es el capture.

### Endpoint `path`

| Estado | Significado | Cómo se sale |
|--------|-------------|--------------|
| `TODO` | `accepted` ausente **y** la capa apuntada no existe todavía | Crear la capa + `accept` |
| `PENDING` | `accepted` ausente y la capa existe | `bilinker accept` |
| `OK` | Los dos valores copiados coinciden con los del vecino | — |
| `CHAIN_DIRTY` | El endpoint estructural adyacente fue re-aceptado | `bilinker accept` |
| `LAYER_UNREACHABLE` | La capa está declarada y no clonada | `stratum pull` |
| `LAYER_UNCONFIGURED` | Ni declarada ni presente, con aceptación previa | Declarar la capa · o · `remove` |
| `BROKEN` | La capa existe y el `.bilink` del UUID no está, o el bilink adyacente no tiene endpoint estructural aceptado | Restaurar + `accept` · o · `remove` |

Los endpoints `path` no tienen capture: apuntan a una capa, no a un fragmento.

`bilinker remove` elimina el bilink de la capa actual. Los vecinos detectan `BROKEN` en el próximo `check` y deciden: reparar o remover. La remoción se propaga hop a hop.

**Las tres ausencias son cosas distintas y se arreglan distinto**, que es por qué no comparten nombre: a una capa declarada le falta traerla, a una sin declarar le falta declararla, y un `.bilink` que desapareció bajo una capa presente es una regresión. Un solo `UNREACHABLE` no distinguía *"me falta traer algo"* de *"algo se rompió"*, que es la diferencia que decide si alguien tiene que mirar. Ver [la frontera](frontier.md) § "Taxonomía de ausencia".

### Endpoint `abstract`

| Estado | Significado | Cómo se sale |
|--------|-------------|--------------|
| `OPEN` | La punta está abierta a quien la consuma | — |

Constante: no hay contra qué compararla. Nunca pide acción y `accept .` nunca la toca.

### Endpoint repo

| Estado | Significado | Cómo se sale |
|--------|-------------|--------------|
| `PENDING` | `accepted` ausente y el clon está | `bilinker accept` |
| `OK` | Los dos valores copiados coinciden con los del proveedor | — |
| `CHAIN_DIRTY` | El proveedor re-aceptó su fragmento | revisar + `bilinker accept` |
| `REJECTED` | La otra punta dejó de ser `abstract` | investigar — el vínculo no se sostiene |
| `REMOTE_UNREACHABLE` | El repo del proveedor no está clonado | lo clona bilinker |
| `BROKEN` | El clon está y el `.bilink` del UUID no | investigar — es regresión |

`CHAIN_DIRTY` no distingue si el proveedor movió el fragmento o cambió su contenido: eso sale de **cuál de los dos valores** difiere, y lo dice `check`. Ver [la frontera](frontier.md) § "La comparación, sin abrir el clon".

## Propagación

Cada nodo ancla en los **valores aceptados** del endpoint estructural de su vecino — no en el hash del archivo vecino:

1. El contenido de un fragmento cambia. `check` reporta `ALTERED`.
2. Alguien revisa y acepta: `accepted.hash` del endpoint estructural se actualiza.
3. El nodo adyacente compara su copia contra ese valor → difieren → `CHAIN_DIRTY`.
4. Alguien acepta el endpoint `path` → su copia se sincroniza.

**Aceptar un endpoint `path` sólo escribe su propio archivo.** Nunca modifica el del vecino, así que no hay cascada circular: la propagación es unidireccional desde el endpoint estructural que cambió hacia los nodos que lo referencian.

Y por eso `check` no propaga nada: refrescar la cache no cambia ningún valor aceptado. Sólo `accept` mueve la cadena.

## Semántica de parseo

- El archivo es YAML. Los tipos están definidos en Rust y el esquema JSON se genera de ellos — ver [versión del formato](format-version.md).
- **Los campos desconocidos se rechazan**, con el nombre del campo. Antes se descartaban en silencio, que es cómo un binario viejo vaciaba las aceptaciones de uno nuevo.
- La aridad es fija: exactamente `0` y `1` bajo `endpoint`. Tres endpoints se rechaza; que falte el `1` también. Deja de ser algo que hay que verificar y pasa a ser algo que no se puede escribir.
- `accepted` sin `hash` se rechaza. Un `hash` suelto fuera del bloque, también.
- Las claves `0:` y `1:` matchean por nombre, no por posición, y no llevan comillas.
- El archivo usa UTF-8 sin BOM.

## Ejemplo completo: cadena de 2 nodos spec → impl

Cuatro archivos: dos bilinks —uno por capa— y un capture en cada una.

```yaml
# capa spec — .bilink/capture/c1a2b3c4e5f6a7b8c9d0e1f2a3b4c5d6.yaml
file: commands/check.md
query: |-
  (section (atx_heading (inline) @n0 (#eq? @n0 "Firma"))) @target
```

```yaml
# capa spec — .bilink/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.yaml   (tip)
endpoint:
  0:
    link: capture c1a2b3c4e5f6a7b8c9d0e1f2a3b4c5d6
    accepted:
      link: capture c1a2b3c4e5f6a7b8c9d0e1f2a3b4c5d6
      hash: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
  1:
    link: path >impl
    accepted:
      link: capture d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0
      hash: b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3
```

```yaml
# capa impl — .bilink/capture/d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0.yaml
file: crates/bilinker/src/check.rs
query: |-
  (function_item name: (identifier) @n0 (#eq? @n0 "check")) @target
```

```yaml
# capa impl — .bilink/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.yaml   (tip)
endpoint:
  0:
    link: path <
    accepted:
      link: capture c1a2b3c4e5f6a7b8c9d0e1f2a3b4c5d6
      hash: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
  1:
    link: capture d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0
    accepted:
      link: capture d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0
      hash: b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3
```

Cada endpoint `path` copia los **dos** valores del endpoint estructural de su vecino: qué ubicación y qué contenido se aprobaron ahí.

## Invariantes

1. El nombre del archivo es un UUID v4 válido con extensión `.yaml`.
2. Existen exactamente los endpoints `0` y `1`. La aridad es fija: la multiplicidad la aporta el capture. Ver [capture.md](capture.md) § "El fan-out vive del lado del capture".
3. Un bilink de misma capa tiene dos endpoints estructurales. Una cadena entre capas tiene exactamente dos tips.
4. `accepted` está completo o ausente. Su ausencia es `PENDING`.
5. `accepted.hash` de un endpoint estructural: SHA-256 del fragmento aprobado.
6. `accepted` de un endpoint `path`: copia de `accepted.link` y `accepted.hash` del endpoint estructural del bilink adyacente. Nunca el hash del archivo vecino.
7. Un endpoint `issue` se hashea como el contenido del archivo del ítem. No tiene capture, así que su `accepted` no lleva `link`.
8. `state.N = OK` si y sólo si `link` == `accepted.link` **y** el hash actual == `accepted.hash`.
9. Un endpoint estructural referencia exactamente un capture de su misma capa.
10. Un bilink no contiene `file`, `query` ni `range`: los dos primeros viven en el capture y el tercero en la cache.
11. Un bilink no contiene `state`, `commit` ni ningún derivado: viven en la cache.
12. La topología de la cadena es lineal — sin ciclos ni bifurcaciones.
13. Sólo se puede aceptar un endpoint sobre un fragmento commiteado.
14. `kind` y `name` son inertes: no afectan ningún hash ni ningún estado. `accepted.agree` tampoco los afecta, pero no es decoración: lo escribe `accept` y es parte de la decisión. Ver [aceptación](accept.md#quiénes-aprobaron).
15. Un campo desconocido se rechaza con su nombre, nunca se descarta.
