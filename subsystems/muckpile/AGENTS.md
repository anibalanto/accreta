# muckpile

Reemplaza a `worklist`/`worklist-server`. La decisión completa —por qué, y con qué forma— vive en [`docs/adr/0001-muckpile-nace-como-subsistema-propio.md`](.stratum/impl/docs/adr/0001-muckpile-nace-como-subsistema-propio.md), del lado del impl. **Ese ADR es la fuente**: este archivo no repite su contenido, sólo dice cómo ejecutarlo.

El repo del impl es propio, hoy anidado en el checkout de accreta —`subsystems/muckpile/.stratum/impl/`— con remoto en `git@github.com:anibalanto/muckpile.git`.

## Antes de razonar sobre cualquier cambio

**Mientras se trabaje acá adentro del checkout de accreta**, leer directo, sin pasar por la herramienta `Skill`:

- `ia/skills/bilinker/SKILL.md`
- `ia/skills/stratum-paths/SKILL.md`

**No la herramienta `Skill`, el archivo.** `Skill` depende de que `.claude/skills` esté registrado en la raíz de la sesión, y `muckpile` puede clonarse y trabajarse solo, sin accreta alrededor — pedirle el nombre en vez del path lo ataría a que alguien más lo haya configurado. Leer el archivo funciona siempre que el checkout de accreta esté a mano; el día que no lo esté, esta sección entera deja de aplicar, porque `bilinker` y `stratum-paths` son de cómo accreta mantiene sus specs en línea con su código — no de lo que `muckpile` hace en producción.

**Ninguna referencia a `worklist` ni a `item-writing`.** La primera es exactamente lo que este subsistema reemplaza — cargarla acá sería instrucción para el sistema que se está sacando. La segunda es sobre cómo se titula un ítem *del worklist*, y `muckpile` todavía no tiene una convención de ítems propia escrita en ningún lado.

## Sin tarea previa, por ahora

`AGENTS.md` de accreta pide un ítem antes de tocar cualquier cosa. Acá no: no se usa `worklist` para trackear el desarrollo de `muckpile`, ni Jira directamente — ninguno de los dos es la herramienta correcta para trackear la herramienta que los reemplaza. El día que `muckpile` pueda trackearse a sí mismo, retomar algo parecido a esa regla es una decisión aparte, no una consecuencia automática de este párrafo.

## El método sigue siendo el de bilinker

Se toca la spec (acá o en `docs/adr/`), `bilinker check .` reporta los endpoints no-OK, cada uno apunta al fragmento de código que hay que tocar, se cambia el código y se acepta. Nada de esto cambia por no tener tarea — el paso 0 que no está es el ítem, no el método.

## Código: inglés entero, sin cita externa

Es la decisión 11 del ADR, y se repite acá porque gobierna cada línea que se escriba: identificadores **y** comentarios en inglés, y un comentario documenta el código — nunca señala hacia un ADR, una spec o un ítem por nombre o número. Si hace falta decir por qué el código es así, se dice en términos del código.
