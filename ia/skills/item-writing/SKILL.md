---
name: item-writing
description: "Cómo se escribe un ítem del worklist — el título, y cómo se lo nombra desde otro texto. Son convenciones de esta organización y de su proveedor, no del formato: un equipo con otro board tiene otras. Cargar al escribir o retitular un ítem."
---

Las convenciones de **redacción** de un ítem. La mecánica —dónde vive el trabajo, qué vista se corta, qué chequeos pasar, qué campos lleva el frontmatter— está en la skill `worklist`, y no se repite acá.

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

**Infinitivo:** *"Especificar e implementar `bilinker history`"* · *"Proteger `refs/bilink/*`"* · *"Reescribir `concepts/migration.md`"*.

**Diagnóstico**, con la forma `<Categoría>: <síntoma observable> <cuándo pasa>` — el síntoma en términos de lo que se ve, no de la causa sospechada:

> **Error de reporte:** un endpoint no resuelto no informa qué query ni qué anchor buscaba

El vocabulario de categorías **se deja crecer**, no se inventa por adelantado.

En los dos casos el **cuerpo** abre con el diagnóstico. Lo que cambia es el título.

### Y el título no lleva sintaxis de comando

> **Un título dice qué pasa, no cómo se escribe el comando.** Nada de `--flag` literal: el nombre del flag va en el cuerpo, que es donde hace falta la precisión.

No es de estilo. **El título es el string con el que el proveedor identifica al ítem** —se busca por `summary` antes de crear— y dos guiones seguidos rompen el parser de JQL. Medido: una corrida de `bootstrap` se cortó en el ítem 24 de 109 por un `--format` en un título.

| En vez de | Escribir |
|---|---|
| `` `graph --format json` no imprime nada `` | `` `graph` en formato JSON no imprime nada `` |
| `` `--no-n1` se ignora cuando hay proveedor `` | el flag que baja el nivel 1 se ignora cuando hay proveedor |

Y la razón no es el guión: **la lista de lo que rompe ese parser no la controlamos**, y ya creció una vez. Un título en prosa no depende de ella.

**No se aplica hacia atrás.** Un ítem que ya tiene clave se identifica por la clave, no por el título, así que el `--` ahí es inerte — y cambiárselo deja el título local diciendo una cosa y el `summary` del board otra, porque lo único que empuja un título es la pasada 4 sobre un push de su ventana. Sobre un sprint cerrado eso no vuelve a pasar. La regla vale para lo que se escribe y para lo que todavía no cruzó.
