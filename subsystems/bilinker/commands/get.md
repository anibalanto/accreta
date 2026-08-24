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

Retorna la lista de endpoints de bilinks cuyo capture tiene un `range` que cubre la posición dada. Cada resultado se identifica como `<UUID>.<N>` y muestra una descripción del fragmento referenciado en el extremo opuesto.

Usa `.bilink/index/index` si está disponible y actualizado (O(1)); si no, escanea los `.bilink` de la layer actual (O(N)).

**Salida:**

```
$ bilinker get src/main/java/ar/example/demo/persona/Persona.java:11:5

7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.1   specs :: voting.yaml#impl
3a4b5c6d-2e3f-4a5b-9c6d-7e8f9a0b1c2d.1   specs :: persona/voting.yaml#impl
```

Si no hay bilinks que cubran esa posición, retorna lista vacía (código 0).

## Forma 2: endpoint → contenido del fragmento referenciado

```
bilinker get <UUID>.<N> [-B <rows>] [-A <rows>] [--diff]
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `<UUID>.<N>` | string | Identificador del endpoint: UUID de la cadena + índice (0 o 1). |
| `-B rows` | int | Líneas de contexto antes del fragmento. |
| `-A rows` | int | Líneas de contexto después del fragmento. |
| `--diff` | flag | Muestra el diff entre el fragmento aceptado y el fragmento actual. |

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

### Flag `--diff`

Requiere `commit.N` presente en el bilink (endpoint aceptado al menos una vez) y el capture resuelto.

- **"antes"**: el fragmento aceptado, recuperado resolviendo la query contra el contenido de `commit.N` y verificado contra `hash.N`. Ver [`check`](check.md) § "Recuperar el texto aceptado".
- **"después"**: resuelve el fragmento actual con la misma query AST que usa `get` normalmente.
- Muestra un unified diff del fragmento, sin contexto extra de archivo.

No se recorta el contenido viejo por el `range` del capture: ese range es la posición **actual**, que `check` reescribe en cada corrida, así que aplicarlo a contenido de otro commit da bytes arbitrarios.

Si la verificación por hash falla —el archivo no existía en ese commit, la query no resuelve ahí— `--diff` cae al recorte por range. Para un diff informativo mostrar algo aproximado es mejor que no mostrar nada; el contraste es con `check`, que ante la misma duda prefiere no detectar, porque de ahí salen decisiones.

```
$ bilinker get 7f3d8e9a.1 --diff

# java-demo :: src/main/java/ar/example/demo/persona/Persona.java  lines 10–12
--- aceptado (commit a3f2b1c)
+++ actual
@@ -1,3 +1,4 @@
 public void vote(String candidato) {
-    repo.save(new Vote(candidato));
+    repo.save(new Vote(candidato, true));
+    audit.log(candidato);
 }
```

Si el fragmento no cambió (estado OK), muestra el contenido sin diff.

`--diff` opera solo sobre el endpoint pedido. Para ver además el diff de todo lo que ese fragmento llama, el traversal del call graph vive en [lattice](../../lattice/overview.md) — bilinker no consulta language servers.

## Forma 3: archivo → todos los endpoints que lo referencian

```
bilinker get <file>
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `file` | path | Path al archivo (absoluto o relativo al directorio de trabajo). |

Retorna todos los endpoints de bilinks que referencian ese archivo, ya sea mediante un capture con query AST, un capture de archivo completo, o cualquier posición dentro del archivo.

Usa `.bilink/index/index` si está disponible y actualizado (O(1)); si no, escanea los `.bilink` de la layer actual (O(N)). En ambos casos, no re-ejecuta queries tree-sitter — el `range` cacheado de cada capture es suficiente.

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
| 1 | Error: archivo no encontrado, UUID inválido, endpoint sin capture, capture sin resolver, capa adyacente no accesible. |

## Propiedades garantizadas

- **Independencia de git**: `get` sin `--diff` no requiere control de versiones.
- **Sin efectos secundarios**: `get` no escribe ningún archivo.
