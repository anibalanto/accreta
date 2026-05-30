# Comando: `worklist show`

Muestra el detalle completo de un item: frontmatter, cuerpo, bilink origen, y lista de hijos.

## Uso

```
worklist show <id>
```

## Comportamiento

1. Localiza el item por ID en todo el árbol de `accreta/.stratum/worklist/`.
2. Imprime el frontmatter formateado, el cuerpo Markdown y el fragmento
   de la capa superior al que apunta el bilink origen.
3. Lista los hijos directos con su estado.

## Salida

```
Task: Implementar método vote
ID: 3
Status: open
Created: 2026-05-26T10:00:00Z
Source: b2c3d4e5-....bilink → specs/voting.yaml:10:5

  (block_mapping_pair
    key: (flow_node) @n0 (#eq? @n0 "impl")
    value: (_) @target)

Children: none
```

## Exit codes

- `0`: item encontrado y mostrado
- `1`: ID no encontrado
