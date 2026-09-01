# Especificación: comando `bilinker chain`

## Propósito

Gestiona las cadenas de bilinks: crear nuevas cadenas, consultar su estado completo y listar todas las cadenas del proyecto.

## Subcomandos

### `bilinker chain new`

Crea una cadena: genera un UUID y escribe un bilink en cada capa.

```
bilinker chain new --tip <STRATUM_PATH[:LINE:COL[,LINE:COL]...]> \
                   [--mid <STRATUM_PATH>]... \
                   --tip <STRATUM_PATH[:LINE:COL[,LINE:COL]...]> \
                   [--kind <valor>] [--name.0 <etiqueta>] [--name.1 <etiqueta>] \
                   [--dry-run] [--yes]
```

| Argumento | Descripción |
|---|---|
| `--tip <ref>` | Extremo de la cadena: path Stratum al archivo con **una o más** posiciones, `abstract`, o `repo <alias>`. Exactamente dos veces. |
| `--mid <layer>` | Capa intermedia. Cero o más veces. |
| `--kind <valor>` | El [`kind`](../concepts/bilink.md) del bilink. |
| `--name.N <etiqueta>` | El `name` del endpoint N. |
| `--dry-run` | Muestra qué capturaría y no escribe nada. |
| `--yes` | No pregunta. Para scripts y para CI. |

Cada `--tip` captura el fragmento —sin posición, el archivo completo— y el endpoint queda apuntando a ese capture. Los mids llevan dos endpoints `path`. Ningún `accepted` se escribe: la cadena nace en `PENDING`.

#### Un tip puede señalar varias partes

Las posiciones extra van separadas por coma, después de la primera:

```bash
bilinker chain new \
  --tip 'concepts/capture.md:66:1' \
  --tip '>impl/crates/bilinker/src/query.rs:109:1,22:1'
```

Cada posición resuelve a su nodo igual que una sola —al ancla estable más cercana—, y de todas sale **una** query con un `@target` por nodo. El fragmento es su concatenación; ver [`concepts/capture.md`](../concepts/capture.md) § "El fragmento son los `@target`".

**Las posiciones se descartan.** Es lo que el formato ya dice para una y vale igual para varias: sirven para *encontrar* los nodos, y lo que se guarda es la query. El orden en que se pasan tampoco se guarda — el fragmento va en orden de archivo.

**La query se ancla una sola vez.** El nodo raíz del patrón es el ancla estable que contiene a todas las partes, así que las partes quedan ancladas *entre sí*: `@RequestMapping` **de la clase que contiene** al método, y no *"el primer `@RequestMapping` del archivo"*.

Si dos posiciones caen en el mismo nodo, es un nodo: no hay parte repetida.

#### La salida deja ver qué se capturó, y qué no

Un capture es opaco después de escrito, así que una query mal generada se descubre tarde. Antes de escribir, `chain new` muestra cada tip que captura posiciones:

```
$ bilinker chain new --tip 'docs/spec.md:1:1' --tip 'src/Service.java:2:5,10:5'

. :: src/Service.java

     1   public class Service {
  ▸  2       public int uno(int a) {
  ▸  3           return a + 1;
  ▸  4       }
     5
     6       public int dos(int b) {
     ⋮
     8       }
     9
  ▸ 10       public int tres(int c) {
  ▸ 11           return c - 3;
  ▸ 12       }
    13   }

2 fragmentos · 2–4, 10–12
queda afuera: todo lo que no está marcado

¿escribir? [y/N/e(ditar)]
```

Cuatro cosas, y cada una atrapa un error distinto:

- **el archivo como encabezado**, una vez y no repetido por parte — un fragmento de cuatro partes con cuatro encabezados se lee como cuatro captures;
- **contexto alrededor**, con `⋮` donde se saltan líneas;
- **`▸` sobre lo capturado**, así lo que no entra se ve *sin marcar* — que es lo que se entiende de un vistazo;
- **una línea que dice qué quedó afuera**, porque es lo que más se malinterpreta.

El error que esto atrapa es señalar el nodo equivocado en un archivo con veinte parecidos: se ve porque la línea marcada queda lejos de donde tenía que estar.

**Con `--dry-run` se muestra lo mismo y no se escribe nada**, ni el capture ni el bilink. Con `--yes` no se pregunta.

**Sin terminal tampoco se pregunta.** Un `chain new` adentro de un script no puede quedarse esperando una tecla que nadie va a apretar; la vista se imprime igual, por stderr, y se escribe. La confirmación existe para la persona que está mirando, y `--yes` es cómo se dice eso explícitamente.

#### Y las marcas se editan

Confirmar con `y/N` obliga a volver a empezar cuando la resolución agarró mal. `e` abre la misma vista en el editor, y ahí se corrige: se saca un `▸`, se pone otro, se guarda. Es el patrón de `git rebase -i` y `git add -e`.

**Las marcas son señales, no rangos.** Cada línea marcada resuelve a su nodo, igual que una posición de la línea de comandos, así que editar el buffer es otra forma de señalar y lo que se guarda sigue siendo la query. Marcar tres líneas de una función marca la función una vez.

**Los dos tips van en un solo buffer.** Abrir un editor por tip haría corregir a ciegas el segundo, y lo que se está revisando es el vínculo, no cada punta por su cuenta. Al guardar, la vista corregida se vuelve a mostrar: la corrección también se revisa.

Un buffer que vuelve sin ninguna marca no escribe nada — es la forma de abortar, la misma que `git commit` con un mensaje vacío.

**El editor es el de git**, resuelto con `git var GIT_EDITOR`: contesta lo que git realmente usaría, respetando `$GIT_EDITOR` → `core.editor` → `$VISUAL` → `$EDITOR` → el fallback del sistema. Es el mismo criterio por el que quién acepta sale de `git var GIT_AUTHOR_IDENT` y no de `user.name`. Si git no contesta, queda el `y/N`.

#### Los dos tips de la frontera

Una cadena que cruza a otro proyecto se crea desde **cada lado por separado**, y no podría ser de otra manera: son dos repos y ninguno escribe en el otro.

**El proveedor publica** una punta abierta:

```bash
bilinker chain new \
  --tip 'src/main/java/…/UserPermissions.java:42:1' \
  --tip abstract
```

Un solo archivo, en su repo, con `link: abstract` del lado 1. Nadie más aparece — el proveedor no sabe quién va a consumirlo.

**El consumidor referencia** ese bilink por su UUID, que es el mismo:

```bash
bilinker chain new --from-repo hsi:8a3f0d21 \
  --tip 'src/permissions.ts:17:1'
```

`--from-repo <alias>:<uuid>` toma el UUID del bilink remoto en vez de generar uno nuevo, y arma el endpoint repo del otro lado. Es la única forma de `chain new` que no genera UUID: **la convención de UUID compartido es lo que hace el rendezvous**, y generar uno propio rompería el vínculo antes de crearlo.

El alias tiene que estar declarado —`.bilink/.hsi.toml`— y el clon tiene que estar. `chain new` sí puede clonar, a diferencia de `check`: es un acto explícito de una persona que está creando un vínculo.

Ver [la frontera](../concepts/frontier.md).

#### El path de un tip atraviesa directorios, no sólo capas

Un tip se escribe con [tokens Stratum](../../stratum/concepts/paths.md), y los tokens de navegación entre capas —`>name`, `<`— se mezclan con componentes de path corrientes:

```bash
bilinker chain new \
  --tip 'subsystems/bilinker/concepts/capture.md:29:1' \
  --tip 'subsystems/bilinker>impl/crates/bilinker/src/capture.rs:523:1'
```

`subsystems/bilinker>impl` es un directorio común y después una capa. **Esa forma es la de este proyecto**, donde la capa raíz contiene las specs de cinco subsistemas y cada uno tiene su impl abajo: sin ella no se puede crear una sola cadena de accreta desde la raíz.

Es lo mismo que el formato acepta desde siempre en un endpoint `path` —`path subsystems/bilinker>impl`—, así que el comando no está agregando una forma nueva sino alcanzando la que ya existe.

**`--kind` existe para no depender de una edición a mano.** `kind` y `name` son campos de declaración, y todo archivo de bilinker sale de un comando: sin el flag, la única forma de poblarlos sería abrir el YAML, que es justamente lo que el formato no pide de nadie.

**Ejemplo:**

```bash
bilinker chain new \
  --tip 'commands/check.md:63:1' \
  --tip '>impl/crates/bilinker/src/check.rs:405:1'
```

**Salida:**

```
Created chain: 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a

  .bilink/7f3d8e9a-….yaml                    (tip)
  .stratum/impl/.bilink/7f3d8e9a-….yaml      (tip)

Los dos endpoints quedan en PENDING. Revisar con `bilinker get` y aprobar con `bilinker accept`.
```

---

### `bilinker chain status <uuid>`

Muestra el estado completo de una cadena recorriendo todos sus nodos.

```
bilinker chain status <uuid>
```

**Salida:**

```
$ bilinker chain status 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a

Chain: 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a  [DIRTY]

  .bilink/                         (tip)   (OK, CHAIN_DIRTY)
    link.0  specs :: voting.yaml#impl       OK
    link.1  → .stratum/impl                CHAIN_DIRTY

  .stratum/impl/                   (tip)   (CHAIN_DIRTY, ALTERED)
    link.0  → spec layer                   CHAIN_DIRTY
    link.1  java-demo :: Persona#vote      ALTERED
              source: commit c7d3e9f "Inline comparator"
```

**Estado global de la cadena:**

| Estado | Condición |
|---|---|
| OK | Todos los nodos y fragmentos en estado OK. |
| DIRTY | Algún nodo tiene CHAIN_DIRTY. |
| BROKEN | Algún nodo tiene estado terminal (ALTERED, DELETED, UNANCHORED, BROKEN). |

---

### `bilinker chain list`

Lista todas las cadenas encontradas en el proyecto a partir del directorio actual.

```
bilinker chain list
```

**Salida:**

```
$ bilinker chain list

7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a  [DIRTY]   spec → impl
3a4b5c6d-2e3f-4a5b-9c6d-7e8f9a0b1c2d  [OK]      spec → impl
f1e2d3c4-5a6b-7c8d-9e0f-1a2b3c4d5e6f  [BROKEN]  spec → impl
```

## Código de salida

| Código | Condición |
|---|---|
| 0 | Operación exitosa. |
| 1 | Error: UUID no encontrado, layer inválida, UUID duplicado en una layer. |
