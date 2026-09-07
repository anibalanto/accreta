# Comando: `worklist push`

Empuja la vista donde uno está parado, dice qué escribió el servidor encima, y traduce el rechazo.

> **No aporta garantías. Aporta que el bucle sea corrible.**

Es la distinción que [`pull.md`](pull.md#empujar-es-una-invariante-y-por-eso-no-necesita-comando) no hizo. Ahí quedó argumentado que empujar **no necesita garantías propias** —sin sustracción no hay borrado huérfano, así que no hace falta atomicidad entre ramas; y el chequeo previo no puede ser local, así que no hay nada que verificar de este lado— y de eso se concluyó que no hacía falta un comando. Las tres invariantes se quedan. La conclusión estaba mal.

## Lo que estaba roto, medido

```
$ git status
En la rama secure/sprint/21
nada para hacer commit, el árbol de trabajo está limpio     ← con cinco commits sin empujar

$ git rev-parse --abbrev-ref @{u}
fatal: no se ha configurado upstream para la rama 'secure/sprint/21'
```

Las vistas nacen sin upstream, porque `git worktree add` no lo configura. Sin upstream, `git status` **no tiene contra qué compararse** y nunca dice *"ahead by 1"*; y `git push` a secas falla.

Así que el bucle que `pull.md` daba por corrible —**`push` rechazado → `pull` → resolver → `push`**— no arrancaba, y el estado en el que uno se quedaba era indistinguible de estar al día.

## Firma

```
worklist push [<vista>] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `<vista>` | Cuál empujar. Por defecto, aquella en la que se está parado. |
| `--dry-run` | Dice qué empujaría, sin empujar. |

**No tiene `--all`**, y es el mismo argumento que se cayó en `pull`: si el comando es de la vista donde estoy parado, no hay asimetría que administrar. Bajar todas de una tiene sentido porque son operaciones independientes; **subir todas de una era lo que pedía atomicidad**, y la atomicidad se cayó.

## Lo que hace, y por qué cada cosa

### Sabe adónde va

El refspec es `HEAD:refs/heads/<vista>`, y el remoto es `srv`. Nadie tiene que acordárselo, y **no depende de que la rama tenga upstream** — que es justamente lo que no tiene.

### Configura el upstream la primera vez

Es una línea, y arregla `git status` para siempre: a partir de ahí dice *"ahead by N"* sin que haga falta ningún comando del worklist.

**Se decidió que lo haga el push y no [`pull`](pull.md)**, aunque `pull` también toca cada vista. El motivo es de significado: el upstream es *"contra qué se compara lo que tengo para subir"*, y eso es la pregunta del push. Que `pull` la contestara de paso sería configurar una relación que su propio comando no usa — `pull` no la mira, porque [pregunta la marca del servidor](pull.md#y-lo-primero-que-hace-el-paso-3-es-preguntar-si-hay-algo-que-replantar).

> **Un comando escribe la configuración que él mismo necesita.** Si la escribe otro, el día que alguien la borre el síntoma aparece lejos de la causa.

### Dice qué escribió el servidor encima

El `post-receive` commitea sobre lo que recibió —el `rename <slug> -> <clave>`, el `normalize:`, la clave del sprint— y **eso no está en tu worktree**. La salida del push ya lo informa; el comando lo repite en su propia voz y cierra con qué hacer:

```
$ worklist push
secure/sprint/21 → srv   2 commit(s)
  @el-push-que-vuelve-corrible-el-bucle -> ACC-325
  estado: ACC-263 -> En curso
  9 commit(s) al panorama
al día en el servidor; para traer lo que escribió: worklist pull
```

**La última línea es la que faltaba.** Sin ella, el que empujó se queda con una ventana que ya no coincide con el servidor y nada se lo dice — que es el mismo agujero que este ítem existe para tapar, un paso después.

### Traduce el rechazo

**Son tres motivos, no dos**, y el tercero no lo pone el hook:

| Rechazo | Quién lo pone | Qué hacer |
|---|---|---|
| deriva del proveedor | el `pre-receive` | mirar el board; el push no puede arreglarlo |
| no aplica sobre el panorama | el `pre-receive` | **`worklist pull`**, resolver, y empujar de nuevo |
| **no-fast-forward** | **git, antes del hook** | **`worklist pull`** y empujar de nuevo |

> **El tercero es el más frecuente de los tres, y es consecuencia directa de que el diseño funcione:** el servidor commitea encima de *cada* push que acepta —el `rename`, el `normalize:`, la clave del sprint—, así que el segundo push de cualquiera choca contra eso.

Y se descubrió usándolo: el primer push de este propio ítem quedó rechazado con un mensaje genérico, porque la traducción cubría los dos del hook y no el de git — que es el que **más** tiene que decir la palabra.

**El orden en que se reconocen importa.** El `rejected` de git también aparece cuando el que rechaza es el hook, así que se miran primero los dos específicos; matchear el genérico antes taparía el motivo real con el más común.

## Lo que no promete, dicho

**No verifica contra el proveedor antes de empujar.** Lo hace el `pre-receive`, que es donde importa porque es donde se escribe. Del lado del cliente sería [un verde falso](pull.md#y-el-chequeo-previo-no-puede-ser-local): habría que probar contra una copia del panorama que el cliente no tiene y no va a tener.

**No empuja varias vistas juntas**, y no hay atomicidad que ofrecer.

**Y no reemplaza a `git push`**, que sigue funcionando. Prohibirlo protegería una atomicidad que no existe — [ya se descartó](pull.md#empujar-es-una-invariante-y-por-eso-no-necesita-comando), y sigue descartado.

## Un push sin nada que empujar es una línea, no un error

Con el bucle corrible va a pasar seguido, y es la misma decisión que hizo a [`pull` idempotente](window-open.md#recortar-sobre-lo-mismo-no-produce-un-corte-nuevo): un comando que se corre de más no puede castigar por eso.

```
$ worklist push
secure/sprint/21: nada que empujar
```

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | se empujó, o no había nada que empujar |
| `1` | el servidor rechazó — la salida dice cuál de los dos motivos, y qué hacer |
| `1` | no es una vista del worklist, o no hay remoto `srv` |
