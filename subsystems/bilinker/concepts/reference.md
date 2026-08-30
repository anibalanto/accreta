# Referencia: tipos de endpoint

Un `link.N` en un archivo `.bilink` puede ser de cuatro tipos. Dos se reconocen por un prefijo explícito; los otros dos, por la forma del valor.

## Discriminación de tipos

Se evalúa en este orden, y el primero que matchea gana:

| # | Condición | Tipo |
|---|---|---|
| 1 | Comienza con `capture ` | **Estructural** — referencia a un capture de esta capa |
| 2 | Comienza con `task ` | **Task** |
| 3 | Contiene `::` | **Estructural embebido** — formato anterior al split |
| 4 | Último componente tiene extensión (contiene `.`) | **Estructural embebido**, archivo completo — formato anterior |
| 5 | Ninguna de las anteriores | **Layer** |

Las filas 3 y 4 son el formato **anterior al split capture/bilink**: la ubicación viajaba dentro del `.bilink` en vez de vivir en un `.capture` aparte. El parser las sigue aceptando para que [`bilinker migrate`](../commands/migrate.md) pueda convertirlas; un endpoint estructural nuevo es siempre la fila 1. Ver [capture](capture.md).

---

## Endpoint estructural

Identifica un archivo o fragmento dentro de un archivo.

### Forma vigente

```
capture <uuid>
```

El fragmento no se describe acá: se describe una sola vez en el [capture](capture.md), que guarda `file`, `query` y `offset`. Varios bilinks pueden referenciar el mismo.

Lo que sigue es el **formato anterior**, con la ubicación embebida. Se documenta porque el parser todavía lo lee y `bilinker migrate` lo convierte; no se escribe más.

### Forma completa

```
file :: query [:: start~end]
```

### Solo archivo

```
file.ext
```

### `file`

Ruta relativa a la raíz de la layer actual.

### `query`

S-expression tree-sitter que identifica el nodo AST contenedor del fragmento. La captura de resultado usa `@target`:

```scheme
; Método en clase Java
(class_declaration
  name: (identifier) @n0 (#eq? @n0 "Persona")
  body: (class_body
    (method_declaration
      name: (identifier) @n1 (#eq? @n1 "vote")) @target))

; Párrafo bajo heading en Markdown
(section
  (atx_heading (inline) @n0 (#eq? @n0 "Decisión"))
  (paragraph) @target)

; Campo en YAML
(block_mapping_pair
  key: (flow_node) @n0 (#eq? @n0 "impl")
  value: (_) @target)

; Función top-level en Rust
(function_item
  name: (identifier) @n0 (#eq? @n0 "process_event")) @target
```

Reglas:
- `@target` va **fuera** del paréntesis de cierre del nodo.
- Capturas auxiliares usan nombres únicos incrementales: `@n0`, `@n1`, `@n2`, …

### `start~end` (opcional)

Offsets en bytes relativos al inicio del nodo `@target`. Se omite cuando la selección es el nodo completo.

### Representación multilínea

Las líneas de continuación (que no comienzan con clave reconocida) se concatenan al valor anterior con un espacio:

```
link.0: src/Persona.java :: (class_declaration
  name: (identifier) @n0 (#eq? @n0 "Persona")
  body: (class_body
    (method_declaration
      name: (identifier) @n1 (#eq? @n1 "vote")) @target))
```

### `hash.N` y `commit.N`

SHA-256 del fragmento y commit del repo al momento de la última aceptación.

---

## Endpoint layer

Identifica una layer adyacente usando un path Stratum.

### Forma

```
link.N: <stratum-path>
```

La ruta es relativa a la raíz de la layer actual. `.bilink/` es implícito — nunca aparece en el valor.

### Resolución

```
resolved = ../<stratum-path>/.bilink/<uuid>.bilink
```

Ejemplos:
```
link.1: .stratum/impl
→ ../.stratum/impl/.bilink/<uuid>.bilink

link.0: ../..
→ ../../../.bilink/<uuid>.bilink
```

### `hash.N` y `commit.N`

Copia del `hash.N` y `commit.N` del endpoint **estructural** del bilink adyacente. No es el hash del archivo `.bilink` adyacente completo.

---

## Endpoint task

Identifica un ítem del worklist.

### Forma

```
task <id>
```

### Resolución

```
<project-root>/.stratum/worklist/<id>.<tipo>.md
```

El project root se encuentra subiendo `depth * 2` componentes desde la layer actual (donde `depth = stratum::depth(layer_root)`).

**El tipo no está en el endpoint y no hace falta que esté.** Los ítems del worklist son archivos sueltos en un solo directorio y sus ids son únicos, así que el archivo se encuentra por prefijo: `<id>.` seguido de cualquier tipo. Si el prefijo no matchea nada el endpoint no resuelve; si matchea más de uno, el worklist tiene dos ítems con el mismo id y eso es un error suyo, no una ambigüedad del formato.

Que el tipo quede afuera es lo que hace que el endpoint sobreviva a la planificación: recolgar un ítem de otra user story cambia un campo del ítem, no el nombre de su archivo. Ver [worklist — ítem](../../worklist/concepts/item.md) § "Jerarquía".

### `hash.N` y `commit.N`

SHA-256 del contenido del archivo del ítem y commit del repo raíz.

---

## Endpoint bilink

Especificado y no implementado: vive en [`proposals/bilink-endpoint.md`](../proposals/bilink-endpoint.md).

---

## Âncoras estables recomendadas

| Tipo de documento | Âncoras estables | Frágil (evitar) |
|---|---|---|
| Código (Java, Rust, Python…) | función, método, clase, declaración | comentario inline |
| Markdown | heading h1–h4, bloque de código | párrafo libre |
| YAML / TOML | clave de mapping | valor string libre |
| JSON | clave de objeto | valor primitivo |
