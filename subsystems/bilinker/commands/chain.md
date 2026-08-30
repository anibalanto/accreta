# Especificación: comando `bilinker chain`

## Propósito

Gestiona las cadenas de bilinks: crear nuevas cadenas, consultar su estado completo y listar todas las cadenas del proyecto.

## Subcomandos

### `bilinker chain new`

Crea una cadena: genera un UUID y escribe un bilink en cada capa.

```
bilinker chain new --tip <STRATUM_PATH[:LINE:COL]> \
                   [--mid <STRATUM_PATH>]... \
                   --tip <STRATUM_PATH[:LINE:COL]> \
                   [--kind <valor>] [--name.0 <etiqueta>] [--name.1 <etiqueta>]
```

| Argumento | Descripción |
|---|---|
| `--tip <ref>` | Extremo de la cadena: path Stratum al archivo, con posición opcional. Exactamente dos veces. |
| `--mid <layer>` | Capa intermedia. Cero o más veces. |
| `--kind <valor>` | El [`kind`](../concepts/bilink.md) del bilink. |
| `--name.N <etiqueta>` | El `name` del endpoint N. |

Cada `--tip` captura el fragmento —sin posición, el archivo completo— y el endpoint queda apuntando a ese capture. Los mids llevan dos endpoints `path`. Ningún `accepted` se escribe: la cadena nace en `PENDING`.

**`--kind` existe para no depender de una edición a mano.** `kind` y `name` son campos de declaración, y todo archivo de bilinker sale de un comando: sin el flag, la única forma de poblarlos sería abrir el YAML, que es justamente lo que el formato no pide de nadie.

**Ejemplo:**

```bash
bilinker chain new \
  --tip 'commands/check.md:63:1' \
  --tip '>impl/crates/bilinker/src/check.rs:405:1'
```

**Salida:**

```
Created chain: 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a

  .bilink/7f3d8e9a-….yaml                    (tip)
  .stratum/impl/.bilink/7f3d8e9a-….yaml      (tip)

Los dos endpoints quedan en PENDING. Revisar con `bilinker get` y aprobar con `bilinker accept`.
```

---

### `bilinker chain status <uuid>`

Muestra el estado completo de una cadena recorriendo todos sus nodos.

```
bilinker chain status <uuid>
```

**Salida:**

```
$ bilinker chain status 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a

Chain: 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a  [DIRTY]

  .bilink/                         (tip)   (OK, CHAIN_DIRTY)
    link.0  specs :: voting.yaml#impl       OK
    link.1  → .stratum/impl                CHAIN_DIRTY

  .stratum/impl/                   (tip)   (CHAIN_DIRTY, ALTERED)
    link.0  → spec layer                   CHAIN_DIRTY
    link.1  java-demo :: Persona#vote      ALTERED
              source: commit c7d3e9f "Inline comparator"
```

**Estado global de la cadena:**

| Estado | Condición |
|---|---|
| OK | Todos los nodos y fragmentos en estado OK. |
| DIRTY | Algún nodo tiene CHAIN_DIRTY. |
| BROKEN | Algún nodo tiene estado terminal (ALTERED, DELETED, UNANCHORED, BROKEN). |

---

### `bilinker chain list`

Lista todas las cadenas encontradas en el proyecto a partir del directorio actual.

```
bilinker chain list
```

**Salida:**

```
$ bilinker chain list

7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a  [DIRTY]   spec → impl
3a4b5c6d-2e3f-4a5b-9c6d-7e8f9a0b1c2d  [OK]      spec → impl
f1e2d3c4-5a6b-7c8d-9e0f-1a2b3c4d5e6f  [BROKEN]  spec → impl
```

## Código de salida

| Código | Condición |
|---|---|
| 0 | Operación exitosa. |
| 1 | Error: UUID no encontrado, layer inválida, UUID duplicado en una layer. |
