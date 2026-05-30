# Especificación: comando `bilinker index`

## Propósito

Construye o reconstruye el índice `.bilink/.index` de una o más layers, permitiendo que `bilinker get` opere en O(1) en lugar de O(N).

## Firma

```
bilinker index [<path>] [--recursive]
```

| Argumento | Tipo | Descripción |
|---|---|---|
| `<path>` | path | Layer o directorio raíz donde construir el índice. Por defecto: layer actual (cwd). |
| `--recursive` | flag | Construye el índice en todas las layers descendientes también. |

## Comportamiento

1. Localiza el directorio `.bilink/` en `<path>`.
2. Escanea todos los `.bilink` de esa layer (excluyendo `.pending/`).
3. Para cada endpoint estructural, extrae `(archivo, uuid, N)`.
4. Escribe `.bilink/.index` con una línea `<archivo>\t<uuid>.<N>` por entrada.
5. Si `--recursive`, repite para cada layer descendiente encontrada en `.stratum/`.

El índice generado reemplaza cualquier versión anterior — la operación es idempotente.

## Salida

```
$ bilinker index --recursive

index: .bilink/.index              (3 entradas)
index: .stratum/impl/.bilink/.index   (7 entradas)
index: .stratum/impl/.bilink/.index   (2 entradas)
```

Con `--quiet`, solo imprime errores.

## Subcomando `status`

```
bilinker index status [<path>] [--recursive]
```

Reporta si el índice de cada layer está actualizado, desactualizado o ausente, sin modificar ningún archivo.

```
$ bilinker index status --recursive

.bilink/.index              OK        (actualizado)
.stratum/impl/.bilink/.index          STALE     (2 bilinks más nuevos)
.stratum/impl/.bilink/.index  MISSING
```

## Cuándo usarlo

- Después de un `git pull` con cambios en `.bilink/`.
- Después de crear o modificar bilinks en lote (`chain new` múltiple, merge).
- En CI, como paso previo a comandos que usan `bilinker get` en proyectos grandes.
- Al agregar bilinker a un proyecto existente con muchos bilinks.

## Relación con otros comandos

`bilinker get` usa el índice si está disponible y actualizado; si no, hace scan O(N) sin error. `bilinker index` es la única forma de construir o actualizar el índice — ningún otro comando lo escribe.

## Exit codes

| Código | Condición |
|---|---|
| 0 | Índice construido exitosamente. |
| 1 | Error de lectura/escritura en alguna layer. |
