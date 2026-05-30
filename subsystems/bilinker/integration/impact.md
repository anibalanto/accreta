# Integración con impact

Bilinker detecta el drift; impact analiza su alcance.

## Flujo

```
bilinker check .  →  ALTERED en specs/voting.yaml
impact scan       →  calcula commits intercedidos desde commit.N
impact thread     →  abre hilo de discusión
```

## Datos compartidos

Impact lee los `.bilink` directamente — no invoca bilinker en tiempo de ejecución. La coordinación es a nivel de datos:

- `commit.N`: impact usa este valor para calcular `git log <commit>..HEAD -- <file>`
- `state.N`: impact filtra endpoints en ALTERED, DELETED, CHAIN_DIRTY

## Tras la resolución

Cuando el thread de impact concluye con un cambio aceptado:

```
bilinker accept <uuid>.<N>   → hash.N y commit.N actualizados, state OK
impact thread resolve        → thread cerrado
```
