# Integración con impact

Bilinker detecta el drift; impact analiza su alcance.

## Flujo

```
bilinker check .  →  ALTERED en specs/voting.yaml
impact scan       →  calcula commits intercedidos desde el commit aceptado
impact thread     →  abre hilo de discusión
```

## Datos compartidos

Impact no lee los `.bilink` ni invoca bilinker: consulta el grafo a través de [lattice](../../lattice/overview.md), que toma las aristas de bilinker vía `bilinker graph --format json` y las entrega ya resueltas entre capas.

Los dos campos que impact necesita viajan en la arista:

- `commit`: baseline de `git log <commit>..HEAD -- <file>`
- `state`: para filtrar endpoints en ALTERED, DELETED, CHAIN_DIRTY

Bilinker sigue siendo el dueño de esos valores — lattice los transporta sin interpretarlos.

## Tras la resolución

Cuando el thread de impact concluye con un cambio aceptado:

```
bilinker accept <uuid>.<N>   → accepted actualizado, state OK
impact thread resolve        → thread cerrado
```
