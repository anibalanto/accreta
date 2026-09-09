---
name: decision-records
description: "Una decisión de diseño no se escribe en el cuerpo del ítem: va en un documento tipado adentro del directorio del ítem, `<id>/files/@<slug>.<tipo>.md`. Cargar al tomar o registrar una decisión de diseño, al escribir un ADR, un DDR o un spike, y al cerrar un ítem que produjo una decisión escrita."
---

Dónde va una decisión de diseño, cómo se nombra el documento, y qué queda en el ítem.

La redacción del ítem —el título, los criterios de cierre, las referencias— está en la skill `item-writing`. La mecánica del worklist está en `worklist`. Acá va sólo el documento de decisión.

## La regla

> **Una decisión de diseño no se escribe en el cuerpo del ítem.** Va en un documento aparte, adentro del directorio del ítem.

```
ACC-9888.task.md                          el ítem: qué hay que hacer y contra qué se mide
ACC-9888/
  files/
    @separar-absorber-de-decidir.adr.md   la decisión, y por qué
```

El ítem dice qué tiene que ser cierto cuando termine. El documento dice por qué se decidió así, qué se descartó y con qué argumento. Son dos lecturas con dos públicos: el ítem lo lee quien va a trabajar, el documento lo lee quien pregunta por qué esto es así.

Y el motivo no es orden: un ítem no se relee si releerlo cuesta lo que cuesta releer una spec, y el que no se relee conserva premisas muertas. Sacar la deliberación del ítem es lo que lo deja corto para poder revisarlo.

## El nombre

```
@<slug>.<tipo>.md
```

**El `@` dice que existe sólo de este lado.** Es la misma marca que lleva un ítem que todavía no cruzó al proveedor, con el mismo significado exacto: el proveedor no lo nombra. Un documento en `files/` no tiene identidad del otro lado y no va a tenerla.

**Y lo pierde cuando aterriza.** Igual que un ítem cambia `@<slug>` por su clave el día que cruza, un documento cambia su nombre el día que se muda a la capa que lo gobierna: pasa a `docs/adr/0007-<slug>.md`, con el número que la capa le asigna ahí. El `@` no sobrevive a la mudanza porque su razón de ser era que el documento no estaba en ningún lado todavía.

**El tipo va en el nombre**, con la misma forma que el ítem —`<id>.<tipo>.md`— y por la misma razón: el tipo lo dice la extensión y no un campo, así que un `ls` alcanza.

| Tipo | Qué es | Qué tiene que contestar |
|---|---|---|
| `adr` | una decisión de **arquitectura**: cambia lo que otra cosa tiene que cumplir | qué se decidió, qué alternativas se descartaron, con qué argumento, y qué queda atado a ella |
| `ddr` | una decisión registrada que **no** es un cambio de arquitectura | lo mismo, y además por qué no toca la arquitectura — si toca, es un ADR |
| `spike` | una exploración con plazo: produce un hallazgo, no una decisión | qué se preguntó, qué se midió, qué se descubrió — y qué decisión queda habilitada, que se toma aparte |

El vocabulario **se deja crecer**, no se inventa por adelantado. Es la misma regla que el vocabulario de categorías de un título diagnóstico: un tipo nuevo aparece cuando hace falta, y lo que hay que escribir en ese momento es qué tiene que contestar — la fila de la tabla, no un permiso.

### Cómo se distingue un ADR de un DDR

> **La pregunta es si algo más tiene que cambiar para que la decisión valga.**

Si la respuesta es sí, es arquitectura y es un `adr`. Si la decisión se puede aplicar sin que ninguna otra pieza tenga que enterarse, es un `ddr`.

| | |
|---|---|
| `adr` | mueve el corte cliente/servidor · cambia qué promete una capa · cambia qué gobierna un bilink · agrega o saca una dependencia entre subsistemas · cambia de dónde sale un dato que otro lee |
| `ddr` | fija un nombre, un formato, un valor por defecto, el orden de dos pasos que no se bloquean, cuál de dos herramientas equivalentes se usa |

Dos ejemplos del proyecto, para calibrar: *"la composición vive en el panorama y no baja al cliente"* es un `adr` —el corte de vistas, el recorte y el cálculo del backlog pasan de lado—; *"el nombre de un sprint es su número y su título en minúscula, entero y sin tope"* es un `ddr`: cambia cómo se escribe un nombre y nada más tiene que enterarse.

Y el tipo **no lo decide el tamaño**. Un `ddr` puede tocar doscientos archivos —un renombre lo hace— y sigue sin cambiar qué tiene que cumplir nadie. Un `adr` puede ser una línea.

**Ante la duda, es un `adr`.** Equivocarse hacia ese lado deja un documento con más contexto del que hacía falta; hacia el otro deja una decisión de arquitectura sin registrar por qué las alternativas se descartaron, que es lo que después nadie puede reconstruir.

## La ubicación es el estado

> Un documento en `files/` **no está decidido**. Uno que está en la capa que lo gobierna, sí.

No hay campo que lo diga, y eso es deliberado: un archivo está en un solo lugar, así que no hay dos fuentes que puedan discrepar. Es la misma forma que la ausencia de `accepted` en un bilink, que *es* el estado pendiente.

```
ACC-9888/files/@separar-absorber-de-decidir.adr.md      no está listo
subsystems/worklist>impl/docs/adr/0007-….md             está listo, porque está acá
```

Tres cosas caen de esa regla:

**El `Estado:` del encabezado de un ADR no hace falta.** Un ADR propuesto no está en `docs/adr/` todavía. Aplica a lo nuevo: los que ya tienen el campo se quedan como están, igual que los identificadores en castellano.

**El número no se quema.** Se asigna cuando el archivo entra a la capa, así que un borrador abandonado no deja un hueco en la numeración ni un ADR fantasma en estado descartado.

**Y el ítem registra dónde aterrizó, una sola vez y al cerrar.** Mientras está abierto no hay campo: el borrador está en `files/` y lo encuentra un `ls`.

```yaml
artifact: subsystems/worklist>impl/docs/adr/0007-separar-absorber-de-decidir.md
```

Es opcional, porque un ítem puede cerrar sin producir documento. Su presencia significa que esto produjo una decisión escrita y está ahí.

## Dónde aterriza

| La decisión es de | Va a |
|---|---|
| la implementación de un subsistema | `docs/adr/` de su capa impl — `subsystems/<nombre>>impl/docs/adr/` |
| el método, el formato, o algo que cruza subsistemas | `docs/adr/` de la raíz de accreta |

El segundo destino está decidido y todavía no abierto. Mientras no exista, una decisión de esa clase se queda en `files/` y eso no es un olvido: es que no tiene dónde ir.

Y el path se compone con tokens Stratum, nunca a mano — ver la skill `stratum-paths`.

## Qué queda en el ítem y qué se va

| Va en el ítem | Va en el documento |
|---|---|
| el diagnóstico con el que abre el cuerpo | por qué se decidió así |
| qué tiene que ser cierto cuando termine | qué alternativas se descartaron, y con qué argumento |
| contra qué se mide | qué premisa cambió, y cuándo |
| de qué depende, en `relation.depends` | qué queda atado a la decisión |

**Y la arqueología se muda, no se borra.** Un ítem que dice *"acá decía X, y el argumento era bueno y la conclusión no"* está llevando el registro de la deliberación adentro del enunciado del trabajo. Ese párrafo tiene su lugar y es el documento — y la prosa del sprint, para lo que es de la iteración y no de una decisión.

## Cómo se cita

**El documento nombra al ítem sólo mientras está en `files/`**, y no hace falta escribirlo: el directorio ya es el ítem. Un `@<slug>.adr.md` adentro de `ACC-9888/` no necesita decir de qué ítem es.

**Y una vez que aterrizó, no cita ningún ítem.** Es la regla que ya vale para las specs: el id de un ítem cambia por diseño cuando cruza al proveedor, así que una cita desde un archivo de otra capa queda apuntando a algo que no existe, y nada lo detecta. La referencia va al revés — el ítem apunta al documento con `artifact:`.

**Lo que sí puede citar el documento es otro documento**, y ahí conviene un bilink en vez de un link de markdown: un ADR que gobierna un fragmento se bilinkea, y eso es lo que hace que el fragmento se entere cuando la decisión cambia. Ver la skill `bilinker`.

## Lo que todavía no está decidido

**Quién mueve el archivo.** A mano, o un comando que lo haga junto con el cierre del ítem. Si es a mano, cerrar un ítem y olvidarse de mudar el documento deja un borrador en el `files/` de algo cerrado — un estado que la regla no contempla.

**Si el corte de una ventana se lleva `<id>/` de cada ítem que entra.** Debería, por el mismo argumento que hizo viajar el vocabulario de estados: la ventana tiene que cerrar adentro, y una decisión sin su documento no se puede revisar. Mientras no esté probado sobre una ventana real, no darlo por hecho.

**Y si un `ddr` aterriza en el mismo `docs/adr/` que un `adr`.** El directorio se llama por uno de los dos tipos, así que o se renombra, o los dos conviven ahí, o un `ddr` aterriza en otro lado. Hoy no está dicho, y es la primera cosa que va a hacer falta el día que se escriba el primero.
