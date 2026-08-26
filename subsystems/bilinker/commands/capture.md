# Especificación: comando `bilinker capture`

## Propósito

Crea un [capture](../concepts/capture.md) a partir de una selección de texto en un archivo, identificada por coordenadas de línea y columna. Escribe `.bilink/capture/<uuid>.capture` y devuelve su UUID, listo para referenciar desde un `link.N`.

## Firma

```
bilinker capture <file> <start_line>:<start_col> <end_line>:<end_col> [--dry-run]
bilinker capture prune [<path>]
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `file` | path | Ruta del archivo, relativa a la raíz de la capa actual. |
| `start_line:start_col` | int:int | Línea y columna de inicio de la selección (1-based). |
| `end_line:end_col` | int:int | Línea y columna de fin de la selección (1-based). |
| `--dry-run` | flag | Imprime el capture que se crearía sin escribir nada. |

## Lenguajes soportados

| Extensión | Lenguaje | Anclas estables |
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
3. Subir en el árbol AST hasta el primer ancestro que sea un ancla estable para el lenguaje.
4. Casos especiales por lenguaje:
   - **YAML `block_sequence_item`**: busca el par `id:` dentro del item y usa su valor como predicado, capturando el item completo.
   - **YAML `block_mapping_pair`**: usa el texto de la clave como predicado.
   - **Markdown `section`**: busca el heading dentro del section y usa su texto inline como predicado, capturando toda la sección (heading + contenido).
5. Construir la query como el camino del AST desde ese ancestro hasta el nodo target. Cada predicado usa un nombre de captura único (`@n0`, `@n1`, …). El `@target` se coloca en el nodo que representa el fragmento completo.
6. Determinar si la selección coincide exactamente con los límites del nodo target:
   - **Exacta**: no escribir `offset`.
   - **Parcial**: calcular offsets en bytes relativos al inicio del nodo target y escribirlos en `offset`.
7. Generar un UUID v4 y escribir `.bilink/capture/<uuid>.capture` con `state: RESOLVED`, `range` absoluto y `resolved_at`.

`capture` no calcula ni almacena hashes: un capture describe ubicación, no contenido aceptado. El hash lo establece `bilinker accept` en el bilink que lo referencie.

## Salida

**stdout** — el UUID del capture creado, para referenciar desde un `link.N`:

```
7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a
```

**stderr** — metadata informativa:

```
created: .bilink/capture/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.capture
file:    src/main/java/ar/example/demo/persona/Persona.java
anchor:  class_declaration "Persona" → method_declaration "vote"
range:   245~389
```

El UUID va solo a stdout para poder usarlo en pipes:

```bash
uuid=$(bilinker capture src/lib.rs 10:1 24:2)
echo "link.1: capture $uuid" >> .bilink/$chain.bilink
```

## Ejemplo

```bash
$ bilinker capture src/main/java/ar/example/demo/persona/Persona.java 10:5 12:5
7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a
```

```
# .bilink/capture/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.capture
file:   src/main/java/ar/example/demo/persona/Persona.java
query:  (class_declaration
  name: (identifier) @n0 (#eq? @n0 "Persona")
  body: (class_body
    (method_declaration
      name: (identifier) @n1 (#eq? @n1 "vote")) @target))

# Cache
range:       245~389
state:       RESOLVED
resolved_at: 2026-08-24T10:00:00Z
```

### Selección parcial

Cuando la selección no coincide con los límites del nodo, se escribe `offset` relativo al inicio del nodo matcheado:

```bash
$ bilinker capture architecture.md 34:10 34:52
3a4b5c6d-2e3f-4a5b-9c6d-7e8f9a0b1c2d
```

```
file:   architecture.md
query:  (section
  (atx_heading
    (inline) @n0 (#eq? @n0 "Decisión"))
  (paragraph) @target)
offset: 42~87
```

## `bilinker capture prune`

Elimina los captures de la capa que ningún `.bilink` referencia.

```
$ bilinker capture prune

3 captures sin referentes:
  9f8e7d6c…  crates/bilinker/src/sciplink.rs
  1a2b3c4d…  crates/bilinker/src/scip_index.rs
  5e6f7a8b…  crates/bilinker-cli/src/main.rs :: (function_item "scip_retrofit")

Eliminar? [y/N]
```

Un capture huérfano no rompe nada — se resuelve en cada `check` sin que nadie lea el resultado. `prune` es higiene, no reparación.

## Código de salida

| Código | Condición |
|---|---|
| 0 | Capture creado (o `prune` completado). |
| 1 | Error: archivo no existe, selección fuera de rango, lenguaje sin gramática. |

## Propiedades garantizadas

- **Determinismo de la referencia**: dos ejecuciones sobre el mismo archivo y selección sin modificaciones intermedias producen la misma `query` y el mismo `range`.
- **Reuso**: capturar dos veces el mismo fragmento devuelve el **mismo** UUID. Antes de crear uno nuevo, `capture` busca un capture de la capa con la referencia exacta `(file, query, offset)` — el mismo criterio que usa la migración. Sin esto, cada cadena nueva volvería a duplicar lo que aquélla unificó.
- **Independencia de git**: `capture` no requiere que el archivo esté bajo control de versiones.
- **No toca bilinks**: `capture` crea el archivo del capture y nada más. Referenciarlo desde un `link.N` es un paso aparte, manual o vía `bilinker chain new`.
