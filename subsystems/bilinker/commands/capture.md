# Especificación: comando `bilinker capture`

## Propósito

Crea un [capture](../concepts/capture.md) a partir de una selección de texto en un archivo, identificada por coordenadas de línea y columna. Escribe `.bilink/capture/<id>.yaml` y devuelve su id, listo para referenciar desde un `link`.

## Firma

```
bilinker capture <file> [<start_line>:<start_col> <end_line>:<end_col>] [--dry-run]
bilinker capture prune [<path>]
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `file` | path | Ruta del archivo, relativa a la raíz de la capa actual. |
| `start_line:start_col` | int:int | Línea y columna de inicio de la selección (1-based). Omitir para capturar el archivo entero. |
| `end_line:end_col` | int:int | Línea y columna de fin de la selección (1-based). |
| `--dry-run` | flag | Imprime el capture que se crearía sin escribir nada. |

**Sin selección, el capture es el archivo entero** y sale sin `query`, que es lo que [`concepts/capture.md`](../concepts/capture.md) § "Formato" define como el archivo completo. No hay nodo AST que encontrar, así que tampoco hay ancla que verificar ni lenguaje que soportar: un archivo entero se captura aunque no haya gramática para él.

Es la forma más usada del lado de las specs, donde el fragmento suele ser el documento.

## Lenguajes soportados

| Extensión | Lenguaje | Anclas estables |
|-----------|----------|-----------------|
| `.java` | Java | `class_declaration`, `interface_declaration`, `enum_declaration`, `method_declaration`, `constructor_declaration`, `field_declaration` |
| `.rs` | Rust | `function_item`, `struct_item`, `enum_item`, `trait_item`, `impl_item`, `const_item`, `static_item` |
| `.yaml`, `.yml` | YAML | `block_sequence_item` (usa `id:` como predicado), `block_mapping_pair` (usa clave) |
| `.md` | Markdown | `section` (usa texto del heading como predicado, captura el contenido completo) |
| `.ts`, `.js` | TypeScript | `class_declaration`, `abstract_class_declaration`, `function_declaration`, `generator_function_declaration`, `enum_declaration`, `interface_declaration`, `type_alias_declaration`, `method_definition`, `method_signature` |
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
   - **Rust `impl_item`**: el discriminante no es un campo `name` sino el tipo implementado (`type:`) y, cuando es la implementación de un trait, también el trait (`trait:`). Con uno solo, `impl Foo` y `impl Bar for Foo` quedan indistinguibles.
5. Construir la query como el camino del AST desde ese ancestro hasta el nodo target. Cada predicado usa un nombre de captura único (`@n0`, `@n1`, …). El `@target` se coloca en el nodo que representa el fragmento completo.
6. **Verificar que la query identifica el fragmento**: resolverla contra el mismo archivo y comprobar que devuelve exactamente un match, en los bytes del nodo seleccionado. Si devuelve otro nodo o más de uno, `capture` falla sin escribir nada.
7. Determinar si la selección coincide exactamente con los límites del nodo target:
   - **Exacta**: no escribir `offset`.
   - **Parcial**: calcular offsets en bytes relativos al inicio del nodo target y escribirlos en `offset`.
8. Calcular `H(file, query, offset)` y escribir `.bilink/capture/<id>.yaml` si no existe. Nada de cache: ni `range`, ni `state`, ni timestamp.

`capture` no calcula ni almacena hashes: un capture describe ubicación, no contenido aceptado. El hash lo establece `bilinker accept` en el bilink que lo referencie.

## Salida

**stdout** — el id del capture, para referenciar desde un `link`:

```
67ba7217e0334051becd4921b55a7872
```

**stderr** — metadata informativa:

```
created: .bilink/capture/67ba7217e0334051becd4921b55a7872.yaml
file:    src/main/java/ar/example/demo/persona/Persona.java
anchor:  class_declaration "Persona" → method_declaration "vote"
```

Si el capture ya existía, `stderr` dice `reused:` en vez de `created:` y no se escribe nada. Es el caso normal de capturar dos veces la misma ubicación, y no es una condición especial: el id sale del contenido, así que **el mismo fragmento produce el mismo archivo**.

El id va solo a stdout para poder usarlo en pipes:

```bash
id=$(bilinker capture src/lib.rs 10:1 24:2)
```

## Ejemplo

```bash
$ bilinker capture src/main/java/ar/example/demo/persona/Persona.java 10:5 12:5
67ba7217e0334051becd4921b55a7872
```

```yaml
# .bilink/capture/67ba7217e0334051becd4921b55a7872.yaml
file: src/main/java/ar/example/demo/persona/Persona.java
query: |-
  (class_declaration
    name: (identifier) @n0 (#eq? @n0 "Persona")
    body: (class_body
      (method_declaration
        name: (identifier) @n1 (#eq? @n1 "vote")) @target))
```

Tres campos posibles y ninguno más: son los que entran en el id. `range`, `state` y `resolved_at` no se escriben — el primero es derivado y vive en [la cache](../concepts/cache.md), y los otros dos no existen.

### Selección parcial

Cuando la selección no coincide con los límites del nodo, se escribe `offset` relativo al inicio del nodo matcheado:

```bash
$ bilinker capture architecture.md 34:10 34:52
3a4b5c6d2e3f4a5b9c6d7e8f9a0b1c2d
```

```yaml
file: architecture.md
query: |-
  (section
    (atx_heading (inline) @n0 (#eq? @n0 "Decisión"))
    (paragraph) @target)
offset: 42~87
```

## `bilinker capture prune`

Elimina los captures de la capa que no alcanza ningún bilink.

**Es mark & sweep sobre dos clases de raíz.** Un capture está vivo si lo referencia un `link` —la ubicación vigente de un endpoint— **o** un `accepted.link` —la ubicación que alguien aprobó. Barrer sólo por la primera borraría el capture que dice dónde estaba lo aceptado, y con él la capacidad de decidir si una ubicación cambió.

```
$ bilinker capture prune

3 captures sin referentes:
  9f8e7d6c…  crates/bilinker/src/sciplink.rs
  1a2b3c4d…  crates/bilinker/src/scip_index.rs
  5e6f7a8b…  crates/bilinker-cli/src/main.rs :: (function_item "scip_retrofit")

Eliminar? [y/N]
```

Un capture huérfano no rompe nada: nadie lo lee. `prune` es higiene, no reparación.

Y hay más huérfanos que antes: como los captures son inmutables, cada vez que `apply` corrige una ubicación acuña uno nuevo y el viejo queda —vivo mientras algún `accepted.link` lo nombre, huérfano cuando esa aceptación se reemplace.

## `bilinker capture remove`

```
bilinker capture remove <uuid> [--force]
```

Elimina un capture puntual. Acepta un prefijo del UUID, y falla si es ambiguo.

**Se niega si tiene referentes**, porque borrarlo dejaría bilinks apuntando a la nada:

```
$ bilinker capture remove 5fdff600
Error: el capture 5fdff600 tiene referentes — usar `bilinker recapture` para repuntarlos, o --force
```

`--force` lo borra igual y avisa que hay que correr `check`. `prune` es lo que corresponde para limpiar en bloque; esto es para un capture concreto.

## Código de salida

| Código | Condición |
|---|---|
| 0 | Capture creado (o `prune` completado). |
| 1 | Error: archivo no existe, selección fuera de rango, lenguaje sin gramática. |

## Propiedades garantizadas

- **Unicidad de la referencia**: la `query` resuelve al nodo que se seleccionó, y a ninguno otro. Un ancla sin discriminante —un `impl` sin tipo, un comentario, un `use`— produce una query que matchea el **primer** nodo de ese tipo del archivo: un capture que apunta a otra cosa y no falla. `capture` verifica antes de escribir y falla si no puede identificar el fragmento unívocamente. Un capture mal anclado es peor que uno roto, porque reporta OK sobre una correspondencia que no existe.
- **Determinismo de la referencia**: dos ejecuciones sobre el mismo archivo y selección sin modificaciones intermedias producen la misma `query` y el mismo `range`.
- **Reuso**: capturar dos veces el mismo fragmento devuelve el **mismo** id, sin buscar nada. El id es `H(file, query, offset)`, así que dos referencias a la misma ubicación son literalmente el mismo archivo. Antes había que escanear la capa buscando un capture equivalente; ahora la deduplicación es por construcción.
- **Independencia de git**: `capture` no requiere que el archivo esté bajo control de versiones. Sí lo requiere [`accept`](accept.md), que necesita el commit del contenido aprobado.
- **No toca bilinks**: `capture` crea el archivo del capture y nada más. Referenciarlo desde un `link` es un paso aparte, vía `bilinker chain new` o `recapture`.
