# Comando: `bilinker accept`

Registra el estado actual de un endpoint como aceptado, estableciendo `hash.N` y `commit.N` en el archivo `.bilink`.

## Uso

```
bilinker accept <uuid>.<N>
bilinker accept <uuid>.<N> <hash> <commit>
```

| Argumento | Descripción |
|-----------|-------------|
| `<uuid>.<N>` | Endpoint a aceptar: UUID del bilink + índice (0 o 1). |
| `<hash>` | SHA-256 del fragmento (o del `.bilink` adyacente para layer). Si se omite, bilinker lo calcula. |
| `<commit>` | SHA-1 del commit del repo. Si se omite, bilinker usa HEAD. |

## Comportamiento

1. Resuelve el archivo `.bilink/<uuid>.bilink`.
2. Calcula o usa el `hash` provisto del contenido actual del endpoint.
3. Calcula o usa el `commit` de HEAD en el repo correspondiente.
4. Establece `hash.N` y `commit.N` en el archivo (sobrescribe valores anteriores).
5. Actualiza `state.N` a `OK` y `resolved_at`.

El archivo `.bilink` cambia — esto dispara `CHAIN_DIRTY` en el nodo adyacente de la cadena en el próximo `check`.

## Ejemplos

```bash
# Aceptar el estado actual (hash y commit calculados automáticamente)
bilinker accept 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.1

# Aceptar con valores explícitos (útil para scripting)
bilinker accept 7f3d8e9a.1 \
  479922a1ee55cc7f9f4f323bb002018e1b4e1cda65e069e0f6f4645926ce25ee \
  d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3
```

## Salida

```
accepted: 7f3d8e9a.1
  hash:   479922a1…
  commit: d4e5f6a7…
  state:  OK

note: nodo adyacente detectará CHAIN_DIRTY en el próximo check
```

## Cuándo usarlo

- Tras `bilinker check` cuando el estado es `PENDING`, `ALTERED` o `CHAIN_DIRTY`
  y el cambio es coherente con la intención del bilink.
- No es necesario para los estados auto-reparables (MOVED, DISPLACED, REANCHORED,
  EXPANDED) — esos se resuelven con `bilinker apply`.

## Exit codes

- `0`: aceptación registrada exitosamente
- `1`: UUID no encontrado, endpoint inválido, o estado no aceptable
