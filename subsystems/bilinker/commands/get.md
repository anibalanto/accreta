# Especificación: comando `bilinker get`

## Propósito

Permite navegar desde una posición en un archivo hacia los fragmentos relacionados a través de bilinks, y recuperar su contenido. Opera en dos formas.

## Forma 1: posición → endpoints que la cubren

```
bilinker get <file>:<line>:<col>
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `file` | path | Path al archivo (absoluto o relativo al directorio de trabajo). |
| `line` | int | Línea (1-based). |
| `col` | int | Columna (1-based). |

Retorna la lista de endpoints de bilinks cuyo `range.N` cubre la posición dada. Cada resultado se identifica como `<UUID>.<N>` y muestra una descripción del fragmento referenciado en el extremo opuesto.

Usa `.bilink/.index` si está disponible y actualizado (O(1)); si no, escanea los `.bilink` de la layer actual (O(N)).

**Salida:**

```
$ bilinker get src/main/java/ar/example/demo/persona/Persona.java:11:5

7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.1   specs :: voting.yaml#impl
3a4b5c6d-2e3f-4a5b-9c6d-7e8f9a0b1c2d.1   specs :: persona/voting.yaml#impl
```

Si no hay bilinks que cubran esa posición, retorna lista vacía (código 0).

## Forma 2: endpoint → contenido del fragmento referenciado

```
bilinker get <UUID>.<N> [-B <rows>] [-A <rows>]
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `<UUID>.<N>` | string | Identificador del endpoint: UUID de la cadena + índice (0 o 1). |
| `-B rows` | int | Líneas de contexto antes del fragmento. |
| `-A rows` | int | Líneas de contexto después del fragmento. |

Resuelve el endpoint `link.N` del bilink `<uuid>.bilink` de la layer actual y retorna el texto del fragmento que referencia.

Si `link.N` es un endpoint layer, resuelve el path Stratum hacia la capa adyacente, localiza el mismo UUID en su `.bilink/`, y retorna el fragmento del endpoint estructural que contiene. Requiere que los archivos de la capa adyacente estén accesibles localmente.

**stdout** — El texto del fragmento.

**stderr** — Metadata:

```
# specs :: persona/voting.yaml  lines 14–16
```

**Salida:**

```
$ bilinker get 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.1

impl: Persona#vote
description: El método vote registra el voto del ciudadano.
```

```
$ bilinker get 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.1 -B 2 -A 2

  id: persona-voting
  name: Voting

impl: Persona#vote
description: El método vote registra el voto del ciudadano.

tests:
```

## Forma 3: archivo → todos los endpoints que lo referencian

```
bilinker get <file>
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `file` | path | Path al archivo (absoluto o relativo al directorio de trabajo). |

Retorna todos los endpoints de bilinks que referencian ese archivo, ya sea mediante un endpoint estructural con query AST, un endpoint de archivo completo (`workspace :: file`), o cualquier posición dentro del archivo.

Usa `.bilink/.index` si está disponible y actualizado (O(1)); si no, escanea los `.bilink` de la layer actual (O(N)). En ambos casos, no re-ejecuta queries tree-sitter — el `range.N` de cada bilink encontrado es suficiente.

**Salida:**

```
$ bilinker get src/main/java/ar/example/demo/persona/Persona.java

7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.1   specs :: voting.yaml#impl          lines 11–18
3a4b5c6d-2e3f-4a5b-9c6d-7e8f9a0b1c2d.1   specs :: persona/voting.yaml#impl  lines 11–18
```

Si no hay bilinks que referencien ese archivo, retorna lista vacía (código 0).

## Flujo típico

```bash
# 1. Cursor en Persona.java:11 → ¿qué bilinks lo referencian?
bilinker get Persona.java:11:5
# → 7f3d8e9a.1  specs :: voting.yaml#impl

# 2. Ver el contenido del otro extremo
bilinker get 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.1
# → texto del campo impl en voting.yaml
```

## Código de salida

| Código | Condición |
|---|---|
| 0 | Operación exitosa (puede haber 0 resultados en forma 1). |
| 1 | Error: archivo no encontrado, UUID inválido, endpoint sin fragmento estructural, query sin match, capa adyacente no accesible. |

## Propiedades garantizadas

- **Independencia de git**: `get` no requiere control de versiones.
- **Sin efectos secundarios**: `get` no escribe ningún archivo.
