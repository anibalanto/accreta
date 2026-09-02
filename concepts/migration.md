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

Liquibase asume una base de datos con una tabla donde anotar. Accreta es distribuido por diseño, y un proyecto Stratum reparte sus metadatos entre varios repos: en este mismo proyecto hay seis capas con `.bilink/` repartidas en cuatro repos git.

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

### Una migración no puede descartar una decisión

> **Si no puede llevar un campo hacia adelante, registra el hueco; nunca lo reemplaza por una respuesta.**

Es la consecuencia de la regla de arriba, y la que menos se ve. Una migración que no resuelve se topa con campos que no puede producir —los que salían de una query, de un language server, de la red— y ahí las salidas parecen dos: descartar el campo, o negarse a migrar. Hay una tercera y es la que corresponde: **conservar lo que sí tenía y declarar que le falta el resto.**

La distinción es entre una **respuesta** y una **imposibilidad**. *"Nadie vigila esto"* es una decisión que alguien tomó; *"no pude traer esto"* es un estado de la migración. Escribir la primera donde iba la segunda no pierde sólo el dato: pierde el registro de que había un dato, y eso es peor, porque el hueco deja de verse.

**Y una decisión que escribe una migración es permanente.** El archivo no dice quién la escribió, así que la herramienta la lee de vuelta como propia y ningún `check` posterior la vuelve a preguntar. Una persona que renuncia cambia de opinión cuando algo se lo recuerda; una renuncia que la migración inventó no tiene a quién recordarle nada.

De ahí sale qué le toca al formato antes que a la migración: **el formato tiene que poder expresar el hueco.** Donde no puede, eso es trabajo previo y no una excusa para inventar un valor — un tipo de salida que no modela el hueco no tiene cómo escribirlo, y entonces la pérdida no es un descuido, está en la firma.

Que fue una migración la que dejó el hueco va donde va la procedencia: el reporte de la corrida, que ya cuenta cuántos campos no pudo traer. El archivo registra lo que **es**, no cómo llegó a serlo.

### Idempotencia

Correrla dos veces sobre la misma capa no hace nada la segunda vez. Es lo que permite migrar capa por capa, a distinto ritmo, sin llevar la cuenta a mano.

### `--dry-run` no escribe

Con `--dry-run` una migración calcula y reporta exactamente lo mismo, **sin escribir un solo archivo**. Es parte del contrato de la migración, no del runner: el runner no puede impedir que una migración escriba.

### El conjunto de migraciones es de sólo-agregar

> **Nunca se borra una migración**, ni siquiera cuando parece que ya nadie está en ese formato.

Es lo único que permite que alguien parado en una versión vieja llegue a la actual corriendo la cadena entera. "Ya nadie está en ese formato" no es verificable: un clon dormido, una rama vieja, un fork. Borrar la migración convierte eso en un archivo que no se puede leer y del que no queda registro de cómo se leía.

Lo que se acumula es el **build**, no la lectura. Si una migración depende de dos versiones del crate de formato, esas dos quedan en el repo para siempre; pero el binario del día a día linkea sólo la última. Es la distinción que hace que esto no sea lo que § "Por qué no alcanza con que la herramienta lo tolere" descarta.

#### Lo que no se puede romper es el camino, no la lista

La regla protege una propiedad: **desde cualquier formato publicado se llega al actual corriendo la cadena**. Quitar un paso la rompe casi siempre, y por eso "nunca se borra" es la formulación operativa correcta.

Casi siempre y no siempre. Un paso cuyo trabajo hace otro —porque el siguiente aprendió a leer el formato de entrada del anterior— se puede **retirar** sin romper nada: el camino sigue completo, con un salto menos. Pasó con `bilinker-001-capture-split`, que `002` absorbió al leer también la forma embebida.

Retirar no es borrar, y la diferencia está en el ledger:

| | Registro de migraciones | Ledger de un repo |
|---|---|---|
| **Retirar** | sale | **se queda** |
| **Borrar** | sale | sale |

El ledger registra qué le pasó a este repo, no qué sabe hacer este binario. Sacar de ahí un id que corrió sería reescribir el pasado, y además volvería a correr esa migración en el próximo `migrate`.

La prueba de que un retiro es legítimo es un test: un repo en el formato de entrada del paso retirado llega al actual. Sin ese test es un borrado con otro nombre.

### Una migración conoce los dos formatos que puentea

Una migración lleva archivos de lo que entiende el formato N a lo que entiende el N+1, así que **declara ese par y depende de las dos versiones**. No hay otra forma de que un componente lea los dos.

De ahí sale algo que no es evidente: **la verificación de que la migración no perdió nada la hace la migración**, no un comando que compare dos árboles. Un comando así linkea un solo parser y sólo puede leer uno de los dos lados.

## Una migración no es todo el proceso

Una migración es lo que este documento define: transformación sintáctica, idempotente, con `--dry-run`, que no consulta git. El **proceso** de migración es la transición completa, y tiene pasos que no son migraciones — sobre todo los **cortes**, donde el formato nuevo reemplaza al viejo en el árbol de trabajo.

Un corte mueve directorios, hace backup, y se puede revertir. No transforma archivos: los intercambia. Aplicarle las reglas de una migración no tendría sentido, y darle una excepción a las reglas las vaciaría.

**Entran en la misma secuencia numerada y el mismo ledger, y los corre otro comando.** La numeración compartida es lo que hace que el orden entre un corte y las migraciones que lo rodean esté escrito en un solo lugar. Que el comando sea otro es lo que salva las invariantes: el runner de migraciones sigue sin consultar git y sin commitear, porque no es él quien corta.

Una consecuencia práctica: **la entrada en el ledger va cuando el paso terminó**, no cuando empezó. Un corte que registrara al generar su carpeta dejaría el repo marcado como migrado mientras sigue corriendo el formato viejo.

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
6. Una migración no descarta una decisión: donde no puede llevar un campo hacia adelante registra el hueco, y nunca lo reemplaza por una respuesta.
7. El runner nunca commitea ni modifica el historial.
