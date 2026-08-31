# Propuesta: verificar una ref ajena, y reparar la disyunción rota

**Estado:** las capas 1 y 2 se implementaron como [`verify-ref`](../commands/verify-ref.md). Lo que queda acá es **el replay en CI** —la capa 3, la única que necesita resolver una query— y la reparación de la disyunción rota.

Diferido por [ADR-0004](../.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md), que decide **dónde viven los bilinks** y deja fuera de esa decisión la superficie de operación que se resuelve mejor con la ref ya andando.

> Lo de abajo es el diseño con el que se escribió `verify-ref`, y se deja como el argumento del que salió. Para qué verifica hoy y cómo se corre, la spec del comando.

## `verify-ref` — la ref que construyó otro

Las [dos verificaciones previas](../concepts/ref.md#las-dos-verificaciones-previas) protegen lo que bilinker escribe. Una ref traída de un proveedor la construyó otra herramienta, y el consumidor no tiene con qué contrastarla: si su árbol de código no es el que su segundo padre dice, calcula drift contra un árbol fabricado sin forma de enterarse.

Un `verify-ref` recorre la ref chequeando los dos enunciados de la [invariante de fidelidad](../concepts/ref.md#la-invariante-de-fidelidad) — comparaciones de tree oids, sin tree-sitter ni rehashear nada.

**No debe chequear que los bilinks estén en `OK`**, por el mismo motivo por el que la invariante no lo dice: una ref con drift es normal, y exigirlo haría imposible [`track`](../commands/track.md).

## Del lado del servidor: tres capas, y casi todo ya existe

La misma verificación, movida a donde puede **rechazar** en vez de sólo avisar. [`ref.md`](../concepts/ref.md#la-ref-es-protegida) enuncia que la ref es protegida; acá va con qué.

### 1 — Config. Cero código.

```
receive.denyNonFastForwards = true
receive.denyDeletes = true
```

`denyDeletes` no es un extra: sin él, *"sólo avanza"* se esquiva borrando la ref y empujándola de nuevo.

### 2 — Un `pre-receive` que reutiliza lo que hay

Nada de esto se construye. `bilink-format` es un crate aparte que *"sólo tiene los tipos y su serialización: no resuelve queries, no consulta git"*, y publica su esquema JSON como artefacto de release **con este propósito escrito**: *"para que un consumidor de la frontera valide antes de interpretar sin adoptar bilinker"*. Un servidor corriendo un hook es exactamente ese actor.

| Se verifica | Con qué | ¿Existe? |
|---|---|---|
| la forma de cada archivo, sin campos desconocidos | el esquema publicado | **sí** |
| el nombre de un capture **es** `sha256(file \0 query \0)[..32]` | `Capture::id()` | **sí, con test** |
| el formato declarado no es más nuevo que el conocido | `.bilink/version` contra la versión del crate | **sí** |
| disyunción y fidelidad | comparación de tree oids | falta el hook |
| [un commit hace una cosa](../concepts/ref.md#un-commit-hace-una-cosa) | cae en uno de los tres tipos y no en dos: los padres y cuál de los dos árboles se movió | falta el hook |
| un capture sólo se **agrega**, nunca se modifica ni se borra | el diff del commit | falta el hook |
| [el mensaje parsea](../concepts/ref.md#el-mensaje-es-el-comando), y su comando es uno del vocabulario | la gramática cerrada | falta el hook |

Ninguna necesita tree-sitter ni ejecutar bilinker.

### 3 — El replay en CI, que cierra la única que falta

Que `accepted.hash` sea de verdad el hash del fragmento **sí** necesita resolver la query. Y ahí sirve que [el mensaje sea el comando](../concepts/ref.md#el-mensaje-es-el-comando): CI no tiene que adivinar qué verificar.

```
1. checkout del árbol del primer padre
2. correr el comando del mensaje, con la versión de Bilinker-Version
3. comparar el tree oid de .bilink/ contra el del commit
```

Si coinciden, el commit es bit a bit lo que la herramienta habría escrito — y eso **subsume** toda la capa 2: no se chequean propiedades una por una, se recalcula todo.

**Barato en bulk.** Los N commits de una aceptación son hijos del mismo merge, así que comparten el árbol de código: un checkout y N replays en cadena.

**Lo que falta de la herramienta es chico**: hoy `accept` escribe y commitea. Para replayar hace falta un modo que produzca el árbol resultante sin commitear —lo que `apply --dry-run` ya hace para su caso— y devuelva su oid.

## Autorización: la fila que falta

[`ref.md`](../concepts/ref.md#autoría-atestación-y-autorización) separa tres cosas y deja la tercera sin dueño: *"Autorización — si tenía derecho a aceptar — no, y está bien que todavía no"*.

El mismo `pre-receive` la llena sin inventar nada: **exigir que todo commit sobre `refs/bilink/*` esté firmado por una clave de una allowlist.** `git verify-commit` ya existe, funciona offline y no pide infraestructura.

### No se puede exigir que lo haya escrito bilinker

Vale dejarlo dicho para que nadie lo intente. Cualquier clave que bilinker tenga en la máquina de alguien es una clave que esa persona tiene: firmar prueba **quién**, nunca **qué programa**. Atestar el productor necesita una frontera de ejecución confiable, que en un clon local no existe.

Y no hace falta. Un commit hecho a mano que cumple las tres capas es indistinguible de uno de bilinker **porque es válido**. Exigir la herramienta además rompería lo que [ADR-0003](../.stratum/impl/docs/adr/0003-formato-captures-y-aceptacion.md) § "El problema real es de bootstrap" pide: *"si la herramienta se rompe, tiene que quedar con qué diagnosticarla."*

La única forma real sería que bilinker corriera como servicio y fuera lo único con permiso de push. Mata el trabajo local y offline, que es el eje del diseño — `check` corre en caliente contra el árbol de trabajo, sin red. No vale el cambio.

## Bilinks sin procedencia

Un `.bilink/` en el árbol que ninguna ref avala, o commiteado en una rama del proyecto, son el mismo problema: **decisiones sin un commit firmado detrás**.

La salida no es descartarlas. Sacarlo con un commit sirve para el caso en que llegaron de un merge de la ref al proyecto, pero destruye trabajo si alguien commiteó bilinks a mano. La salida es **adoptarlas**, con el commit de adopción atribuyéndolas a quien adopta, que es lo único cierto que se puede decir de ellas.

Sin base de merge, toda diferencia es conflicto: ése es el costo de haber perdido la procedencia.

Es también lo que le falta a la verificación de disyunción para poder **reparar** y no sólo abortar. Hoy [`sync`](../commands/sync.md) detecta que el commit del proyecto trae `.bilink/` en su árbol y se para; cómo volver a sacarlo de la rama, y qué hacer con lo que había ahí, es este mismo problema.
