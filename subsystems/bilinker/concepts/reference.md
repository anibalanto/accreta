# Especificación: Formato de referencia bilinker

## Dos tipos de endpoint

Un `link.N` en un archivo `.bilink` es de uno de dos tipos. La presencia de `::` es el discriminador sin ambigüedad:

| Tipo | Forma | Distinguido por |
|---|---|---|
| **Estructural** | `workspace :: file :: query [:: start~end]` | Contiene `::` |
| **Layer** | `<ruta-relativa-a-layer>` | Sin `::` |

---

## Endpoint estructural

Identifica un fragmento dentro de un archivo de código o documento.

### Estructura

```
workspace :: file :: query [:: start~end]
```

### `workspace`

Nombre lógico definido en `.bilinker.toml`. Provee:
- El directorio raíz para resolver rutas relativas.
- El lenguaje (gramática tree-sitter) con el que parsear los archivos del workspace.

Un archivo pertenece a exactamente un workspace. El workspace actúa como namespace: dos archivos con el mismo nombre en distintos workspaces son referencias distintas.

### `file`

Ruta relativa al directorio raíz del workspace. Forma parte de la identidad del extremo: distingue `ar/example/Persona.java` de `ar/other/Persona.java`. Aplica igual a archivos de código y de documentación.

### `query`

S-expression tree-sitter que identifica el nodo AST contenedor del fragmento. Se construye subiendo en el árbol AST desde la selección del usuario hasta el primer ancestro con nombre estable (función, método, clase, heading, campo de mapping).

**Reglas de construcción:**
1. El nodo target se identifica con la captura `@target`, colocada **fuera** del
   paréntesis de cierre: `(method_declaration ...) @target`.
2. Cada captura auxiliar usa un nombre único incremental: `@n0`, `@n1`, `@n2`, …
   Los nombres duplicados causan error "Impossible pattern" en tree-sitter.
3. Los predicados usan el tipo real del nodo hijo según el AST — no se asumen tipos
   (`identifier`, `type_identifier`, etc. varían por gramática y versión).

**Ejemplos:**

```scheme
; Método vote en la clase Persona (Java)
(class_declaration
  name: (identifier) @n0 (#eq? @n0 "Persona")
  body: (class_body
    (method_declaration
      name: (identifier) @n1 (#eq? @n1 "vote")) @target))

; Párrafo bajo el heading "Decisión" (Markdown)
(section
  (atx_heading
    (inline) @n0 (#eq? @n0 "Decisión"))
  (paragraph) @target)

; Valor del campo "impl" en un mapping YAML
(block_mapping_pair
  key: (flow_node) @n0 (#eq? @n0 "impl")
  value: (_) @target)

; Función top-level (Rust)
(function_item
  name: (identifier) @n0 (#eq? @n0 "process_event")) @target
```

### `start~end` (opcional)

Offsets en bytes relativos al inicio del texto del nodo matcheado por `@target`. Se incluye únicamente cuando la selección es un sub-fragmento del nodo.

- Si se omite: la referencia apunta al nodo completo.
- Si se incluye: identifica el sub-fragmento exacto dentro del nodo.

`start~end` es auto-corregible (estado DISPLACED): si el hash se encuentra en otro offset dentro del mismo nodo, bilinker actualiza `start~end` en el auto-fix.

### `hash.N` y `commit.N` para endpoints estructurales

SHA-256 del fragmento y commit del repo en el momento de la última aceptación.

- `hash.N` ausente → PENDING.
- Hash actual == `hash.N` → OK.
- Hash actual en otro offset del mismo nodo → DISPLACED.
- Hash actual ≠ `hash.N` → contenido cambió (ALTERED) o fue eliminado.

### Representación multilínea

Cuando la query es larga puede ocupar varias líneas. Las líneas de continuación son las que no comienzan con una clave reconocida:

```
link.0: java-demo :: src/Persona.java :: (class_declaration
  name: (identifier) @n0 (#eq? @n0 "Persona")
  body: (class_body
    (method_declaration
      name: (identifier) @n1 (#eq? @n1 "vote")) @target))
```

---

## Endpoint layer

Identifica una layer del proyecto que contiene un nodo de la misma cadena.

### Forma

```
link.N: <ruta-relativa-a-layer>
```

La ruta es relativa a la **raíz de la layer actual** (el directorio que contiene la carpeta `.bilink/`). La carpeta `.bilink/` nunca aparece en el valor — es implícita.

### Resolución

Dado `link.N: <layer-path>` en `<current-layer>/.bilink/<uuid>.bilink`:

```
resolved = ../<layer-path>/.bilink/<uuid>.bilink
```

El `../` sube del directorio `.bilink/` a la raíz de la layer actual, navega `<layer-path>`, y busca `.bilink/<uuid>.bilink` en esa layer.

**Ejemplos:**

```
# tip en spec layer root
# archivo: .bilink/7f3d8e9a.bilink
link.1: .stratum/impl
→ ../.stratum/impl/.bilink/7f3d8e9a.bilink

# tip en impl layer
# archivo: .stratum/impl/.bilink/7f3d8e9a.bilink
link.0: ../..
→ ../../../.bilink/7f3d8e9a.bilink
```

### `hash.N` y `commit.N` para endpoints layer

SHA-256 del archivo `.bilink` adyacente completo y commit del repo adyacente en el momento de la última aceptación. Si cualquier campo dentro del `.bilink` referenciado cambia (fragmento, estado, cache), su hash cambia y difiere de `hash.N`, lo que habilita la propagación reactiva en la cadena.

---

## Âncoras estables por tipo de documento

| Tipo de documento | Âncoras estables (recomendado) | Frágil (evitar) |
|---|---|---|
| Código (Java, Rust, Python…) | función, método, clase, declaración de variable | comentario inline |
| Markdown | heading h1–h4, bloque de código, ítem de lista | párrafo libre sin heading |
| YAML / TOML | clave de mapping | valor string libre |
| JSON | clave de objeto | valor primitivo |
| SQL | nombre de tabla, función, vista | expresión inline |
