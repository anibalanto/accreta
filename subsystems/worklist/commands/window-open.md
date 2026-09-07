# Comando: `worklist-server window open`

Recorta una ventana: produce la rama `secure/sprint/<id>` con **sólo los ítems de ese sprint y sus ancestros**, cortada desde el panorama.

Sin esto, una rama segura no puede prometer lo que su nombre dice. Nacida de un `git checkout -b` sobre `insecure/all`, se trae todos los ítems, y *"me hago responsable de verificarme entera"* deja de ser sostenible — ver [`concepts/sync.md`](../concepts/sync.md#dos-clases-de-rama-y-el-nombre-dice-qué-se-puede-hacer).

**Es del servidor**, y por los dos criterios a la vez: lee el panorama, que [vive de un solo lado](../concepts/sync.md#el-panorama-vive-en-un-solo-lado-y-la-ventana-en-los-dos), y escribe la rama de la ventana, que es un artefacto. Ver [el corte entre el cliente y el servidor](../concepts/distribution.md#y-window-open-cambió-de-lado).

## Firma

```
worklist-server window open <sprint-id> [--from <rama>] [--dry-run] [--force]
```

| Argumento | Descripción |
|---|---|
| `<sprint-id>` | El id del sprint: recorta `_sprints/<id>.sprint.md` y lo que declara. |
| `--from` | De dónde cortar. Por defecto `insecure/all`, el panorama — que existe sólo acá. |
| `--dry-run` | Lista qué entraría, sin escribir la rama. |
| `--force` | No replanta: el corte nuevo reemplaza a la ventana, **descartando** lo que tenía encima. |

## Qué entra

1. **Los ítems que el `items` del sprint declara**, y para cada uno **todo su subárbol** — los hijos van con el padre, porque [lo que entra a un sprint es un subárbol entero](../concepts/hierarchy.md#la-regla-del-ancestro).

   El `items` sale de [`.metadata/product.yaml`](../concepts/composition.md), que está en el panorama de donde se corta. **No entra a la ventana**: es del servidor, y una copia de la composición del lado del cliente es una fuente de verdad que sólo puede quedarse vieja — el mismo motivo por el que el panorama tampoco baja.
3. **Los ancestros de cada uno, de sólo lectura** — en la práctica la épica, que [no entra a un sprint](../concepts/hierarchy.md#épicas) y viaja para que la cadena `parent` cierre adentro de la ventana.
4. **El vocabulario de estados**, `.metadata/states.yaml`, si el proyecto lo declara — por lo mismo que la épica: [la ventana tiene que cerrar adentro](../concepts/states.md#y-el-vocabulario-viaja-con-el-recorte), y el cliente ya no tiene el panorama de donde leerlo.

Y nada más. **Lo que no es de la ventana no está**, que es el punto: parado adentro, un ítem ajeno no puede confundirse con uno tuyo, porque no está.

## Para leer algo que no es de la ventana

El panorama lo tiene todo, y está donde este comando corre:

```
git -C <bare> show insecure/all:ACC-3.task.md
```

Traerlo a la ventana sería ensanchar el conjunto que la ventana promete verificar — exactamente lo que el recorte evita.

## La rama nace acá, y el cliente la trae

El comando escribe `refs/heads/secure/sprint/<id>` en el repo del servidor. Del otro lado son dos comandos de git y ninguno del worklist:

```
git fetch srv
git worktree add .worklist/secure/sprint/10 secure/sprint/10
```

> **La ventana baja; el panorama no.** Es la asimetría que separa un artefacto de un tronco, y acá es lo único que el cliente necesita saber.

## Salida

```
$ worklist-server window open 10
secure/sprint/10: 5 archivo(s)
  _sprints/10.sprint.md
  ACC-2.task.md        ← de items
  ACC-3.task.md        ← subárbol de ACC-2
  ACC-9.task.md        ← de items
  ACC-1.epic.md        ← ancestro, sólo lectura
recortado desde insecure/all (a1b2c3d) -> 9z8y7x6
```

## Recortar sobre lo mismo no produce un corte nuevo

> **El corte es derivado. Si re-derivarlo da el mismo árbol, el que ya está sigue siendo válido y la rama no se mueve.**

No es una optimización, y hay que decir por qué: **recortar escribe un commit, y un commit lleva la hora adentro del hash.** Así que recortar igual produce un objeto nuevo para decir lo que ya decía, y todo el que tenga la rama clonada se queda sin poder fast-forwardear — por nada. Medido el 2026-09-07, tres recortes seguidos sobre un panorama quieto:

```
66a671d  padre=4ed96b0  tree=62b15c4   08:26
9e4aba8  padre=4ed96b0  tree=62b15c4   08:34
```

Mismo padre, mismo árbol, dos shas. **Son el mismo corte con otra lectura del reloj.**

Y pasó a importar cuando [`pull`](pull.md) empezó a recortar en cada invocación: mientras recortar era algo que alguien hacía a mano cada tanto, reescribir la rama de más era un accidente raro; con un comando que recorta cada vez que alguien se pone al día, sería lo normal.

La pregunta se hace en dos pasos, del barato al exacto:

| | |
|---|---|
| **el panorama no se movió** desde que se hizo el corte | no hay de dónde salga una diferencia — ni el `items`, que vive arriba. Se contesta sin construir nada |
| **se movió, pero no para esta ventana** | cualquier push a cualquiera de las otras adelanta el panorama. Se compara **el árbol** del recorte nuevo contra el del corte que está, que es lo que el corte significa |

El sha no sirve para esto, porque lleva la hora y el padre adentro; el árbol es exactamente *"qué archivos, con qué contenido"*, que es lo que el recorte decide.

## Abrir dos veces regenera: el corte se recalcula y el trabajo se replanta

**Todo lo que sigue es para cuando sí cambió algo** — si no cambió nada, la sección de arriba ya contestó y no se llega hasta acá.

> **Recortar de nuevo no reemplaza la ventana: la pone al día.**

Es lo que decidió [la propagación](../concepts/propagation.md#hacia-abajo-regenerar-y-el-rebase-es-sólo-el-rescate). El comando hace dos cosas y en este orden:

1. **Recalcula el corte** contra el `items` de hoy, desde el panorama de hoy. No re-aplica el commit de corte anterior, que borra una lista fija de rutas y por eso dejaría entrar todo lo que el panorama ganó desde entonces.
2. **Replanta encima** lo que la ventana tenía sobre su corte viejo: lo que el servidor escribió —los `rename`, los `normalize:`— y lo que alguien haya editado adentro.

```
antes:    all(viejo) ─▶ corte(viejo) ─▶ W1 ─▶ W2
después:  all(nuevo) ─▶ corte(nuevo) ─▶ W1' ─▶ W2'
```

**Y en el caso sano el replante queda vacío solo.** Si `W1` y `W2` ya subieron al panorama, el corte nuevo los contiene y el cherry-pick no aporta nada, así que se deja caer. Nadie tiene que acordarse de propagar antes de regenerar.

### Sin corte no regenera, y se niega

El corte es lo que dice dónde empieza el trabajo de la ventana. Una rama `secure/**` nacida de un `checkout -b` no lo tiene, y sin él no hay forma de separar su trabajo del árbol que arrastró:

```
$ worklist-server window open 7
error: no encuentro el corte de esta ventana: el primer commit sobre el panorama es
  a2b035d edito ACC-93
```

### Y `--force` baja de categoría

Ya no es el flujo del `items` que cambió —eso ahora es regenerar y ya—: **es tirar lo local a sabiendas** cuando el replante no entra.

```
$ worklist-server window open 7
error: el corte nuevo esta, pero un commit de la ventana no se pudo replantar:
  a2b035d edito ACC-93
  choca en: ACC-93.task.md

  pasa cuando el trabajo toca un item que el `items` de hoy ya no lleva.
  Para tirarlo a sabiendas: --force
```

**El motivo del choque es siempre el mismo**, y por eso el mensaje lo nombra: el trabajo toca un ítem que el `items` de hoy no lleva, así que el corte nuevo no lo tiene y el parche no encuentra dónde apoyarse. Con `--force` el replante no corre, y la salida dice cuántos commits quedan afuera.

**Y había un caso que el mensaje diagnosticaba mal**, medido el 2026-09-07: el commit chocaba por la normalización del round-trip —el panorama guarda la vuelta, el commit guarda lo que se tipeó— y ahí el `items` no tenía nada que ver. **Un diagnóstico falso es peor que ninguno**: manda a mirar un `items` que está bien.

Un commit ya contenido en el corte **módulo normalización** se deja caer antes de llamarlo conflicto, y se dice:

```
  dejado caer: 980d772 — ya esta en el corte modulo normalizacion
```

Es la misma decisión que el caso vacío —lo que la ventana hizo ya está arriba— con la diferencia de que git no la puede tomar solo: el patch-id compara bytes. Ver [`commands/pull.md`](pull.md#el-paso-3-deja-caer-lo-que-ya-fue-superado), que es donde está el argumento, y [`concepts/propagation.md`](../concepts/propagation.md#salvo-que-primero-hay-que-descontar-la-normalización-y-eso-no-es-un-conflicto).

**Y lo mismo con el `.sprint.md`**, por otro motivo: `status` e `items` se editan en el panorama y bajan regenerando, así que un choque que cae entero sobre `_sprints/**` lo gana el corte.

```
  dejado caer: a680093 — la planificacion del sprint es del panorama
```

### Y no se mueve una rama que alguien tiene abierta

`git branch -f` **se niega** a mover una rama con worktree activo. `update-ref` no: la mueve, y deja ese worktree con el índice del árbol anterior.

El síntoma no dice la causa. `git status` muestra archivos *"modificados"* que nadie tocó, y un `git merge` se niega a seguir por cambios locales que no existen. Costó una vuelta entera de diagnóstico averiguar que el trabajo perdido no era trabajo, sino un índice desactualizado.

```
$ worklist-server window open 10
error: secure/sprint/10 está checkouteada en un worktree y no se puede mover:
  /home/…/.worklist/secure/sprint/10

  moverla dejaría ese worktree con el índice del árbol anterior. Sacá el
  worktree primero.
```

**Ni con `--force`**: forzar es para descartar commits a sabiendas, no para dejar un checkout inconsistente. Son dos cosas distintas y el flag sólo autoriza una. Y el chequeo corre **antes de escribir nada**, para que un error no deje un corte a medio hacer.

#### Pero el worktree del cliente no se ve desde acá

El chequeo mira los worktrees **del repo donde el comando corre**, y desde que el comando es del servidor eso es un bare: ahí normalmente no hay ninguno, así que el error de arriba deja de dispararse por el caso para el que se escribió. **El worktree que alguien tiene abierto está en otro repo, y ningún chequeo local lo puede ver.**

Lo que queda no es un agujero nuevo, es lo de siempre con una rama reescrita: regenerar cambia la historia de la ventana, así que el clon que la tenga no va a poder fast-forwardear. Se pone al día como cualquier rama replantada, **y con un comando que se niega en vez de descartar**:

```
worklist pull
```

Que es este comando y las dos mitades del cliente en uno solo — ver [`commands/pull.md`](pull.md). El rebase que hace adentro saltea solo lo que el corte nuevo ya trae —el servidor replantó todo lo que le habían empujado—, así que lo que queda encima es exactamente lo que el clon tenía sin empujar. **Que se pare en un conflicto es el dato**, igual que antes lo era el error: el trabajo local toca un ítem que el `items` de hoy ya no lleva.

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | la ventana quedó recortada |
| `1` | el sprint no existe en `--from`, o su `items` nombra algo que no está |
| `1` | la rama existe y no tiene corte, o un commit no se pudo replantar — salvo `--force` |
| `1` | la rama está checkouteada en un worktree — tampoco con `--force` |
