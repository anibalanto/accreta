# Especificación: comando `bilinker capture`

## Propósito

Genera una referencia bilinker a partir de una selección de texto en un archivo, identificada por coordenadas de línea y columna. La salida está lista para pegar en un archivo `.bilink`.

## Firma

```
bilinker capture <workspace> <file> <start_line>:<start_col> <end_line>:<end_col>
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `workspace` | string | Nombre del workspace tal como está definido en `.bilinker.toml`. |
| `file` | path | Ruta del archivo relativa a la raíz del workspace. |
| `start_line:start_col` | int:int | Línea y columna de inicio de la selección (1-based). |
| `end_line:end_col` | int:int | Línea y columna de fin de la selección (1-based). |

## Lenguajes soportados

| Extensión | Lenguaje | Âncoras estables |
|-----------|----------|-----------------|
| `.java` | Java | `class_declaration`, `method_declaration`, `interface_declaration`, `enum_declaration` |
| `.rs` | Rust | `function_item`, `struct_item`, `enum_item`, `trait_item`, `impl_item` |
| `.yaml`, `.yml` | YAML | `block_sequence_item` (usa `id:` como predicado), `block_mapping_pair` (usa clave) |
| `.md` | Markdown | `section` (usa texto del heading como predicado, captura el contenido completo) |
| `.ts`, `.js` | TypeScript | `function_declaration`, `class_declaration`, `abstract_class_declaration`, `enum_declaration`, `interface_declaration`, `type_alias_declaration`, `method_definition`, `method_signature` |
| `.tsx`, `.jsx` | TSX | igual que TypeScript, con parser TSX para archivos con JSX |

## Flujo interno

1. Leer el archivo y parsearlo con la gramática tree-sitter del lenguaje detectado por extensión.
2. Encontrar el nodo AST más pequeño que contiene la selección completa
   (`named_descendant_for_point_range`).
3. Subir en el árbol AST hasta el primer ancestro que sea un âncora estable para el lenguaje.
4. Casos especiales por lenguaje:
   - **YAML `block_sequence_item`**: busca el par `id:` dentro del item y usa su valor como predicado, capturando el item completo.
   - **YAML `block_mapping_pair`**: usa el texto de la clave como predicado.
   - **Markdown `section`**: busca el heading dentro del section y usa su texto inline como predicado, capturando toda la sección (heading + contenido).
5. Construir la query como el camino del AST desde ese ancestro hasta el nodo
   target. Cada predicado usa un nombre de captura único (`@n0`, `@n1`, …). El `@target` se coloca en el nodo que representa el fragmento completo.
6. Determinar si la selección coincide exactamente con los límites del nodo target:
   - **Exacta**: no incluir `start~end`.
   - **Parcial**: calcular offsets en bytes relativos al inicio del nodo target e incluir `start~end`.
7. Calcular el hash SHA-256 del texto exacto del fragmento seleccionado.

## Salida

**stdout** — La referencia lista para copiar:

```
link.N: java-demo :: src/main/java/ar/example/demo/persona/Persona.java :: (class_declaration
  name: (identifier) @n0 (#eq? @n0 "Persona")
  body: (class_body
    (method_declaration
      name: (identifier) @n1 (#eq? @n1 "vote")) @target))
```

**stderr** — Metadata informativa:

```
hash: 479922a1ee55cc7f9f4f323bb002018e1b4e1cda65e069e0f6f4645926ce25ee
```

El hash va a stderr para que pueda usarse `capture` en pipes sin contaminar la referencia en stdout.

## Código de salida

| Código | Condición |
|---|---|
| 0 | Captura exitosa. |
| 1 | Error: workspace no encontrado, archivo no existe, selección fuera de rango, lenguaje sin gramática. |

## Ejemplos

### Selección que coincide exactamente con un método Java

```bash
$ bilinker capture java-demo \
    src/main/java/ar/example/demo/persona/Persona.java \
    10:5 12:5

link.N: java-demo :: src/main/java/ar/example/demo/persona/Persona.java :: (class_declaration
  name: (identifier) @n0 (#eq? @n0 "Persona")
  body: (class_body
    (method_declaration
      name: (identifier) @n1 (#eq? @n1 "vote")) @target))
```
```
# stderr:
hash: 479922a1ee55cc7f9f4f323bb002018e1b4e1cda65e069e0f6f4645926ce25ee
```

### Selección parcial dentro de un párrafo Markdown

```bash
$ bilinker capture docs architecture.md 34:10 34:52

link.N: docs :: architecture.md :: (section
  (atx_heading
    (inline) @n0 (#eq? @n0 "Decisión"))
  (paragraph) @target) :: 42~87
```
```
# stderr:
hash: c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4
```

## Propiedades garantizadas

- **Determinismo**: dos ejecuciones sobre el mismo archivo y selección sin
  modificaciones intermedias producen la misma referencia y el mismo hash.
- **Independencia de git**: `capture` no requiere que el archivo esté bajo
  control de versiones.
- **Sin efectos secundarios**: `capture` no escribe ningún archivo; solo imprime.
