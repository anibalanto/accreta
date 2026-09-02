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

Usa `.bilink/index/index` si está disponible y actualizado (O(1)); si no, escanea los bilinks de la capa actual (O(N)).

**Salida:**

```
$ bilinker get src/main/java/ar/example/demo/persona/Persona.java:11:5

7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.1   specs :: voting.yaml#impl
3a4b5c6d-2e3f-4a5b-9c6d-7e8f9a0b1c2d.1   specs :: persona/voting.yaml#impl
```

Si no hay bilinks que cubran esa posición, retorna lista vacía (código 0).

## Forma 2: endpoint → contenido del fragmento referenciado

```
bilinker get <UUID>.<N> [-B <rows>] [-A <rows>] [--diff] [--raw]
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `<UUID>.<N>` | string | Identificador del endpoint: UUID de la cadena + índice (0 o 1). |
| `-B rows` | int | Líneas de contexto antes del fragmento. |
| `-A rows` | int | Líneas de contexto después del fragmento. |
| `--diff` | flag | Muestra el diff entre el fragmento aceptado y el fragmento actual. |
| `--raw` | flag | El texto del fragmento y nada más: sin números de línea y sin huecos. |

Resuelve el endpoint N del bilink `<uuid>.yaml` de la capa actual y retorna el texto del fragmento que referencia.

Si el endpoint es de tipo `path`, resuelve el path Stratum hacia la capa adyacente, localiza el mismo UUID en su `.bilink/`, y retorna el fragmento del endpoint estructural que contiene. Requiere que los archivos de la capa adyacente estén accesibles localmente.

**stdout** — El fragmento, **una línea por línea del archivo, con su número**.

**stderr** — Metadata:

```
# specs :: persona/voting.yaml  lines 14–16
```

**Con [alias](chain.md#el-alias-el-verbo-y-la-ruta-compuestos-del-fragmento), el encabezado lo lleva**: es lo que contesta *qué* se está mirando, y el archivo y las líneas contestan dónde.

```
# GET /public-api/user/info/from-token
# hsi :: …/UserPublicController.java  lines 22–22, 36–37
```

**Un fragmento de varios `@target` se muestra por partes**, y la metadata lleva un tramo por parte:

```
# impl :: PermissionController.java  lines 12–12, 40–43
```

**Salida:**

```
$ bilinker get 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.1

14:   impl: Persona#vote
15:   description: El método vote registra el voto del ciudadano.
```

```
$ bilinker get 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.1 -B 2 -A 2

12:     id: persona-voting
13:     name: Voting
14:   impl: Persona#vote
15:   description: El método vote registra el voto del ciudadano.
16:
17:   tests:
```

### Un hueco es un hueco, y se marca en las dos escalas

> `⋮` para las líneas que no entran. `...` para lo que no entra **adentro** de una línea.

```
$ bilinker get 67ba7217.0

22:   @RequestMapping("/public-api/user")
 ⋮
36:   	@GetMapping(value = "/info/from-token")
37:   	... PublicUserInfoDto ... (@RequestHeader("user-token") String userToken) ...
```

**Los `...` son el límite entre partes, y por eso no hace falta marcarlo aparte.** En la línea 37 dicen tres cosas de una: que `public` no entra, que el nombre del método no entra —la decisión de [`32`](../../../.stratum/worklist-accreta/32.task.md), ahora visible— y que el ` {` tampoco. Dónde termina una parte y arranca la otra se ve porque lo que hay en el medio no está.

**Y hace falta porque el texto pelado mentía.** Todo capture de `spring-controller` tiene cuatro `@target`, y el tipo de retorno y los parámetros **comparten línea** siempre que la firma quepa en una. Concatenados, esa línea salía dos veces y se leía como una duplicación:

```
	public PublicUserInfoDto fetchUserInfoFromToken(@RequestHeader("user-token") String userToken) {
	public PublicUserInfoDto fetchUserInfoFromToken(@RequestHeader("user-token") String userToken) {
```

No estaba capturada dos veces: eran dos partes en la misma línea. Con la vista, una línea del archivo sale **una vez**, aunque la toquen varias partes.

La sangría se conserva tal cual: es espacio en blanco, no aporta contenido, y sin ella el código no se lee.

### `--raw` es el texto, y no es el default

**Hasta acá el stdout era el fragmento exacto**, unido por el mismo separador que lo une en el `hash` — o sea que lo que se imprimía *era* el fragmento, el mismo que `check` compara. Eso pasa a `--raw`.

El argumento no es de gustos. Si alguien quiere el texto exacto es **para compararlo**, y comparar lo hacen `check` y `--diff`, no una persona en una terminal. Un default que sirve para pipear y no para leer optimiza el uso raro.

**Y `get` es el comando de después.** La [vista previa](chain.md#la-salida-deja-ver-qué-se-capturó-y-qué-no) existe porque *"un capture es opaco después de escrito"*; `get` se usa justo cuando esa opacidad ya se cobró, y mostraba **menos** que la vista de antes de escribir. Ver [`concepts/capture.md`](../concepts/capture.md) § "El fragmento son los `@target`" para qué es una parte.

### Flag `--diff`

Requiere el `commit` del endpoint —el commit en que el contenido aceptado quedó establecido— y el capture resuelto. Vive en [la cache](../concepts/cache.md); con cache fría se re-deriva.

- **"antes"**: el fragmento aceptado, recuperado resolviendo la query contra el contenido del `commit` del endpoint y verificado contra `accepted.hash`. Ver [`check`](check.md) § "Recuperar el texto aceptado".
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

### Cruzando la frontera, el clon se profundiza a pedido

Un endpoint [repo](../concepts/frontier.md#el-endpoint-repo) apunta a un fragmento de otro proyecto, y su clon **arranca superficial**: el árbol actual de la rama declarada, sin historia. Alcanza para [`check`](check.md), que no puede andar profundizando clones como efecto colateral.

`get --diff` es el que sí necesita historia, y la trae **sólo para ese bilink**:

```
1. Recorrer el .bilink remoto hacia atrás por la ref del proveedor,
   hasta la versión cuyo `accepted` coincide con lo que este repo aceptó.
2. Leer el capture de entonces y el archivo de entonces.
3. Comparar contra lo que el proveedor publica ahora.
```

El paso 1 es lo que hace que **ningún commit del proveedor se copie**: se descubre, no se guarda. Y el fetch es incremental —`--deepen`, o los blobs de un clon parcial— así que el costo se paga únicamente donde hay un humano mirando.

**Ése es el reparto: `check` es masivo y barato; ver el diff es puntual y caro.** El conocimiento mínimo queda como default, no como límite.

Si el clon no está, `--diff` no lo crea: dice que falta y sale, igual que `check` reporta `REMOTE_UNREACHABLE`. Traer un repo ajeno por primera vez es un acto explícito.

`--diff` opera solo sobre el endpoint pedido. Para ver además el diff de todo lo que ese fragmento llama, el traversal del call graph vive en [lattice](../../lattice/overview.md) — bilinker no consulta language servers.

## Forma 3: archivo → todos los endpoints que lo referencian

```
bilinker get <file>
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `file` | path | Path al archivo (absoluto o relativo al directorio de trabajo). |

Retorna todos los endpoints de bilinks que referencian ese archivo, ya sea mediante un capture con query AST, un capture de archivo completo, o cualquier posición dentro del archivo.

Usa `.bilink/index/index` si está disponible y actualizado (O(1)); si no, escanea los bilinks de la capa actual (O(N)). En ambos casos, no re-ejecuta queries tree-sitter — el `range` cacheado de cada capture es suficiente.

**Salida:**

```
$ bilinker get src/main/java/ar/example/demo/persona/Persona.java

7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.1   specs :: voting.yaml#impl          lines 11–18
3a4b5c6d-2e3f-4a5b-9c6d-7e8f9a0b1c2d.1   specs :: persona/voting.yaml#impl  lines 11–18
```

Si no hay bilinks que referencien ese archivo, retorna lista vacía (código 0).

## Cuando el capture no resuelve, igual dice a dónde apuntaba

Un endpoint que no resuelve no tiene contenido que mostrar, y hasta acá `get` fallaba nombrando el archivo y nada más:

```
$ bilinker get be2e3fd6.1
Error: query matched nothing in crates/bilinker/src/apply.rs
```

**Falta justamente el dato que hace falta para arreglarlo.** El estado ya dijo que el fragmento no está; lo que no se sabe es **cuál era**, para decidir a dónde repuntarlo. Y `UNRESOLVED` es el estado que *obliga* a intervenir a mano: no tiene fix automático y no se resuelve aceptando.

Así que la referencia se imprime igual, y el error va después:

```
$ bilinker get be2e3fd6.1
# crates/bilinker/src/apply.rs
# capture 1a2b3c4d…  (UNANCHORED)
query: (function_item name: (identifier) @n0 (#eq? @n0 "compute_fix")) @target

Error: el anchor `compute_fix` no está en el archivo.
  Repuntar con `bilinker recapture be2e3fd6.1 <archivo> <línea>:<col>`.
```

**Sigue fallando** —el código de salida es 1— porque no hay fragmento que devolver. Lo que cambia es que la salida ya no obliga a abrir el `.yaml` del capture para leer la query, que es exactamente lo que el formato evita pedirle a nadie: ningún archivo de bilinker se escribe ni se lee a mano, y además el capture se referencia por un id de 32 hex que hay que sacar del bilink primero.

Es la misma regla que gobierna [`apply`](apply.md#cuando-el-fix-no-se-puede-calcular) al no poder calcular un fix: **la salida es fiel a lo que la herramienta sabe.** El capture está ahí, se leyó para intentar resolverlo, y lo que se imprime lo incluye.

### La causa se re-deriva, no se supone

*"La query no matcheó"* es cierto y no sirve: es la observación, no la causa. Así que el estado del capture se vuelve a resolver, y de ahí sale qué comando corresponde:

| Estado | Qué pasó | Qué hay que hacer |
|---|---|---|
| `MOVED` | el archivo se movió | `bilinker apply` |
| `REANCHORED` | el anchor se renombró y se localizó por similitud | `bilinker apply` |
| sin fix, y el anchor aparece en un archivo **sin trackear** | git no puede ver el rename | `git add` ese archivo |
| sin fix, y el anchor no aparece en ninguna parte | el fragmento ya no existe | `recapture` o `remove` |

La tercera fila es la que [`apply`](apply.md#y-las-otras-dos-causas-no-son-de-apply) no puede explicar: sin rename detectado el capture queda en un estado sin fix, y `apply` no lo mira. Se decide buscando el anchor entre los archivos sin trackear — un hecho, no una sugerencia genérica.

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
- **Un endpoint que no resuelve imprime su referencia igual**: archivo, id del capture, y query. Falla después.
- **`--raw` imprime el fragmento y nada más**: el mismo texto que `check` hashea, byte por byte. Es la única salida de la que eso se puede afirmar, y por eso el flag existe.
- **Una línea del archivo sale una sola vez**, aunque la toquen varias partes.
