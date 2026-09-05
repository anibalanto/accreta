# Comando: `worklist bootstrap`

Le da clave del proveedor a lo que no la tiene, sobre el panorama, **sin prometer que la rama se verifique**. Es el camino que el backlog no tenía.

Es la operación que separa las dos cosas que el modelo trataba como una — ver [`concepts/sync.md`](../concepts/sync.md#tener-clave-es-del-ítem-verificarse-es-de-la-rama).

## Firma

```
worklist bootstrap --project <clave> [--ref <rama>] [--base <url>] [--dry-run]
```

| Argumento | Descripción |
|---|---|
| `--project` | El proyecto del proveedor: `ACC`. |
| `--ref` | Sobre qué rama. Por defecto `refs/heads/insecure/all`, que es donde vive todo. |
| `--base` | Base del proveedor, para traducir los links a otros ítems al convertir el cuerpo. |
| `--dry-run` | Dice a qué le pediría clave, sin hablar con nadie ni mover ninguna ref. |

## Qué hace, y sobre todo qué no

1. Lista los `*.md` del árbol cuyo nombre **no** es una clave de proveedor: ésos son los pedidos. Los `_sprints/*.sprint.md` no entran — un sprint no es un issue.
2. Los ordena topológicamente por `parent`, para que una épica exista antes que lo que cuelga de ella.
3. Por cada uno: [`create-or-find`](create-or-find.md), el renombre con su reescritura, y el `--parent` de su épica ancestro.
4. Y el cuerpo, **sólo donde el issue se creó**.

**Y nada más.** Ni vínculos, ni sprint, ni actualizar lo que ya tenía clave: todo eso es sincronización, y sincronizar es lo que esta operación explícitamente no promete. Las cinco pasadas de [`assign-keys`](assign-keys.md) son para una ventana, que sí lo promete.

### Encontrado no es creado

`create_or_find` puede encontrar en vez de crear — una corrida anterior que se cayó después de crear y antes de comitear el renombre deja exactamente eso. Ahí el cuerpo **no** viaja.

Crear un issue con su descripción escribe sobre algo que no existía: no hay nada que pisar. Escribírselo a uno que ya estaba es una escritura sobre contenido ajeno, y no hubo compare-and-swap que probara que partimos de su estado actual. La salida lo dice en vez de callarlo:

```
  5e -> ACC-207  (ya existia: el cuerpo no se toco)
```

## Corre en el servidor, y no es un push

`insecure/**` [rechaza el push](../concepts/sync.md#dos-clases-de-rama-y-el-nombre-dice-qué-se-puede-hacer) — **del cliente.** Que el servidor escriba en el panorama es lo que [la propagación](../concepts/propagation.md) ya hace en cada push aceptado; esto es lo mismo por otro motivo.

Así que se corre parado en el repo bare, como `assign-keys`, y con la misma credencial.

## Salida

```
$ worklist bootstrap --project ACC
refs/heads/insecure/all: resolvio 126 item(s)
  51 -> ACC-205  (7 refs reescritas)  [parent ACC-14]
  5e -> ACC-206  (1 refs reescritas)  [parent ACC-14]
  5z -> ACC-207
refs/heads/insecure/all: a1b2c3d -> 9z8y7x6
```

Y la segunda corrida:

```
refs/heads/insecure/all: no hay ningun item sin clave
```

**No es idempotencia por suerte**: los ítems que ya tienen clave no son pedidos, así que la lista del paso 1 sale vacía sin preguntarle nada al proveedor.

## Códigos de salida

| Código | Condición |
|--------|-----------|
| `0` | los pedidos quedaron con clave, o no había ninguno |
| `1` | un ciclo entre pedidos, o el proveedor falló — la ref queda como estaba |
