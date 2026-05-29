# Comando: `impact scan`

Lee los `accepted.N` almacenados en los `.bilink` del proyecto y calcula qué
archivos tienen commits no analizados desde el último anchor aceptado.

## Uso

```
impact scan [--path <dir>] [--chain <uuid>]
```

## Comportamiento

1. Recorre todos los `.bilink` bajo el directorio raíz (buscando `.bilink/`).
2. Para cada endpoint estructural con entradas en `accepted.N`, toma el commit
   de la última entrada aceptada y ejecuta:
   ```
   git log <accepted.N.last.commit>..HEAD -- <file>
   ```
3. Si el resultado es no vacío, el archivo tiene commits intercedidos desde
   el anchor — se genera un Impact Report en `.impact/reports/<uuid>.impact`.
4. Si ya existe un reporte abierto para la misma combinación (chain + file +
   commit_anchor), no genera uno nuevo.
5. Imprime el resumen en stdout.

## Flags

| Flag | Descripción |
|------|-------------|
| `--path <dir>` | Restringe el scan a `.bilink` files bajo ese subdirectorio |
| `--chain <uuid>` | Restringe el scan a una cadena específica |

## Salida

```
scan: 2 bilink(s) con cambios desde anchor

  src/Persona.java   chain abc12345  2 commits
  specs/voting.yaml  chain abc12345  1 commit

generados: .impact/reports/f3a1...  .impact/reports/9c2b...
```

Si no hay cambios:

```
scan: todos los bilinks están en su anchor  (12 revisados)
```

## Exit codes

- `0`: sin cambios detectados
- `1`: se generaron reportes (hay cambios para revisar)
- `2`: error en la ejecución

## Cuándo usarlo

`impact scan` es la entrada principal del ciclo de análisis. Puede invocarse:
- manualmente antes de una revisión
- como parte de un git hook post-merge o post-fetch
- en un CI pipeline para detectar drift entre ramas

No requiere `bilinker watch` en ejecución — la fuente de verdad es el
historial de git accesible localmente.
