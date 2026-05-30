# Especificación: `stratum tree`

## Propósito

Muestra el árbol de capas Stratum a partir de un path dado. Cada nodo del árbol es un directorio común; sus hijos son las sub-capas (`>name`) o los subdirectorios que contienen capas más profundas.

## Sintaxis

```
stratum tree [<path>]
```

- `<path>`: expresión Stratum de inicio. Default: `*` (raíz del proyecto).

## Salida

Árbol en texto con conectores `├──` / `└──` / `│`. La primera línea es la expresión de inicio; cada hijo indentado es una sub-capa (`>name`) o un subdirectorio que contiene capas.

```
*
└── subsystems
    ├── bilinker
    │   └── >impl
    └── stratum
        └── >impl
```

```
*/subsystems/stratum
└── >impl
```

## Reglas de construcción del árbol

1. **Sub-capas directas** — los directorios dentro de `.stratum/` del nodo actual se muestran como `>name`, ordenados alfabéticamente, antes que los subdirectorios.
2. **Subdirectorios con contenido Stratum** — los subdirectorios no ocultos que contienen capas (directa o recursivamente) se muestran como nodos intermedios, ordenados alfabéticamente.
3. **Subdirectorios sin contenido Stratum** — se omiten.
4. La recursión continúa en cada hijo hasta que no haya más capas.

## Códigos de salida

| Código | Condición |
|--------|-----------|
| 0 | Éxito. |
| 1 | Path no encontrado. |
| 2 | Expresión Stratum inválida. |

## Propiedades

- **Solo lectura**: no modifica el filesystem.
- **Composable**: salida a stdout, errores a stderr.
