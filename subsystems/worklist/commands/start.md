# Comando: `worklist start`

Marca un item como en curso.

## Uso

```
worklist start <prefix>
```

## Comportamiento

1. Resuelve el item por prefijo.
2. Actualiza `status: in-progress` y `updated_at` en el frontmatter.

## Salida

```
in-progress: 7f3d8e9a  Implementar método vote
```

## Exit codes

- `0`: item actualizado
- `1`: prefijo no encontrado o ambiguo
