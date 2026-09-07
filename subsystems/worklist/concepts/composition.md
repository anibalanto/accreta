# La composición del producto

Qué ítems lleva cada sprint, en qué orden está el backlog, y qué coordenada tiene cada sprint en el proveedor. **Vive en un solo archivo, en el panorama, y es del servidor.**

```yaml
# .metadata/product.yaml
sprints:
  - id: 21
    name: "El worklist se muda"     # menos de 20 caracteres
    status: in-progress
    key: 6525
    items: [ACC-315, ACC-316, …]
backlog:
  - ACC-96
  - …
```

`.metadata/` **con punto adelante** por lo mismo que `_sprints/` llevaba `_`: garantiza que nunca choque con un id. Y el panorama ya tiene `.bilink/` en su raíz, así que la forma no es nueva.

## El `.sprint.md` era tres cosas mezcladas

|  | Dónde va ahora |
|---|---|
| **la composición** — qué ítems, el orden del backlog, `key`, `status` | acá |
| **la prosa** — por qué esos ítems, en qué orden, qué quedó afuera, cómo cerró | `*>graviton/sprints/` |
| el archivo | **desaparece** |

Y no es una mudanza por prolijidad: **son tres cosas con tres dueños distintos.** La composición la arma el servidor, la prosa la escribe una persona, y mezclarlas en un archivo hacía que las dos direcciones de la propagación se pelearan por el mismo texto — el `.sprint.md` es hoy [el único archivo que las dos direcciones tocan](propagation.md#y-el-tercero-es-la-clave-del-sprint-por-un-motivo-que-no-es-de-alcance-sino-de-autoría), y por eso el único donde hubo que partir la regla campo por campo.

## Por qué en el servidor y no como campo del ítem

Hubo una decisión anterior que proponía lo contrario —sacar `items` del sprint y ponerlo como campo del ítem— con el argumento de que *"tenerlo escrito obligaría a editar dos lugares al mover algo"*.

**Esto la revierte y respeta su principio.** El problema no era el contenedor: era la **duplicación** entre el `items` del sprint y el ítem. Con un solo archivo hay **un solo lugar** — y uno que el cliente no necesita.

> **Quién arma los sprints y el backlog es responsabilidad del servidor.** Al cliente le interesa ver su carpeta, esté o no configurado el proveedor.

## Y la membresía deja de ser una propiedad de la rama

Es el cambio que más cosas destraba, y conviene decirlo solo.

> **Hoy un ítem está en un sprint porque su archivo está en esa rama.** Con la composición en un campo, está porque una lista lo nombra.

| | Sacar un ítem de un sprint era | Pasa a ser |
|---|---|---|
| en el worklist | **borrar un archivo de una rama** — un commit que alguien hace | **editar una lista** en el panorama, y la ventana se regenera |
| desde el proveedor | no tenía cómo entrar: nadie compara ramas contra sprints | **comparar dos listas**, que es lo que [`absorb`](../commands/absorb.md) ya sabe hacer con los otros campos |

De ahí salen tres cosas que estaban trabadas:

**La invariante de [sólo edición](sync.md#una-vista-sólo-puede-aportar-ediciones) deja de tener excepción.** Que un archivo desaparezca de una vista pasa a ser **consecuencia de regenerar** y no un commit que alguien hace.

**Y sincronizar la membresía deja de ser una traducción y pasa a ser un espejo.** Jira ya modela sprints con issues adentro; comparar `items` contra lo que el board tiene es comparar dos listas de claves. Antes había que preguntarle a la forma del recorte, que no es una lista de nada.

**Y absorber un cambio de membresía deja de ser peligroso.** Traer *"el board sacó este ítem del sprint"* significaba sacar un archivo de una ventana —cambiar el recorte, con trabajo adentro— y pasa a significar editar una línea del YAML: lo que sigue es el recorte de siempre, que es idempotente y ya está probado.

## El nombre sale del título entero, y sin tope

> **El nombre de un sprint es su número y su título, en minúscula y con guiones medios.** Entero.

```
21-la-estructura-del-worklist-se-muda-al-servidor
```

Antes acá decía *"menos de 20 caracteres"*, con el argumento de que un nombre de 20 que sale de cortar uno de 62 no nombra nada. **El argumento era bueno y la conclusión no**: si recortar rompe el nombre, lo que sobra es el tope, no el título.

Y eso es lo que vuelve la migración de los 22 **mecánica** en vez de 22 decisiones a mano.

### Y no es el nombre que ve el proveedor

Son dos nombres, y confundirlos hacía parecer que el tope se podía borrar.

| | |
|---|---|
| **el `name` del YAML** | el nombre de este lado, sin tope. Reemplaza al del archivo |
| **el nombre del sprint en Jira** | `<id> <título>`, **y sigue recortándose a 29** porque Jira no acepta 30 |

Así que la maquinaria del recorte **no se borra**: el nombre de acá se hizo más largo, no más corto. Lo que sí queda claro es de quién es cada límite — uno es del proveedor y el otro no existe.

**Y el nombre del sprint no es su clave.** En la interfaz de Jira el id de un sprint no se muestra, no se puede buscar y nadie lo escribe: un sprint renombrado a `6524` queda imposible de encontrar justo para la persona que iba a usar ese nombre. La clave vive en el YAML como **coordenada**, que es lo que es.

## La mudanza es en dos mitades, y hoy está la primera

**El `items` ya sale de acá.** El recorte lee la composición, el renombre la mantiene, y la clave del sprint se anota acá cuando el servidor la consigue.

**El `.sprint.md` todavía viaja en la ventana**, porque las pasadas que sincronizan sprints —resolver el sprint del proveedor, anotarle la clave— lo leen de ahí. Se va con ellas.

> **La composición es la fuente de la membresía desde hoy. El archivo es una copia que todavía se lee para otra cosa.**

Y el orden no es arbitrario: mover la lectura del `items` no toca el proveedor, y mover las pasadas de sprint sí — son las que crean sprints y meten issues en el board. La primera mitad se puede hacer con la suite como única red; la segunda necesita medirse contra un board.

**Y el renombre necesitó una regla propia.** En un `.md` una referencia a un ítem es un link —`](@o.task.md)`— y se reescribe con el texto. Acá es una **entrada de una lista**, sin sintaxis alrededor: un renombre que sólo mire markdown deja el sprint nombrando un slug que ya no existe, y el próximo recorte falla con *"la composición nombra a `@o`, y no está"*. Se reescribe sobre la estructura, porque `@o` como texto también aparece adentro de `@otro`.

## Y lo que el proveedor perdió se saca, pero no se borra

Un ítem puede tener clave acá y no existir del otro lado. Medido el 2026-09-07: `ACC-268` tenía su `rename 6m -> ACC-268` en el log y Jira contestaba **404**.

Y hoy eso **traba el ítem para siempre**: `is_unassigned` es falso, así que ninguna pasada vuelve a mirarlo — no se recrea, no se corrige, y su clave muerta [se lleva el lote del sprint entero](../commands/assign-keys.md#una-clave-que-el-board-no-tiene-no-puede-llevarse-el-lote) en cada push.

> **Sale del árbol y entra a `.metadata/removes/`, entero.**

```
.metadata/removes/ACC-268.task.md
```

El archivo se mueve, no se destruye: **el ítem deja de ser un ítem y su contenido queda a la vista.** No hay que reconstruir nada de la historia de git para saber qué decía, ni por qué no está.

### Por qué lógico y no físico

*"Se recupera de git"* es cierto y no alcanza. Un borrado físico deja una ausencia, y **una ausencia no dice por qué**: quien la encuentra tiene que sospechar que alguna vez hubo algo, y recién ahí buscar. Un archivo en `removes/` contesta las dos preguntas sin arqueología — qué era y por qué se fue.

Y hace la vuelta barata: **devolverlo es moverlo de nuevo.** Si el 404 fue un error —alguien borró de más en el board— restaurar es un `git mv`, no un rescate.

### Se decide por el código, nunca por el mensaje

El proveedor contesta *"la incidencia no existe **o no tienes permiso para verla**"* — **una sola frase para dos casos que no se parecen en nada**. Sacar un ítem porque una credencial perdió permiso sería el mismo error de forma que confundir `sin verificar` con `coincide`, con el costo subido a destruir.

|  |  |
|---|---|
| **404** | no existe → se saca |
| **403** | no se puede ver → **no se toca**, y se reporta |

Se decide por el status HTTP, que sí los distingue. La frase no.

### Y no se libera la clave

`ACC-268` queda muerta y no se reasigna. El ítem tampoco vuelve a nacer con clave nueva: **salió**, y si el trabajo hace falta se escribe uno nuevo, que es una decisión de una persona.

Recrearlo automáticamente sería el sistema discutiéndole al board sobre algo que alguien borró a mano allá.

## Lo que no contesta

**Dónde está el archivo de un ítem.** El YAML dice **en qué sprint está**, que es otra pregunta — y es la que un endpoint `issue` de bilinker necesita.

**Y la composición no baja al cliente**, por lo mismo que el panorama no baja: es del servidor, y una copia que se queda vieja es una fuente de verdad falsa. Lo que el cliente ve es su ventana, que sale de acá por el recorte.
