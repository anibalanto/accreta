# La frontera entre proyectos

Un bilink que cruza de un proyecto a otro. Una punta [`abstract`](#el-endpoint-abstract) del lado del proveedor —que publica sin saber quién consume— y un [endpoint repo](#el-endpoint-repo) del lado del consumidor, que lo referencia por un alias local.

La decisión y su motivación viven en [ADR-0005](../.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md). Acá está lo que hay que cumplir para que funcione.

## Lo que no puede pasar

**Ningún extremo conoce al otro.** Ni su hash de contenido, ni su ubicación, ni su existencia:

- El **proveedor no guarda absolutamente nada** de ningún consumidor. Publica un bilink con una punta abierta y no se entera de quién la toma.
- El **consumidor guarda dos SHA-256 opacos** y un alias. De esos hashes no se reconstruye path, query, texto ni commit.

La asimetría es el punto. El consumidor nombra al proveedor porque hay que saber de qué se depende; lo que se elimina es que el proveedor tenga que saber de nadie — que es lo que hacía impracticable el modelo anterior, donde cada lado copiaba el hash del otro.

**Y el proveedor no paga por consumidor.** Un fragmento publicado es **un** archivo, sin importar cuántos lo consuman. Cada consumidor tiene el suyo, con el mismo nombre, en su propio repo. Sumar o quitar un consumidor no toca el repo del proveedor.

## El endpoint `abstract`

Una punta que **no es responsabilidad de quien la declara**:

```yaml
# hsi/.bilink/8a3f0d21-….yaml — el bilink abstracto
endpoint:
  0:
    link: capture 8a3f0d214e5b4c3d9f2a1b2c3d4e5f6a
    accepted:
      link: capture 8a3f0d214e5b4c3d9f2a1b2c3d4e5f6a
      hash: c4e1770b93a5f8d2…
  1:
    link: abstract        # lo aporta quien lo consuma
```

**Es un valor, no un campo ausente.** La aridad fija en dos es la invariante más defendida del formato y no vale la pena romperla por esto. Hay precedente en `TODO`, que ya es *"una intención declarada — no es un error"*; acá la intención es permanente y abierta a cualquiera.

**No lleva bloque `accepted`.** No hay nada que bendecir del lado abierto, y con el bloque entero ausente eso es una ausencia y no una lista de campos que enumerar.

### Por qué un bilink y no un capture suelto

Un capture tiene estado de resolución y no de aceptación. La pregunta que el proveedor necesita poder contestar —*"¿lo que publiqué sigue coincidiendo con lo que aprobé?"*— es **puramente local** y sólo un bilink la sostiene.

Y le da al fragmento una **identidad durable** que sobrevive a sus propios cambios: el UUID del bilink es lo que los consumidores referencian, y sigue valiendo cuando el proveedor mueve el fragmento o cambia su contenido.

### El estado es `OPEN`, constante

Siempre sana, nunca pide acción. No hay contra qué compararla, así que no puede tomar otro valor.

Se le da un nombre en vez de dejar el slot vacío porque la tupla `(state.0, state.1)` la consumen [`check`](../commands/check.md) para su código de salida, `accept .` para elegir a quién aceptar, [`status`](../commands/status.md) para imprimir y lattice para el campo `state` de la arista. **Un valor constante lo maneja cada uno sin ramas; un hueco obliga a todos a tratar el caso nulo.** Es la misma forma que `TODO`.

`accept .` nunca la toca.

## El endpoint repo

El UUID es el mismo que el del bilink remoto —así que no se escribe— y el otro repo se nombra por un **alias local**:

```yaml
# retinar/.bilink/8a3f0d21-….yaml
endpoint:
  0:
    link: repo hsi
    accepted:
      link: 8f2a4c6e…      # id del capture del proveedor — ubicación publicada
      hash: c4e1770b…      # hash del fragmento del proveedor — contenido publicado
  1:
    link: capture 3d9b7e152a4c4b6d8e0f1a2b3c4d5e6f
    accepted:
      link: capture 3d9b7e152a4c4b6d8e0f1a2b3c4d5e6f
      hash: a71f5c3d…
```

**Es el endpoint `path` generalizado.** Misma convención de UUID compartido, mismo `.bilink/` implícito, misma forma de resolución — sólo cambia que la dirección se resuelve por alias en vez de por path relativo:

```
path:  resolved = ../<stratum-path>/.bilink/<uuid>.yaml
repo:  resolved = <clon de .{alias}.toml @ refs/bilink/{branch}>/.bilink/<uuid>.yaml
```

### El alias se declara una vez, en un `.toml`

```toml
# .bilink/.hsi.toml
remote = "git@gitlab…:minsal/hsi.git"
branch = "rc-2.32"
```

**Vive en `.bilink/`, no en `.stratum/`.** Un proveedor externo no es una capa inferior del consumidor, y declararlo bajo `.stratum/` diría que sí. La forma del nombre —un dotfile que describe a su hermano sin punto— es la de Stratum, aplicada a un repo que no es una subcapa.

El clon va al lado de su declaración, en `.bilink/<alias>/`, y está **gitignoreado**: no se commitea el checkout de otro repo.

**El `.bilink` no contiene ninguna URL**, sólo un nombre local. Toda la identidad del proveedor —dónde está, qué rama— queda concentrada en un archivo por proveedor. Si `hsi` cambia de host, se edita un archivo y no N bilinks.

### La rama es el pin de versión, y va en el `.toml`

`branch` declara la rama **del proyecto**, y la herramienta traduce a [su ref de bilinks](ref.md): `branch = "rc-2.32"` se busca en `refs/bilink/rc-2.32`. Una sola fuente de verdad, y nadie tipeando namespaces de refs.

Como esa ref lleva **el árbol del proyecto más `.bilink/`**, un solo fetch trae las declaraciones del proveedor y el código al que apuntan, coherentes por construcción. Es lo que hace simple el caso remoto, y la razón por la que la frontera necesita que ADR-0004 esté antes.

Todos los vínculos hacia un proveedor siguen la misma rama; cambiar de `main` a `rc-2.32` es editar una línea. Soportar dos versiones a la vez son **dos branches del consumidor**, cada una con su `.toml` — no un campo de rango de versiones. La branch dice *qué línea seguir*; los valores aceptados dicen *qué vi en esa línea*.

## La comparación, sin abrir el clon

El consumidor copia los **dos** valores aceptados del bilink remoto, igual que un endpoint `path` copia el valor estructural de su vecino. Dos SHA-256 opacos, comparados por separado:

| Comparación | Significado |
|---|---|
| las dos iguales | `OK` |
| `accepted.link` cambió, `accepted.hash` igual | el proveedor **movió** el fragmento y lo aceptó — revisar |
| `accepted.hash` cambió | el proveedor **cambió** el fragmento — revisar |

Con eso la frontera obtiene el mismo vocabulario que el caso local, **sin leer un solo archivo del proveedor**.

**Un `apply` del proveedor sin aceptar no mueve nada.** Es trabajo en curso, y el consumidor no tiene por qué enterarse — ni podría resolverlo. Ésa es la razón de copiar campos con nombre y no el hash del archivo `.bilink` remoto entero.

Cada valor cambia por una sola razón y por ninguna otra, y los dos son inmunes a etiquetas, comentarios y reordenamientos del archivo remoto.

**Y ninguno invalida la referencia.** `link` sigue apuntando al UUID durable; sólo se mueven los valores aceptados. Poner el id del capture en el *endpoint* dejaría la referencia colgada en cada cambio del proveedor, sin poder distinguir "cambió" de "nunca existió"; ponerlo en la *aceptación* no.

### Y una lectura más: que siga siendo `abstract`

El consumidor lee además el `link` de la otra punta del bilink remoto para verificar que **sigue siendo `abstract`**. Si dejó de serlo, es `REJECTED`.

Dos lecturas, dos hechos, cero normalización. `REJECTED` es un hecho distinto de "el fragmento cambió" y no se mezcla en el mismo token: la otra punta ya no admite ser ampliada, así que el vínculo no puede sostenerse. **El nombre describe la condición desde el lado que la sufre**, que es el único que puede observarla — el proveedor no rechaza a nadie en particular, ni sabe que hay alguien.

### Ningún commit del proveedor se copia

Cuando hace falta ubicar el suyo, se descubre recorriendo su ref hacia atrás hasta que su `accepted` coincida con lo aceptado. Es lo que hace [la profundidad a pedido](#profundidad-a-pedido).

## La versión del proveedor se verifica

El consumidor lee el [`.bilink/version`](format-version.md) que viene **adentro de la ref del proveedor**, y se niega si no lo entiende — en vez de malinterpretar.

**Es la razón de fondo del campo.** Dentro de un proyecto, formato y binario se mueven juntos; entre proyectos son repos con ciclos de release independientes, así que la divergencia de versiones no es un accidente sino lo normal.

Y arrastra algo contraintuitivo: **un cambio aditivo también sube la versión.** `link: abstract` no rompe a quien escribe —ningún archivo existente lo usa— pero un parser viejo lo leería como un path de capa, en silencio y sin fallar. El ledger de migraciones no puede expresarlo porque no hubo migración que registrar. Ver [versión del formato](format-version.md) § "Se verifica, no se declara".

Negarse es lo correcto y no una limitación: el modo de falla alternativo es reportar estados sobre un archivo que no se entendió.

## El clon: `check` nunca hace red

`check` es masivo y **completamente offline**. Si el repo del proveedor no está clonado, reporta `REMOTE_UNREACHABLE` y sigue.

Las operaciones de red son explícitas y puntuales: el clon inicial, el fetch de la ref, y la profundización de `get --diff`.

### El sparse se calcula, no se declara

El conjunto de archivos a traer **sale de los bilinks**, no de un campo del `.toml`:

```
1. Clonar sólo `.bilink/` — alcanza para el paso siguiente.
2. Por cada bilink local con endpoint repo hacia el alias:
     resolver <clon>/.bilink/<uuid>.yaml
     seguir su endpoint estructural hasta <clon>/.bilink/capture/<id>.yaml
     leer su `file`.
3. Ampliar el sparse-checkout a ese conjunto, sin volver a clonar.
```

No hace falta persistirlo: git ya lo guarda en el clon, que además está gitignoreado. Y es **incremental por naturaleza** — sumar un vínculo de frontera agrega un archivo, sacarlo lo quita. Un conjunto fijo en el `.toml` quedaría desactualizado con el primer bilink nuevo, y sería un valor derivado metido en un archivo de declaración.

Hacen falta los archivos y no sólo los `.bilink` porque **detectar el drift y entenderlo son cosas distintas**: los valores aceptados dicen que algo cambió; para mirar el fragmento, correr `get` y decidir si se acepta, hay que tener el archivo del proveedor.

### Profundidad a pedido

El clon arranca **superficial**: el árbol actual de la rama declarada, sin historia. Alcanza para `check`, que corre sobre todo y no puede andar profundizando clones como efecto colateral.

Cuando alguien pide ver qué cambió entre lo aceptado y lo actual de **un** bilink, recién ahí se trae lo necesario: se recorre el `.bilink` remoto hacia atrás hasta la versión cuyo `accepted` coincide con lo aceptado, se lee el capture de entonces, y se compara.

**El reparto es: `check` es masivo y barato; ver el diff es puntual y caro.** El conocimiento mínimo queda como default, no como límite — y lo que la profundidad acota es la *historia*, no el alcance: el sparse-checkout ya entrega el archivo entero, que es más que el fragmento.

### Del otro lado no hay capas

Un alias nombra un **repo**, no una capa: bilinker busca el `.bilink/` de la raíz del clon y nada más.

La estructura interna del proveedor —si tiene `.stratum/`, cuántas capas— no es asunto del consumidor, y meterla en el `.toml` sería volver a saber de más.

## Taxonomía de ausencia

Bajo un solo nombre no se puede distinguir *"me falta traer algo"* de *"algo se rompió"*, que es la diferencia que decide si alguien tiene que mirar. Y no basta con separar en dos: **la subcapa y el proyecto ajeno se arreglan con comandos distintos**.

| Situación | Estado | Arreglo |
|---|---|---|
| `.toml` presente, directorio ausente | `LAYER_UNREACHABLE` | `stratum pull` |
| Ni `.toml` ni directorio, con aceptación previa | `LAYER_UNCONFIGURED` | declarar la capa · o `bilinker remove` |
| Ni `.toml` ni directorio, sin aceptación previa | `TODO` | crear la capa |
| Contenedor presente, `.bilink` del UUID ausente | `BROKEN` | investigar — es regresión |
| Repo ajeno declarado, clon ausente | `REMOTE_UNREACHABLE` | lo clona bilinker |

Las dos ausencias que se arreglan trayendo algo son **normales** —trabajar sin clonar todas las capas es lo esperado—, `LAYER_UNCONFIGURED` pide una declaración o un `remove`, y sólo `BROKEN` es una regresión.

**`UNREACHABLE` a secas desaparece del vocabulario.** Sus tres situaciones quedan repartidas entre `LAYER_UNREACHABLE`, `REMOTE_UNREACHABLE` y `BROKEN` — que es lo que `BROKEN` ya significaba.

**El nombre sale de qué es la cosa, no de cómo se la trae.** Una capa es una capa aunque su `.toml` la busque por URL; un proyecto git ajeno es, en el vocabulario de git, un remoto. No sirve `REPO_`, porque una capa Stratum **también** es un repo y no contrastaría con nada.

## La limitación conocida

Con el UUID implícito, un consumidor **no puede vincular dos fragmentos locales a la misma abstracción remota**: el nombre de archivo colisiona. Haría falta una segunda abstracción del lado del proveedor.

Se acepta, porque los consumidores reales ya centralizan el consumo en un único método por operación.

## Invariantes

1. Un endpoint `abstract` no tiene ni va a tener contraparte en su propio repo, y su estado es `OPEN` siempre.
2. Un endpoint repo se nombra con el prefijo `repo` más un alias, y el alias se resuelve por un `.bilink/.{alias}.toml`.
3. El proveedor no guarda nada de ningún consumidor, y su costo no crece con la cantidad de consumidores.
4. Lo que el consumidor guarda del proveedor son dos SHA-256 opacos y un alias. Ninguna URL entra en un `.bilink`.
5. `check` no clona, no fetchea y no hace red. Ausencia se reporta, no se resuelve.
6. El conjunto sparse es derivado de los bilinks, y nunca se declara.
7. El consumidor verifica la versión de formato del proveedor antes de interpretar sus archivos, y se niega si no la entiende.
8. La aridad sigue en dos, y una cadena sigue siendo lineal: la bifurcación ocurre **entre** cadenas, en el UUID del bilink abstracto.
