# El corte entre el cliente y el servidor

Worklist se instala en dos lados que no son el mismo público: la máquina de quien trabaja, y el servidor git que recibe los push. **Son dos binarios, `worklist` y `worklist-server`**, y esta página dice qué cae de cada lado y por qué.

> **El cliente no tiene credenciales del proveedor porque no tiene con qué usarlas.**

Es la misma preferencia que el resto del proyecto: que lo inválido no se pueda representar, en vez de que sea improbable. Todo este diseño se apoya en que el servidor es el único que habla con el proveedor —de ahí sale que el push sea un [compare-and-swap](sync.md#la-ventana-y-el-compare-and-swap), que la ventana acote el alcance de la pregunta, y que la instalación tenga una cuenta de servicio—, y un binario que le da `assign-keys` a cada máquina de desarrollo contradice eso mismo. La regla existía; lo que no existía era nada que la hiciera cumplir.

## Dos criterios, y hacen falta los dos

| Criterio | Qué protege | Qué atrapa que el otro no |
|---|---|---|
| **la credencial** | lo que se escribe **afuera**, en el proveedor | `create-or-find`, que no corre desde ningún hook |
| **la propiedad del artefacto** | lo que se escribe **adentro**, en las ramas del servidor | `propagate`, que no toca el proveedor |

El segundo sale entero de que [`insecure/all` sea una rama del servidor](sync.md#y-el-que-escribe-es-el-servidor-que-ya-lo-hacía): **quien commitea en el panorama es el servidor, tenga credencial o no.** Sin ese criterio, `propagate` —que es cherry-pick y `git mv`, cero credenciales— queda del lado del cliente, y ahí ya se cobró: corriéndolo desde el clon, el panorama recibió catorce commits que el diseño le asigna al servidor. Nada lo impidió, porque no había nada que lo impidiera — era el comando que estaba a mano.

**Y el disparador no es criterio.** Se descartó llamarlo `worklist-hook`: el nombre decía *"corre desde un hook"* y dejaba adentro a `create-or-find`, que no es uno. Los hooks son **una forma de invocar** al servidor —`worklist-server check-push --stdin`—, no su definición.

## El reparto

| Subcomando | ¿Habla con el proveedor? | ¿Escribe en una rama del servidor? | Binario |
|---|---|---|---|
| `check-push` | sí — `snapshot` de N claves | no | `worklist-server` |
| `assign-keys` | sí — crea, edita, vincula, mete en el sprint | sí | `worklist-server` |
| `bootstrap` | sí — crea los issues del backlog | sí | `worklist-server` |
| `reconcile` | sí — adopta lo que ya existe | sí | `worklist-server` |
| `adopt` | sí — crea del otro lado todo un corpus | sí | `worklist-server` |
| `create-or-find` | sí | no | `worklist-server` |
| `propagate` | **no — es git puro** | sí, el panorama | `worklist-server` |
| `install-hooks` | no | no — escribe `hooks/` del bare | `worklist-server` |
| `provider set-status` | el de **prueba**, que es un archivo | no | `worklist-server` |
| `window open` | no | **sí — lee el panorama y escribe la rama de la ventana** | `worklist-server` |
| `new` | **no — es escribir un archivo** | no — escribe en la vista | `worklist` |
| `state change` | **no — propone, no consuma** | no — escribe en la vista | `worklist` |
| `remove` | **no — propone `dropped`** | no — escribe en la vista | `worklist` |

**`provider set-status` no es de ninguno de los dos públicos: es de las pruebas.** Va igual en el servidor, porque el proveedor de prueba **sustituye al real en el lugar donde el real se usa** —`check-push --provider-file`—, y ese comando ya está de ese lado. Ponerlo en el cliente le daría a la máquina de desarrollo la única forma de manipular un proveedor que el sistema tiene; un tercer binario sería una instalación más para algo que sólo corre en pruebas.

### El cliente queda con dos comandos, y los dos escriben una propuesta

Hoy `worklist` tiene **dos subcomandos** —`state change` y `remove`—, y no es un accidente del recorte: es dónde está el proyecto. Lo que va a llenarlo ya está decidido y sin implementar —`sync`, `view add`, `new`, `status`, `is-secure`—, y **todo eso nace del lado correcto sólo si el binario existe antes**.

**Y los dos que hay hacen lo mismo con el push**: escriben en la vista y esperan. Es la forma que le queda al cliente cuando no tiene ni el panorama ni la credencial — proponer en un archivo, y que el servidor lo consuma cuando el push llega.

**`state change` y `remove` son del cliente por el mismo criterio, y son el caso que lo pone a prueba**: los dos terminan en una operación del proveedor —una transición—, y aun así ninguno la ejecuta. Escriben la propuesta en la vista, y el servidor la consuma cuando el push llega. **Que el efecto final sea del proveedor no hace que el comando lo sea**; lo que decide el lado es quién habla, no en qué termina.

**Y `new` es del cliente sin discusión**, que antes no era obvio: mientras el id salía de un contador del servidor, crear un ítem necesitaba conectividad y el comando quedaba a mitad de camino. Con un id local marcado, crear un ítem es escribir un archivo — ver [`commands/new.md`](../commands/new.md). Es el motivo de que este corte se haga ahora y no después: lo que se escriba de acá en adelante es del servidor, y con un binario sin partir se escribe del lado del cliente y hay que mudarlo.

### Y `window open` cambió de lado

Estaba anunciado acá mismo —*"se lo lleva la task que saca el panorama del cliente"*— y ya pasó: **el cliente no tiene `all`**, así que no tiene de dónde cortar.

Y cruza por los dos criterios a la vez, que es lo que lo vuelve un caso limpio y no una excepción:

| | |
|---|---|
| **lee el panorama** | que existe en un solo lado |
| **escribe la rama de la ventana** | que es un artefacto del servidor, igual que el panorama del que sale |

> **Recortar es derivar un artefacto de otro. Las dos puntas son del servidor, y el cliente no era ninguna de las dos.**

Lo que le queda al cliente no es un comando más chico: es `git fetch` y un worktree. **La ventana baja y el panorama no**, y esa asimetría es la que define a cada uno — ver [`sync.md`](sync.md#el-panorama-vive-en-un-solo-lado-y-la-ventana-en-los-dos).

## Lo que el cliente necesita del proveedor, se lo pide al servidor

`view add assigned/julio` necesita saber quién está asignado, y eso es una consulta al proveedor. Con este corte el cliente no puede hacerla, y **la salida no es darle credenciales**: es que le pregunte al servidor, igual que `sync` pide verificación en vez de verificar.

Y deja al descubierto lo que falta: **el único canal cliente→servidor es `git push`.** Dos comandos independientes pidiendo el mismo canal es la señal de que el canal es la pieza.

### Y el tercero no pide nada del proveedor

`window open` es el que corrige el título de esta sección. No necesita una credencial ni una consulta a Jira: necesita **el panorama**, que también está de un solo lado. Así que lo que el cliente le pide al servidor no es *"preguntale al proveedor por mí"* — es *"hacé esto, que sólo vos podés hacer"*, y la credencial era una de las razones y no la razón.

**Hoy no hace falta ningún canal, y eso es una casualidad de la instalación**: el bare está en la misma máquina, así que recortar una ventana es correr el comando parado ahí y traerla con un `fetch`. **El día que el servidor sea un GitLab, los tres comandos se quedan sin cómo pedir.** Se acepta a sabiendas, porque el canal es una pieza propia y no un rincón de ninguno de los tres.

## La lib se parte también, y no es lo mismo que partir los binarios

Partir los binarios no obliga a partir la lib, y dejarla entera dejaría `board.rs` y `port.rs` en la máquina del cliente aunque ningún comando los llame. Es menos grave —sin comando no hay entrada— y es exactamente la diferencia entre *"no se puede"* y *"no hay por dónde"*. **Se parte.**

```
crates/
  worklist-core/       el renombre, la reescritura, la ventana, el cuerpo
  worklist-provider/   el puerto y sus dos transportes, el compare-and-swap
  worklist-cli/        el binario `worklist`         → sólo core
  worklist-server/     el binario `worklist-server`  → core + provider
```

| Crate | Módulos | Por qué ahí |
|---|---|---|
| **core** | `lib` (renombre y reescritura), `window`, `body`, `git` | el cliente los necesita: recortar y rebasear una vista es suyo, y la conversión del cuerpo es del formato, no del proveedor |
| **provider** | `port`, `board`, `provider`, `check_push`, `assign`, `propagate` | todo lo que enlaza el puerto, más lo que escribe en las ramas del servidor |

### El corte de la lib no es el mismo que el de los comandos, y hay que decirlo

> **Los comandos se cortan por dos criterios. La lib se corta por uno solo: enlazar el puerto.**

`propagate` es el caso que lo muestra. Como **comando** es del servidor por el segundo criterio —escribe en el panorama—, pero su **código** no toca el proveedor en absoluto: es `merge-base`, `cherry-pick` y lectura de frontmatter. Cortar la lib por propiedad del artefacto no significaría nada, porque una lib no escribe en ninguna rama: la escribe quien la llama.

Y el cliente necesita parte de esa maquinaria: **recortar una ventana ya usa el cherry-pick**, porque replantar el trabajo que la vista tenía encima del corte es lo que la vuelve segura de re-cortar.

**Lo que se hace entonces es separar las primitivas de git de la propagación que las usa:**

| | |
|---|---|
| `core::git` | `cut_commit`, `cherry_pick_one`, el sha nulo — **son git, no propagación** |
| `provider::propagate` | qué commits suben al panorama, cómo se reescriben y qué se anota |

Así el cliente tiene con qué replantar su vista y **no** tiene la función que escribe en el panorama. Es la misma distinción de siempre aplicada al código: no alcanza con que no haya comando, si la función está ahí.

**Y el nombre mejora al separarlas.** `cut_commit` vivía en `propagate` por quién la llamó primero, no por lo que hace.

**Y `worklist-cli` no depende de `worklist-provider`.** Es lo único que hay que verificar para que la invariante se sostenga: es una línea de un `Cargo.toml`, y el día que alguien la agregue el corte se deshace en silencio. La verificación no es una convención — es que el grafo de dependencias no la tenga.

Los nombres de los crates son asimétricos —`worklist-cli` produce `worklist`— y los binarios no: lo que se instala es `worklist` y `worklist-server`, que es lo que alguien escribe en una terminal.

## Los hooks los escribe el servidor

```bash
worklist-server install-hooks --repo <bare>
```

Escribe `hooks/pre-receive` y `hooks/post-receive` apuntando **al path del binario que los está escribiendo**, resuelto por él y no tipeado por nadie.

> **Un hook que apunta al binario equivocado es un fix verde en los tests y ausente en producción.**

Ya pasó: el hook apuntó una vez a un `target/debug` viejo. Dos `sh` de tres líneas parecen demasiado poco como para generarlos, y lo que se generó no son las tres líneas — es el path, que es la única parte que puede estar mal.

Ver [`commands/install-hooks.md`](../commands/install-hooks.md).

## Dos binarios son dos formas de quedar viejo

Es el costo de este corte y hay que poder verlo, no confiar en que no pase:

- **`worklist --version` y `worklist-server --version`**, las dos disponibles sin credencial ni repo.
- **El `pre-receive` anuncia la suya en la salida del push.** Es el único momento en que el cliente y el servidor se hablan, así que es el único lugar donde la diferencia se puede notar sin ir a mirar. Quien empuja ve contra qué versión está hablando, junto al resto de lo que el hook informa.

Contra el costo: hoy el que instala tiene que poner el CLI entero en el servidor y el CLI entero en cada cliente, y las dos instalaciones son la misma cosa con permisos distintos que nadie chequea. Partirlo hace que el tutorial de instalación tenga dos secciones cortas en vez de una con salvedades.

## La tabla envejece, y ya van tres veces

`propagate` no estaba cuando el criterio se escribió; `reconcile` tampoco; y la composición del producto —qué ítems tiene cada sprint, el orden del backlog— todavía no tiene ningún subcomando que la toque. Cae del lado del servidor por el segundo criterio, no por el primero: no habla con el proveedor, pero es propiedad del artefacto.

> **La tabla no estaba mal — estaba completa cuando se escribió.**

Por eso lo que decide no es la tabla sino los dos criterios de arriba. Un subcomando nuevo se ubica preguntándole a los dos, y la fila se agrega después.
