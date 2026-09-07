# Comando: `worklist-server check-push`

El compare-and-swap de [`concepts/sync.md`](../concepts/sync.md#la-ventana-y-el-compare-and-swap), pensado para correr desde `hooks/pre-receive`. No mira el contenido que llega: compara el estado en vivo del proveedor contra lo que el tip actual de la rama —el del servidor, antes de este push— tiene escrito.

## Firma

```
worklist-server check-push (--provider-file <archivo> | --project <clave> [--base <url>] [--account <email>])
                           [--states-map <archivo>] [--stdin] [--dry-run [--ref <rama>]]
```

| Argumento | Descripción |
|---|---|
| `--provider-file` | El proveedor **de prueba**: un archivo `clave -> status`. Ver [`worklist-server provider set-status`](provider-set-status.md). |
| `--project` | El proveedor **real**. |
| `--states-map` | El [mapeo de estados](../concepts/states.md) de esta instalación. **Obligatorio con `--project`**: sin él el comando se niega a arrancar, en vez de rechazar todas las ventanas. |
| `--base` | La URL del proveedor. Con `--project` se le piden por REST las transiciones que el workflow admite, que es lo que decide un rechazo por regla. |
| `--account` | El email de la cuenta con la que REST autentica. Ver [`install-hooks`](install-hooks.md). |
| `--stdin` | Lee `<viejo> <nuevo> <ref>` por línea — el protocolo de un `pre-receive`. Sin esto, toma el rango de la rama actual. |
| `--dry-run` | Compara y **no rechaza**: reporta lo que difiere y sale con cero. Ver § "Comparar sin rechazar". |
| `--ref` | Sobre qué rama comparar. **Sólo con `--dry-run`**, y por defecto el panorama. |

**El proveedor real habla por dos transportes, y esto no es un detalle de adentro.** El estado en vivo se lee con `acli`; las transiciones que el workflow admite se piden por REST, porque [ningún CLI puede listarlas](../concepts/sync.md#y-el-tercero-llegó-por-una-razón-que-no-era-la-prevista). De ahí que `--project` traiga `--base` y `--account` de la mano: son la dirección y la mitad no secreta de la credencial.

**Uno de los dos, y no los dos.** El de prueba sólo informa el `status`, así que con él **el título y el cuerpo no se comparan** — el paso 4 de abajo se achica a una tercera parte. Es lo que hoy tiene la instalación, porque el `status` del worklist y el de Jira no son el mismo campo y compararlos rechazaría todas las ventanas. La task `74` es ese mapeo.

## Qué hace

1. Para cada `ref` que llega: si es `refs/heads/insecure/**`, **rechaza el push** — una rama insegura no se puede verificar, así que no puede aceptar escrituras. Si no es `refs/heads/secure/**`, la ignora: este comando no opina sobre ramas que no son del worklist.
2. Lee, del árbol de `<viejo>`, todos los `*.md` cuyo nombre ya es una clave de proveedor (`is_unassigned` en falso), y su campo `status`.
3. Le pregunta al proveedor su estado ahora mismo para esas mismas claves.
4. Si algún valor difiere, rechaza — imprime cuál clave y los dos valores.
5. Y **prueba contra el panorama**: si lo que la ventana traería no aplica sobre `insecure/all`, rechaza. Ver [`concepts/propagation.md`](../concepts/propagation.md#y-el-panorama-nunca-guarda-un-conflicto).

Si las dos pruebas pasan, acepta.

## Son dos árbitros, y el segundo sólo ve lo que el primero no

El proveedor arbitra todo lo que tiene clave: dos ventanas que editan el mismo ítem chocan en el paso 4, un paso antes de que el panorama se entere. Lo que llega al paso 5 es lo que **no tiene contraparte allá** — en la práctica, un ancestro de sólo lectura que alguien editó adentro de su ventana.

**Se prueba en memoria, con `merge-tree`.** Un `pre-receive` que rechaza no puede dejar un worktree ni objetos atrás, y el paso 5 corre en todos los pushes, no sólo en los que fallan.

**Y no poder probar no es que entre.** Un repo sin panorama —hoy, el de la instalación— no puede correr el paso 5, y eso se **avisa**: dar por bueno lo que nadie miró es el mismo defecto de forma que confundir *"no había trabajo"* con *"el trabajo no se hizo"*.

```
aviso: refs/heads/secure/sprint/10 no se pudo probar contra el panorama — este repo no tiene refs/heads/insecure/all
```

No rechaza el push: informa. Que el panorama no esté del lado del servidor es un problema de la instalación, no de quien empuja.

**Y no prueba los renombres**, porque no se copian: [se rehacen](../concepts/propagation.md#el-renombre-es-el-único-que-no-se-copia-y-el-motivo-es-de-alcance) sobre el árbol del panorama, y ahí no hay parche que pueda no aplicar.

## Comparar sin rechazar

> **Antes de darle poder de veto al proveedor conviene saber cuánto va a vetar.** `--dry-run` es esa corrida: las mismas comparaciones, ninguna decisión.

Es lo que hace falta para cruzar la instalación al proveedor real. El `status` ya se midió —lo mide [`push-states --dry-run`](push-states.md)—, pero el título y el cuerpo **nunca se compararon contra nada**: el proveedor de prueba no los informa, así que esas dos comparaciones no corrieron ni una vez. Encenderlas de golpe sobre todo el inventario es enterarse de cuántas difieren cuando ya rechazan.

**Y no es un comando aparte a propósito.** Lo que hay que medir es exactamente *lo que `check-push` va a rechazar*, y la única forma de que la medición no mienta es que la haga el mismo código. Una copia con la misma intención empieza igual y se separa después, que es el defecto que este subsistema ya evita en todos lados: un inventario paralelo se usa como si fuera el bueno.

### Sin push que proteger, los dos recortes se caen

`--dry-run` no es sólo callar el rechazo: **cambia dos alcances, y los dos por el mismo motivo.**

| | Verificando un push | En `--dry-run` |
|---|---|---|
| una rama `insecure/**` | se rechaza — no se puede verificar, así que no acepta escrituras | **se lee**, y es el default |
| el título y el cuerpo | sólo sobre las claves que el push escribe | sobre **todas** las del tip |

Los dos recortes existen por el push, y acá no hay push. Al panorama no se le empuja, pero leerlo no es empujarle — es la misma forma que [`push-states`](push-states.md#corre-sobre-el-panorama-y-lee), y por la misma razón: es el único que tiene el inventario completo, y medir sobre un recorte contesta sobre sus ítems callando el resto. Y el alcance angosto del título y del cuerpo es la regla de [`concepts/sync.md`](../concepts/sync.md#dos-alcances-porque-son-dos-promesas) —*"un push que no toca un ítem no puede pisarlo"*—, que dice qué puede rechazar una escritura, no qué se puede mirar.

### El resumen es el número que decide

La última línea cuenta por campo, porque de eso depende qué se hace después: **cero es cruzar, y ochenta es arreglar antes de cruzar.** Un listado sin total obliga a contarlo a mano y a equivocarse en el orden de magnitud, que es justo lo que la decisión necesita.

**Y la primera línea dice sobre cuántas claves habla**, que es la otra mitad del mismo número: cero diferencias sobre cero claves se lee igual que cero sobre doscientas. Una clave que el proveedor **no informa** no entra en esa cuenta y se lista aparte — no es una que coincida, es una que no se vio, y lo que hay que hacer con ella es averiguar por qué no está. Es la misma regla que [`push-states`](push-states.md#una-clave-que-el-proveedor-no-informa-se-reporta-no-se-saltea).

## Un cuerpo que difiere dice dónde, no cuánto

Un título entra en una línea y se muestra entero de los dos lados. Un cuerpo no: son cientos de líneas, y ponerlas al lado convierte un reporte en un volcado del archivo.

Se informa **la primera línea que difiere, con su número**, y cada lado recortado:

```
reject: ACC-140 cuerpo difiere — linea 7
        tip:       **Replanificado el 2026-09-06.** Antes se llamaba *"El modelo y la…
        proveedor: **Replanificado el 6/9.** Antes se llamaba *"El modelo y la fronte…
```

**Decir sólo `difiere` es un rechazo que no se puede accionar**: el que empuja sabe que algo cambió y no dónde, así que su único camino es leer los dos cuerpos enteros y compararlos a ojo — el trabajo que el hook acaba de hacer y tiró.

## Salida

```
$ worklist-server check-push --provider-file provider.json --stdin <<< "a1b2c3d e4f5g6h refs/heads/sprint/10"
reject: ACC-101 status era "open" en el tip, el proveedor dice "done"
```

Y la corrida que mide. **Es la misma línea en los dos casos** —lo único que cambia es la etiqueta—, porque es el mismo hallazgo: dos formatos serían dos cosas que mantener de acuerdo, y la medición existe justamente para anticipar lo que el rechazo va a decir.

```
$ worklist-server check-push --project ACC --states-map … --dry-run
worklist-server 0.1.0
refs/heads/insecure/all: 291 claves comparadas
  sin informar (1): ACC-3
  difiere: ACC-101 titulo era "Separar absorber de decidir" en el tip, el proveedor dice "separar absorber de decidir"
  difiere: ACC-140 cuerpo difiere — linea 7
          tip:       **Replanificado el 2026-09-06.** Antes se llamaba *"El modelo…
          proveedor: **Replanificado el 6/9.** Antes se llamaba *"El modelo y la f…
resumen: status 0, titulo 1, cuerpo 1
```

Y el segundo árbitro:

```
reject: a2b035d edito ACC-14 no entra al panorama — choca en ACC-14.epic.md
        alguien mas escribio eso desde otra ventana. Regenera la tuya y volve a aplicarlo.
```

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | todas las ventanas del rango coinciden con el proveedor |
| `0` | `--dry-run`, difiera lo que difiera |
| `1` | al menos una clave difiere — push rechazado |
| `1` | lo que la ventana traería no aplica sobre el panorama — push rechazado |

**`--dry-run` sale con cero incluso cuando encuentra ochenta diferencias**, y no es una omisión: lo que ese código informa es si la medición se pudo hacer. Que el resultado se lea de la salida y no del retorno es la misma regla que [`concepts/sync.md`](../concepts/sync.md#el-éxito-se-lee-de-la-salida-nunca-del-código-de-retorno) le pide al proveedor, acá aplicada al propio comando.
