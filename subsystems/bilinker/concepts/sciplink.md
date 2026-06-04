# Concepto: sciplink

## Propósito

Un **sciplink** es una referencia unidireccional a un símbolo del índice SCIP, con el hash del código fuente de ese símbolo en el momento de la última aceptación. Permite que `bilinker check` detecte cambios en cualquier elemento del subgrafo de llamadas alcanzable desde un endpoint bilink.

Los sciplinks son siempre generados automáticamente por `bilinker check` cuando detecta callees nuevos en el subgrafo. No se crean ni se editan manualmente. SCIP es infraestructura transparente — el usuario no lo invoca directamente.

## Ubicación

```
.bilink/
  index/
    index           ← índice bilinker
    index.scip      ← caché SCIP (gitignored)
  sciplink/
    <id-normalizado>.sciplink
```

Viven dentro de `.bilink/sciplink/` para no mezclarse con los archivos `.bilink` de UUID.

## Nombre de archivo

El nombre se deriva del symbol ID SCIP aplicando estas transformaciones:
- Eliminar el prefijo `scip://`
- Reemplazar `/` con `.`
- Reemplazar `#` con `..`
- Eliminar `()` y espacios

Ejemplos:

| Symbol ID | Nombre de archivo |
|-----------|------------------|
| `scip://rust . voting/Repo#save().` | `rust.voting.Repo..save.sciplink` |
| `scip://rust . voting/Audit#log().` | `rust.voting.Audit..log.sciplink` |
| `scip://typescript npm voting 1.0.0 src/repo.ts/Repo#save().` | `typescript.npm.voting.1.0.0.src.repo.ts.Repo..save.sciplink` |

## Formato

```
symbol: scip://rust . voting/Repo#save().
file:   src/repo.rs
range:  45~89
hash:   a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
commit: d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3
state:  OK
resolved_at: 2026-06-04T10:00:00Z
```

| Campo | Descripción |
|-------|-------------|
| `symbol` | Symbol ID SCIP completo. Identifica unívocamente el elemento en el índice. |
| `file` | Ruta del archivo fuente relativa a la raíz de la layer. Provista por SCIP. |
| `range` | Byte range absoluto (`start~end`) del símbolo en `file`. Provisto por SCIP. |
| `hash` | SHA-256 del código fuente en `file[range]` al momento de la última aceptación. |
| `commit` | SHA-1 del commit HEAD del repo al momento de la última aceptación. |
| `state` | Estado de consistencia calculado por `bilinker check`. |
| `resolved_at` | Timestamp UTC del último check. |

`hash`, `commit`, `state` y `resolved_at` están ausentes hasta el primer `bilinker accept`.

## Estados

| Estado | Condición | Resolución |
|--------|-----------|------------|
| `OK` | Hash actual del código en `file:range` == `hash` | — |
| `ALTERED` | Symbol existe; hash del código difiere | `bilinker accept` |
| `RENAMED` | Symbol ID ausente del índice; encontrado por `file:range` similar o `RENAMED_FROM` | `bilinker check` lo resuelve automáticamente |
| `DELETED` | Symbol ID ausente del índice; sin candidato de rename | `bilinker check --prune` para eliminar |

## Ciclo de vida

```
bilinker chain new   →  crea .bilink con campo subgraph: (si SCIP disponible)
bilinker check       →  descubre callees, crea .sciplink con hash (state: OK)
                     →  en runs posteriores: detecta ALTERED / RENAMED / DELETED
bilinker accept      →  confirma ALTERED (el callee cambió intencionalmente)
bilinker check --prune → elimina .sciplink con state DELETED
```

No hay paso de aceptación inicial — `bilinker check` crea el `.sciplink` con `hash` y `state: OK` directamente al descubrir un callee nuevo.

## Relación con bilinks

Un sciplink no es un bilink — no tiene dos endpoints, no forma cadenas, no tiene semántica de layer. Es un checkpoint de hash sobre un símbolo SCIP.

El vínculo con un bilink es indirecto: los campos `subgraph.N` del `.bilink` declaran los símbolos raíz; `bilinker check` atraviesa el grafo SCIP desde cada símbolo y verifica los `.sciplink` correspondientes.

Un mismo `.sciplink` puede ser alcanzable desde múltiples bilinks raíz (si dos funciones distintas llaman al mismo callee). El archivo `.sciplink` es compartido — sus campos `hash`, `commit` y `state` reflejan el estado global del símbolo, no el de una cadena particular.
