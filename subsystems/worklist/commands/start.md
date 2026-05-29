# Comando: `worklist start`

Marca un item como en curso.

## Uso

```
worklist start <id>
```

## Comportamiento

1. Resuelve el item por ID.
2. Actualiza `status: in-progress` y `updated_at` en el frontmatter.

## Salida

```
in-progress: 3  Implementar método vote
```

## Exit codes

- `0`: item actualizado
- `1`: ID no encontrado
