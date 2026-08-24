# Especificación: comando `bilinker chain`

## Propósito

Gestiona las cadenas de bilinks: crear nuevas cadenas, consultar su estado completo y listar todas las cadenas del proyecto.

## Subcomandos

### `bilinker chain new`

Crea una nueva cadena generando un UUID y los archivos `.bilink` en las layers especificadas.

```
bilinker chain new --tip <layer> "<referencia-estructural>" \
                   [--mid <layer>]... \
                   --tip <layer> "<referencia-estructural>"
```

| Argumento | Descripción |
|---|---|
| `--tip <layer> "<ref>"` | Extremo de la cadena: layer donde vive el nodo + referencia estructural. Se especifica exactamente dos veces. |
| `--mid <layer>` | Layer intermedia. Se puede especificar cero o más veces. |

Genera el UUID, crea el archivo `.bilink/<uuid>.bilink` en cada layer especificada con los endpoints correctos (estructurales en los tips, layer en los mids) y la sección de cache vacía (se completa en el primer `check`).

**Ejemplo:**

```bash
bilinker chain new \
  --tip . "specs :: voting.yaml :: (block_mapping_pair key: (flow_node) @n0 (#eq? @n0 \"impl\") value: (_) @target)" \
  --tip .stratum/impl "java-demo :: src/.../Persona.java :: (class_declaration ...)"
```

**Salida:**

```
Created chain: 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a

  .bilink/7f3d8e9a-….bilink                    (tip)
  .stratum/impl/.bilink/7f3d8e9a-….bilink      (tip)

Run 'bilinker check' to populate cache.
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
