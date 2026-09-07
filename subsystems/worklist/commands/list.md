# Comando: `worklist list`

Lista los items del árbol donde se lo corre, con su estado y jerarquía.

**Y eso es una vista, no el corpus.** El cliente no tiene el panorama —[vive de un solo lado](../concepts/sync.md#el-panorama-vive-en-un-solo-lado-y-la-ventana-en-los-dos)—, así que parado en una ventana este comando contesta sobre sus ítems y no sobre todos. La pregunta por el corpus entero es del servidor, que es el que tiene el tronco.

## Uso

```
worklist list [--status <estado>] [--under <id>] [--flat]
```

## Comportamiento

Sin flags, muestra el árbol completo de items indentado por jerarquía, ordenado por ID (orden de creación) dentro de cada nivel.

## Flags

| Flag | Descripción |
|------|-------------|
| `--status <estado>` | Filtra por `open`, `in-progress`, o `done` |
| `--under <id>` | Muestra solo el subárbol del item con ese ID |
| `--flat` | Lista plana, una línea por item sin indentación |

## Salida

```
1   [open]         Implementar hash.N en Rust           .epic
  2   [in-progress]  Parsear hash.N en BiLinkFile        .user-story
    3   [open]        Actualizar struct BiLinkFile        .task
    4   [open]        Actualizar semántica de parseo      .task
  5   [open]          Actualizar check.rs                .user-story
6   [open]           Comando bilinker accept             .epic
7   [open]           Specs de impact                     .epic
```

Con `--flat`:

```
1   [open]         Implementar hash.N en Rust
2   [in-progress]  Parsear hash.N en BiLinkFile
3   [open]         Actualizar struct BiLinkFile
```

## Exit codes

- `0`: lista impresa exitosamente
- `2`: error de lectura
