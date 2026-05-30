# Comando: `impact watch`

Ejecuta `impact scan` automáticamente en respuesta a cambios en el filesystem.

## Uso

```
impact watch [--path <dir>]
```

## Comportamiento

1. Observa el directorio raíz del proyecto (o `--path`) con un watcher de
   filesystem.
2. Cuando un archivo modificado es referenciado por algún `.bilink`, ejecuta
   `impact scan --path <dir-del-archivo>` de forma automática.
3. Imprime en stdout cada scan disparado y su resultado.
4. Corre indefinidamente hasta Ctrl-C.

## Salida

```
watching /home/anibal/proyecto  (Ctrl-C to stop)

[2026-05-20T14:32:10Z] cambio detectado: src/Persona.java
  → impact scan  →  1 reporte generado: f3a19c2b
[2026-05-20T14:45:02Z] cambio detectado: specs/voting.yaml
  → impact scan  →  sin cambios
```

## Relación con `bilinker watch`

`bilinker watch` detecta drift estructural en los `.bilink` files. `impact watch` reacciona a esos mismos cambios pero con un análisis más profundo: calcula el blast radius, obtiene el historial de git y genera Impact Reports.

Pueden correr en paralelo o de forma independiente. `impact watch` no depende de que `bilinker watch` esté activo — observa el filesystem directamente.

## Cuándo usar

En un servidor de integración continua o en un entorno de desarrollo donde se quiere análisis de impacto en tiempo real. Para análisis puntuales o en CI, `impact scan` es suficiente.

## Exit codes

- `0`: detenido con Ctrl-C
- `2`: error al inicializar el watcher
