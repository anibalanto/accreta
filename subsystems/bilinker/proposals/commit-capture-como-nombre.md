# Propuesta: `<commit>:<capture>` como el nombre de un fragmento

**Estado:** en discusión. Sin diseño. Vive acá porque la observación que la origina es chica y las consecuencias no.

[`discutir-una-aceptacion`](discutir-una-aceptacion.md) descartó el fragmento como ancla con una línea de tabla — *"se mueve, y con él el capture"*— y eligió el commit de la ref. La elección estuvo bien y el descarte fue de más:

> **Un capture se mueve a través de la historia. Adentro de un commit no se mueve nada.**

## La observación

Un capture es una ubicación —`file` y `query`— y su nombre es el hash de esos dos campos, así que es [inmutable por construcción](../concepts/capture.md). Un commit de la ref fija el árbol de código: por la [invariante 2](../concepts/ref.md#invariantes) es idéntico al del commit del proyecto absorbido más recientemente, y por la [6](../concepts/ref.md#invariantes) ese commit no se reescribe nunca.

Fijadas las dos cosas, resolver la query da **un fragmento y uno solo**. El par no es una referencia aproximada: es una función total sobre objetos que ya existen y que nadie puede mover.

```
<commit de la ref> : <capture>   →   un nodo del AST, y siempre el mismo
```

Es la misma forma que `<rev>:<path>` de git, un nivel más fino. git nombra un archivo adentro de un commit; esto nombra un nodo adentro de un commit.

## Y el par ya existe — lo que no existe es el nombre

Esto no propone una capacidad nueva. La resolución ya está implementada: [`get --diff`](../commands/get.md) recupera el fragmento aceptado *"resolviendo la query contra el contenido del `commit` del endpoint"*, y [`history`](../commands/history.md) reconstruye contra qué código se aceptó cada vez recorriendo la ref.

Lo que falta es que eso **se pueda decir**. Hoy el commit de una aceptación es implícito: no está en el archivo de bilink —`accepted` lleva `agree`, `link`, `hash`, `hash_ast`— y se averigua caminando la ref hasta encontrar el commit de decisión que la escribió. Se puede llegar; no se puede nombrar.

**Y el commit de absorción es un artefacto, no un detalle de implementación.** Hoy aparece en la spec como mecanismo —*"la correspondencia con el proyecto es el segundo padre"*— y no como algo que alguien pase por parámetro, escriba en un documento o pegue en un chat. Tiene hash, es inmutable, es firmado y es append-only: tiene todo lo que hace falta para ser una dirección, y ninguna sintaxis que lo use.

## Qué le contesta a `discutir-una-aceptacion`

Su tabla de anclas gana una fila, y no reemplaza a las otras:

| Ancla | Problema |
|---|---|
| un fragmento | se mueve, y con él el capture |
| un bilink por UUID | su contenido cambia con cada aceptación |
| un commit de la ref | inmutable y firmado |
| **`<commit>:<capture>`** | **inmutable, y además dice de qué se habla** |

El commit solo contesta *cuándo se decidió*. El par contesta *sobre qué*, que es lo que una discusión necesita nombrar: nadie discute un commit, discute una función.

**Y el alcance deja de ser la aceptación.** Un commit de la ref sólo existe donde hubo una decisión, así que anclar en él permite discutir lo aceptado y nada más. El par existe sobre **cualquier fragmento** que un capture alcance, se haya aceptado o no — que es lo que hace falta para que comentar sea una operación del ecosistema y no una nota al pie de `accept`.

## Dónde vive lo firmado

De la misma observación sale una consecuencia que no es sobre discusiones.

Una aceptación es una firma sobre un contenido. Ese contenido hoy está en el bilink como **hashes sueltos** —`hash`, `hash_ast`— y el bilink cambia con cada aceptación, así que lo firmado no es un objeto: es un campo de un archivo mutable, verificable pero no direccionable.

> **Se propone que lo que se firma sea un capture.**

Un capture ya es la unidad direccionable del sistema, y `accepted.link` ya lleva uno — la **ubicación** de lo aceptado. Que el **contenido** también lo sea cierra la simetría: firmar deja de ser *"guardo el hash de lo que vi"* y pasa a ser *"referencio el artefacto que vi"*, que otro puede abrir, mostrar y comentar con las mismas herramientas.

El caso que lo vuelve concreto es un **contrato**: un documento que se firma y contra el cual se verifica después. Hoy tendría que vivir como fragmento de algún archivo y ser referenciado por hash; con esto, `accepted` referencia el capture del contrato y la firma es sobre algo nombrable.

## Lo que cierra

Tres piezas que ya existen por separado y no se tocan entre sí:

| | hoy | con el nombre |
|---|---|---|
| [`watch`](../commands/watch.md) | avisa que un **archivo** vinculado cambió | puede vigilar un fragmento nombrado |
| comentar | no existe; `discutir-una-aceptacion` busca dónde colgarlo | cuelga de un par, sobre cualquier fragmento |
| `accepted` | hashes de un contenido | referencia al capture de un contrato |

**El seguimiento cierra el círculo cuando lo vigilado, lo discutido y lo firmado son la misma clase de cosa.** Hoy son tres: un path, un commit y un hash.

## Lo que habría que decidir, y esta propuesta no decide

**Qué commit va del lado izquierdo.** Todo commit de la ref tiene árbol de código —la invariante 2 vale para los tres tipos—, así que cualquiera resuelve. Pero el de **absorción** es el único donde el árbol *cambió*, y usar los otros daría muchos nombres para el mismo fragmento. Si el nombre tiene que ser canónico, hay que elegir uno.

**Si se resuelve sin working tree.** Debería: `cat-file` del blob más tree-sitter alcanza, y es lo que permite nombrar un fragmento de un commit que no está checkouteado. Es la diferencia entre una dirección y un atajo.

**Qué pasa con un commit que no está local.** Es el problema que [`get`](../commands/get.md) ya resolvió para la frontera, descubriendo y no copiando. Vale mirar si aplica igual.

**Si el nombre se escribe o sólo se pasa.** Un nombre que sólo viaja por línea de comando no necesita formato; uno que se guarda en un archivo entra al formato y tiene versión.

**Si el capture de lo firmado es un capture normal.** Uno normal apunta a una ubicación del proyecto y se mueve con el código. Un contrato probablemente no deba moverse — y ahí la pregunta se toca con [`abstract`](../concepts/reference.md), que ya está especificado y no implementado.

**Y qué relación tiene con el endpoint [`bilink`](bilink-endpoint.md).** Los dos hablan de referenciar algo del sistema desde adentro del sistema, y conviene que no sean dos mecanismos.

## Lo que esta propuesta no dice

No dice que haya que implementar comentarios. Dice que **el nombre es la pieza que falta para que se puedan implementar**, y que hoy no está por una razón que ya no se sostiene: que un fragmento se mueve.
