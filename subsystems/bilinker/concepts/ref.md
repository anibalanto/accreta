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

**Absorber no es un comportamiento por comando: es una precondición de todo commit sobre la ref.** No hay una tabla de quién absorbe y cuándo: hay una condición que se verifica y, si no se cumple, se cumple absorbiendo **en un commit propio, inmediatamente antes**. Cuando ya se cumple —el proyecto no se movió desde la última absorción— no se absorbe nada, y el enunciado 1 sigue valiendo porque el árbol de código no cambió.

### Un commit hace una cosa

De ahí sale la regla que gobierna toda la forma de la ref:

> **Un commit sobre la ref trae código, o decide, o sincroniza decisiones. Nunca dos de las tres.**

Nunca hay un commit que absorba y decida a la vez, y por eso la historia se lee sin abrir un solo archivo:

| Tipo | Padres | Árbol de código | `.bilink/` |
|---|---|---|---|
| **1 · absorción** — trae el código del proyecto | dos: la ref y el commit del proyecto | **cambia** | **sin tocar** |
| **2 · decisión** — `accept`, `apply` | uno | sin cambios | **cambia** |
| **3 · sincronización** — trae bilinks aceptados por otro | dos, los dos de la ref | sin cambios | **cambia** |

Los tres se distinguen con git a secas, sin leer un bilink: **por la cantidad de padres, de dónde vienen, y cuál de los dos árboles se movió.**

`sync` no es un tipo propio: **es la absorción invocada explícitamente**, con la misma forma que la que `accept` dispara cuando le falta.

#### El tipo 3 tiene dos casos, y también se distinguen solos

Sincronizar decisiones es traer lo que aceptó otro. De dónde viene cambia qué puede pasar:

| | De dónde | Los dos lados vienen de… |
|---|---|---|
| **3.a** — otra rama, que puede haberse rebaseado sobre la trackeada | [`adopt <rama>`](../commands/adopt.md) | **dos absorciones distintas** |
| **3.b** — la misma rama: alguien tenía una versión anterior y aceptó | [`pull <remoto>`](../commands/pull.md) | **la misma absorción** |

El discriminador es de git: se busca hacia atrás la absorción más cercana de cada lado y se comparan. Si es la misma, es 3.b; si son dos, es 3.a.

Y la diferencia importa porque en **3.a los dos lados describen código distinto** —cada rama absorbió lo suyo—, así que el árbol de código del resultado tiene que salir del primer padre y no fusionarse. Es lo que `adopt` ya hace: *"la fusión de contenido queda confinada a `.bilink/`: el árbol se **construye**, no se fusiona"*. En **3.b los dos lados describen el mismo código**, así que no hay nada que elegir.

#### Y 3.b converge por construcción

Es el caso que el formato ya resolvió sin proponérselo: dos personas que aceptan **el mismo contenido** escriben **los mismos bytes**, porque `link`, `hash` y `hash_ast` direccionan por contenido. El merge de 3.b no tiene qué conflictuar.

Eso vale **porque `commit` no está en `accepted`**. Es el único campo candidato que nunca converge —el mismo contenido aceptado en dos ramas vive en dos commits distintos—, y meterlo adentro habría hecho que 3.a y 3.b conflictuaran en cada endpoint aceptado de los dos lados, sobre algo que nadie decidió. Ver [ADR-0003](../.stratum/impl/docs/adr/0003-formato-captures-y-aceptacion.md) § "`commit` es el commit del contenido".

**3.b es [`pull`](../commands/pull.md)**, y es el más simple de los dos: los dos lados cuelgan de la misma absorción, así que el árbol de código no se elige — sale del primer padre. `adopt` cubre 3.a, donde cada rama absorbió lo suyo y por eso hay dos códigos distintos descritos.

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
rc-2.35:             X ─────── B ─────── C ───────────────────── D ─────── E
                     │                   ╲                       ╲         ╲   ← 2º padre: lo que se absorbe
refs/bilink/rc-2.35: ●0 ───────────────── ●1 ─ ●2 ─ ●3 ─ ●4 ────── ●5 ─ ●6 ─── ●7
                     corte                └ absorción             └ absorción  └ sync
```

| | Padres | Acto | Diff de `.bilink/` contra su 1er padre |
|---|---|---|---|
| `●0` | `X` | el corte | agrega `.bilink/` entero |
| `●1` | `●0`, `C` | **absorción** de `C` | **vacío** |
| `●2` | `●1` | `accept 7f3d8e9a.0` | el `accepted` de un endpoint |
| `●3` | `●2` | `accept 7f3d8e9a.1` | el `accepted` del otro |
| `●4` | `●3` | `apply -y` (3 renames) | tres commits, uno por `link` repuntado — acá abreviados; ningún `accepted` tocado → los tres quedan `RELOCATED` |
| `●5` | `●4`, `D` | **absorción** de `D` | **vacío** |
| `●6` | `●5` | `accept . --place` (3 endpoints) | tres commits, uno por endpoint — acá abreviados |
| `●7` | `●6`, `E` | `sync` = absorción de `E` | **vacío** |

Los tres merges —`●1`, `●5`, `●7`— tienen el diff de `.bilink/` **vacío**, y eso es lo que los identifica: traen código y no deciden nada. Sus hijos son al revés: tocan sólo `.bilink/` y su árbol de código no cambia.

`●4` muestra por qué la fidelidad se enuncia con "el merge más cercano": nadie tocó el proyecto entre `●1` y él, así que no hay absorción nueva y su árbol de código sigue siendo el de `C`.

Y como `●6`, es **una fila para varios commits**: las decisiones son una por objeto, así que tres endpoints repuntados son tres commits encadenados. La fila los abrevia para que el dibujo quepa; la ref no.

El corte (`●0`) es el único commit de la ref sin ningún commit del proyecto absorbido *por debajo*: nace de `X` como padre único, y ahí la fidelidad se lee contra `X` mismo. Es también el único caso en que una decisión puede no tener una absorción arriba.

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
b1e3f55  Kim     absorb e91f0c4: rc-2.35 al día
9c1f0ab  Ana     accept --place a1b2c3d4.0: scip/link.rs
5d20e81  Ana     accept --place 8e9f0a1b.0: scip/link.rs
3f8b41c  Ana     accept --place 7f3d8e9a.1: scip/link.rs
c7e0d92  Ana     absorb d0b7a12: el rename ya commiteado
4e77d20  Ana     apply 7f3d8e9a.1 3ca90f81…: sciplink.rs → scip/link.rs
77a0c94  Luis    accept 7f3d8e9a.0: spec de check ↔ check_structural
2b1a5f0  Luis    absorb c4e1770: rc-2.35 hasta C
0af3c12  Luis    track rc-2.35: corte 005, los bilinks salen de la rama
```

Ése es el registro de decisiones: quién aceptó qué y cuándo, sin una sola línea del historial del proyecto de por medio.

**Granularidad: un commit por decisión, no por invocación.** `accept <uuid>.0` da un commit; `accept .` sobre veinte endpoints da **veinte**, encadenados, todos hijos del mismo merge. Vale igual para `apply`, que también escribe decisiones: un `apply -y` que repunta tres `link` escribe tres.

La granularidad sigue al **objeto** y no al acto, por tres razones:

- **Atribución por decisión.** *"La responsabilidad vive en el commit que escribió el valor, no en el valor"*, y una firma sobre un commit que aprueba veinte fragmentos dice mucho menos que veinte firmas sobre uno cada una.
- **Varias personas, varios caminos.** Un mismo capture puede tener N bilinks, y aceptarlos es trabajo de gente distinta en momentos distintos. Con el commit como unidad de decisión, cada aprobación es un objeto propio que se lee, se firma y se audita sola.
- **Y hace caro esconder una aprobación masiva.** Un `accept .` sobre una capa recién cambiada *"fabrica aprobaciones que nadie miró"* — ver [aceptación](accept.md#lo-que-no-se-acepta-a-ciegas). Un commit disimula cien decisiones; cien commits firmados las denuncian.

**Deshacer una aceptación** no necesita `git revert`: es reescribir su `accepted` con los valores anteriores, un commit nuevo, leídos de `refs/bilink/<branch>~n`. La unidad de movimiento es el contenido del archivo, no el commit — así que esto vale con cualquier granularidad, y no es lo que la decide.

### El mensaje es el comando

**La primera línea de todo commit sobre la ref empieza con el comando canónico que lo produjo**, de un vocabulario cerrado:

```
absorb <commit-del-proyecto>                    ← tipo 1
track  <rama>                                   ← tipo 1: la ref nace
accept [--place|--content] <uuid>.<N>           ← tipo 2
apply  <uuid>.<N> <capture-nuevo>               ← tipo 2
adopt  <rama>                                   ← tipo 3.a
pull   <remoto>                                 ← tipo 3.b
```

**Cinco verbos, y cada uno es un comando que existe.** No hay verbo para el corte: el comando que lo escribe es [`track`](../commands/track.md) en su caso *"no hay de quién heredar"*, y un verbo propio nombraría un comando que nadie puede correr. Los dos nacimientos se distinguen por los padres —el corte tiene uno solo, y no es de la ref— que es como se distingue [todo lo demás de un commit de la ref](#un-commit-hace-una-cosa).

**Cada comando nombra el objeto sobre el que actuó**, y nada más: el endpoint para una decisión, la rama de origen para una sincronización, el commit traído para una absorción, la rama que nace para un `track`. Lo que los padres ya dicen no se repite salvo donde hace legible el log — `absorb` nombra el commit que además es su segundo padre, y eso es deliberado: el registro se lee sin abrir el DAG.

`adopt` no lleva endpoint. Trae **todo** lo que el vecino decidió entre la base y su tip, en [un solo commit y no N](../commands/adopt.md#son-dos-commits-y-por-qué), y el conjunto sale del merge a tres puntas entre los dos padres — que ya están en el objeto. Un endpoint en el mensaje no agregaría nada que reproducir y sugeriría una granularidad que el acto no tiene: adoptar no es decidir, es traer decisiones ya firmadas por otro.

`pull` y `adopt` son los dos casos del tipo 3, y llevan verbos distintos porque nombran fuentes distintas: `adopt` una rama vecina, `pull` la copia que el remoto tiene de esta misma ref.

Después del comando puede ir `: ` y prosa libre para quien lee, y el cuerpo es libre. Al final, un trailer obligatorio:

```
accept 7f3d8e9a.0: spec de check ↔ check_structural

Bilinker-Version: 0.4.1
```

**Canónico quiere decir derivado del acto, no tipeado por la persona.** Un `accept .` de veinte endpoints escribe veinte commits, y cada uno lleva **su** comando —`accept <uuid>.<N>`—, no `accept .`. Lo que la persona tipeó puede ir como trailer `Invocation:`, que es dato de auditoría y no de verificación.

De ahí sale la propiedad que hace útil el formato: **el comando más el árbol del primer padre determinan el árbol resultante.** Es la [invariante 4 de la aceptación](accept.md#invariantes) —*"aceptar el mismo fragmento en el mismo estado y sobre el mismo commit del proyecto produce siempre el mismo `accepted`"*— leída como contrato de reproducción. Quien quiera verificar un commit corre el comando contra el árbol del padre y compara tree oids.

**`Bilinker-Version` es obligatorio, y no alcanza con la versión del formato.** [ADR-0006](../.stratum/impl/docs/adr/0006-formato-como-crate-versionado.md) ata la versión del formato a la del crate `bilink-format`, pero `hash`, `hash_ast` y [el recorte de bordes](capture.md) viven en `bilinker`: un cambio ahí movería los hashes sin bumpear el formato, y la reproducción de commits viejos empezaría a fallar sin que nada esté mal.

**Un mensaje se parsea, nunca se ejecuta.** Es texto que escribe cualquiera, así que un verificador que se lo pase a una shell tiene ejecución remota. El vocabulario es cerrado justamente para eso: se parsea a una forma estructurada, se valida cada argumento contra su tipo —un UUID es un UUID, un índice de endpoint es `0` o `1`—, y el proceso se lanza con argv armado. Un verbo desconocido invalida el mensaje; no es texto libre que se pasa por alto.

Y un comando por commit, que es la misma regla de [un commit hace una cosa](#un-commit-hace-una-cosa) dicha desde el mensaje. De ahí que `apply` sobre tres endpoints escriba **tres** commits y no uno: `apply <uuid>.<N> <capture-nuevo>` nombra un endpoint, y un mensaje que nombrara tres no sería reproducible contra el árbol de un solo padre.

#### La gramática no es retroactiva

Los commits que ya están en la ref **no se pueden reescribir**: es append-only. Exigirles la gramática rechazaría la historia entera, así que la regla es de alcance y no de contenido:

> **Un verificador valida los commits del push, no la historia alcanzable desde ellos.**

Y el discriminador no hace falta inventarlo: **la ausencia de `Bilinker-Version` significa "anterior a la gramática"**, y eso no es un error — es el mismo patrón con que [`check`](../commands/check.md) trata el cambio de definición de `hash_ast`. No hay commit de corte ni marcador: el trailer se describe solo. Con el trailer puesto, el mensaje **tiene** que parsear; sin él, no se lo interroga.

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

## La ref es protegida

`refs/bilink/*` **sólo avanza**, y nadie la escribe a mano: sus únicos escritores son los comandos de bilinker. Eso no es higiene, es carga estructural: la ref es lo único que conserva los commits del proyecto contra los que se aceptó —los alcanza como segundos padres— y lo único que vuelve verificable una decisión, porque el commit firmado es el artefacto. Reescribirla deja sin baseline y sin atestación a toda aceptación del repo.

Tres niveles, y sólo el último es exigible:

| Nivel | Qué da | Con qué |
|---|---|---|
| el clon local | **detección** | la disyunción y la fidelidad, chequeadas antes de cada commit |
| el refspec | que no se fuerce por accidente | el de [`init`](../commands/init.md), **sin `+`** |
| el servidor | **rechazo** | `receive.denyNonFastForwards`, `receive.denyDeletes`, y un `pre-receive` |

`denyDeletes` no es un extra: sin él, *"sólo avanza"* se esquiva borrando la ref y empujándola de nuevo.

El nivel del servidor es [`verify-ref`](../commands/verify-ref.md), y el `pre-receive` que lo llama son dos líneas.

### El servidor puede verificarlo sin ejecutar bilinker

Es la parte que el diseño ya pagó y no había dicho: **las invariantes de forma de la ref son comparaciones de tree oids**, no análisis de contenido. Un `pre-receive` las chequea con git a secas, sin instalar nada:

| Se verifica | Cómo |
|---|---|
| avanza y no se borra | `denyNonFastForwards` + `denyDeletes` |
| [disyunción](#las-dos-verificaciones-previas) | el árbol del commit absorbido no contiene `.bilink/` |
| [fidelidad](#la-invariante-de-fidelidad) | el árbol de código del commit nuevo es el del absorbido |
| [un commit hace una cosa](#un-commit-hace-una-cosa) | cae en uno de los tres tipos y no en dos: los padres y cuál de los dos árboles se movió lo deciden |

Ninguna necesita tree-sitter ni abrir un bilink, que es exactamente lo que la invariante de fidelidad dice de sí misma al enunciarse *"verificable sin tree-sitter"*. Lo que el servidor **no** puede verificar es si lo aceptado es correcto — eso es una decisión humana y no una propiedad del árbol.

### La autorización es la firma, más una regla de una línea

El mismo hook llena la fila que [más abajo](#autoría-atestación-y-autorización) queda sin dueño: **todo commit sobre la ref tiene que estar firmado por una clave de una allowlist**, que `git verify-commit` chequea offline y sin infraestructura.

Del lado de quien escribe, la condición la pone git y no bilinker: se firma si `commit.gpgsign` está puesto, igual que cualquier otro commit del repo.

Y con eso alcanza para que [`agree`](accept.md#quiénes-aprobaron) deje de ser auto-declarado, **sin traducir ningún nombre a ninguna clave**:

> **Un commit sólo puede agregar a su propio autor a un `agree`.**

La firma ata el commit a una clave, y con ella al autor que declara; la regla ata los nombres agregados a ese mismo autor. Las dos juntas cierran la cadena: `- ana` sólo puede haberlo escrito un commit firmado cuya autora es Ana. **Nadie aprueba en nombre de otro.**

Sacar un nombre no está restringido, y no puede estarlo: es lo que hace `adopt` al traer valores distintos, y lo que hace un `accept` cuando los valores cambian y la lista se vacía. Lo que se protege es **agregar**, que es lo único que afirma algo sobre otra persona.

### Y el prefijo anterior a la gramática pasa una vez

La ref es append-only, así que exigirle la forma a lo que ya está rechazaría el primer push de todo repo que cortó antes de que la gramática existiera. El discriminador es el mismo que para [el mensaje](#la-gramática-no-es-retroactiva) —la ausencia de `Bilinker-Version`— y la puerta que eso abre se cierra con una regla de orden:

> **Una vez que un commit de la ref lleva el trailer, ninguno de sus descendientes puede no llevarlo.**

Un commit sin trailer empujado encima de uno que lo tiene no es historia vieja: es alguien esquivando la verificación.

### La protección de ramas de la forja no alcanza

GitHub y GitLab protegen `refs/heads/*` y `refs/tags/*`; un namespace propio queda afuera de esa UI. Es el precio de estar [fuera de `refs/heads/`](#fuera-de-refsheads): se gana invisibilidad y se pierde la protección declarativa, así que hay que ponerla como config o hook del servidor. Conviene saberlo antes de asumir que "la rama está protegida" cubre algo de esto.

### Localmente no se puede impedir, sólo detectar

Un `git update-ref` en el clon de alguien no lo frena nadie: es su repo. Vale la misma distinción que ya hace la disyunción — *"no es exigible como invariante —nadie puede impedir que alguien mergee— pero sí detectable antes de que contamine nada"*. La frontera donde el rechazo es posible es la compartida, y por eso el enunciado fuerte vive en el servidor y no en el cliente.

## Auditoría: contra la ref, no contra la rama

La rama del proyecto no está protegida: se rebasea, se force-pushea, se cambia. Pero la ref **absorbe los commits del proyecto como segundos padres**, así que todo commit alguna vez absorbido queda alcanzable desde la ref para siempre.

```
git merge-base --is-ancestor <commit> refs/bilink/<branch>   ← precondición
git log <commit>..refs/bilink/<branch> -- <file>              ← la ventana
```

Si aun así no es ancestro —commit nunca absorbido, clon superficial del proveedor— degradar a *"fuente desconocida"* en vez de devolver un rango inflado.

Y de ahí sale la propiedad que la task `16` verifica: **la ref es lo que vuelve inmutable el `commit` de una aceptación**. Un rebase de la rama del proyecto saca ese commit de `main`, pero no lo destruye, porque `refs/bilink/<branch>` lo alcanza como segundo padre de una absorción. Protege al `commit` guardado en [la cache](cache.md) y protege también a su re-derivación, que camina la historia del archivo y necesita que esos commits sigan existiendo.

**Y es lo que hace que `commit` no necesite estar en `accepted`.** La firma de un commit cubre el objeto commit, y el objeto incluye sus padres: el commit del proyecto contra el que se aceptó ya está atestado por la firma de la aceptación, vía el DAG, sin ningún campo en el archivo. Ver la resolución en [ADR-0003](../.stratum/impl/docs/adr/0003-formato-captures-y-aceptacion.md) § "`commit` es el commit del contenido" — donde el argumento decisivo es otro: `commit` es el único campo candidato que **nunca converge**, y meterlo en `accepted` haría que [`adopt`](../commands/adopt.md) reportara un conflicto por cada endpoint aceptado de los dos lados.

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
3. Ningún commit sobre la ref se escribe sin que el commit del proyecto contra el cual se calculó esté absorbido, y la absorción ocurre en un commit propio, nunca en el mismo.
4. **Un commit sobre la ref hace una cosa**, y es una de tres: **absorción** —dos padres, uno del proyecto, diff de `.bilink/` vacío—, **decisión** —un padre, árbol de código sin cambios—, o **sincronización** —dos padres, los dos de la ref, árbol de código sin cambios. Ninguno hace dos.
5. Una aceptación es un commit de tipo decisión por endpoint aceptado, y las de una misma invocación son hijas de la misma absorción.
6. La ref no se rebasea ni se cherry-pickea nunca. Es append-only, todo fetch es fast-forward, y sus únicos escritores son los comandos de bilinker.
7. La ref es **protegida**: en el servidor, todo push que no sea fast-forward y todo delete se rechazan. Localmente la violación no se impide, se detecta.
8. El árbol del commit de un commit sobre la ref se construye con `read-tree` del absorbido más `update-index` de `.bilink/`. Nada del árbol de trabajo fuera de `.bilink/` entra.
9. La ref es por repo: cubre todas las capas de ese repo y ninguna de un repo anidado.
10. `.bilink/head` dice a qué rama y a qué commit de la ref corresponde el `.bilink/` del árbol, lo escriben tanto la materialización como todo commit sobre la ref, y ningún comando opera sobre un `.bilink/` que no corresponde a la rama actual.
11. La puesta a punto de un clon —exclusión, refspec y materialización— es [`init`](../commands/init.md), es explícita, y sin ella ningún comando corre.
