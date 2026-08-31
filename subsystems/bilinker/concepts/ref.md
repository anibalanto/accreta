# La ref de bilinks

Ninguna rama del proyecto contiene `.bilink/`. Los bilinks viven en **`refs/bilink/<branch>`**, una ref por rama del proyecto: los de `rc-2.35` están en `refs/bilink/rc-2.35`.

La decisión y su motivación viven en [ADR-0004](../.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md). Acá está lo que todo comando que escribe sobre la ref tiene que cumplir — [`init`](../commands/init.md), [`sync`](../commands/sync.md), [`track`](../commands/track.md), [`adopt`](../commands/adopt.md), y también [`accept`](../commands/accept.md) y [`apply`](../commands/apply.md), que commitean como parte de su acto.

## Fuera de `refs/heads/`

La ref no es una rama. `git branch -a` no la lista, la UI de la forja tampoco —los listados de ramas muestran `refs/heads/*`— y `git log --branches` la ignora. Es lo que hacen `git notes` con `refs/notes/*` y Gerrit con `refs/changes/*`.

Requiere refspecs explícitos para push y fetch. Los pone [`init`](../commands/init.md); nadie los tipea.

## Qué lleva adentro cada commit

**El árbol del proyecto más `.bilink/`.** No una rama huérfana con sólo los bilinks: cada commit es un snapshot consistente por construcción, y eso es lo que hace simple el caso remoto — el consumidor trae una sola ref y obtiene las declaraciones del proveedor junto con exactamente el código al que apuntan.

El árbol no se duplica: git comparte los objetos con la rama del proyecto, así que el costo marginal de cada snapshot es la carpeta `.bilink/` y nada más.

## La invariante de fidelidad

Son dos enunciados, los dos verificables sin tree-sitter:

> **1.** El árbol de código de todo commit de `refs/bilink/<branch>` es **idéntico al del commit del proyecto absorbido más recientemente** — el segundo padre del merge más cercano siguiendo primeros padres, o el propio si el commit *es* ese merge.
> **2.** El commit del proyecto contra el cual `accept` o `apply` calcularon su trabajo tiene que estar absorbido antes de commitear.

El primero es una comparación de tree oids: exacta, y barata porque el merge más cercano casi siempre es el commit mismo o el anterior. El segundo garantiza que el commit absorbido sea **el correcto**. Y la exigencia de [`accept`](accept.md) —falla sobre un archivo sucio— es lo que impide que el enunciado 2 se cumpla sobre un commit que no contiene lo que se aceptó.

**Absorber no es un comportamiento por comando: es una precondición de todo commit sobre la ref.** No hay una tabla de quién absorbe y cuándo, ni un `sync` implícito adentro de cada comando: hay una condición que se verifica y, si no se cumple, se cumple absorbiendo **en el mismo commit**. Cuando ya se cumple —el proyecto no se movió desde la última absorción— el acto es un commit común de un solo padre, y el enunciado 1 sigue valiendo porque su árbol de código no cambió.

### Lo que la invariante no dice

Que los bilinks del commit estén en `OK`. Una versión más fuerte —*"todo `accepted.link` resuelve, contra el árbol de ese commit, a su `accepted.hash`"*— prohibiría que la ref contenga drift, cuando el drift es el estado normal que la herramienta existe para reportar. Con [`track`](../commands/track.md) queda evidente: una rama nueva hereda bilinks que describen código de otro commit, y casi seguro arranca con drift. Correcto que lo haga.

## Cómo se arma el commit

```
read-tree     <commit del proyecto absorbido>       ← el nuevo, si hay que absorber;
update-index  únicamente .bilink/                     el vigente, si ya está absorbido
```

Nada del árbol de trabajo fuera de `.bilink/` entra jamás al commit de la ref. `cache/`, `index/`, `head` y `.bilink-migrate-*` quedan fuera del índice de bilinker, no sólo del índice del proyecto. Y de adentro de `.bilink/` queda afuera **el clon de un proveedor** —`.bilink/<alias>/`— que es otro repo entero y no contenido de esta capa.

**La lista es la regla, y no hay ningún `.gitignore` detrás.** El árbol se construye enumerando: lo que no está en la lista entra, y lo que no tiene que entrar se saca de la lista — no agregando una línea a un archivo versionado. La exclusión del lado del proyecto ya la puso [`init`](../commands/init.md) con un solo patrón, y `.bilink/.gitignore` gobierna otra cosa: los derivados, para el repo que **todavía no cortó** y tiene `.bilink/` en su rama.

Y por eso la fusión de contenido de un [`adopt`](../commands/adopt.md) queda confinada a `.bilink/`: el árbol se **construye**, no se fusiona. Un `git merge` a secas entre dos refs fusionaría dos árboles de código enteros.

### Las dos verificaciones previas

Corren antes de cada commit, sobre lo que bilinker está por escribir:

1. **Disyunción.** El **árbol** del commit del proyecto que se absorbe no contiene `.bilink/`. Si lo contiene, alguien mergeó la ref al proyecto o commiteó bilinks a mano. Se aborta.

   Es sobre el árbol y no sobre el diff a propósito: el commit que *borra* `.bilink/` tiene un diff que lo toca y un árbol que no, y es exactamente el commit que hay que poder absorber — el `X` del corte es exactamente eso.

2. **Fidelidad.** El árbol de código del commit nuevo es idéntico al del commit del proyecto absorbido vigente. Comparación de tree oids; si falla, abortar.

Con eso, *"la ref nunca se mergea de vuelta"* deja de ser una regla de buena conducta y pasa a ser una condición **chequeada**. No es exigible como invariante —nadie puede impedir que alguien mergee— pero sí detectable antes de que contamine nada.

**Verificar una ref ajena** —la de un proveedor, construida por otra herramienta, donde el consumidor no tiene otra fuente contra la cual contrastar— es un problema distinto, y está en [`proposals/verificar-ref-ajena.md`](../proposals/verificar-ref-ajena.md).

## Evolución: merge, y en una sola dirección

Cuando la rama del proyecto avanza, la ref la absorbe con un merge. Nunca rebase —reescribiría la historia que el `get --diff` de [ADR-0005](../.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md) necesita recorrer hacia atrás— y nunca cherry-pick, que copia los commits en vez de referenciarlos.

Los merges no conflictúan: la ref **no modifica archivos del proyecto** —su único diff es agregar `.bilink/`— y el proyecto no toca `.bilink/`. Los dos lados escriben conjuntos disjuntos, y eso se cumple **desde el corte**, porque la ref nace de un commit del proyecto en el que `.bilink/` ya no existe.

```
rc-2.35:             X ─────── B ─────── C ─────────────────── D ─────── E
                     │                   ╲                     ╲         ╲    ← 2º padre: lo que se absorbe
refs/bilink/rc-2.35: ●0 ───────────────── ●1 ─────── ●2 ─────── ●3 ────── ●4
                     corte                accept     apply      accept    sync
                                          (merge C)  (1 padre)  (merge D) (merge E)
```

| | Acto | 2º padre | Diff de `.bilink/` contra su 1er padre |
|---|---|---|---|
| `●0` | el corte | — | agrega `.bilink/` entero |
| `●1` | `accept 7f3d8e9a.0` | `C` | `accepted` completo de un endpoint |
| `●2` | `apply -y` (3 renames) | — | tres `link` repuntados; ningún `accepted` tocado → los tres quedan `RELOCATED` |
| `●3` | `accept . --place` | `D` | los tres `accepted.link` → los tres vuelven a `OK` |
| `●4` | `sync` | `E` | **vacío** — el proyecto avanzó y nadie aceptó nada |

`●2` es el caso que hace la diferencia: nadie tocó el proyecto entre `●1` y él, así que no hay merge — y sin embargo su árbol de código sigue siendo el de `C`, que es lo que la invariante de fidelidad pide.

El corte (`●0`) es el único commit de la ref sin ningún commit del proyecto absorbido *por debajo*: nace de `X` como padre único, y ahí la fidelidad se lee contra `X` mismo.

## La correspondencia con el proyecto es el segundo padre

Y por lo tanto un hecho de git y no una convención de nombres: se recorre con `git log --parents`, y `git branch --contains` y `git merge-base` la responden solas. Un commit de un solo padre la hereda del merge más cercano hacia atrás.

No hace falta ningún identificador propio — el hash del commit ya lo es, y el estado anterior es `refs/bilink/rc-2.35~1`.

**El recorrido se frena al salir de la ref, y el freno es la disyunción.** Los commits de la ref llevan `.bilink/` en su árbol y los del proyecto no, así que buscar hacia atrás el commit absorbido se para en el primero que no lo lleva — y ése *es* la respuesta.

Sin ese freno el corte daría la respuesta equivocada: `●0` tiene un solo padre, `X`, así que el recorrido seguiría hacia atrás por la historia del proyecto y devolvería el segundo padre del primer merge ajeno que encontrara. Con el freno, `●0` contesta `X`, que es exactamente contra lo que su fidelidad se lee.

Es la segunda cosa que la disyunción compra. La primera es detectar que alguien mergeó la ref al proyecto; ésta es poder distinguir los dos lados sin marcarlos.

**Y vale para todo recorrido de la ref, no sólo para ése.** `git log --first-parent refs/bilink/<branch>` tampoco se detiene solo: al llegar al corte sigue hacia atrás por la historia del proyecto. Los commits propios de la ref son los que llevan `.bilink/` en su árbol, y ésa es la lista sobre la que se leen el registro de decisiones y los candidatos de [`track`](../commands/track.md).

Sin el freno, `track` sobre una rama vieja elige mal de forma sistemática: el commit más viejo del proyecto es ancestro de cualquier rama, así que siempre califica y siempre gana. Un `--first-parent` sin acotar convierte *"heredá los bilinks más nuevos que te sirvan"* en *"heredá de la raíz"*.

Y se lee bien: **`git log --first-parent refs/bilink/rc-2.35` muestra sólo la evolución de los bilinks**, ocultando la historia del proyecto absorbida.

```
$ git log --first-parent --format='%h %an  %s' refs/bilink/rc-2.35
b1e3f55  Kim     sync: rc-2.35 hasta E
9c1f0ab  Ana     accept --place: 3 endpoints tras el rename de sciplink.rs
4e77d20  Ana     apply: 3 renames — sciplink.rs → scip/link.rs
77a0c94  Luis    accept 7f3d8e9a.0: spec de check ↔ check_structural
0af3c12  Luis    corte: bilinker-003-immutable-captures
```

Ése es el registro de decisiones: quién aceptó qué y cuándo, sin una sola línea del historial del proyecto de por medio.

**Granularidad: un commit por invocación de `accept`, no por aceptación.** La granularidad sigue al **acto**, no al objeto. `accept <uuid>.0` da un commit; `accept .` da un commit, con el mensaje enumerando los endpoints — porque es una persona mirando y decidiendo **una vez**, y partirlo en N commits firmados no agrega verdad.

**Deshacer una aceptación** no necesita `git revert`: es reescribir su `accepted` con los valores anteriores, un commit nuevo, leídos de `refs/bilink/<branch>~n`.

### Antes del corte no hay ref, y el commit no ocurre

Un repo que todavía no corrió el corte `005` tiene sus bilinks en la rama del proyecto y ninguna `refs/bilink/*`. Ahí `accept` y `apply` no escriben ningún commit sobre la ref — no hay ninguna— y los cambios quedan en el árbol, visibles con `git status`, para que los commitee quien trabaja. Es como funcionaba antes y sigue siendo correcto para ese repo.

**La existencia de la ref de la rama es lo que enciende el commit del acto.** No hace falta un flag ni una versión de formato: el corte *es* el interruptor, y lo que lo vuelve honesto es que crear la ref sea un acto explícito de [`track`](../commands/track.md).

Eso es lo que permite que el binario nuevo corra sobre repos que todavía no cortaron — que es exactamente lo que la transición necesita, y lo que [ADR-0003](../.stratum/impl/docs/adr/0003-formato-captures-y-aceptacion.md) § "El problema real es de bootstrap" pide: si la herramienta se rompe, tiene que quedar con qué diagnosticarla.

## El índice propio

Los `.bilink/` están en el árbol de trabajo, no en un worktree aparte: quien usa bilinker o lattice quiere esos archivos a mano. El proyecto los ignora vía `.git/info/exclude` — **no vía `.gitignore`**, que está versionado y modificarlo tocaría la rama del proyecto.

Ignorados a secas, los cambios que escribe `accept` no aparecerían en ningún `git status`: para el índice del proyecto son archivos ignorados, y la ref donde cuentan no está checkouteada.

**Bilinker usa su propio `GIT_INDEX_FILE` sobre el mismo árbol de trabajo**, contra `refs/bilink/<branch>`. El mismo `.bilink/` queda ignorado por el índice del proyecto y trackeado por el de bilinker, que así recupera `status` y `diff` reales sin ensuciar los del proyecto. Es el patrón conocido de los dotfiles en repo bare.

El índice vive en `.git/bilink/index` — dentro de `.git/`, porque es por clon y no se versiona, y con nombre propio porque no es el índice del proyecto.

### La ref es por repo, y el recorrido se para en su frontera

Un solo `refs/bilink/<branch>` cubre **todas las capas de un repo**, estén donde estén: el commit lleva el árbol del proyecto entero, así que los `.bilink/` de todas sus capas entran en el mismo snapshot. Es la misma razón por la que la exclusión de [`init`](../commands/init.md) es un patrón y no una lista.

Pero un subdirectorio con su propio `.git` **es otro repositorio**, y sus bilinks son suyos. El recorrido se para ahí.

No es hipotético: en accreta cada subsistema tiene su capa de implementación en un repo propio, gitignoreado por el padre. Sin ese freno, el corte del padre se traga los bilinks de los hijos — y quedan en un snapshot cuyo árbol de código **no los contiene**, así que ni la disyunción ni la fidelidad hablan de ellos. Los dos chequeos pasarían, y estarían mirando otra cosa.

Es [`configuration.md`](configuration.md) § "Uso con múltiples capas y repositorios" dicho desde la ref: *"cada capa puede ser un repositorio git independiente, y bilinker siempre opera en el contexto de una sola"*. La frontera del repo es la de la ref.

**`check` corre en caliente.** Bilinks y código vivo en el mismo directorio, con los cambios sin commitear a la vista, que es lo que [`check`](../commands/check.md) ya exige al comparar contra el árbol de trabajo y no contra HEAD. Si se comparara contra la copia de código de la ref, quien rompe un vínculo no se enteraría hasta que alguien sincronice.

**La asimetría local/remoto.** Localmente el código sale del árbol de trabajo, así que la foto de la ref puede estar atrasada sin afectar un `check`. Remotamente el consumidor sólo tiene la ref, así que esa foto **es** el código con el que verifica. Por eso la absorción no es un requisito para observar, sino para que el snapshot sea cierto para quien no tiene otra cosa.

## `.bilink/head` — de dónde salió el árbol

Materializar `.bilink/` en el árbol y excluirlo del índice del proyecto tiene un filo: **`git checkout` no lo toca**. Cambiar de rama mueve el código y deja los bilinks donde estaban. Quedás con el código de `B` y los bilinks de `A`, y nada te avisa.

`.bilink/head` es un archivo con dos valores —la rama y el commit de `refs/bilink/<branch>` que le corresponde:

```
branch rc-2.35
commit 9c1f0abf3e21d5c4b7a08e6f2d1934ac5b7e0f18
```

Lo escribe **tanto la materialización como todo commit sobre la ref**: si `accept` avanza la ref, el árbol pasa a corresponder al commit nuevo y `head` tiene que decirlo, o la guarda se dispararía después de cada aceptación.

No se commitea: se suma a `cache/` y `index/` en la lista de lo que queda fuera del índice de bilinker. No es configuración, es estado del árbol de trabajo — exactamente lo que `HEAD` es para git.

**Sin punto**, aunque `.bilink/` tenga adentro un `.{alias}.toml`. La regla que gobierna ese punto es la de Stratum, donde un dotfile describe a su hermano sin punto; `head` no describe a nadie. Y `.bilink/` ya es un directorio oculto: adentro, el punto no esconde nada, que es por lo que `cache/state` e `index/index` tampoco lo llevan.

### La materialización es automática

Cualquier comando compara `head` contra la rama del proyecto, y si no coinciden materializa el `.bilink/` de la ref correcta y sigue. No hay comando de más que tipear, ni pregunta que contestar — no hay nada que decidir.

### La guarda es una sola, la de git

Si el `.bilink/` del árbol difiere del commit que `head` nombra, hay trabajo que no está en ninguna parte —`.bilink/` está fuera del git del proyecto— y materializar lo destruiría. Ahí se para y se avisa, igual que `git checkout` se niega a pisar cambios.

En el flujo diseñado esa ventana no existe: `accept` y `apply` commitean sobre la ref como parte del acto, así que el árbol queda limpio contra el índice propio. Queda abierta sólo para un `apply` que crashea a mitad o una edición a mano, y las dos merecen un humano.

### Por qué un archivo y no inferirlo de la cache

Porque [la cache](cache.md) es un derivado y estar fría es un estado normal —clon fresco, otra máquina, `rm -rf .bilink/cache/`—, o sea que no puede responder justo cuando más falta hace. Y peor: haría que un derivado autorice sobrescribir la fuente.

Son **dos marcadores para dos preguntas distintas**, y los dos hacen falta: `head` responde *"¿son éstos los bilinks que corresponden?"* y protege la fuente; el commit que anota `cache/state` responde *"¿estos estados se calcularon sobre estos bilinks?"* y protege el derivado.

### En `HEAD` desacoplado no se materializa nada

A mitad de un rebase con conflictos no hay rama actual contra la cual comparar, y adivinar una sería peor que no hacer nada. Ahí los comandos de lectura corren contra lo que `head` dice, avisando; los que commitean sobre la ref se niegan hasta volver a una rama.

## Tres desajustes, tres arreglos

`head` no se mete con el rebase, y está bien que no. Un `git rebase main` estando en `feature/x` te deja en `feature/x`, y no toca `refs/bilink/feature/x`: `head` sigue diciendo `(feature/x, ●a)` y todo coincide. No hay materialización que disparar, porque los bilinks no se movieron — se movió el código debajo.

| Qué se movió | Cómo se detecta | Qué lo arregla |
|---|---|---|
| la rama (`checkout`) | `head` ≠ rama actual | materializar — automático, sin comando |
| el código de la rama (commit, rebase, merge) | el commit absorbido ≠ tip de la rama | [`sync`](../commands/sync.md), o el `accept`/`apply` siguiente, que absorbe igual |
| las decisiones del vecino (rebase sobre otra rama) | `status` lo avisa: la ref del vecino avanzó | [`adopt <rama>`](../commands/adopt.md) — con `--dry-run` para verlo antes |

Sólo el primero puede resolverse solo, y por eso es el único que se resuelve solo: los otros dos escriben un commit sobre la ref, y ninguno de los dos es una decisión que la herramienta pueda tomar por su cuenta.

## Auditoría: contra la ref, no contra la rama

La rama del proyecto no está protegida: se rebasea, se force-pushea, se cambia. Pero la ref **absorbe los commits del proyecto como segundos padres**, así que todo commit alguna vez absorbido queda alcanzable desde la ref para siempre.

```
git merge-base --is-ancestor <commit> refs/bilink/<branch>   ← precondición
git log <commit>..refs/bilink/<branch> -- <file>              ← la ventana
```

Si aun así no es ancestro —commit nunca absorbido, clon superficial del proveedor— degradar a *"fuente desconocida"* en vez de devolver un rango inflado.

Y de ahí sale la propiedad que la task `16` verifica: **la ref es lo que vuelve inmutable el `commit` de una aceptación**. Un rebase de la rama del proyecto saca ese commit de `main`, pero no lo destruye, porque `refs/bilink/<branch>` lo alcanza como segundo padre de un merge. Protege al `commit` guardado en [la cache](cache.md) y protege también a su re-derivación, que camina la historia del archivo y necesita que esos commits sigan existiendo.

## Autoría, atestación y autorización

Tres cosas distintas que conviene no mezclar:

| | Qué es | Con qué se hace | ¿Existe hoy? |
|---|---|---|---|
| **Atribución** | quién *dice* que fue | `author`/`committer` del commit de la ref | no — `accept` no registra a nadie |
| **Atestación** | prueba de que fue | `git commit -S` · `gpg.format=ssh` · `git verify-commit` | disponible, offline, sin infraestructura |
| **Autorización** | si tenía derecho a aceptar | gobernanza | no, y está bien que todavía no |

Para auditar y revertir alcanza con la atribución; ante alguien que no confía, con la firma. El autor de git es auto-declarado: lo que constituye atestación es la firma, no el campo.

Ésa es la respuesta a por qué la aceptación no necesita un archivo propio para ser revisable: **la superficie de revisión es la de bilinker, no la de la forja.** La forja no muestra la ref, pero `status`, `diff` y `log` sobre el índice y la ref propios sí, y el artefacto firmable es el commit.

**Bendecir lo mismo no diluye la responsabilidad.** Dos bilinks pueden tener un `accepted` idéntico, pero los actos que los escribieron son dos commits distintos, cada uno con su autor y su firma. La responsabilidad vive en el commit que escribió el valor, no en el valor.

## Invariantes

1. Los bilinks viven en `refs/bilink/<branch>` y ninguna rama del proyecto los contiene.
2. El árbol de código de todo commit de la ref es idéntico al del commit del proyecto absorbido más recientemente.
3. Ningún commit sobre la ref se escribe sin que el commit del proyecto contra el cual se calculó esté absorbido.
4. La ref no se rebasea ni se cherry-pickea nunca. Es append-only, y todo fetch es fast-forward.
5. El árbol del commit de un commit sobre la ref se construye con `read-tree` del absorbido más `update-index` de `.bilink/`. Nada del árbol de trabajo fuera de `.bilink/` entra.
6. La ref es por repo: cubre todas las capas de ese repo y ninguna de un repo anidado.
7. `.bilink/head` dice a qué rama y a qué commit de la ref corresponde el `.bilink/` del árbol, lo escriben tanto la materialización como todo commit sobre la ref, y ningún comando opera sobre un `.bilink/` que no corresponde a la rama actual.
8. La puesta a punto de un clon —exclusión, refspec y materialización— es [`init`](../commands/init.md), es explícita, y sin ella ningún comando corre.
