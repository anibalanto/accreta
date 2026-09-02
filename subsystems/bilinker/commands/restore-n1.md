# Especificación: comando `bilinker restore-n1`

## Propósito

Devuelve a un `accepted` el vecindario de nivel 1 que la migración [`bilinker-003-accepted-list`](migrate.md#bilinker-003-accepted-list--de-38-a-40) descartó, leyéndolo del backup que dejó el corte.

Escribe **el contrato** —`hash` y `hash_ast`— y declara con [`link: unknown`](../concepts/bilink.md#el-link-de-un-nivel-del-vecindario-y-su-tercera-forma) que la ubicación no se recuperó, porque el backup no la tiene: los captures de los vecinos entraron al formato con la misma versión que los tiró.

**Es de un solo uso.** Existe porque una migración descartó un campo que tenía en la mano, y su § "Cuándo se retira" dice cuándo deja de existir.

## Firma

```
bilinker restore-n1 [<path>] [--recursive] [--dry-run] [--from <dir>]
```

| Argumento | Descripción |
|---|---|
| `path` | Capa a restituir. Default: capa actual (cwd). |
| `--recursive` | Alcanza también las capas descendientes encontradas en `.stratum/`. |
| `--dry-run` | Muestra qué escribiría sin escribir nada. |
| `--from <dir>` | De dónde leer el backup. Default: `.bilink-formato-3/` al lado del `.bilink/` de la capa. |

`--from` existe porque **el backup puede no estar donde el corte lo dejó.** No está en git —lo tapa el glob `.bilink-formato-*`— así que un clon fresco no lo trae y un árbol limpiado lo perdió; la copia que quedó es un tarball fuera de git. Ver [la propuesta de retención](../proposals/retencion-del-backup-del-corte.md), que existe para que el próximo corte no tenga este problema.

## No es una migración, y la diferencia es verificable

La tentación es registrarla en el ledger con un id, como cualquier otra. **No entra**, y no por gusto:

**Una migración tiene que ser reproducible desde lo que el repo contiene.** Dos personas que corren la misma migración sobre el mismo commit obtienen lo mismo, y eso es lo que hace que el ledger signifique algo. Esto lee un directorio que no está en git y que puede no existir, así que el resultado depende de qué tenga cada máquina. **Un paso así no puede llevar un id en un registro que afirma qué le pasó al repo.**

Y hay tres más, cada una suficiente:

| | |
|---|---|
| **no cruza una versión de formato** | los archivos son `4.1.0` válidos antes y después. Un ledger de migraciones de formato con una entrada que no mueve el formato deja de decir una sola cosa |
| **no la necesitan todos los repos** | sólo los que cortaron con la `003` teniendo vecindarios adquiridos. Una migración se registra igual en un repo donde no había nada que hacer |
| **es condicional y parcial** | saltea endpoints y tiene que decir cuáles. *"La migración corrió"* no distingue haber restituido todo de haber restituido la mitad |

La idempotencia, que es lo que el ledger habría dado gratis, **sale de las condiciones**: después de una corrida el `n` ya no es `declined`, así que la segunda corrida es no-op sin que nadie tenga que recordarlo.

## Las tres condiciones

Un nivel se restituye **sólo si las tres se cumplen**, y **fallar no es lo mismo en la primera que en las otras dos**: la primera dice que no hay hueco que llenar, las otras dos que hay un hueco y no se puede. Sólo las segundas se cuentan como salteadas — contar las primeras sería reportar como pendiente cada endpoint del repo.

### 1 — El `accepted` vivo tiene el `n` en `declined`

Un `n` adquirido después del corte es más nuevo que el backup: alguien lo resolvió con un language server y eso es la verdad de hoy. Un `n` ausente dice que el fragmento no tiene firma resoluble, y el backup no puede contradecirlo.

**Sólo `declined` es el hueco que la `003` dejó**, y es el único caso donde el backup sabe algo que el archivo vivo no. Que no se cumpla no es un salteo: es que ahí no había nada que devolver.

### 2 — El `hash` del fragmento no se movió

**El discriminador está en el archivo**: el `hash` del `accepted` vivo contra el del `accepted` del backup.

El backup describe el código de antes del corte. Si el fragmento cambió desde entonces, ese vecindario era de otra versión de la firma y restituirlo **afirmaría algo falso**: un contrato aprobado para un fragmento que ya no es el que hay.

> **Y esta condición se vence sola.** Cada `accept` sobre un endpoint degradado le mueve el `hash`, y con eso el backup deja de aplicarle — sin que `accept`, `check` ni nada lo diga. **La ventana no se cierra sólo por borrar el backup: se cierra por trabajar.** Al 2026-09-02 ya había 8 endpoints así.

### 3 — Hay exactamente un `accepted`

Con más de una entrada el endpoint está [`CONSENSUS_DIVERGED`](../concepts/bilink.md#más-de-un-accepted-es-un-estado-no-una-forma-de-trabajar) y no hay un valor contra el cual comparar el `hash`. Elegir una sería resolver un desacuerdo entre personas leyendo un backup, que no es algo que un comando pueda hacer.

## Qué escribe

Del `accepted` del backup, donde el nivel era dos hashes sin `link`:

```yaml
      n:
        1:
          hash: d3fba7c71764b82cd17875f63ddba6864e015ddf8bc20adf62419ac44040327a
          hash_ast: 319cfc6fd8d01fa928f12876d9f8a93ff84b98ba92c0fdad3603118b04d2e816
```

Al `accepted` vivo, cuyo `n` decía `declined`:

```yaml
      n:
        1:
          link: unknown
          hash: d3fba7c7…
          hash_ast: 319cfc6f…
```

**Los niveles se copian todos**, no sólo el 1: si el backup tuviera un nivel 2 se restituye igual, con su propio `link: unknown`. Enumerar el 1 sería la clase de lista que envejece con el próximo nivel.

**Y no toca nada más.** Ni el `link` del endpoint, ni su `hash`, ni `agree`, ni la declaración de afuera. Lo único que cambia es el `n` de la entrada aceptada — el campo que la `003` reemplazó.

### Es una decisión, y por eso la escribe esto y no `accept`

Restituir escribe adentro de `accepted`, que es territorio de [`accept`](accept.md). No es una excepción al reparto: **el valor que se escribe es una decisión que alguien ya tomó** —los hashes *son* la aprobación, con su `agree` intacto al lado— y esto la devuelve a donde estaba. No aprueba nada nuevo, y por eso no toca `agree`.

Lo que **no** hace es acuñar captures ni proponer ubicaciones. Eso es `apply`, y llenar los `unknown` es su propia tarea.

## Salida

Se agrupa por capa, y **nombra los que no pudo**:

```
$ bilinker restore-n1 . --recursive

accreta
  restituidos  11
  salteados     4   el hash del fragmento se movió: el backup es de otra versión
    696c6d76.1  8b893c60.1  a5f2ebba.1  e671c12e.1

subsystems/bilinker/.stratum/impl
  restituidos  11
  salteados     4   el hash del fragmento se movió
    696c6d76.1  8b893c60.1  a5f2ebba.1  e671c12e.1

131 restituidos, 8 salteados, en 8 capas
```

**Los salteados van con su uuid y no sólo contados.** Un endpoint salteado se queda en `declined`, y `declined` es una decisión: `check` lo reporta limpio y **no vuelve a aparecer en ningún inventario**. La salida de este comando es el único registro de que ahí había un contrato, así que el commit que lo corre tiene que poder nombrarlos.

## Código de salida

| Código | Condición |
|---|---|
| 0 | La corrida terminó. Restituyó lo que las condiciones permitían y dijo qué salteó. |
| 1 | No pudo leer el backup o no pudo escribir un bilink. Nada queda a medias. |
| 2 | La versión de formato de la capa no se entiende. No se restituyó nada. |

**Los salteados no lo hacen fallar.** No son un error de la corrida sino un hecho sobre el pasado, y el comando hizo lo único que podía hacer con ellos: decirlo. Salir con 1 los volvería indistinguibles de un backup ilegible, que sí se puede arreglar.

## Propiedades garantizadas

- **`--dry-run` no escribe nada.**
- **Idempotente**: la segunda corrida no restituye nada, porque la condición 1 ya no se cumple.
- **No inventa un `link`.** Un vecindario restituido queda con la ubicación declarada como desconocida, nunca con captures adivinados.
- **No baja cobertura.** Un endpoint que no cumple las condiciones se queda exactamente como estaba.
- **Cada nivel restituido conserva su `hash_ast` o su ausencia**, sin recalcular: el fold es todo-o-nada sobre los vecinos y este comando no tiene con qué recomponerlo.

## Cuándo se retira

Cuando ningún repo alcanzable tenga un `accepted` con `n: declined` cuyo backup del corte diga otra cosa — o cuando ya no queden backups del corte, que es lo mismo desde el lado de esta herramienta.

Es la misma asimetría de [`bilinker-001-capture-split`](migrate.md#bilinker-001-capture-split-retirada): el código de un paso de un solo uso se va, y lo que queda es el registro de que corrió. Acá ese registro no es una entrada de ledger —esto no es una migración— sino los commits que lo corrieron, con los uuids salteados adentro.
