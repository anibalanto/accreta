# Referencia: tipos de endpoint

Un `link` en un archivo de bilink lleva **el tipo adelante**, en un prefijo. El resto del valor se interpreta en el lenguaje que el prefijo nombra.

## Discriminación de tipos

```
link: <prefijo> <resto>
```

Partir en el primer espacio y matchear el prefijo. Nada más.

| Prefijo | El resto es | Estado |
|---|---|---|
| `capture <id>` | un id de capture de esta capa | implementado |
| `path <stratum-path>` | un path Stratum hacia una capa vecina | implementado |
| `issue <id>` | un id de ítem del worklist | implementado |
| `repo <alias>` | un alias de repo ajeno | implementado |
| `abstract` | nada — no lleva valor | implementado |
| `bilink <uuid>` | otro bilink | [`proposals/bilink-endpoint.md`](../proposals/bilink-endpoint.md) |

**Un prefijo desconocido es un error, no un fallback.** Antes el endpoint layer era lo que quedaba cuando ninguna otra forma matcheaba, y eso obligaba a una regla de precedencia entre prefijos, palabras reservadas y paths. Sin fallback, esa regla no existe.

> **Los prefijos reconocidos se publican en el esquema**, y salen de la misma tabla que usa el parser. Agregar un tipo obliga a tocarla, y eso cambia el hash del esquema. Ver [versión del formato](format-version.md) § "Qué tiene que aparecer en el esquema".

---

## Endpoint capture

Identifica un fragmento, referenciando el [capture](capture.md) que lo ubica.

```
link: capture 67ba7217e0334051becd4921b55a7872
```

El id es el hash de la ubicación — sus campos, cada uno seguido de un `\0`. El endpoint no describe el fragmento: pregunta.

### `accepted`

```yaml
accepted:
  link: capture 67ba7217e0334051becd4921b55a7872   # la ubicación aprobada
  hash: c00e07602bd5…                              # el contenido aprobado
  hash_ast: 1b9e44a2f0c8…                          # opcional
```

Las dos dimensiones se aprueban por separado. Ver [aceptación](accept.md).

---

## Endpoint path

Identifica una capa vecina con un [path Stratum](../../stratum/concepts/paths.md).

```
link: path >impl
link: path <
```

La ruta es relativa a la raíz de la capa actual. `.bilink/` es implícito — nunca aparece en el valor.

### Resolución

```
resolved = ../<stratum-path>/.bilink/<uuid>.yaml
```

### `accepted`

Copia de `accepted.link` y `accepted.hash` del endpoint **estructural** del bilink adyacente. No es el hash del archivo vecino completo.

Cada valor cambia por una sola razón: `hash` cuando cambia el contenido publicado, `link` cuando cambia su ubicación aprobada. Los dos son inmunes a etiquetas, comentarios y reordenamientos del archivo vecino.

---

## Endpoint issue

Identifica un ítem del worklist — una épica, una user story o una task.

```
link: issue 3a
```

### Resolución

```
<project-root>/.stratum/worklist/<id>.<tipo>.md
```

El project root se encuentra subiendo `depth * 2` componentes desde la capa actual.

**El tipo no está en el endpoint y no hace falta que esté.** Los ítems son archivos sueltos en un solo directorio y sus ids son únicos, así que el archivo se encuentra por prefijo. Si no matchea nada el endpoint no resuelve; si matchea más de uno, el worklist tiene dos ítems con el mismo id y eso es un error suyo.

Que el tipo quede afuera es lo que hace que el endpoint sobreviva a la planificación: recolgar un ítem de otra user story cambia un campo del ítem, no el nombre de su archivo. Ver [worklist — ítem](../../worklist/concepts/item.md) § "Jerarquía".

**Se llama `issue` y no `task`** porque apunta a cualquiera de los tres tipos, y `task` es además el nombre del tipo hoja del worklist: el mismo término significaría "cualquier ítem" acá y "el ítem sin hijos" allá. El nombre sale de qué es la cosa del otro extremo, no de quién la provee — por eso tampoco es `worklist <id>`.

### `accepted`

Lleva `hash` y no lleva `link`: no hay capture que aprobar, porque la ubicación de un ítem es su id.

---

## Endpoint `abstract`

Una punta abierta: no la resuelve quien la declara, la aporta quien la consuma.

```
link: abstract
```

**No lleva valor y no lleva `accepted`.** No hay nada que bendecir del lado abierto, y con el bloque entero ausente eso es una ausencia y no una lista de campos vacíos.

Es palabra reservada, y con el tipo adelante no hace falta ninguna regla de desempate: es la única forma sin valor, y ninguna otra se le parece.

### Estado

`OPEN`, constante. Siempre sana, nunca pide acción, y `accept .` nunca la toca. Ver [la frontera](frontier.md) § "El estado es `OPEN`, constante".

---

## Endpoint repo

Identifica un fragmento **de otro proyecto**, por un alias local.

```
link: repo hsi
```

El UUID es el mismo que el del bilink remoto, así que no se escribe. El alias se declara en `.bilink/.{alias}.toml`, y es el único lugar del consumidor que sabe algo del otro repo — el `link` no contiene ninguna URL.

### Resolución

```
resolved = <clon de .{alias}.toml @ refs/bilink/{branch}>/.bilink/<uuid>.yaml
```

Es el [endpoint path](#endpoint-path) generalizado: misma convención de UUID compartido, mismo `.bilink/` implícito — sólo cambia que la dirección se resuelve por alias en vez de por path relativo.

El `.toml` declara la rama **del proyecto**; la traducción a `refs/bilink/<branch>` la hace la herramienta.

### `accepted`

Copia de `accepted.link` y `accepted.hash` del endpoint **estructural** del bilink remoto. Dos SHA-256 opacos: ninguno revela path, query, texto ni commit del proveedor.

```yaml
accepted:
  link: capture 8f2a4c6e…   # el capture del proveedor — ubicación publicada
  hash: c4e1770b…           # hash del fragmento del proveedor — contenido publicado
```

**Es una copia opaca: se compara, no se resuelve.** El `capture <id>` que lleva adentro es de la capa del proveedor, no de ésta, y buscarlo acá no encontraría nada — pero eso ya vale para un endpoint [`path`](#endpoint-path), donde el id copiado es del vecino. La forma es la misma en los dos casos, y por eso el campo se lee igual.

Lo que la vuelve inofensiva es que nadie la resuelve: de esos dos SHA-256 no se reconstruye path, query, texto ni commit del proveedor.

Ver [la frontera](frontier.md) para lo demás: qué se compara, la verificación de versión, el clon y la taxonomía de ausencia.

---

## Endpoints que todavía no existen

`bilink <uuid>` está especificado y no implementado, en [`proposals/bilink-endpoint.md`](../proposals/bilink-endpoint.md).

---

## Ánclas estables recomendadas

Qué nodo conviene capturar, por tipo de documento:

| Tipo de documento | Ánclas estables | Frágil (evitar) |
|---|---|---|
| Código (Java, Rust, TypeScript…) | función, método, clase, declaración con nombre | comentario, `use`/`import` |
| Markdown | heading h1–h4, bloque de código | párrafo libre |
| YAML / TOML | clave de mapping | valor string libre |
| JSON | clave de objeto | valor primitivo |

El criterio es que **el ancla se nombre a sí misma**. Un nodo sin nombre propio produce una query que matchea el primero de su tipo en el archivo, y un capture así apunta a otra cosa sin fallar. `bilinker capture` lo verifica antes de escribir y falla si no puede identificar el fragmento unívocamente — ver [`capture`](../commands/capture.md) § "Propiedades garantizadas".

La lista exacta por lenguaje está en [`commands/capture.md`](../commands/capture.md) § "Lenguajes soportados".
