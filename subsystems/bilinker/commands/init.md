# Especificación: comando `bilinker init`

## Propósito

Pone a punto el clon para que bilinker pueda operar. Es **lo primero que corre cualquiera que vaya a usar bilinker**, y sin él ningún otro comando corre.

Todo lo que [la ref](../concepts/ref.md) necesita son tres cosas puestas en el clon, y ninguna viaja con él: la exclusión en `.git/info/exclude`, el refspec en `.git/config`, y el `.bilink/` materializado en el árbol. Son **por clon**, no por rama ni por commit, porque las tres viven en `.git/` o fuera de git.

## Firma

```
bilinker init [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--dry-run` | Muestra qué haría sin escribir nada. |

No toma path: es **por repo**, no por capa. Un solo patrón `.bilink/` en el exclude cubre todas las capas de ese repo, estén donde estén.

## Qué hace

```
1. .git/info/exclude  ←  .bilink/  y  .bilink-migrate-*
2. .git/config        ←  refspec de refs/bilink/*   → desde acá, git fetch las trae
3. fetch de refs/bilink/*  +  materializar el .bilink/ de la rama actual  +  escribir head
```

### 1 — La exclusión

En `.git/info/exclude`, **no en `.gitignore`**. `.gitignore` está versionado, y agregarlo modificaría la rama del proyecto — justo lo que este diseño evita. `info/exclude` es local, no se commitea y no aparece en ningún MR.

`.bilink-migrate-*` va al lado: esas carpetas son temporales y nunca se commitean. [`migrate`](migrate.md) ya las escribe ahí al empezar; tenerlas también acá vuelve el clon correcto antes de la primera migración.

Un exclude que ya exista se respeta: se agregan las líneas que falten y no se toca lo demás.

### 2 — El refspec

```
[remote "origin"]
    fetch = refs/bilink/*:refs/bilink/*
```

**Con el refspec puesto, `git fetch` trae las refs de bilinks junto con las ramas.** Sin él, una rama al día puede convivir con bilinks viejos, que es la clase de desajuste silencioso que este diseño existe para eliminar.

No compromete la invisibilidad: `refs/bilink/*` sigue sin aparecer en `git branch -a` ni en la forja, porque no está bajo `refs/heads/` ni bajo `refs/remotes/`. Es lo que [la ref](../concepts/ref.md#fuera-de-refsheads) ya anticipa al decir que los refspecs los pone bilinker.

**Se mapea a sí mismo, no a `refs/remotes/`.** La ref del remoto y la local son la misma cosa: no hay un flujo de trabajo donde alguien tenga bilinks locales adelantados que quiera comparar contra los del remoto sin traerlos.

**Y va sin `+`.** El `+` de un refspec significa *"actualizá incluso si no es fast-forward"*, y acá es exactamente lo que no se quiere. Como la ref es append-only, en operación normal el fetch es fast-forward y el `+` no aporta nada; lo único que agrega es que, si alguien la reescribió, el fetch **pisa la ref local en silencio** — y con ella los commits a los que apunta cada [`accepted.commit`](../concepts/ref.md#la-ref-es-protegida) del repo. Sin `+`, ese fetch falla y el problema se ve. Es la mitad del clon de una regla que del lado del servidor es un rechazo.

Si hay más de un remoto, el refspec va en todos. Si ya está, no se duplica.

### 3 — Fetch, materializar y escribir `head`

Se traen las refs, se materializa el `.bilink/` de la rama actual desde `refs/bilink/<branch>`, y se escribe [`.bilink/head`](../concepts/ref.md#bilinkhead-de-dónde-salió-el-árbol) con la rama y el commit.

[`.bilink/version`](../concepts/format-version.md) **llega sola**: está versionada, así que viaja en el árbol de la ref como cualquier otro archivo de `.bilink/`, y la materialización la escribe con los demás. `init` no la calcula ni la elige — sería la única cosa del directorio que no saliera del commit, y entonces podría discrepar de los archivos que describe.

Es la distinción que [`cache.md`](../concepts/cache.md) ya hace por otro lado: `cache/`, `index/` y `head` quedan fuera del índice de bilinker porque son derivados o estado del árbol; `version` entra porque describe archivos versionados y viaja con ellos.

## El paso 3 no pisa nada

Si hay un `.bilink/` en el árbol y no hay `head`, `init` **no puede saber de dónde salió**, así que lo deja intacto y se limita a los pasos 1 y 2 — y lo dice.

```
$ bilinker init
exclude: + .bilink/  + .bilink-migrate-*
refspec: + refs/bilink/*:refs/bilink/*

.bilink/ presente sin head: no se materializa nada.
  Es lo esperado en el paso 3 del corte 005; en un clon fresco, revisar de
  dónde salió antes de seguir.
```

Es lo que hace que el paso 3 del [corte `005`](../.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md) pueda ser un `init` a secas: ahí el `.bilink/` del árbol todavía no está en la ref, y materializar lo borraría.

Y eso absorbe el setup propio de la migración: **el corte corre el mismo `init` que corre cualquier clon**. Un camino menos que mantener, y uno que se ejercita todos los días en vez de una sola vez.

## Sin `init`, los comandos fallan; no se auto-configuran

**Bilinker arregla solo lo que es suyo, y pide lo que es del repo del usuario.** Materializar `.bilink/` es automático porque `.bilink/` le pertenece; escribir en `.git/config` y `.git/info/exclude` es tocar la configuración de otro, y eso merece un acto explícito por más inofensivo que sea.

La alternativa —que el primer `check` configure el repo de callado— convierte un comando de lectura en uno que modifica el entorno.

```
$ bilinker check .
error: el repo no está inicializado para bilinker.
  Correr `bilinker init`.
```

La detección pide **las dos** piezas que `init` escribe: el exclude, y el refspec en cada remoto que haya. El refspec es la que no puede estar por accidente —un `.bilink/` en el árbol puede venir de antes del corte, y el exclude lo pudo escribir alguien a mano— pero en un repo sin remoto no existe, y pedirla sola lo dejaría sin forma de estar nunca inicializado.

**Y sólo se exige donde corresponde: en un repo que ya cortó**, que es lo que dice su [ledger](migrate.md). Antes del corte los bilinks viven en la rama, no hacen falta ni exclude ni refspec, y exigirlos rompería todos los repos que todavía no cortaron — incluida la herramienta con la que se corta. Lo dice el ledger y no el filesystem porque el ledger está commiteado: un clon fresco de un repo que cortó lo sabe **antes** de tener una sola `refs/bilink/*` local, que es exactamente el caso en el que hay que exigirlo. Mirar si hay refs daría la respuesta contraria justo ahí.

## Es idempotente

Correrlo dos veces no hace nada la segunda: las líneas ya están, el fetch es fast-forward y no trae nada, y la materialización encuentra el árbol al día.

Correrlo después de un `git clone` en una máquina nueva es el caso normal, no una recuperación.

## Salida

```
$ bilinker init
exclude: + .bilink/  + .bilink-migrate-*
refspec: + refs/bilink/*:refs/bilink/*  (origin)
fetch:   refs/bilink/main  0af3c12..b1e3f55
árbol:   .bilink/ materializado desde refs/bilink/main @ b1e3f55
         5 capa(s), 63 bilink(s), formato 3.0.0
```

Cuando no hay nada que hacer:

```
$ bilinker init
ya inicializado (origin) · refs/bilink/main @ b1e3f55 · árbol al día
```

Cuando la rama actual no tiene ref:

```
$ bilinker init
exclude: ya estaba
refspec: ya estaba
fetch:   sin cambios

refs/bilink/feature/x no existe.
  Correr `bilinker track feature/x` para crearla.
```

No es un error: una rama nueva todavía no tiene bilinks propios, y quién los hereda es una decisión de [`track`](track.md), no de `init`.

## Códigos de salida

| Código | Condición |
|---|---|
| 0 | Inicializado, o ya estaba. |
| 1 | No es un repo git; o el fetch falló; o hay un `.bilink/` sucio que la materialización habría pisado. |

## Propiedades garantizadas

- **Idempotente**: correrlo dos veces no hace nada la segunda.
- **No toca ninguna rama del proyecto**: escribe en `.git/`, y en el árbol sólo `.bilink/`.
- **No commitea**: `init` no escribe ningún commit, ni sobre la ref ni sobre el proyecto.
- **No pisa un `.bilink/` sin procedencia**: sin `head`, no materializa.
