# Estados

Un ítem tiene un `status`, y **cuáles existen es del proyecto, no del formato**. El formato dice que el campo está y que su valor sale de un vocabulario declarado; qué palabras lo forman lo decide quien usa el worklist.

> **Un equipo que quiere `review` lo agrega sin tocar el worklist.**

## El vocabulario está en git; el mapeo, en la instalación

Son dos archivos porque son dos cosas, y el corte es el mismo que el del [id del board](sync.md#el-puerto-son-las-operaciones-no-los-comandos): lo que **todos comparten** se versiona, lo que depende **de este proveedor y de este board** no.

| | Dónde | Por qué |
|---|---|---|
| **el vocabulario** | `.metadata/states.yaml`, en el panorama | todos tienen que estar de acuerdo en qué significa `review`, y eso es historia del proyecto |
| **el mapeo** | la instalación, al lado de la credencial | depende del workflow del board, que puede cambiar sin que el proyecto cambie |

```yaml
# .metadata/states.yaml — en el panorama, versionado
states: [open, in-progress, done, dropped]
```

```json
// en la instalación, junto a la credencial — `states.json`
{
  "open":        "To Do",
  "in-progress": "In Progress",
  "done":        "Done",
  "dropped":     { "status": "Done", "resolution": "Won't Do" }
}
```

**Y los formatos no son los mismos porque los lugares no lo son.** Lo que vive en git se edita a mano al lado de frontmatter markdown, así que es YAML como todo lo demás del árbol; lo que vive en la instalación va al lado de `provider.json`, que ya es JSON. Elegir un formato único obligaría a que uno de los dos lados desentonara con sus vecinos, que es lo que se lee mal cuando alguien abre el directorio.

**El vocabulario de arriba es el que worklist trae por defecto**, no una lista cerrada. Un proyecto que declara otro reemplaza el archivo entero: no hay estados heredados que convivan con los declarados, porque un vocabulario a medias es peor que uno chico.

**Y va en `.metadata/`**, que es el espacio de lo que el worklist sabe de sí mismo y no es un ítem — ahí va también la composición de los sprints y del backlog. Es la misma señal que el `_` de `_sprints/`, y por la misma razón: [ninguno de los dos puede ser un id](item.md#el-alfabeto-de-un-id), así que un directorio no se confunde nunca con un ítem.

### Y el vocabulario viaja con el recorte

> **Vive en el panorama, y el cliente no tiene el panorama.** Así que `.metadata/states.yaml` entra en cada ventana, como un archivo más del corte.

Se leía del panorama —*"los worktrees comparten refs, así que `insecure/all` está a mano incluso parado en una ventana"*— y eso dejó de ser cierto cuando el panorama [se fue del cliente](sync.md#el-panorama-vive-en-un-solo-lado-y-la-ventana-en-los-dos).

Y lo que se perdía era exactamente lo que esta página promete: `state change` es del cliente **porque validar un estado no necesita al proveedor**, y sin el vocabulario la validación cae al que worklist trae por defecto. **Un proyecto que declara `review` vería su propio estado rechazado**, y por una razón que no tiene nada que ver con el estado.

Es el mismo argumento que hace viajar a la épica de sólo lectura: **la ventana tiene que cerrar adentro.** La cadena `parent` cierra con los ancestros; el vocabulario cierra con este archivo. Y cuesta un archivo por recorte.

### Un estado sin mapeo no arranca

> **Si el vocabulario declara un estado que el mapeo de la instalación no cubre, la instalación falla al configurarse.**

La alternativa —aceptarlo y que ese ítem no sincronice su estado— es peor de la peor manera: **funciona**. El worklist queda con ítems que no cruzan sin que nada lo diga, y el síntoma aparece semanas después como una divergencia sin causa visible. Fallar al configurar es la misma preferencia de siempre: que lo inválido no se pueda representar.

Y la comprobación es barata: dos listas y una diferencia.

### El mapeo no siempre es un status

`dropped` lo muestra: en Jira, *"descartado"* no es un `status` sino un `status` **más una resolución**, y son campos distintos del issue.

```yaml
done:     "Done"                                    # forma corta
dropped:  { status: "Done", resolution: "Won't Do" }  # forma completa
```

**La forma corta es azúcar de la completa**, no otro tipo de entrada: `"Done"` es `{ status: "Done" }`. Un mapeo que sólo supiera hablar de `status` habría obligado a que `dropped` no existiera, o a que mintiera diciendo `Done` a secas — y en el board no se distinguiría de algo que se hizo.

## `dropped`: una decisión que se revirtió

`open | in-progress | done` no tiene dónde poner algo que **se decidió no hacer**.

> **No es `done` —no se hizo nada— y no es `open` —no hay nada que hacer.**

Y como el backlog se calcula, quedarse en `open` por descarte tiene una consecuencia medible: en el momento en que este estado se agregó, **cuatro ítems descartados aparecían en el inventario de trabajo pendiente**. El inventario de lo que falta hacer incluía cosas que ya se había decidido no hacer.

**Borrarlos no era la salida.** El valor de una decisión revertida es el registro de por qué se descartó, que es lo que evita volver a proponer lo mismo dentro de un mes. Se puede borrar cuando ese porqué ya está absorbido en el ítem que la reemplazó, y ahí es un `remove` — ver abajo.

| | |
|---|---|
| **cuenta como cerrado** | un ítem `dropped` no bloquea el sprint que lo lleva: no queda nada por hacer |
| **no cuenta en el backlog** | que es el punto entero de que exista |
| **el archivo se queda** | con su historia y su porqué |

## worklist no tiene reglas de transición. El proveedor sí

> **`worklist` no decide si de `open` se puede pasar a `done`.**

No es una omisión: es que **no puede saberlo**. El workflow de un board dice qué transiciones son legales, lo configura quien administra el proyecto, y cambia sin avisarle a nadie. Una tabla de transiciones acá sería una copia que empieza desactualizada.

De ahí sale [`worklist state change`](../commands/state-change.md), y su forma: **hace un commit propuesto**, no un hecho. Si el proveedor rechaza la transición —porque su workflow exige pasar por `review` antes—, el commit no llega y el motivo se reporta.

### Y hay dos formas de rechazo, que no se pueden confundir

Este sistema ya rechazaba, pero por un solo motivo. Ahora son dos, y **el que lee tiene que poder distinguirlos porque lo que hay que hacer es distinto**:

| | Qué pasó | Qué hacer |
|---|---|---|
| **por deriva** | el proveedor se movió desde el tip: alguien tocó el ítem del otro lado | traer lo que cambió y volver a intentar |
| **por regla** | lo que se pidió **no es una transición legal** en ese workflow | pedir otra, o cambiar el workflow |

El primero es el [compare-and-swap](sync.md#la-ventana-y-el-compare-and-swap) de siempre y se arregla solo con volver a sincronizar. El segundo **no se arregla reintentando**: reintentar una transición ilegal la vuelve a rechazar para siempre.

> **Un rechazo por regla no dice sólo que falló: dice cuáles sí se puede.**

Jira devuelve las transiciones disponibles para un issue, y usarlas es la diferencia entre un error y una respuesta. Es la misma preferencia que [el éxito se lee de la salida](sync.md#el-éxito-se-lee-de-la-salida-nunca-del-código-de-retorno): un mensaje que no permite decidir qué hacer después no informó nada.

#### Un rechazo por regla informa siempre, porque listar ya pasó

La primera forma de esto trataba el listado como un extra: se pedía **sólo tras un rechazo**, porque preguntarlo siempre era una llamada por ítem para un dato que casi nunca se usa. Y de ahí salía una degradación — si el listado fallaba, el rechazo decía *"no se pudieron listar"*, que es honesto y no sirve para nada.

**Con la transición pedida por id, ese orden se da vuelta.** El id sale de listar, así que para intentar la transición **ya hubo que listar**. Ver [`sync.md`](sync.md#y-pedir-por-id-es-lo-que-saca-una-ambigüedad-no-sólo-una-llamada).

| | antes | ahora |
|---|---|---|
| cuándo se lista | tras un rechazo | antes de intentar, siempre |
| qué cuesta | una llamada por rechazo | una llamada por ítem **que cambia de estado** |
| qué informa un rechazo por regla | las disponibles, **o nada** | las disponibles, siempre |

**El costo no creció como parece.** No es una llamada por ítem: es una por ítem cuyo estado cambió, que en un push típico son unos pocos. Lo que se pagaba antes por rechazo se paga ahora por cambio, y a cambio la degradación desaparece.

> **Y desaparece de verdad, no se vuelve improbable.** Si el listado falla, no hay id, así que no hay transición que intentar: el fracaso pasa a ser *"no se pudo preguntar"* antes de tocar nada, en vez de un rechazo que no supo explicarse.

## Qué se compara, y contra qué se compara

El compare-and-swap mira el `status` de **todas** las claves del tip. Hasta que existió el mapeo eso no podía correr contra Jira:

> El `status` del worklist y el de Jira **no son el mismo campo**, así que compararlos rechazaba **todas** las ventanas de entrada.

Por eso la instalación apuntaba al proveedor de prueba, cuyo archivo lleva valores con la forma del worklist. **Con el mapeo, la comparación es entre magnitudes comparables**: el `status` del tip se traduce al del proveedor y recién ahí se compara.

```
tip:        status: in-progress
mapeo:      in-progress -> "In Progress"
proveedor:  "In Progress"                  → coincide, el push entra
```

**Y la traducción va en esa dirección y no en la otra.** Traducir lo que el proveedor devuelve hacia el vocabulario del proyecto necesitaría el mapeo inverso, que no siempre existe: dos estados del proyecto pueden mapear al mismo status del proveedor —`dropped` y `done` lo hacen, y se distinguen por la resolución— y ahí el inverso no es una función.

## Invariantes

1. El `status` de un ítem es uno de los estados que `.metadata/states.yaml` declara. No hay estados implícitos.
2. Todo estado declarado tiene entrada en el mapeo de la instalación, o la instalación no se configura.
3. `worklist` no evalúa la legalidad de una transición. La propone y el proveedor decide.
4. Un rechazo informa cuál de las dos clases es, y uno por regla informa **siempre** qué transiciones sí están disponibles — no puede no saberlo, porque listarlas es lo que precede al intento.
5. La traducción de estados va del proyecto al proveedor. El mapeo no se usa al revés.
