# Comando: `impact thread`

Gestiona los hilos de discusión asociados a Impact Reports.

## Uso

```
impact thread new <report-uuid> [--title <string>]
impact thread list [--status open|resolved|discarded]
impact thread show <thread-uuid>
impact thread reply <thread-uuid> [--kind reply|analysis|resolution] [--message <text>]
impact thread resolve <thread-uuid>
impact thread discard <thread-uuid>
```

## Subcomandos

### `impact thread new`

Abre un nuevo thread a partir de un Impact Report existente.

1. Verifica que el report existe en `.impact/reports/`.
2. Crea `.impact/threads/<uuid>/thread.md` con `status: open`.
3. Copia el Impact Report como `0001.md` (`kind: report`).
4. Imprime el UUID del thread creado.

```
thread abierto: 7f3d8e9a
  title: cambio en src/Persona.java afecta chain abc12345
  .impact/threads/7f3d8e9a-...
```

### `impact thread list`

Lista todos los threads con su estado y título.

```
7f3d8e9a  open      cambio en src/Persona.java afecta chain abc12345
a1b2c3d4  resolved  actualizar spec voting.yaml tras refactor
```

### `impact thread show`

Muestra el contenido completo de un thread: metadata y todos los mensajes en orden de secuencia.

### `impact thread reply`

Agrega un mensaje al thread. El texto puede venir de `--message` o de stdin si se omite.

```
impact thread reply 7f3d8e --kind analysis --message "El cambio es compatible, el invariante sigue válido"
```

Crea el siguiente archivo en secuencia (`0002.md`, `0003.md`, etc.).

### `impact thread resolve`

Agrega un mensaje `kind: resolution` y cambia el estado a `resolved`. Requiere que el thread esté `open`.

### `impact thread discard`

Marca el thread como `discarded`. Para falsos positivos o cambios irrelevantes.

## Flags globales

| Flag | Descripción |
|------|-------------|
| `--status <estado>` | Filtra por estado en `list` |
| `--title <string>` | Título del thread en `new` |
| `--kind <kind>` | Tipo de mensaje en `reply` |
| `--message <text>` | Cuerpo del mensaje en `reply` |

## Exit codes

- `0`: operación exitosa
- `1`: thread o report no encontrado, estado inválido, o error
