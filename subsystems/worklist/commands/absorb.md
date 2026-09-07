# Comando: `worklist-server absorb`

Escribe en la rama de la ventana lo que el proveedor dice y el tip no.

> **Es la única dirección que faltaba.** [`push-states`](push-states.md) sube el `status`, [`propagate`](propagate.md) sube lo que la ventana resolvió, [`reconcile`](reconcile.md) adopta issues que ya existen del otro lado. Un cambio hecho **en el board** no tenía por dónde entrar.

Y no es que se perdiera: se **detectaba**, en el peor momento. [`check-push`](check-push.md) rechaza el push por deriva, así que uno se enteraba con el trabajo ya hecho — y el rechazo no tiene salida asistida, porque bajar no lo arregla.

## Es del servidor, y por eso `pull` puede traerlo

Los dos criterios del [corte](../concepts/distribution.md) dan lo mismo: **habla con el proveedor** y **escribe en una rama del servidor**.

Y esa es justamente la propiedad que hace que el cliente no cambie:

```
absorb  ──escribe en la rama de la ventana──▶  propagate  ──▶  el panorama
                                                                    │
                                          el recorte lo incluye ◀───┘
                                                    │
                                          worklist pull lo baja
```

> **Ningún mecanismo nuevo.** `pull` trae lo que el servidor escribió, y esto es una cosa más que el servidor escribe.

[`pull`](pull.md) lo invoca en su paso 0, igual que ya invoca [`window open`](window-open.md). Con eso *"poner la vista al día"* pasa a incluir el proveedor **sin que el cliente pida una credencial**.

## Firma

```
worklist-server absorb --ref <rama> (--provider-file <archivo> | --project <clave> [--base <url>] [--account <email>])
                       [--states-map <archivo>] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--ref` | La ventana. Se leen las claves de su tip. |
| `--dry-run` | Dice qué absorbería, sin escribir. |

El resto es el mismo reparto que [`check-push`](check-push.md), y por la misma razón: **lo que absorbe tiene que ser exactamente lo que el otro iba a rechazar.** Dos formas de preguntar lo mismo se separan en el primer cambio.

## Y no cuesta lo que parecía

El puerto tiene [una sola operación](../concepts/sync.md#tener-clave-es-del-ítem-verificarse-es-de-la-rama) —`Provider::snapshot(keys)`— y toma todas las claves juntas: contra Jira es **una JQL paginada**, no una llamada por ítem. Para una ventana de 19 es una consulta.

`pull.md` había dejado esto afuera diciendo que *"le pone el costo caro al comando barato"*, y **ese número estaba mal atribuido**: lo caro es el panorama de 292 claves, no una ventana.

## Los tres campos no se absorben igual

| Campo | | Por qué |
|---|---|---|
| **título** | **se absorbe** | la vuelta es directa |
| **`status`** | **sólo si la vuelta es única** | ver abajo |
| **cuerpo** | **no, todavía** | ver abajo |

### El `status` no siempre se puede volver, y no es un defecto del comando

El [mapeo de estados](../concepts/states.md) va del worklist al proveedor, y **no tiene por qué ser inyectivo**. Medido en esta instalación:

```json
{ "open": "Tareas por hacer", "in-progress": "En curso",
  "done": "Finalizada", "dropped": "Finalizada" }
```

`Finalizada` vuelve a `done` **y** a `dropped`.

> **La vuelta no es una función, así que no se absorbe: se reporta con las dos candidatas.**

```
ACC-101  status  el board dice "Finalizada", que vuelve a `done` o a `dropped` — no se elige solo
```

**Y no se arregla arreglando el mapeo.** Que dos estados del worklist compartan uno del proveedor es una limitación del board: su workflow tiene tres status para seis tipos de issue y `resolution` no está en el `editmeta`, así que `dropped` no se puede distinguir de `done` allá. Va a seguir pasando, y lo que el comando tiene que hacer es **no adivinar**.

### Y el cuerpo espera, con un número al lado

Medido el 2026-09-07 sobre 295 claves: **149 cuerpos difieren, y 113 de ésos son del conversor** — el round-trip no devuelve lo que guardó, así que el ítem no coincide consigo mismo sin que nadie haya tocado el board.

Absorber hoy reescribiría más de cien ítems con cambios que nadie hizo, y cada uno es un commit que después sube al panorama.

> Reportar de más es ruido. **Absorber de más es escribir.**

Se enciende cuando el round-trip cierre. Hasta entonces el cuerpo se **reporta**, que es lo que ya hace `check-push`.

## Absorber pisa, así que no decide solo

Si edité un ítem acá y alguien lo editó allá, absorber en silencio elige por mí. Es el mismo argumento que [`--ff-only` contra `reset --hard`](pull.md): **la negativa es el dato**.

La línea es lo que **todavía no subió**: el rango entre [la marca de propagación](../concepts/propagation.md#hasta-dónde-subió-se-anota-en-una-ref-no-se-deduce) y el tip de la ventana es exactamente el trabajo que nadie más vio.

| | |
|---|---|
| el ítem no fue tocado desde la marca | **se absorbe** — no hay nada de este lado que pisar |
| el ítem cambió después de la marca | **se reporta y no se toca**: los dos lados escribieron |

## Una clave que el proveedor no informa se cuenta aparte

> **Cero absorbidos sobre veinte claves no es *"todo coincide"*.**

Es la misma regla que [`check-push`](check-push.md#el-resumen-es-el-número-que-decide) y [`push-states`](push-states.md) ya aplican, y **este comando la rompió el día que se escribió**. Medido el 2026-09-07, contra la instalación real:

```
refs/heads/secure/sprint/21: 20 clave(s)
resumen: 0 absorbido(s), 0 reportado(s)          ← se leía como "está todo bien"
```

Las veinte claves habían caído en el mismo `else { continue }`: el proveedor de esta instalación es [el de prueba](check-push.md), y su archivo está **vacío**, así que no informó ni una. El comando calló, y callar convirtió *"nadie miró"* en *"coincide"*.

```
refs/heads/secure/sprint/21: 20 clave(s)
  sin informar (20): ACC-259, ACC-263, ACC-268, ACC-275, ACC-302, …
resumen: 0 absorbido(s), 0 reportado(s), 20 sin informar
```

**Y la cuenta sube.** [`pull`](pull.md) sólo cierra diciendo *"al día"* si además el proveedor informó todo; con una sola sin informar dice qué no miró. Lo mismo [`status --verify`](status.md), que ya no dice `coincide` sino `el proveedor no informo`.

**Una clave informada sin ningún campo cuenta igual**, porque es lo mismo: el de prueba sólo lleva `status`, así que sobre título y cuerpo no vio nada.

## Salida

```
$ worklist-server absorb --ref refs/heads/secure/sprint/21 --project ACC …
refs/heads/secure/sprint/21: 19 clave(s)
  ACC-140  titulo   "Separar absorber de decidir" -> "separar absorber de decidir"
  ACC-101  status   el board dice "Finalizada", que vuelve a `done` o a `dropped` — no se elige solo
  ACC-263  cuerpo   difiere — linea 7   (el cuerpo no se absorbe: ver ACC-321)
  ACC-275  titulo   cambiado alla y aca desde la ultima propagacion — no se toca
resumen: 1 absorbido, 3 reportados
absorb: refs/heads/secure/sprint/21 <- el proveedor  (1 commit)
```

**El resumen cuenta las dos cosas**, porque son decisiones distintas: lo absorbido ya está, y lo reportado espera a alguien.

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | se pudo preguntar, absorbiera o no |
| `1` | no se pudo preguntar al proveedor, o la rama no existe |

**Cero aunque haya diez reportados**, por lo mismo que en [`check-push --dry-run`](check-push.md#códigos-de-salida): lo que el retorno informa es si la medición se pudo hacer.

## Y lo que el proveedor perdió lo saca otro comando

[`worklist-server removes`](removes.md) — porque no es lo mismo. `absorb` reconcilia **campos** de ítems que existen de los dos lados; sacar un ítem del árbol es otra decisión, y se toma con otro dato: el código HTTP de una lectura por clave, no el `snapshot` del lote.

Y corre sobre **el panorama**, no sobre una ventana: ahí están las dos cosas que hay que tocar —el archivo y el `items` que lo nombra— y las ventanas se regeneran.
