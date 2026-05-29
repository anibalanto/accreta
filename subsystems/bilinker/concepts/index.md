# Especificación: índice de bilinks por archivo

## Problema

Encontrar todos los bilinks que referencian un archivo dado requiere escanear
todos los `.bilink` del proyecto — O(N) donde N es el número de bilinks totales.
Para proyectos grandes esto puede ser lento.

## Solución: `.bilink/.index`

Cada layer puede tener un archivo `.bilink/.index` que mapea archivos fuente
a los endpoints de bilinks que los referencian. La búsqueda pasa de O(N) a O(1).

El índice es **opcional y regenerable**. Si no existe o está desactualizado,
los comandos que lo usan caen al scan O(N). Nunca es fuente de verdad — siempre
puede reconstruirse a partir de los `.bilink`.

## Formato

```
docs/api-spec.md	7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.0
src/Service.java	a3f9c821-4e5b-4c3d-9f2a-1b2c3d4e5f6a.1
src/Service.java	7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.0
```

- Una entrada por línea: `<archivo>\t<uuid>.<N>`, separados por tabulador.
- Un mismo archivo puede tener múltiples entradas.
- Las líneas que comienzan con `#` son comentarios y se ignoran.
- Encoding UTF-8 sin BOM.

La ruta del archivo es relativa a la raíz de la layer que contiene el `.bilink/`.
El resto de la información (query, range, state) vive en el `.bilink` y se lee
de ahí cuando es necesario.

## Ubicación

```
<layer-root>/
  .bilink/
    .index              ← índice de esta layer
    7f3d8e9a-….bilink
    a3f9c821-….bilink
```

Cada layer tiene su propio índice. El índice solo cubre los endpoints estructurales
de los bilinks que viven en esa layer — no endpoints layer ni bilinks de otras layers.

## Detección de obsolescencia

El índice se considera válido si su mtime es mayor o igual al mtime del `.bilink`
más reciente en el mismo directorio. Si algún `.bilink` es más nuevo que el índice,
el índice está desactualizado.

Los comandos que usan el índice lo verifican antes de usarlo:
- Índice válido → lookup O(1).
- Índice ausente o desactualizado → scan O(N) sobre los `.bilink` de la layer.

El scan de fallback nunca regenera el índice automáticamente — eso es
responsabilidad explícita de `bilinker index`.

## Relación con `range.N`

El índice responde "¿qué bilinks referencian este archivo?" en O(1).
Una vez recuperados los UUIDs relevantes, leer `range.N` de cada `.bilink`
es O(1) por bilink — ya está resuelto en el archivo.

El índice no almacena ranges ni queries. Esa información vive en el `.bilink`
y puede cambiar con cada `check` o `accept`.

## Git

El índice puede estar bajo control de versiones o en `.gitignore`, según la
preferencia del equipo:

- **Bajo git**: el índice refleja el estado del repo; los merges pueden
  generar conflictos triviales que se resuelven con `bilinker index`.
- **En `.gitignore`**: cada desarrollador construye su índice localmente.
  Recomendado para proyectos con muchos cambios frecuentes en bilinks.

## Invariantes

1. El índice es un derivado — nunca modifica la fuente de verdad (los `.bilink`).
2. Un índice válido es consistente con los `.bilink` actuales de su layer.
3. La ausencia del índice nunca es un error — degrada a O(N) silenciosamente.
4. El índice no cubre endpoints layer, solo endpoints estructurales.
