# Draft: el desacuerdo como una entrada más

**Estado:** borrador. La forma está propuesta y nada está decidido — salió de una conversación de diseño y no de un caso corrido.

`agree` es el único vocabulario que el formato tiene sobre personas. Quien mira un fragmento y no está de acuerdo **no escribe nada**, y nada es exactamente lo que escribe quien no lo miró.

> **El silencio tiene dos causas y una sola representación.**

Y es el hueco que le saca sentido a [`4r`](../../../.stratum/worklist-accreta/4r.task.md): un estado que reporta *"falta que firme quien tenía que firmar"* no puede distinguir al que todavía no llegó del que llegó y dijo que no.

## La mitad que ya existe, y hay que verla antes de agregar nada

**Una contrapropuesta ya se escribe.** Es una segunda entrada con otros valores, y `CONSENSUS_DIVERGED` es el formato sosteniendo dos posiciones sin elegir ninguna. Eso ya está hecho, y está hecho por la afirmativa.

Lo que falta es la posición que **no propone otros valores**: la que rechaza éstos y no tiene una versión alternativa que aprobar.

Y no se resuelve con un veto. Un `no` que bloquea cierra la conversación y pone al formato a arbitrar entre personas, que es lo que [`bilink.md`](../concepts/bilink.md#más-de-un-accepted-es-un-estado-no-una-forma-de-trabajar) evita al decir que *"de qué lado está el desacuerdo es de las personas, no del código"*.

## La forma

Una entrada más en la lista, con **la misma tupla** y el signo contrario:

```yaml
endpoint:
  0:
    link: capture 7287f233dc696b13e5af79affc0fbc68
    n:
      1:
        link: capture f33f7641452a6039a607bff32ec8c264
    accepted:
    - agree:
      - Anibal
      - Pablo
      link: capture 7287f233dc696b13e5af79affc0fbc68
      hash: 9c040e13…
      hash_ast: c6420b70…
      n:
        1:
          link: capture f33f7641452a6039a607bff32ec8c264
          hash: a0b0ae96…
          hash_ast: ca5f4384…

    - disagree:
      - Ignacio: Rompe el contrato con el consumidor de la capa de arriba.
      link: capture 7287f233dc696b13e5af79affc0fbc68
      hash: 9c040e13…
      hash_ast: c6420b70…
      n:
        1:
          link: capture f33f7641452a6039a607bff32ec8c264
          hash: a0b0ae96…
          hash_ast: ca5f4384…
```

Ignacio no propuso otro contenido: rechazó **este**, y para poder decir cuál lo escribe entero.

## Por qué una entrada y no un campo

La alternativa —un `remark:` colgado de la entrada aprobada— falla por dos motivos, y los dos son del formato y no de gusto:

**La objeción viviría adentro de lo que objeta.** Y se iría con ella: `accept` colapsa las entradas cuyos valores difieren, así que la primera aceptación nueva se llevaría puesta la objeción sin que nadie la conteste.

**Y una posición es sobre *estos* valores.** [`accept.md`](../concepts/accept.md#quiénes-aprobaron) ya lo dice de `agree` —*"la identidad de una entrada es su tupla entera"*— y vale igual para el rechazo: si el fragmento cambia, Ignacio no rechazó lo nuevo, igual que Pablo no lo aprobó. Escribir la tupla entera es lo que hace que eso sea verdad por construcción y no por una regla aparte.

De ahí sale el mejor argumento a favor: **el colapso no necesita ninguna regla nueva.** `accept` se queda con la entrada de los valores que se están aprobando y las demás se van — y un rechazo de valores viejos se va con los valores viejos, sin caso especial.

## Lo que redefine, y hay que enunciarlo

Hoy dos entradas con la misma tupla **no pueden existir**: `accept` une al que aprueba con la entrada que ya tiene esos valores. Con esto, la identidad de una entrada pasa a ser **la tupla más su signo**.

> Una entrada no es *"un conjunto de valores"*: es **una posición sobre un conjunto de valores.** Dos posiciones opuestas sobre la misma tupla son dos entradas, no una duplicada.

La unión sigue existiendo y se vuelve por signo: quien aprueba se suma al `agree` de la entrada que coincide, quien rechaza se suma al `disagree` de la que coincide.

## El estado, y el agujero que abre

**`CONSENSUS_DIVERGED` no cambia de nombre, cambia de definición**: de *"hay más de un `accepted`"* a *"hay más de una posición"*. Con eso cubre lo de siempre —dos personas que aprobaron valores distintos— y además lo que hoy no se puede escribir: uno dijo que sí y otro que no. Ningún token nuevo.

**Y hay una regla que se rompe si no se toca.** [`bilink.md`](../concepts/bilink.md#más-de-un-accepted-es-un-estado-no-una-forma-de-trabajar) dice que *"un endpoint sólo puede estar `OK` con exactamente una entrada"*. Una entrada sola de `disagree` cumple eso al pie de la letra —una entrada, hashes que coinciden— y **saldría `OK` siendo un rechazo sin contestar**. La regla pasa a ser:

> **`OK` pide exactamente una entrada, y que esa entrada sea un `agree`.**

Es el mismo espíritu que el resto del formato: fallar hacia reportar. Un fragmento que alguien miró y rechazó, y que nadie más aprobó, no es un fragmento sobre el que no hay nada que hacer.

## Qué no es: la discusión sigue afuera

[`discutir-una-aceptacion.md`](discutir-una-aceptacion.md) contesta dónde se discute una decisión, y contesta que en los threads de impact — porque *"una discusión es mutable por naturaleza y la ref es inmutable por diseño"*. Nada de eso cambia.

**Una posición no es una discusión.** Es una decisión, tiene autor, es inmutable una vez escrita, y las decisiones viven en el bilink. Lo que la nota de al lado lleva es **el motivo**, no el hilo: una línea que dice por qué, atribuible por `git blame` como cualquier endoso. El ida y vuelta sigue siendo de impact.

Lo que sí hace es **retirarle su pregunta abierta 2**. Esa propuesta observaba que objetar *antes* de que exista la aceptación no tiene ancla, porque el ancla era el commit. Acá la objeción **es su propia ancla**: se escribe sin que exista ninguna aceptación previa, y produce su propio commit firmado.

## Lo que queda por decidir

**El nombre del campo.** `no-agree` es lo que se escribió primero y es el único campo del formato con guión. `disagree` es una palabra y no lo necesita.

**Que una entrada lleve exactamente uno de los dos.** Con dos campos hermanos queda escribible una entrada con los dos, que no quiere decir nada — y si a alguien le parece que sí quiere decir algo, eso ya son dos entradas. Lo tiene que rechazar el tipo, con el precedente de *"`accepted` sin `hash` es rechazado"*.

**De quién es la nota.** Escrita al nivel de la entrada, dos personas que rechazan comparten un bloque y **se pierde la atribución por línea**, que es toda la razón por la que `agree` es bloque y no flow. Por eso el ejemplo la pone colgada del nombre. Falta decidir si `agree` toma la misma forma —*"apruebo, y además tengo algo que decir"* es un caso real— o si quedan asimétricos.

**Si rechazar necesita el vecindario resuelto.** Por simetría sí: se rechaza una tupla, y la tupla incluye `n`. El costo es que decir que no **necesita el proveedor**, y con la cache fría eso son los 25 segundos que el sprint [`10`](../../../.stratum/worklist-accreta/_sprints/10.sprint.md) está midiendo. Es el punto donde esta propuesta y ese sprint se tocan.

**Con qué código de salida.** Un endpoint con un rechazo sin contestar no está `OK`, así que 1 — con el mismo riesgo que [`4r`](../../../.stratum/worklist-accreta/4r.task.md) ya nombra: pone a fallar el CI, que es el punto y también el costo.

**Qué hace `adopt`.** Une campo por campo y ya une `agree` sin preguntarle a nadie. Unir `disagree` es la misma operación, pero trae al repo de uno el rechazo de alguien de otro equipo — y si eso cruza la frontera es la misma pregunta abierta 4 de la otra propuesta, sobre otro objeto.

## Qué haría falta antes de decidir

Un desacuerdo real escrito a mano en un bilink de `hsi`, para ver si la tupla repetida molesta al leer el archivo. Es la única parte de esto que no se puede razonar de antemano.
