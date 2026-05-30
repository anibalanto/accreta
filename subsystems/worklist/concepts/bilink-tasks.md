# Archivo `.tasks` por bilink

Un bilink puede tener cero, una o múltiples tasks asociadas. La asociación se registra en un archivo `<bilink-uuid>.tasks` en la raíz del worklist.

## Ubicación y nombre

```
accreta/.stratum/worklist/
  7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.tasks   ← tasks del bilink con ese UUID
  1.task
  2.task
  ...
```

El nombre del archivo es el UUID del bilink — el mismo UUID que identifica al bilink en el sistema bilinker. La extensión `.tasks` lo distingue de los ítems.

## Formato

Una línea por task ID (base-36):

```
3
1a
2f
```

## Creación y mantenimiento

El servidor worklist crea o actualiza el archivo `.tasks` automáticamente cuando se crea una task con `worklist new capture <selector>`. El cliente nunca escribe este archivo directamente.

## Lookup O(1)

La relación bilink → tasks es una comprobación de existencia de archivo:

- **¿Tiene este bilink tasks?** → comprobar si `<uuid>.tasks` existe
- **¿Cuáles son?** → leer el archivo, una línea por ID
- **¿Dónde vive la task `3`?** → `3.task` (o dentro de una carpeta padre)

No hay scan, no hay índice adicional.

## Relación inversa

Cada ítem `.task` tiene `source_bilink: <uuid>` en su frontmatter — la relación inversa (task → bilink) también es O(1).

## Invariantes

1. El nombre del archivo es un UUID v4 válido con extensión `.tasks`.
2. Cada línea es un ID base-36 válido que corresponde a un ítem existente.
3. El archivo solo existe si hay al menos una task asociada al bilink.
4. Solo el servidor worklist crea y modifica este archivo.
