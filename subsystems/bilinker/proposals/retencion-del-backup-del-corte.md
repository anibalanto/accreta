# Propuesta: el backup de un corte tiene un hogar y una fecha

**Estado:** no implementado. Sale de [`1x`](../../../.worklist/insecure/all/1x.task.md), que reclama la política, y del costo que la `003` cobró por no tenerla. Para el corte de la `003` **no se aplica**: su backup ya existe y su ventana se cierra sola cuando [`43`](../../../.worklist/insecure/all/43.user-story.md) restituya. Esto es para el próximo.

`migrate --cut` deja el formato anterior en `.bilink-formato-<N>/` y lo dice al terminar. **Nadie dice dónde vive eso ni cuándo se va**, y las dos ausencias son la misma: un directorio suelto no tiene ni dueño ni vencimiento.

## Lo que la `003` demostró

El corte a 4.0.0 degradó 139 vecindarios a `n: declined` y con eso tiró dos sha256 por endpoint. Se pueden recuperar, y **están vivos por accidente**: siguen en los `.bilink-formato-3/` de cada capa porque nadie los borró todavía.

Y no están en ningún lado más:

```
$ git ls-files .bilink-formato-3        → 0
$ git check-ignore -v .bilink-formato-3 → .git/info/exclude:12:.bilink-formato-*
$ git ls-tree -r <ref> | grep -c formato → 0
```

Ni en la rama, ni en la ref, tapados por el glob. **Existen como archivos sueltos en un clon local y nada más**, así que un `git clean` o un reclone se lleva 98 contratos de `hsi` sin que nada falle hasta que alguien intente leerlos.

> **Un backup que nadie borra no es una política de retención, es un accidente que a veces sale bien.** Que esta vez saliera bien es lo que hace difícil verlo: el mecanismo que salvó los contratos fue otro defecto sin arreglar.

Y salió bien **a medias**, que es lo que un accidente hace. El backup no se borró, y aun así **8 de los 139 quedaron sin recuperar**: aceptar un endpoint cuyo campo la migración descartó le mueve el `hash`, y el backup deja de aplicarle. Ninguna herramienta lo dice, porque nada sabe que hay algo pendiente. **Un backup sin fecha se vence solo**, y no el día que alguien lo borra sino el día que alguien sigue trabajando.

## Lo que se propone

**El backup de un corte se commitea, y el commit dice cuándo se puede borrar.**

`migrate --cut` escribe el formato anterior bajo `recovery/<migración>/` en el repo de la capa, en un commit propio, y el mensaje lleva la condición de borrado: *"se borra cuando `<verificación>` pase"*. El glob del exclude deja de taparlo porque el path deja de matchear.

Con eso el backup gana las tres cosas que un directorio suelto no puede tener: **está en más de una máquina**, porque se pushea con la capa; **tiene historia**, así que borrarlo no lo pierde; y **tiene vencimiento escrito donde se lo va a leer**.

### Por qué no una ref

Es lo primero que se piensa, porque [los bilinks ya viven en una ref](../concepts/ref.md) justamente para no estar en la rama, y un backup es más de lo mismo: datos que no son el proyecto.

**Y no alcanza, por una razón operativa y no de diseño: una ref no se pushea sola.** Sin `push refs/backup/*` explícito el backup queda igual de local que el directorio suelto, y el problema que esta propuesta existe para resolver es exactamente ése. Una ref arregla *dónde no molesta* y no arregla *que haya una sola copia*.

`refs/bilink/*` no sufre eso porque su push está en el flujo de todos los días. Un backup se escribe una vez cada corte de formato, así que cualquier paso que dependa de que alguien se acuerde, no va a pasar.

### Y por qué no en cada capa sin más

Commitear el backup en la capa que describe es lo correcto —viaja con lo que respalda— y es lo que la propuesta dice. Lo que **no** alcanza es hacerlo sin el path nuevo: el glob `.bilink-formato-*` lo tapa, y exceptuarlo capa por capa es N ediciones de `.git/info/exclude` que no viajan en ningún clon.

De ahí que el hogar sea un path que el glob no matchea, y no una excepción al glob.

## La regla, que es lo que falta en la spec

> **Un corte que deja un backup deja también la condición para borrarlo.** Sin condición no hay retención: hay un directorio que sobrevive hasta que alguien lo limpie por error o por prolijidad, y las dos son la misma pérdida.

Y la condición no puede ser *"cuando el corte quedó bien"* en abstracto. Para el `003` la condición real era **que los 139 vecindarios estuvieran restituidos**, que es un enunciado que la migración misma no podía escribir: descartó el campo, así que no sabía que había algo pendiente. De ahí la otra mitad: **una migración que no puede llevar un campo hacia adelante registra el hueco**. El valor con que se escribe ya existe —[`unknown` en el `link` de un nivel](../concepts/bilink.md#el-link-de-un-nivel-del-vecindario-y-su-tercera-forma)— y la regla general la escribe [`49`](../../../.worklist/insecure/all/49.task.md). Con el hueco escrito, la condición de borrado es *"no queda ningún `unknown`"*: se calcula en vez de estimarse.

Las dos van juntas y en ese orden: el hueco declarado es lo que hace que la fecha del backup se pueda derivar en vez de estimar.

## Qué toca cuando se haga

[`commands/migrate.md`](../commands/migrate.md), que es hoy el único lugar donde el backup del corte se nombra: dónde lo escribe, qué dice al terminar, y la regla. `--rollback` restaura desde el directorio suelto, así que mover el hogar lo mueve a él también, o lo saca — es una de las tres preguntas abiertas de [`1x`](../../../.worklist/insecure/all/1x.task.md).
