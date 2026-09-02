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
    n:
      1:
        link: capture fe74f8b4e9fd72eeae03ea41ce520155 1b06e7c6750d68696653c9112925a54e
    accepted:
    - agree:
      - pablo
      link: capture 67ba7217e0334051becd4921b55a7872
      hash: c00e07602bd560755096b57df1ddb9ed49d816fb8af58a4ec9cde82f21f38db3
      hash_ast: 1b9e44a2f0c8d3e7a5b1c9d4e2f6a8b0c3d5e7f9a1b3c5d7e9f1a3b5c7d9e1f3
      n:
        1:
          link: capture fe74f8b4e9fd72eeae03ea41ce520155 1b06e7c6750d68696653c9112925a54e
          hash: ebdaf622a00a28d0d45d27a793ebe10a8c8c14637259fe11e4f5b82aa739b6b7
          hash_ast: 49b10d85fc2f5a7a6ecb55108007419f31acf617cceb62601fd8b14890d7b856
  1:
    link: path >impl
```

**Dos ejes por endpoint, y cada uno con su declaración y su decisión.** El fragmento y su [vecindario](accept.md#el-cierre-de-firma) se escriben con la misma forma, y cada campo tiene un escritor y uno solo:

| | declaración | decisión |
|---|---|---|
| el fragmento | `link` | `accepted[].link` |
| su vecindario | `n.1.link` | `accepted[].n.1.link` |

> **`apply` escribe las declaraciones. `accept` escribe las decisiones. `check` no escribe nada en el bilink.**

La frontera deja de ser una convención de nombres y pasa a ser estructura. Ver [aceptación](accept.md).

**Y `accepted` es una lista**, porque dos personas pueden haber aprobado versiones distintas del mismo fragmento y **ninguna de las dos se descarta**. Más de una entrada es un estado —`CONSENSUS_DIVERGED`— y no un modo de operación: ver [§ Más de un `accepted`](#más-de-un-accepted-es-un-estado-no-una-forma-de-trabajar).

No existe campo `id`: el UUID del nombre es el identificador. No existe `range`: la ubicación vive en el [capture](capture.md) que el `link` referencia. No existe `resolved_at`, ni `state`, ni `commit`: son derivados y viven en [la cache](cache.md).

### Más de un `accepted` es un estado, no una forma de trabajar

**Un endpoint sólo puede estar `OK` con exactamente una entrada.** Con dos o más el estado es `CONSENSUS_DIVERGED`, y `check` falla.

Eso es lo que vuelve sana a la lista: **no es una estructura para sostener dos verdades, es una forma de no perder ninguna mientras se resuelve.** Es transitoria por construcción — alguien mira, acepta, y colapsa a una.

La alternativa —*"una gobierna y las demás son historia"*— daría un endpoint verde con una aprobación vieja adentro, que es la clase de mentira que el resto del formato existe para impedir.

**Es un eje que no describe al fragmento.** Los demás estados dicen dónde está, qué dice y qué tipos menciona. Éste dice *"sobre esos tres no hay una sola respuesta"*, y de qué lado está el desacuerdo es de las personas, no del código.

#### Cómo colapsa, que es la regla que ya existía

`accept` sobre un endpoint divergido **deja una sola entrada: la de los valores que se están aprobando.** Las entradas cuyos valores difieren se van.

No es una regla nueva — es exactamente lo que [`agree`](accept.md#quiénes-aprobaron) ya hacía:

> quien aprobó el hash anterior no aprobó el nuevo… los aprobadores anteriores quedan donde siempre estuvieron, **en los commits que escribieron los valores anteriores**.

Lo único que la lista cambia es **qué pasa entre las dos aceptaciones**: antes la segunda pisaba a la primera en silencio, ahora conviven visibles hasta que alguien resuelve. El destino final de la aprobación desplazada es el mismo de siempre: la ref.

Y si los valores que se aceptan **coinciden** con los de una entrada existente, no hay colapso que hacer: quien acepta se suma a su `agree`, y las demás entradas siguen ahí. Sigue divergido, y es correcto — sumarse a un lado no resuelve un desacuerdo.

#### Una entrada es completa, y por eso lleva un solo `agree`

**Si una entrada está escrita, se aceptó entera.** No hay endoso parcial de una entrada, y por eso un `agree` adentro de `n.1` no nombra nada.

**Y no hay cómo aceptar a medias.** Con firma resoluble y sin proveedor, `accept` **se niega** — no tenés el mapa completo del vecindario, así que no hay nada que aprobar. La única alternativa es declararlo con `--no-n1`, que es renunciar y no abstenerse. No existe el camino *"apruebo la firma y el vecindario no lo miré"*.

**Lo que sí existe es que dos personas aprueben el mismo fragmento y vecindarios distintos** — y eso ya tiene forma: son **dos entradas**.

```yaml
accepted:
- agree:
  - anibal
  link: capture <a>
  hash: h1
  n: { 1: { link: capture <n-a>, hash: hA } }

- agree:
  - juan                   # mismo fragmento que Pedro…
  link: capture <b>
  hash: h2
  n: { 1: { link: capture <n-b>, hash: hB } }

- agree:
  - pedro                  # …y otro vecindario
  link: capture <b>
  hash: h2
  n: { 1: { link: capture <n-c>, hash: hC } }
```

**`agree` va en bloque incluso acá, donde hay un nombre por entrada.** Compactarlo a `agree: [anibal]` no rompe nada hoy y le saca al campo lo que lo hace servir: [`git blame` sólo atribuye una línea a un commit](accept.md#un-nombre-por-línea-y-por-eso-no-guarda-su-commit), así que el día que la entrada tenga dos firmantes en una línea el primero se pierde. Que en `n` de acá abajo sí esté compactado no es una excepción: ahí la forma es tipografía, y en `agree` es el mecanismo.

Juan y Pedro coinciden en `link` y `hash` y difieren en `n`. **Son dos contratos distintos**, y por lo tanto dos entradas — no una entrada con dos endosos parciales.

**Es la regla que el formato ya tenía**, extendida una palabra:

> Como los valores direccionan por contenido, *"estar de acuerdo"* no es ambiguo: es haber aprobado este hash y esta ubicación — **y este vecindario** — y no otros.

La identidad de una entrada es su tupla entera. Dos personas convergen en una sola entrada **sólo si coinciden en todo**, y si difieren en cualquier nivel, difieren.

#### Lo que no está resuelto: si cruza la frontera

Un endpoint `repo` copia el `accepted` del proveedor. **Con el proveedor divergido no hay uno solo que copiar**, y las tres salidas son malas de distinta forma:

| | |
|---|---|
| no copiar nada | el consumidor queda bloqueado por un desacuerdo interno del proveedor, del que no es parte |
| copiar la lista | el consumidor hereda un desacuerdo ajeno y su propio `accepted` deja de ser una decisión suya |
| rechazar y decirlo | honesto —*"no se puede consumir un contrato que sus autores no acuerdan"*— pero pide un estado propio del lado del consumidor, porque **el consumidor no está divergido** |

La tercera es la que se parece más al resto del diseño, y aun así abre un nombre nuevo. **Queda abierta a propósito**: es un caso que no se puede alcanzar hasta que haya un proveedor real con divergencia, y decidirlo antes sería inventar el nombre de un estado que nadie vio.

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

### `as`

Con qué generador se capturó ese extremo. El valor es el mismo nombre que tomó [`--as.N`](../commands/chain.md#--as-quién-genera-la-query) al escribirlo —`interface`, `spring-controller`—, y su ausencia significa que no se sabe con qué se capturó: es lo que dice cualquier archivo escrito antes de este campo, y lo que dice un capture del núcleo.

```yaml
endpoint:
  0:
    link: capture 67ba7217e0334051becd4921b55a7872
    as: spring-controller
  1:
    link: abstract
```

**Va por endpoint porque el hecho es de un extremo.** Un tip puede ser un endpoint de Spring y el otro `abstract`; un campo arriba, al lado de `kind`, afirmaría sobre la relación entera algo que vale de un lado solo. Es la misma razón por la que `--as` va por tip y no global, así que el registro de lo que hizo `--as` va donde va `--as`.

**Y no entra en `kind`**, que ya contesta otra pregunta: `kind` clasifica *qué clase de relación* se declara y `as` dice *con qué receta se capturó este extremo*. En el mismo campo no pueden convivir.

**Es la receta, no el valor.** Lo que se guarda no es el nombre ni la ruta que el generador sabe componer —eso sale del fragmento cada vez que se lee, que es lo que no puede mentir— sino con qué componerlos. Un valor derivado y guardado envejece en silencio; la regla que lo deriva no cambia cuando cambia el valor.

**Un `as` que nombra un generador que no está instalado es un dato que no se pudo usar, nunca un error.** El capture sigue resolviendo y `check` sigue contestando: lo único que se degrada es lo que ese generador sabía componer. Ver [`chain new`](../commands/chain.md#el-capture-no-deja-rastro-y-el-bilink-sí).

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
| `CONTRACT_UNVERIFIED` | Hay `n` adquirido y **nadie pudo resolver el vecindario** | levantar el proveedor · o nada |

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
4. `accepted` es una lista de cero o más entradas, y cada entrada está completa. La lista vacía y la ausencia son lo mismo: `PENDING`.
5. `accepted.hash` de un endpoint estructural: SHA-256 del fragmento aprobado.
6. Una entrada de `accepted` de un endpoint `path`: copia de `link`, `hash` y `n` de la entrada del endpoint estructural del bilink adyacente. Nunca el hash del archivo vecino. Un vecino divergido no se copia — ver § "si cruza la frontera".
7. Un endpoint `issue` se hashea como el contenido del archivo del ítem. No tiene capture, así que su `accepted` no lleva `link`.
8. `state.N = OK` si y sólo si **hay exactamente una entrada** en `accepted`, y para ella `link` == `accepted[0].link` y el hash actual == `accepted[0].hash`.
8.b Con más de una entrada, `state.N = CONSENSUS_DIVERGED`, **sin evaluar los otros ejes**: no hay un valor contra el cual compararlos.
8.c El vecindario se compara igual y un nivel más abajo: `n.1.link` contra `accepted[0].n.1.link`, y el fold de hoy contra `accepted[0].n.1.hash`. Sin proveedor ese eje degrada y los otros se deciden igual.
9. El `link` de un endpoint estructural referencia exactamente un capture de su misma capa. Un `n.1.link` referencia cero o más, todos de su misma capa.
10. Un bilink no contiene `file`, `query` ni `range`: los dos primeros viven en el capture y el tercero en la cache. Vale igual para los captures de `n.1.link`.
11. Un bilink no contiene `state`, `commit` ni ningún derivado: viven en la cache.
12. La topología de la cadena es lineal — sin ciclos ni bifurcaciones.
13. Sólo se puede aceptar un endpoint sobre un fragmento commiteado.
14. `kind`, `name` y `as` son inertes: no afectan ningún hash ni ningún estado. `accepted.agree` tampoco los afecta, pero no es decoración: lo escribe `accept` y es parte de la decisión. Ver [aceptación](accept.md#quiénes-aprobaron).
15. Un campo desconocido se rechaza con su nombre, nunca se descarta.
16. Ningún `accept` descarta una entrada de `accepted` cuyos valores coincidan con los que se están aprobando: se une el `agree`. Sólo se descartan las entradas que aprobaban **otros** valores.
17. Un capture referenciado por un `n.1.link` —de la declaración o de una decisión— cuenta como referenciado para [`prune`](../commands/capture.md).
