---
name: item-writing
description: "Cómo se escribe un ítem: el título, el cuerpo, y cómo se lo nombra desde otro texto. Son convenciones de esta organización y de su proveedor, no del formato: un equipo con otro board tiene otras."
when_to_use: "Al escribir, retitular o revisar un ítem, o al nombrar uno desde otro texto."
---

Las convenciones de **redacción** de un ítem. La mecánica —dónde vive el trabajo, qué vista se abre, qué campos lleva el frontmatter— está en el README del impl de `muckpile`, y no se repite acá.

> **Esto es lo que cambia si cambia la organización.** El vocabulario de categorías es de un equipo, y la regla del parser de abajo es de un proveedor. Una instalación con otro board hereda el formato y no hereda esta página.

## Cómo se nombra un ítem

> **`<id> <título>`.** Nunca el id solo.

En markdown el link envuelve las dos cosas:

```markdown
[`3z` Error de cobertura: el vecindario de una firma resuelve a la firma misma, y el tipo que importa no se pregunta](3z.task.md)   ← adentro del worklist, donde el renombre sí llega
```

El id solo obliga a abrir el archivo para saber de qué se habla, y **el que lo abre es el que menos contexto tiene**: quien escribió la referencia ya sabía cuál era. Vale en los ítems y en los sprints.

**El título va verbatim, no una glosa.** Una glosa envejece cuando el título cambia y nada lo detecta; un título copiado envejece igual, pero se ve al lado del que cambió. Y donde el título no describa al ítem, lo que hay que arreglar es el título.

## El título

> **Infinitivo cuando ya sabés qué hacer. Diagnóstico cuando lo que tenés es el síntoma.**

No son dos gustos: un título en infinitivo **obliga a nombrar la solución**. Sobre un bug recién visto eso es inventarla antes de decidirla, y el título queda casado con una hipótesis. Sobre trabajo ya decidido es al revés — un título diagnóstico hace parecer que la tarea *causa* el problema en vez de resolverlo.

**Infinitivo:** *"Especificar e implementar el comando history de bilinker"* · *"Proteger las refs de bilink"* · *"Reescribir la spec de migración"*.

**Diagnóstico**, con la forma `<Categoría>: <síntoma observable> <cuándo pasa>` — el síntoma en términos de lo que se ve, no de la causa sospechada:

> Error de reporte: un endpoint no resuelto no informa qué query ni qué anchor buscaba

El vocabulario de categorías **se deja crecer**, no se inventa por adelantado.

En los dos casos el **cuerpo** abre con el diagnóstico. Lo que cambia es el título.

### Y tampoco lleva markdown: ni negrita ni comillas de código

> Un título es texto plano. Sin `**negrita**` y sin `` `comillas de código` ``.

Es la misma razón que la de abajo, con otro final. El título **no pasa por el conversor**: [`concepts/sync.md`](../../../subsystems/worklist/concepts/sync.md) dice que va al `summary` *"como texto plano, por su propio camino — no es parte de la conversión del cuerpo"*.

Así que el markdown no se renderiza: **aparece literal en el board.** Un título escrito `` Especificar `<id>/`: el directorio de datos `` se lee del otro lado con las comillas invertidas puestas, y una negrita se lee con los dos asteriscos.

| En vez de | Escribir |
|---|---|
| Especificar `<id>/`: el directorio de datos de un ítem | Especificar el directorio de datos de un ítem |
| Barrer `_sprints/` de las specs y de `AGENTS.md` | Sacar el directorio de sprints de las specs y del método |
| **Error de reporte:** status y push no coinciden | Error de reporte: status y push no coinciden |

**Y el precio es nombrar en prosa.** Un identificador entre comillas es cómodo justamente porque no obliga a describirlo; sin ellas hay que decir qué es, y eso hace el título más largo y más legible para quien lo lee en el board sin el repo a mano.

**No aplica hacia atrás**, por lo mismo que la regla de abajo: un ítem con clave se identifica por la clave, y cambiarle el título deja el local diciendo una cosa y el `summary` del board otra. Vale para lo que se escribe y para lo que todavía no cruzó.

### Y el título no lleva sintaxis de comando

> **Un título dice qué pasa, no cómo se escribe el comando.** Nada de `--flag` literal: el nombre del flag va en el cuerpo, que es donde hace falta la precisión.

No es de estilo. **El título es el string con el que el proveedor identifica al ítem** —se busca por `summary` antes de crear— y dos guiones seguidos rompen el parser de JQL. Medido: una corrida de `bootstrap` se cortó en el ítem 24 de 109 por un `--format` en un título.

| En vez de | Escribir |
|---|---|
| graph --format json no imprime nada | graph en formato JSON no imprime nada |
| --no-n1 se ignora cuando hay proveedor | el flag que baja el nivel 1 se ignora cuando hay proveedor |

Y la razón no es el guión: **la lista de lo que rompe ese parser no la controlamos**, y ya creció una vez. Un título en prosa no depende de ella.

**No se aplica hacia atrás.** Un ítem que ya tiene clave se identifica por la clave, no por el título, así que el `--` ahí es inerte — y cambiárselo deja el título local diciendo una cosa y el `summary` del board otra, porque lo único que empuja un título es la pasada 4 sobre un push de su ventana. Sobre un sprint cerrado eso no vuelve a pasar. La regla vale para lo que se escribe y para lo que todavía no cruzó.

### Y el prefijo del título puede nombrar el subsistema

Un título ambiguo sin decir de qué se está hablando lleva adelante el subsistema: `bilinker`, `stratum`, `lattice`, `lspd`, `impact`, `worklist`. Es el mismo movimiento que el título diagnóstico —un prefijo que clasifica, después el contenido— y no reemplaza a ninguna de las dos formas de arriba: se usa cuando el título solo no dice de qué es.

El separador es dos puntos o un guión medio suelto, nunca dos guiones seguidos: `Worklist -- cortar la vista` rompe el parser por la razón de la sección anterior.

## El cuerpo

Cuatro secciones, y el orden importa: el cuerpo **abre con el diagnóstico** —eso ya está dicho arriba— y después van éstas.

| | Para qué |
|---|---|
| `## Precondiciones` | qué tiene que ser cierto para que el ítem se pueda ejecutar |
| el *como / quiero / para* | el para qué, y va en la user story — no en cada task |
| `## Cuándo está hecha` | contra qué se mide que terminó. Es la que decide |
| las referencias | la spec, el ADR, y las dependencias |

### Precondiciones

Tres clases, y ninguna es el enunciado del trabajo:

| | |
|---|---|
| la instalación | la capa que tiene que estar traída, el archivo de configuración que tiene que existir, la variable que tiene que estar exportada |
| de qué lado corre | cliente o servidor, y si hace falta la credencial del proveedor |
| lo que tiene que estar hecho antes | otro ítem |

> Y una precondición que es otro ítem **no se escribe en prosa: va en `relation.depends`.**

Es la distinción que más rinde de las tres. Lo que declara el frontmatter lo puede leer un comando —el orden topológico, el grafo de dependencias, el corte de una ventana— y lo que está en prosa no lo lee nadie. Un párrafo que dice *"depende de que la composición viva en el YAML"* sin el campo es una dependencia que existe para el que lee y no para el proyecto.

Y las precondiciones se escriben **sin repetir el diagnóstico**. Si una precondición es la mitad del enunciado del ítem, sobra en uno de los dos lugares.

### El *como / quiero / para* va en la user story, y una sola vez

Es el título de la user story, que es donde ya se escribe:

> Como quien abre el worklist, quiero cortar sólo las vistas que necesito, y que mover un archivo entre dos sea reasignar el ítem

Una task que cuelga de una user story **no lo repite**. Repetirlo es reescribir el enunciado del padre en cada hijo, y con la regla del ancestro —una user story entra a un sprint con todas sus tasks— eso son cinco o siete copias del mismo párrafo que envejecen por separado.

Una task suelta, sin `parent`, tampoco lo lleva. Si el para qué de una task no se puede leer del ítem del que cuelga, lo que falta es la user story.

### Cuándo está hecha

Los criterios de cierre, y son la sección que más cuidado lleva.

> **El criterio de cierre es lo único que distingue *"está hecho"* de *"el proveedor dice que está hecho"*.**

Sin él, el estado del board no se puede constatar: se le cree. Y un ítem que el board da por cerrado sin que nadie pueda mirar contra qué es un ítem que no se puede auditar después — que es justamente lo que el paso 0 del método existe para evitar.

Tres propiedades, cada una con su falla:

| | |
|---|---|
| atómico | un criterio que dice dos cosas no se puede chequear a medias. Si se cumple una mitad, no hay respuesta — se parte en dos |
| verificable | tiene que nombrar algo que se pueda mirar: un archivo que existe, un comando que sale con cero, una spec que dice tal cosa. *"Queda claro"* y *"funciona bien"* no se miden |
| no redundante | dos criterios que se cumplen siempre juntos son uno, y el segundo hace parecer que hay más cerrado de lo que hay |

Y una duda **no entra como criterio: entra como pregunta**, en `## Lo que hay que decidir`. Un criterio que en realidad es una duda no se puede cumplir, así que el ítem no cierra y nada dice por qué.

### Referencias

Tres vínculos, y los tres tienen dirección obligatoria.

**La spec que el ítem toca.** El ítem cita la spec, y la spec **no cita al ítem** — es la misma regla que `decision-records` aplica a las decisiones, y el motivo es que el id de un ítem cambia por diseño cuando cruza al proveedor. Un endpoint `issue <id>` de bilinker es la excepción, porque es una referencia verificada y repuntable.

**La decisión que el ítem ejecuta**, en `docs/decisions/` —ver la skill `decision-records`—. Si la justificación de un ítem sólo existe adentro del ítem, lo que falta es la decisión.

**Las dependencias, en `relation.depends`** y no en prosa, por lo dicho en § Precondiciones.

### El ítem dice qué tiene que ser cierto, no cómo se decidió

Es la regla que más cuesta y la que más rinde.

Un ítem que lleva adentro la arqueología de sí mismo —*"acá decía X, y el argumento era bueno y la conclusión no"*— cuesta releerlo lo que cuesta releer una spec. Y un ítem que no se relee conserva premisas muertas, que es peor que no tener nada escrito: se le cree, y nadie mira si la premisa sigue en pie.

| Va en el ítem | Va en la decisión |
|---|---|
| qué tiene que ser cierto cuando termine | por qué se decidió así |
| contra qué se mide | qué alternativas se descartaron, y con qué argumento |
| de qué depende | qué premisa cambió, y cuándo |

**Y el registro de lo que cambió de opinión no se borra: se muda.** Va a la decisión y a la prosa del sprint, que es donde ya vive, y ahí no le pone peso al ítem que alguien tiene que leer antes de trabajar.

## Lo que todavía no está escrito acá

**La revisión por otro.** Un ítem escrito por una persona y ejecutado por la misma no tiene quién le encuentre el criterio que falta. El día que haya dos, la revisión necesita un lugar en el ítem y no sólo en el proceso: quién lo revisó, contra qué, y qué pasa cuando el alcance cambia después de aprobado.

Queda anotado para no redescubrirlo.
