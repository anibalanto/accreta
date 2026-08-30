# Migración de metadatos

Las herramientas de Accreta guardan su estado en archivos —`.bilink`, `.capture`, `.task`, reportes `.impact`— cuyo formato evoluciona con las specs. Una **migración** es una transformación de esos archivos al formato vigente, aplicada una sola vez por repo y registrada.

No migra contenido del proyecto: solo los metadatos que las herramientas mantienen sobre él.

## Por qué no alcanza con "que la herramienta lo tolere"

La alternativa a migrar es que cada herramienta lea los dos formatos para siempre. Eso funciona hasta el segundo cambio, y a partir de ahí cada lectura carga con toda la historia del formato. La tolerancia sirve como puente durante la transición; la migración es la que la termina.

## Por qué no es Liquibase

El modelo es el de Liquibase —un conjunto ordenado de cambios y un registro de cuáles se aplicaron— pero buena parte de su maquinaria existe para compensar que **una base de datos no está versionada**. Acá los metadatos sí lo están:

| Liquibase resuelve | Acá lo resuelve |
|---|---|
| bloques de rollback | `git revert` |
| checksums de changeset | el changeset está en git |
| coordinar al equipo | el commit de migración se propaga solo |
| ver qué cambió | `git diff` |

Lo que queda después de tachar eso es el núcleo: **ids ordenados, un ledger de aplicadas, y un runner idempotente**.

## No hay centro

Liquibase asume una base de datos con una tabla donde anotar. Accreta es distribuido por diseño, y un proyecto Stratum reparte sus metadatos entre varios repos: en este mismo proyecto hay cuatro capas con `.bilink/` repartidas en tres repos git.

Por eso **el ledger es por repo**, no global. Cada repo sabe qué formato tiene lo suyo, y un clone trae esa información con el resto del contenido.

## El ledger

```
<repo-root>/.accreta/
  migrations        ← un id aplicado por línea, ordenado
```

```
# Migraciones aplicadas en este repo.
# Una por línea, ordenadas. Los merges se resuelven por unión.
bilinker-001-capture-split
```

### Es un conjunto de ids, no un número de versión

Si dos ramas agregan migraciones distintas, un entero da conflicto y al resolverlo se pierde el registro de una de las dos. La unión de dos conjuntos, en cambio, siempre es la respuesta correcta — y es lo que un merge de texto hace naturalmente con líneas ordenadas.

### Es un archivo, no el historial de git

La tentación es anotar la migración en el mensaje del commit y consultarla con `git log --grep`. No: **el historial de git es reescribible y por lo tanto no sirve como registro**. Un rebase o un squash borra los trailers, un cherry-pick los duplica, y un clone shallow no ve historia y concluye que no se aplicó nada.

Un registro tiene que ser contenido. El archivo lo es; el historial no.

## La migración

```
<herramienta>-<NNN>-<slug>
bilinker-001-capture-split
```

El prefijo evita colisiones entre subsistemas, el número fija el orden, el slug la hace legible en el ledger.

Una migración es **código**, no una descripción declarativa. Las transformaciones reales no son renombres de campos: `bilinker-001` parte cada endpoint estructural en dos archivos y genera UUIDs nuevos. Ningún formato declarativo expresa eso sin volverse un lenguaje de programación.

### Es una transformación sintáctica, no una resolución

Una migración **no resuelve queries ni consulta git**. Copia lo que había y deja los campos derivados vacíos, para que el `check` siguiente los recalcule.

El motivo es que una migración que resuelve puede fallar por razones ajenas al formato —un archivo que se movió, una query que se rompió— y dejar la capa a mitad de camino, con parte de los archivos en un formato y parte en otro. Una transformación puramente sintáctica no puede fallar por el estado del proyecto, y eso vale más que tener el dato fresco.

### Idempotencia

Correrla dos veces sobre la misma capa no hace nada la segunda vez. Es lo que permite migrar capa por capa, a distinto ritmo, sin llevar la cuenta a mano.

### `--dry-run` no escribe

Con `--dry-run` una migración calcula y reporta exactamente lo mismo, **sin escribir un solo archivo**. Es parte del contrato de la migración, no del runner: el runner no puede impedir que una migración escriba.

### El conjunto de migraciones es de sólo-agregar

> **Nunca se borra una migración**, ni siquiera cuando parece que ya nadie está en ese formato.

Es lo único que permite que alguien parado en una versión vieja llegue a la actual corriendo la cadena entera. "Ya nadie está en ese formato" no es verificable: un clon dormido, una rama vieja, un fork. Borrar la migración convierte eso en un archivo que no se puede leer y del que no queda registro de cómo se leía.

Lo que se acumula es el **build**, no la lectura. Si una migración depende de dos versiones del crate de formato, esas dos quedan en el repo para siempre; pero el binario del día a día linkea sólo la última. Es la distinción que hace que esto no sea lo que § "Por qué no alcanza con que la herramienta lo tolere" descarta.

### Una migración conoce los dos formatos que puentea

Una migración lleva archivos de lo que entiende el formato N a lo que entiende el N+1, así que **declara ese par y depende de las dos versiones**. No hay otra forma de que un componente lea los dos.

De ahí sale algo que no es evidente: **la verificación de que la migración no perdió nada la hace la migración**, no un comando que compare dos árboles. Un comando así linkea un solo parser y sólo puede leer uno de los dos lados.

## El runner

```
<herramienta> migrate [<path>] [--recursive] [--dry-run]
```

Para bilinker, ver [commands/migrate.md](../subsystems/bilinker/commands/migrate.md).

Cada herramienta expone su propio subcomando y registra sus migraciones; el runner compartido aporta el ledger, el orden, el dry-run y el reporte. No hay un `accreta migrate` único: tendría que conocer el formato de todos los subsistemas, que además viven en repos separados.

### Ledger por repo, migraciones por capa

Son dos granularidades distintas y conviene tenerlo presente:

- Una **capa** es donde viven los metadatos (`<layer>/.bilink/`).
- Un **repo** es la unidad que git versiona, y donde va el ledger.

Una capa puede ser su propio repo, o varias capas pueden compartir uno. El runner agrupa las capas por repo antes de aplicar nada.

De ahí sale una regla que no es obvia: **una migración se marca como aplicada cuando corrió sobre todas las capas de ese repo**, no sobre la primera. Si no, migrar una capa dejaría el repo marcado como migrado con las demás sin tocar.

El corolario práctico es que hay que alcanzar todas las capas de un repo en la misma corrida — `--recursive` desde la raíz, no invocaciones sueltas.

## Flujo

```
<herramienta> migrate --recursive --dry-run   → revisar el alcance
<herramienta> migrate --recursive             → transformar
git diff                                       → auditar
<herramienta> check .                          → recalcular lo derivado
git commit                                     → registrar, ledger incluido
```

El runner no commitea. `apply` sí lo hace porque aplica muchos fixes chicos y semánticos; una migración es un cambio mecánico masivo y conviene mirarlo antes de fijarlo.

## Invariantes

1. El ledger vive en `<repo-root>/.accreta/migrations` y es un conjunto de ids.
2. Una migración se registra solo después de correr sobre todas las capas de ese repo.
3. Una migración es idempotente: sobre una capa ya migrada no hace nada.
4. Con `--dry-run` no se escribe ningún archivo.
5. Una migración no resuelve queries ni consulta git; solo transforma formato.
6. El runner nunca commitea ni modifica el historial.
