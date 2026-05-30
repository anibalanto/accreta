# Comando: `impact report`

Genera un Impact Report puntual para un archivo o bilink específico.

## Uso

```
impact report <file>
impact report --bilink <uuid>
```

## Comportamiento

1. Resuelve el target: archivo en el filesystem o UUID de un `.bilink`.
2. Busca todos los chains que referencian ese target.
3. Para cada endpoint estructural con entradas en `accepted.N`, toma el commit
   de la última entrada aceptada y ejecuta:
   ```
   git log <accepted.N.last.commit>..HEAD -- <file>
   ```
4. Construye el Impact Report con la lista de chains afectadas, el diff
   acumulado y los commits intercedidos.
5. Guarda el reporte en `.impact/reports/<uuid>.impact`.
6. Imprime el reporte en stdout.

## Flags

| Flag | Descripción |
|------|-------------|
| `--bilink <uuid>` | Genera el reporte para un bilink específico por UUID |
| `--no-save` | Imprime el reporte sin guardarlo en `.impact/reports/` |

## Salida

```
Impact Report  f3a19c2b

trigger:  manual
file:     src/Persona.java
commit:   a1b2c3d4

chains afectadas:
  abc12345  src/Persona.java ↔ specs/voting.yaml
    ALTERED  src/Persona.java
    2 commits desde anchor a1b2c3d4:
      d5e6f7a8  Anibal  2026-05-20  refactor vote method signature
      9b0c1d2e  Anibal  2026-05-22  add validation in vote()

reporte guardado: .impact/reports/f3a19c2b-...
```

## Relación con `impact scan`

`impact report` es la versión puntual y manual de `impact scan`. Scan barre todo el proyecto; report se enfoca en un target específico. Ambos generan el mismo formato de artefacto `.impact`.

## Exit codes

- `0`: reporte generado sin cambios significativos
- `1`: reporte generado con cambios detectados
- `2`: target no encontrado o error en la ejecución
