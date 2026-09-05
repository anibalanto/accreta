# Comando: `worklist create-or-find`

La integración real con el proveedor: [`concepts/sync.md`](../concepts/sync.md#asignar-una-clave-crear-o-encontrar) — busca un issue por título antes de crear, para que un reintento después de una falla nunca duplique.

## Firma

```
worklist create-or-find --project <clave> --type <tipo> --source <ruta> <título> [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--project` | La clave del proyecto de Jira — `ACC` para accreta. |
| `--type` | `task`, `user-story` o `epic`. El vocabulario del worklist, nunca el de Jira: la traducción la hace quien habla con el proveedor, ver [`concepts/sync.md`](../concepts/sync.md#el-tipo-es-del-worklist-no-de-jira). |
| `--source` | La ruta del ítem en el repo del worklist. Arma la descripción: `Fuente: <ruta>`. |
| `<título>` | El título del ítem. Viaja **intacto** al crear, y reducido al buscar. |
| `--dry-run` | No llama a `acli`. Imprime qué buscaría y qué crearía. |

**No hay `--parent`, y no es un olvido.** Este comando resuelve **un ítem suelto**, y la jerarquía no se puede calcular sobre uno solo: hay que subir la cadena de ancestros hasta la épica, que es lo que [`assign-keys`](assign-keys.md) hace sobre la ventana entera. El puerto sí recibe un padre —`Board::create_or_find(título, tipo, descripción, parent)`—; este comando le pasa `None`.

## Comportamiento

1. Busca `project = <clave> AND summary ~ "<título reducido>"`. **El título de búsqueda no es el título del issue**: se le sacan los caracteres que rompen el parser de JQL, que no tiene forma de escaparlos. Ver [`concepts/sync.md`](../concepts/sync.md#el-título-de-búsqueda-no-es-el-mismo-string-que-el-título-del-issue).
2. Si algún resultado tiene el `summary` **exactamente** igual al título, devuelve su clave. No crea nada. La query busca de más a propósito; quien decide es la comparación exacta, no lo que la JQL trajo.
3. Si no encuentra nada, crea el issue con `--summary <título>` —el real, intacto—, `--type <el tipo traducido a Jira>`, `--project <clave>`, y `--description` en una sola línea plana: `Fuente: <ruta>`.
4. Devuelve la clave.

**El cuerpo del ítem no viaja acá.** No porque no deba —la regla general es que [la descripción lleva el cuerpo convertido a ADF](../concepts/sync.md#el-cuerpo-viaja-y-vuelve-convertido)—, sino porque este comando **no lo tiene**: recibe una ruta, no un ítem. El cuerpo lo sube la pasada 2, con `set_description`, y sólo sobre lo que se creó.

## Salida

```
$ worklist create-or-find --project ACC --type task --source 50.task.md "Migrar dependencias a relation.depends"
ACC-101

$ worklist create-or-find --project ACC --type task --source 50.task.md "Migrar dependencias a relation.depends" --dry-run
would search: project = ACC AND summary ~ "Migrar dependencias a relation.depends"
would create: --project ACC --type Tarea --summary "Migrar dependencias a relation.depends" --description "Fuente: 50.task.md"
```

El `--type` que el `--dry-run` imprime es el **de Jira** — `Tarea`, no `task`: es lo que se le va a pedir a `acli`, y mostrar el del worklist escondería justo el paso que puede fallar.

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | encontrado o creado, clave impresa |
| `1` | el título no deja nada con que buscarse — todos sus caracteres rompen la JQL |
| `1` | `--type` no es del vocabulario del worklist, así que no hay a qué tipo de Jira traducirlo |
| `1` | error de `acli` — red, autenticación, o proyecto inválido |
