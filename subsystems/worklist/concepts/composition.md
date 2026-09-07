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

## Lo que el nombre corto borra

Los nombres van **bajo 20 caracteres**, y eso no es estética.

Hoy el nombre del sprint en el proveedor se arma como `<id> <título>` y **se recorta a 29** porque Jira no acepta más, con la regla de *"se recorta el título y nunca el número, y se marca el recorte"*. Medido sobre los 22 de hoy, los títulos van de **6 a 62** caracteres y varios pasan los 37, así que el recorte ocurre de verdad.

Con el nombre bajo 20, `<id> <nombre>` da **23 como máximo**: el recorte no ocurre nunca, y esa maquinaria se borra.

> **El límite deja de ser un trim y pasa a ser estructural.**

El título largo no se pierde: pasa a ser la primera línea del documento del sprint en graviton, que es donde la prosa vive.

## La mudanza es en dos mitades, y hoy está la primera

**El `items` ya sale de acá.** El recorte lee la composición, el renombre la mantiene, y la clave del sprint se anota acá cuando el servidor la consigue.

**El `.sprint.md` todavía viaja en la ventana**, porque las pasadas que sincronizan sprints —resolver el sprint del proveedor, anotarle la clave— lo leen de ahí. Se va con ellas.

> **La composición es la fuente de la membresía desde hoy. El archivo es una copia que todavía se lee para otra cosa.**

Y el orden no es arbitrario: mover la lectura del `items` no toca el proveedor, y mover las pasadas de sprint sí — son las que crean sprints y meten issues en el board. La primera mitad se puede hacer con la suite como única red; la segunda necesita medirse contra un board.

**Y el renombre necesitó una regla propia.** En un `.md` una referencia a un ítem es un link —`](@o.task.md)`— y se reescribe con el texto. Acá es una **entrada de una lista**, sin sintaxis alrededor: un renombre que sólo mire markdown deja el sprint nombrando un slug que ya no existe, y el próximo recorte falla con *"la composición nombra a `@o`, y no está"*. Se reescribe sobre la estructura, porque `@o` como texto también aparece adentro de `@otro`.

## Lo que no contesta

**Dónde está el archivo de un ítem.** El YAML dice **en qué sprint está**, que es otra pregunta — y es la que un endpoint `issue` de bilinker necesita.

**Y la composición no baja al cliente**, por lo mismo que el panorama no baja: es del servidor, y una copia que se queda vieja es una fuente de verdad falsa. Lo que el cliente ve es su ventana, que sale de acá por el recorte.
