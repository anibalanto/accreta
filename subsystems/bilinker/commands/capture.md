# Especificación: comando `bilinker capture`

## Propósito

Genera una referencia bilinker a partir de una selección de texto en un archivo,
identificada por coordenadas de línea y columna. La salida está lista para pegar
en un archivo `.bilink`.

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

## Flujo interno

1. Localizar `.bilinker.toml` desde el directorio de trabajo hacia arriba.
2. Resolver el workspace por nombre y obtener `path` y `language`.
3. Leer el archivo y parsearlo con la gramática tree-sitter del lenguaje.
4. Encontrar el nodo AST más pequeño que contiene la selección completa
   (`named_descendant_for_point_range`).
5. Subir en el árbol AST desde ese nodo hasta el primer ancestro que sea un
   âncora estable para el lenguaje (función, método, clase, heading, campo de
   mapping, etc.).
6. Construir la query como el camino del AST desde ese ancestro hasta el nodo
   target, usando los nombres de campo reales y los tipos de nodo reales del
   árbol. Cada predicado usa un nombre de captura único (`@n0`, `@n1`, …).
   El `@target` se coloca después del paréntesis de cierre del nodo.
7. Determinar si la selección coincide exactamente con los límites del nodo target:
   - **Exacta**: no incluir `start~end`.
   - **Parcial**: calcular offsets en bytes relativos al inicio del nodo target
     e incluir `start~end`.
8. Calcular el hash SHA-256 del texto exacto del fragmento seleccionado.

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

El hash va a stderr para que pueda usarse `capture` en pipes sin contaminar la
referencia en stdout.

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
