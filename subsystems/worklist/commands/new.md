# Comando: `worklist new`

Crea un nuevo ítem en `accreta/.stratum/worklist/` y opcionalmente establece un bilink abierto al fragmento que lo origina. Requiere conectividad con el servidor worklist.

## Uso

```
worklist new <tipo> "<título>" [capture <selector>] [--under <id>]
```

Donde `<tipo>` es `epic`, `user-story` o `task`.

`capture <selector>` captura un fragmento como origen del trabajo y crea un bilink abierto (`link.0: fragmento`, `link.1: todo`). El selector se resuelve desde el directorio actual en la terminal:

| Forma del selector | Descripción |
|--------------------|-------------|
| `archivo:línea:col` | Posición exacta en un archivo |
| `archivo` | Archivo completo |
| `<uuid-bilink>` | Bilink existente |

Sin `capture`, el ítem se crea sin bilink — válido para epics de alto nivel.

## Comportamiento

1. Si se especifica `capture <selector>`, ejecuta `bilinker capture` sobre el
   fragmento y crea un bilink con `link.0: fragmento`, `link.1: todo`.
2. Empuja la solicitud de creación al servidor worklist.
3. El servidor asigna el próximo ID base-36, crea el archivo `<id>.<tipo>`,
   actualiza `<bilink-uuid>.tasks` si hay bilink, y hace commit.
4. Hace fetch del servidor.
5. Imprime el ID y el path del ítem creado.

Si no hay conectividad, el comando falla con código de salida `3`.

## Ejemplos

```bash
# Task con capture: marca trabajo pendiente en la spec
cd accreta/bilinker/specific
worklist new task "implementar bilinker accept" capture bilink-format.md:104:1

# Task con capture desde impl: spec necesita actualización
cd accreta/bilinker/.stratum/impl/src
worklist new task "actualizar spec hash.N" capture lib.rs:88:1

# Epic sin capture: objetivo de alto nivel sin fragmento de origen
worklist new epic "implementar hash.N"

# User story hija de un epic
worklist new user-story "parsear hash.N en BiLinkFile" capture bilink-format.md:45:1 --under 1

# Task hija de una user story, sin capture
worklist new task "actualizar struct BiLinkFile" --under 2
```

## Salida

```
created:  3
type:     task
path:     accreta/.stratum/worklist/3.task
bilink:   a3f9c821-4e5b-4c3d-9f2a-1b2c3d4e5f6a  (bilink-format.md:104:1 → todo)
```

## Exit codes

| Código | Condición |
|--------|-----------|
| `0` | ítem creado exitosamente |
| `1` | selector inválido, ID padre no encontrado, o archivo sin historial git |
| `2` | error de escritura |
| `3` | sin conectividad con el servidor worklist |
