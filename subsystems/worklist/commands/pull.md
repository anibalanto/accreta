# Comando: `worklist pull`

Pone una vista al día: **trae el corte de hoy y replanta encima lo que no se empujó.**

Sin esto el cliente no tiene una operación, tiene un `git push` que no alcanza. Cada push deja en el servidor commits que el que empujó no tiene —el `rename <slug> -> <clave>`, el `normalize:`, el `key` del sprint— y hasta acá eso se despachaba con un *"el cliente que empujó tiene que hacer `fetch` para ver los ids reales"*, como si `fetch` fuera una respuesta y no una instrucción a medias.

**Y medido el 2026-09-05:** traer las dieciséis ventanas al día fueron dieciséis invocaciones a mano, cada una un commit atrás, y **ninguna se veía pendiente** — `git status` decía *"limpio"* en las dieciséis.

## Firma

```
worklist pull [<vista>] [--all] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `<vista>` | Qué vista poner al día. Por defecto, aquella en la que se está parado. |
| `--all` | Todas las vistas del clon. **Sólo bajada** — ver § "`--all` es sólo bajada". |
| `--dry-run` | Dice qué traería y qué replantaría, sin mover ninguna rama. |

## Son tres pasos, y ningún mecanismo nuevo

```
worklist-server window open <n>       el corte de hoy, donde está el panorama
git fetch srv                         baja
git rebase srv/secure/sprint/<n>      lo que no se empujó, encima
```

Los tres existen. Lo que no existe es que sean **uno**, y que alguien los corra por las dieciséis.

**El paso 1 no es opcional y es el que más se olvida.** Sin él se baja el corte de la última vez que alguien recortó, que puede ser de hace tres semanas: la ventana queda al día con una foto vieja del panorama, que es la peor forma de estar al día — se ve idéntica a estarlo de verdad.

### Y no es opcional, pero es gratis cuando no hay nada

> **`pull` sobre una vista que no cambió no toca nada.** Ni el servidor, ni la rama, ni el worktree.

Es la otra mitad de que el paso 1 sea incondicional. Preguntar siempre está bien; **escribir siempre no**, y las dos puntas escribían de más por el mismo motivo — un commit lleva la hora adentro del hash, así que rehacer lo mismo produce un objeto distinto:

| | Escribía de más | Ahora |
|---|---|---|
| el paso 1 | recortaba en cada corrida, y la rama se movía para decir lo que ya decía | [`window open` compara el árbol](window-open.md#abrir-dos-veces-sobre-lo-mismo-no-hace-nada) y no escribe si es el mismo |
| el paso 3 | cherry-pickeaba mi trabajo sobre su propio padre, cambiándole el sha | si mi HEAD **ya contiene** la punta del servidor, estoy adelantado y no divergido: no hay nada que replantar |

**Estar adelantado no es estar divergido**, y confundirlos es lo que hacía que ponerse al día reescribiera trabajo que estaba bien donde estaba.

## Son dos actores, no dos comandos

Rearmar una vista son dos operaciones con dueños distintos, y confundirlas es lo que hacía difícil el comando:

| | Quién | Por qué él y no el otro |
|---|---|---|
| **calcular el corte de hoy** | el **servidor** | un recorte nuevo puede traer ítems que el cliente no tiene. Se calcula donde está el tronco, o no se calcula |
| **replantar lo mío encima** | el **cliente** | son los commits que no empujé: el servidor no los tiene, y por eso no puede replantarlos él |

> **`pull` es traer el corte de hoy y replantar encima lo que no se empujó.**

Eso vale para las dos clases de vista, y es lo que hace que el comando no dependa del modelo: **lo que cambia entre un artefacto y una vista dinámica no es quién hace cada mitad, es el transporte** — si el corte llega como una rama que ya está en el servidor, o como algo que baja una vez y no vuelve. Para lo que existe hoy —las ventanas de sprint, que son [ramas de las dos puntas](../concepts/sync.md#el-panorama-vive-en-un-solo-lado-y-la-ventana-en-los-dos)— el transporte es `fetch`, y por eso el paso 3 es literalmente un `rebase`.

**Es del cliente aunque su primer paso sea del servidor.** El [corte entre los dos](../concepts/distribution.md) ubica un comando por quién habla con el proveedor y quién es dueño del artefacto que se escribe, y `pull` escribe **la rama local y el worktree**: eso es del cliente. Que su paso 1 sea una invocación a [`window open`](window-open.md) no lo muda de lado, igual que `state change` no es del servidor por terminar en una transición del proveedor — **lo que decide el lado es quién escribe el resultado, no en qué se apoya para producirlo.**

### Y lo primero que hace el paso 3 es preguntar si hay algo que replantar

> **Si la vista subió entera, el corte nuevo la contiene por construcción.** No hay nada que replantar, y no hace falta mirar un solo commit.

Lo contesta [la marca del servidor](../concepts/propagation.md#hasta-dónde-subió-se-anota-en-una-ref-no-se-deduce), que es quien la anota — el mismo paso 1 que ya le pide el corte se la pregunta de paso.

**Y no se contesta con `refs/remotes/srv/**`**, aunque esté a mano y parezca lo mismo: esa ref dice *lo último que traje*, que es otra cosa, y **la mueve cualquier `fetch`** — incluido el de este propio comando en `--dry-run`. Una fuente que la consulta desplaza no puede contestar la consulta.

Tampoco se contesta por patch-id, que sería lo natural: medido el 2026-09-07 sobre las siete ventanas, **no reconoce ni uno solo** de sus commits, porque arriba el cuerpo quedó en forma canónica y abajo como se tipeó. Lo que sigue es para la vista que subió a medias.

## El paso 3 deja caer lo que ya fue superado

**Medido el 2026-09-07, al querer bajar un ítem recién agregado al `items` de la ventana 21: el recorte falló.** El cherry-pick de un commit del cliente choca contra el corte nuevo, y reproducido el merge de tres puntas las dos diferencias son de formato:

```
el corte nuevo (del panorama)   _"un nombre bajo 20"_     |---|---|---|
el commit del cliente           *"un nombre bajo 20"*     | --- | --- | --- |
```

Es la normalización del round-trip, que [`propagate`](propagate.md) **rehace** arriba. Así que el panorama guarda la vuelta y el commit guarda lo que se tipeó.

> **La vuelta del round-trip es la buena, y el que escribió se adapta.** No hay dos versiones que reconciliar: hay una superada y una vigente.

Por eso el paso 3 **no negocia**. Si el commit ya está contenido en el corte nuevo **módulo normalización**, se deja caer; se replanta sólo lo que el corte no tiene. Y se puede preguntar, porque la conversión converge en una pasada: normalizar las dos puntas y comparar contesta sí o no.

### Y el `.sprint.md` lo gana el corte

El segundo caso en que un choque no informa nada, y no es normalización.

> **`status` e `items` se editan arriba y bajan regenerando.** Así que el commit de la ventana arrastra como contexto un `items` que ya quedó viejo, y choca sobre algo que no estaba tratando de cambiar.

Medido sobre la ventana 21: un commit que pasaba el sprint de `open` a `in-progress` chocó contra un `items` que había crecido de 6 ítems a 17. El `status` que ese commit propone **ya está en el panorama**; lo que se deja caer es el commit, no el cambio.

Es [la asimetría de la clave del sprint](../concepts/propagation.md#y-el-tercero-es-la-clave-del-sprint-por-un-motivo-que-no-es-de-alcance-sino-de-autoría) leída para el otro lado, y por eso vale **sólo bajando**: hacia arriba las ediciones que la ventana le hace al `.sprint.md` sí viajan.

### Lo que queda es el conflicto que sí informa

**Un conflicto que quede después de descontar esos dos sí es el dato**: dice que el trabajo local toca un ítem que el `items` de hoy ya no lleva. Hasta acá el error decía **siempre** eso, incluso cuando la causa era una de las otras dos, y mandaba a mirar un `items` que no tenía nada que ver.

### Y la versión fuerte de lo mismo queda anotada

**Que el cliente normalice antes de commitear.** Ahí no hay nada que dejar caer, porque nunca existe una versión sin normalizar — es la que haría que esto no pudiera volver a pasar, en vez de detectarlo cada vez.

Cuesta que el cliente reescriba lo que la persona escribió sin que nadie se lo pida, así que **no entra acá**: se decide con la medición de este comando delante, que es la que dice cuántos commits se dejan caer por normalización y cuántos por otra cosa.

## `--all` es sólo bajada

Las dieciséis de una. **No hace falta ninguna atomicidad entre ellas**: son dieciséis operaciones independientes, y que una conflictúe no dice nada de las otras.

**Lo que sí hace falta es que la salida diga cuáles quedaron a medias**, porque el modo de falla de esto es exactamente que no se note — es el mismo defecto de forma que la vista atrasada que se ve limpia, un nivel más arriba.

## Empujar es una invariante, y por eso no necesita comando

Esta task pedía además un comando que empujara **todas** las vistas con atomicidad entre ellas, que probablemente prohibiera `git push`, y que avisara por las que no verificó. Nada de eso hace falta, y no porque se descarte: se cae solo, contra [la invariante de sólo edición](../concepts/sync.md#una-vista-sólo-puede-aportar-ediciones).

| Lo que se cae | Por qué |
|---|---|
| **empujar todas las vistas con atomicidad** | se apoyaba en que mover un ítem entre vistas fuera un borrado allá y un agregado acá. Eso es cierto mientras la membresía sea *"el archivo está en esta rama"*, y la composición la muda a un campo: mover pasa a ser **una edición en un lugar** |
| **pedir verificación de una vista y avisar por las demás** | la asimetría existía porque el comando empujaba siete y preguntaba por una. **Si el comando es de la vista donde estoy parado, no hay asimetría que administrar**: una y una |
| **prohibir `git push`** | existía para proteger la atomicidad. Sin atomicidad que proteger, prohibir el canal barato no compra nada |

### Y el chequeo previo no puede ser local

La tentación es que el push mire antes de subir: sacar el commit de corte, probar si las ediciones aplican arriba, y avisar. **Ese chequeo ya existe y ya corre — en el servidor.** El `pre-receive` prueba el cherry-pick contra el panorama con `merge-tree`, en memoria, en todos los pushes; si no aplica, [rechaza](check-push.md). Y el compare-and-swap rechaza además por deriva del proveedor. Las dos vuelven con las diferencias.

Hacerlo del lado del cliente necesitaría el panorama para probar contra qué, y el cliente [no lo tiene](../concepts/sync.md#el-panorama-vive-en-un-solo-lado-y-la-ventana-en-los-dos).

> **Un chequeo previo contra una copia del panorama es un verde falso**, y es exactamente la clase de error que sacar el panorama del cliente evitó: leer una copia que se quedó vieja y actuar como si fuera la verdad. Allá escribía en el proveedor; acá diría *"subí tranquilo"*.

Así que el bucle es el de git: **`push` rechazado → `pull` → resolver → `push`.**

### Pero *"se explica solo"* era falso, y por eso `push` existe igual

Acá decía que el bucle *"se explica solo"*, y medido el 2026-09-07 **ni siquiera arranca**: las vistas nacen sin upstream —`git worktree add` no lo configura— así que `git push` a secas falla, y `git status` no tiene contra qué compararse y nunca dice *"ahead by 1"*.

> **Se confundió *"no necesita garantías propias"* con *"no necesita comando"*.**

Las tres invariantes de la tabla de arriba se quedan enteras: no hace falta atomicidad, el chequeo previo no puede ser local, y prohibir `git push` no compra nada. Lo que se cae es la conclusión. [`worklist push`](push.md) no agrega ninguna garantía — sabe el refspec, configura el upstream la primera vez, nombra lo que el servidor escribió encima, y traduce el rechazo a `worklist pull`.

## Lo que no promete, dicho

**No le pregunta a Jira.** La verificación contra el proveedor llega en el `push`, que es cuando importa porque es cuando se escribe; y para preguntarla sin escribir ya existe [`check-push --dry-run`](check-push.md#comparar-sin-rechazar). Meterla adentro de `pull` le pone el costo caro al comando barato.

**Y si algún día la lleva, no antes de que el round-trip cierre**: hoy 113 de 149 cuerpos que difieren son del conversor y no deriva real, así que un `pull` que los reportara le mostraría al que lo corre una pila de diferencias que no existen.

**Y el paso 1 supone que el servidor está a mano.** Hoy el bare está en la misma máquina, así que `pull` lo puede invocar. El día que sea un GitLab, este comando necesita el canal cliente→servidor que hoy no existe — el mismo que ya está anotado en [`concepts/distribution.md`](../concepts/distribution.md#lo-que-el-cliente-necesita-del-proveedor-se-lo-pide-al-servidor). No es una razón para no hacerlo: es la línea que la spec tiene que decir.

## Salida

El caso sano no tiene nada que contar, y por eso es una línea:

```
$ worklist pull
secure/sprint/21: al dia
```

La vista que subió a medias sí, porque ahí hubo decisiones:

```
$ worklist pull
  2 commit(s) replantado(s)
  dejado caer: 980d772 ACC-304: el nombre del sprint… (ya esta en el corte modulo normalizacion)
  dejado caer: a680093 el sprint 21 arranca (la planificacion del sprint es del panorama)
secure/sprint/21: al dia
```

Y `--dry-run` dice lo que pasaría, **sin `fetch`**: la punta de hoy se lee del servidor, que la tiene. Un dry-run que mueve refs no es un dry-run.

```
$ worklist pull --dry-run
secure/sprint/21: 0 commit(s) sin empujar; la punta de hoy es 9cb59d6 y el corte se recalcularia
```

**No dice *"al día"***, que sería afirmar algo que esta corrida no hizo.

Y las dieciséis, donde lo que importa es la última línea:

```
$ worklist pull --all
secure/sprint/19: al día
secure/sprint/20: al día
secure/sprint/21: al día
…
16 vistas, 15 al día, 1 a medias:
  secure/sprint/13: conflicto al replantar 4c1e9a2 — ACC-88.task.md
    el trabajo toca un item que el `items` de hoy ya no lleva.
```

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | la vista quedó al día |
| `0` | `--dry-run`, replante limpio o no |
| `1` | el replante conflictúa — la vista queda como estaba, y la salida dice en qué archivo |
| `1` | con `--all`, al menos una quedó a medias |

**Con `--all` una vista que conflictúa no detiene a las demás**: las quince siguen, y el código de retorno dice que hubo una. Es la otra mitad de que no haya atomicidad entre ellas — si son independientes para avanzar, lo son también para fallar.
