# muckpile

Reemplaza a `worklist`/`worklist-server`. La decisión completa —por qué, y con qué forma— vive en [`docs/adr/0001-muckpile-nace-como-subsistema-propio.md`](.stratum/impl/docs/adr/0001-muckpile-nace-como-subsistema-propio.md), del lado del impl. **Ese ADR es la fuente**: este archivo no repite su contenido, sólo dice cómo ejecutarlo.

El repo del impl es propio, hoy anidado en el checkout de accreta —`subsystems/muckpile/.stratum/impl/`— con remoto en `git@github.com:anibalanto/muckpile.git`.

## Antes de razonar sobre cualquier cambio

**Mientras se trabaje acá adentro del checkout de accreta**, leer directo, sin pasar por la herramienta `Skill`:

- `ia/skills/bilinker/SKILL.md`
- `ia/skills/stratum-paths/SKILL.md`

**No la herramienta `Skill`, el archivo.** `Skill` depende de que `.claude/skills` esté registrado en la raíz de la sesión, y `muckpile` puede clonarse y trabajarse solo, sin accreta alrededor — pedirle el nombre en vez del path lo ataría a que alguien más lo haya configurado. Leer el archivo funciona siempre que el checkout de accreta esté a mano; el día que no lo esté, esta sección entera deja de aplicar, porque `bilinker` y `stratum-paths` son de cómo accreta mantiene sus specs en línea con su código — no de lo que `muckpile` hace en producción.

**Y ese path cae afuera de este repo — verificado, no es una hipótesis.** `stratum '*'` desde adentro de `subsystems/muckpile/.stratum/impl/` da la raíz de accreta, no la de este repo: `*` es "el ancestro git más externo", y accreta es un ancestro real de este checkout anidado. Es el lugar correcto para encontrar `ia/skills/` — y también donde vive el `AGENTS.md`/`CLAUDE.md` de accreta, con reglas que no son las de acá. **Ir a buscar el archivo de skill puntual no es leer nada más de esa raíz.** No cargar el `AGENTS.md` de accreta de paso porque esté ahí sentado: sus reglas —`worklist`, la tarea previa— son exactamente las que esta página ya dijo que no aplican.

**Ninguna referencia a `worklist` ni a `item-writing`.** La primera es exactamente lo que este subsistema reemplaza — cargarla acá sería instrucción para el sistema que se está sacando. La segunda es sobre cómo se titula un ítem *del worklist*, y `muckpile` todavía no tiene una convención de ítems propia escrita en ningún lado.

## Sin tarea previa, por ahora

`AGENTS.md` de accreta pide un ítem antes de tocar cualquier cosa. Acá no: no se usa `worklist` para trackear el desarrollo de `muckpile`, ni Jira directamente — ninguno de los dos es la herramienta correcta para trackear la herramienta que los reemplaza. El día que `muckpile` pueda trackearse a sí mismo, retomar algo parecido a esa regla es una decisión aparte, no una consecuencia automática de este párrafo.

## El método sigue siendo el de bilinker

Se toca la spec (acá o en `docs/adr/`), `bilinker check .` reporta los endpoints no-OK, cada uno apunta al fragmento de código que hay que tocar, se cambia el código y se acepta. Nada de esto cambia por no tener tarea — el paso 0 que no está es el ítem, no el método.

## Código: inglés entero, sin cita externa

Es la decisión 11 del ADR, y se repite acá porque gobierna cada línea que se escriba: identificadores **y** comentarios en inglés, y un comentario documenta el código — nunca señala hacia un ADR, una spec o un ítem por nombre o número. Si hace falta decir por qué el código es así, se dice en términos del código.

## Un identificador traduce el término de la spec, nunca inventa uno propio

Si el ADR dice "vista", el código dice `view` en todos lados — no `workspace` en un módulo y `context` en otro. Si dice "canónico", es `canonical`; si dice `to-work`, el comando y la función se llaman así. La traducción es la única libertad: elegir un sinónimo distinto en cada lugar donde el ADR ya usó una palabra es la misma clase de error que tener dos nombres para el mismo concepto, con el costo de que acá ni siquiera se nota en el mismo idioma.

## Commits

Mensaje de una línea. Un commit hace una cosa — si un cambio hace dos, son dos commits.

**Sin prefijo de ítem, porque no hay ítems todavía** (ver "Sin tarea previa" arriba). No inventar uno mientras tanto — el día que `muckpile` trackee su propio trabajo, ahí se decide cómo se prefija, con el sistema real delante y no a ciegas.

**Nunca empujar accreta.** Este trabajo commitea y empuja en el repo del impl —tiene su propio `.git` y su propio remoto—; accreta es el checkout que lo contiene, no el destino del push. Si hace falta tocar algo del lado de accreta (este mismo archivo, la declaración de capa), el commit local es aceptable —así se hizo durante todo el diseño—, pero el `push` de accreta queda para una decisión humana aparte, nunca como paso de la implementación de punta a punta.

## Pruebas: TDD, con un proveedor de prueba que nunca es el real

Se escribe el test antes que el código, para todo lo que tenga un contrato verificable — que es casi todo: la canonicidad (decisión 10), el renombre y la reescritura de referencias (decisión 4), el escape de título para búsqueda, la categoría de un estado (decisión 8). Son funciones puras, con entrada y salida claras, y es exactamente lo que hace TDD barato acá.

**La suite automatizada nunca llama a Jira real.** Antes de escribir nada que hable con el proveedor, existe un proveedor de prueba — la misma idea que ya tiene `worklist` — y es contra ése que corren los tests que se ejecutan en cada ciclo. Pegarle a la API real desde una suite que corre repetida no sólo puede romper por rate limit: puede tocar un board de una organización que no es la del que está probando.

**Y aun así, la API real tiene su lugar: la exploración puntual, nunca la suite.** Cuando hay duda genuina sobre cómo se comporta un endpoint —qué trae `statusCategory`, cómo lista las transiciones un board con un workflow raro—, medirlo contra `ACC` (el board de accreta, no uno de una empresa) es el método correcto, el mismo que `sync.md` ya usa y documenta con fecha. Esa medición informa el test que se escribe contra el proveedor de prueba; no reemplaza al test, y no vive en el mismo lugar que corre en cada `cargo test`.

**Se avanza de punta a punta y se para sólo por un hallazgo de riesgo, no por etapas fijas.** La sección "Lo que este ADR no decide" es el inventario de qué cuenta como riesgo: si aparece algo de esa lista, o algo con la misma forma —una decisión de diseño que el ADR no cerró, una escritura contra el board real fuera del caso de exploración de arriba—, es motivo de parada. Lo rutinario no necesita permiso: para eso está la suite.
