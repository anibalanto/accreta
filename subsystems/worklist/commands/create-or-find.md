# Comando: `worklist create-or-find`

La integración real con el proveedor: [`concepts/sync.md`](../concepts/sync.md#asignar-una-clave-crear-o-encontrar) — busca un issue por título antes de crear, para que un reintento después de una falla nunca duplique.

## Firma

```
worklist create-or-find --project <clave> --type <tipo> --source <ruta> <título> [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--project` | La clave del proyecto de Jira — `ACC` para accreta. |
| `--type` | `Task`, `Historia` o `Epic`. |
| `--source` | La ruta del ítem en el repo del worklist. Arma la descripción: `Fuente: <ruta>`. |
| `<título>` | El título del ítem. Se escapa antes de entrar en cualquier JQL. |
| `--dry-run` | No llama a `acli`. Imprime qué buscaría y qué crearía. |

## Comportamiento

1. Escapa el título y busca `project = <clave> AND summary ~ "<título escapado>"`.
2. Si encuentra un resultado, devuelve su clave. No crea nada.
3. Si no encuentra nada, crea el issue con `--summary <título>`, `--type <tipo>`, `--project <clave>`, y `--description` en una sola línea plana: `Fuente: <ruta>` — nunca el cuerpo del ítem, ver [`concepts/sync.md`](../concepts/sync.md#asignar-una-clave-crear-o-encontrar).
4. Devuelve la clave nueva.

## Salida

```
$ worklist create-or-find --project ACC --type Task "Migrar dependencias a relation.depends"
found: ACC-101

$ worklist create-or-find --project ACC --type Task "Migrar dependencias a relation.depends" --dry-run
would search: project = ACC AND summary ~ "Migrar dependencias a relation.depends"
would create: --project ACC --type Task --summary "Migrar dependencias a relation.depends" --description "Fuente: 50.task.md"
```

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | encontrado o creado, clave impresa |
| `1` | error de `acli` — red, autenticación, o proyecto/tipo inválido |
