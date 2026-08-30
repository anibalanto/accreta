# La cache

Todo lo que se puede reconstruir de los demás archivos vive en `.bilink/cache/state`, fuera de git.

El criterio es uno solo: **si un valor se puede recalcular, no va en el bilink.** Lo que queda versionado es lo que nadie puede reconstruir — la declaración (`link`) y la decisión (`accepted`).

## Qué contiene

```
.bilink/
  cache/
    state        ← un archivo por capa
```

| Campo | De quién | Qué es |
|---|---|---|
| `range` | del capture | Dónde cayó el fragmento en la última resolución. |
| `state` | del capture | Si la ubicación resuelve. |
| `state.N` | del bilink | Si lo que hay coincide con lo aceptado, por endpoint. |
| `commit` | del bilink | En qué commit el fragmento quedó con el contenido aceptado. |

**Es un archivo por capa**, igual que `index/index`: reescritura atómica, menos inodos, y cero conflictos de merge porque no está versionado.

## Estar fría es normal

La cache no está en git, así que no tenerla es un estado corriente y no una anomalía: un clon fresco, un cambio de rama, después del corte de formato, `rm -rf .bilink/cache/`, u otra máquina.

Adentro conviven **dos clases de derivado con garantías distintas**, y conviene no tratarlas como una sola:

| | Cache fría significa | Quién la escribe |
|---|---|---|
| `state` · `state.N` · `range` | **no disponible** — hay que correr `check` | `check` |
| `commit` | **más lento**, nunca no disponible: se re-deriva a demanda | `accept`; re-derivable por quien lo necesite |

Por eso que `commit` viva acá no bloquea a nadie. Donde hace falta —recuperar el texto aceptado, atribuir un cambio a un commit— sólo corre sobre endpoints **ya no-OK**, así que el costo está acotado por lo que está roto y no por el total.

### `commit` lo escribe `accept`, y sigue siendo derivado

Es raro que un derivado lo escriba `accept` y no `check`, y es deliberado: se calcula una vez, en el momento en que hay todo el contexto a mano. Lo que define un derivado es que se pueda reconstruir, no quién lo escribió.

## Sabe a qué rama corresponde

`cache/state` registra el commit de `refs/bilink/<branch>` del que salió.

Sin eso, una cache por capa devolvería estados de otra rama en silencio cuando el árbol de trabajo cambia de rama. Con el commit anotado **la cache se invalida sola**: si no coincide, no sirve, y hay que correr `check`.

Es la mitad derivada de un par. La otra es `.bilink/head`, que dice a qué rama y a qué commit corresponde el `.bilink/` del árbol — ver [ADR-0004](../.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md). La cache protege al derivado; `head` protege a la fuente, y por eso `head` no puede vivir acá adentro.

## Por qué `resolved_at` no está

No se mudó a la cache: **se fue del formato.**

Tenía un solo uso funcional —ser el baseline de `git log --since=<resolved_at>` para atribuir un cambio a un commit— y ese uso está dominado por `commit`. Un timestamp es un mal baseline para arqueología en git: las fechas se desordenan con rebases, cherry-picks y skew de reloj, mientras que `git log <commit>..HEAD -- <file>` recorre por ancestría y es exacto.

No era un campo que sobraba: era una segunda respuesta, peor, a una pregunta ya contestada.

Y de paso arregla un defecto visible: `check` escribía `resolved_at` en cada corrida, así que **ensuciaba archivos versionados sin cambio semántico**. Al escribir ADR-0003, `git status` sobre accreta mostraba 16 `.bilink` modificados y el diff completo de cada uno era una línea de `resolved_at`.

## Invariantes

1. Todo lo que está en la cache es reconstruible sin ella.
2. La cache nunca es fuente de verdad. Si contradice a los archivos versionados, gana el archivo.
3. `check` es el único que escribe `state`, `state.N` y `range`.
4. La cache registra el commit de `refs/bilink/<branch>` al que corresponde, y se descarta sola si no coincide.
5. La cache no se versiona.
