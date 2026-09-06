# Comando: `worklist new`

Crea un ítem en la vista donde se está trabajando, y opcionalmente un bilink abierto al fragmento que lo origina.

> **No necesita conectividad.** Crear un ítem es escribir un archivo: el id nace del lado local —`@<slug>`— y el servidor recién interviene al sincronizar. Ver [`item.md`](../concepts/item.md) § "Un ítem tiene un solo id a la vez".

Esto es lo que cambió: durante meses el comando estuvo especificado y sin implementar porque *"el servidor asigna el próximo ID base-36"*, y ese servidor no existía. **Sin contador que pedir no hay nada que delegar.**

## Uso

```
worklist new <tipo> "<título>" [capture <selector>] [--under <id>] [--as <slug>]
```

Donde `<tipo>` es `epic`, `user-story` o `task`.

| | |
|---|---|
| `capture <selector>` | captura un fragmento como origen del trabajo y crea un bilink abierto contra el ítem |
| `--under <id>` | el `parent`, con la marca si el padre tampoco cruzó todavía |
| `--as <slug>` | el slug, cuando el derivado del título no es el que se quiere |

El selector se resuelve desde el directorio actual en la terminal:

| Forma del selector | Descripción |
|--------------------|-------------|
| `archivo:línea:col` | Posición exacta en un archivo |
| `archivo` | Archivo completo |
| `<uuid-bilink>` | Bilink existente |

Sin `capture`, el ítem se crea sin bilink — válido para épicas de alto nivel.

## El slug sale del título, y se puede dar

> **Por defecto, el slug es el título slugificado**: minúsculas, sin acentos, y todo lo que no sea del alfabeto de un id colapsado a un guión medio.

```
"Arreglar el hook que no arranca"   →   @arreglar-el-hook-que-no-arranca.task.md
```

**Derivarlo del título es lo que evita nombrar dos veces.** El título ya se escribió, y además es lo que el proveedor usa para reconocer un pedido; pedir un segundo nombre sería pedir la misma decisión otra vez. `--as` existe para cuando el título es largo o el slug conviene más corto, no como el camino normal.

**No hay tope de largo.** Un nombre recortado deja de contestar la pregunta para la que existe, y el alfabeto no impone ninguno — ver [`item.md`](../concepts/item.md) § "El alfabeto de un id".

**Y una colisión es el filesystem diciendo algo.** Si `@arreglar-el-hook.task.md` ya existe, el comando falla en vez de desambiguar solo: dos ítems que se nombran igual el mismo día probablemente sean el mismo ítem, y eso es información. La unicidad no hace falta más allá de la vista — el `@<slug>` nunca llega al panorama.

## Comportamiento

1. Deriva el slug del título, o toma el de `--as`, y verifica que el archivo no exista.
2. Si se especifica `capture <selector>`, ejecuta `bilinker capture` sobre el fragmento y crea un bilink con el ítem en un endpoint `issue @<slug>`.
3. Escribe `@<slug>.<tipo>.md` en la raíz de la vista actual, con `title`, `status: open`, `created_at`, `updated_at` y el `parent` si vino `--under`.
4. Imprime el id y el path.

**No commitea.** El ítem entra al repo con el resto del trabajo, que es lo que hace que un cambio y la tarea que lo ejecuta viajen juntos.

**Y se escribe en la vista, no en el panorama.** El panorama es una rama insegura: lo que se escriba ahí no se verifica y no cruza al proveedor. Ver [`sync.md`](../concepts/sync.md) § "Dos clases de rama, y el nombre dice qué se puede hacer".

## Ejemplos

```bash
# Task con capture: marca trabajo pendiente en la spec
cd accreta/subsystems/bilinker
worklist new task "Implementar bilinker accept" capture concepts/bilink.md:104:1

# Task con capture desde impl: la spec necesita actualización
cd accreta/subsystems/bilinker/.stratum/impl/src
worklist new task "Actualizar la spec de hash.N" capture lib.rs:88:1

# Épica sin capture: objetivo de alto nivel sin fragmento de origen
worklist new epic "Implementar hash.N"

# User story hija de una épica que ya cruzó
worklist new user-story "Parsear hash.N en BiLinkFile" --under ACC-14

# Task hija de una user story que todavía no cruzó
worklist new task "Actualizar el struct BiLinkFile" --under @parsear-hash-n-en-bilinkfile

# Con el slug elegido a mano, porque el título es largo
worklist new task "El vecindario de una firma resuelve a la firma misma" --as @vecindario-de-firma
```

## Salida

```
created:  @implementar-bilinker-accept
type:     task
path:     .worklist/secure/sprint/20/@implementar-bilinker-accept.task.md
bilink:   a3f9c821-4e5b-4c3d-9f2a-1b2c3d4e5f6a  (concepts/bilink.md:104:1 → issue @implementar-bilinker-accept)
```

## Y el bilink se entera cuando el ítem cruza

Un endpoint `issue @<slug>` apunta a un ítem que todavía no tiene su id definitivo. En el primer push el servidor lo renombra, el endpoint se repunta —`issue @arreglar-el-hook` → `issue ACC-347`— y queda `RELOCATED`, pidiendo un `accept --place`.

**No es una vez: es cada vez que un ítem nuevo cruza.** Hoy no duele porque no hay endpoints `issue` en el proyecto, pero [`bilink-tasks.md`](../concepts/bilink-tasks.md) existe para que sean muchos — así que conviene decidir **antes** de crearlos si ese churn se acepta, se automatiza o se evita.

## Exit codes

| Código | Condición |
|--------|-----------|
| `0` | ítem creado |
| `1` | selector inválido, id padre no encontrado, archivo sin historial git, o el slug ya existe |
| `2` | error de escritura |
