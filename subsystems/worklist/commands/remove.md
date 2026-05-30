# Comando: `worklist remove`

Elimina un item de worklist cuando ya no aplica.

## Uso

```
worklist remove <id>
```

## Comportamiento

1. Resuelve el item por ID.
2. Si el item tiene hijos, pide confirmación explícita (o requiere `--force`).
3. Elimina el archivo y la carpeta homónima si existe.
4. Si el item tiene un `source_bilink`, pregunta si también eliminar ese bilink
   de `.bilink/`. No lo elimina automáticamente — el bilink puede ser útil aunque el item de worklist ya no aplique.

## Flags

| Flag | Descripción |
|------|-------------|
| `--force` | Elimina sin pedir confirmación aunque haya hijos |
| `--remove-bilink` | Elimina también el bilink origen |

## Cuándo usar

Un item se elimina cuando el requerimiento de la capa superior cambió, fue eliminado, o el item fue creado por error. No es un substituto de `worklist done` — `remove` es para items que no deberían haberse creado o que ya no tienen sentido.

## Salida

```
removed: 3  Implementar método vote
bilink b2c3d4e5 kept (use --remove-bilink to delete it)
```

## Exit codes

- `0`: item eliminado
- `1`: ID no encontrado o tiene hijos sin `--force`
