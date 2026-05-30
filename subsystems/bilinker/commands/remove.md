# Comando: `bilinker remove`

Elimina un archivo `.bilink` cuando el bilink ya no aplica.

## Uso

```
bilinker remove <uuid>
```

## Comportamiento

1. Resuelve `.bilink/<uuid>.bilink` en la layer actual.
2. Elimina el archivo.
3. Los nodos adyacentes de la cadena detectarán `BROKEN` en el próximo `check`
   y deberán decidir: reparar o también remover. La remoción se propaga hop a hop — no es automática.

## Cuándo usarlo

Para los estados `DELETED` y `BROKEN` donde el bilink ya no tiene sentido: el fragmento fue eliminado definitivamente, el repo fue removido, o el bilink fue creado por error.

No es un sustituto de `bilinker accept` — si el fragmento cambió pero sigue siendo válido, usar `accept`.

## Salida

```
removed: .bilink/7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink

note: nodos adyacentes detectarán BROKEN en el próximo check
```

## Exit codes

- `0`: archivo eliminado
- `1`: UUID no encontrado
