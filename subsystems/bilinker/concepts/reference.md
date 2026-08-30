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
| `repo <alias>` | un alias de repo ajeno | [ADR-0005](../.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md) |
| `abstract` | nada — no lleva valor | [ADR-0005](../.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md) |
| `bilink <uuid>` | otro bilink | [`proposals/bilink-endpoint.md`](../proposals/bilink-endpoint.md) |

**Un prefijo desconocido es un error, no un fallback.** Antes el endpoint layer era lo que quedaba cuando ninguna otra forma matcheaba, y eso obligaba a una regla de precedencia entre prefijos, palabras reservadas y paths. Sin fallback, esa regla no existe.

> **Los prefijos reconocidos se publican en el esquema**, y salen de la misma tabla que usa el parser. Agregar un tipo obliga a tocarla, y eso cambia el hash del esquema. Ver [versión del formato](format-version.md) § "Qué tiene que aparecer en el esquema".

---

## Endpoint capture

Identifica un fragmento, referenciando el [capture](capture.md) que lo ubica.

```
link: capture 67ba7217e0334051becd4921b55a7872
```

El id es `H(file, query, offset)` — el hash de la ubicación. El endpoint no describe el fragmento: pregunta.

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

## Endpoints que todavía no existen

`repo <alias>` y `abstract` llegan con la frontera entre proyectos, en [ADR-0005](../.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md). Son aditivos: ningún archivo existente los usa.

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
