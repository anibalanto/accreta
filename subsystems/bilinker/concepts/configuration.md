# Especificación: Raíz del proyecto

## Resolución de la raíz

bilinker determina la raíz del proyecto buscando desde el directorio de trabajo
hacia arriba (directory walk). Se detiene en el primer directorio que contenga
alguno de los siguientes marcadores:

1. `.bilink/` — directorio de bilinks del proyecto (marcador primario)
2. `.git/` — raíz de repositorio git (marcador secundario)

Si ninguno se encuentra, se usa el directorio de trabajo actual como raíz.
Esto permite usar bilinker en proyectos nuevos sin ningún paso de inicialización.

No existe ningún archivo de configuración.

## Uso con múltiples capas y repositorios

Cada capa puede ser un repositorio git independiente. bilinker siempre opera en
el contexto de **una sola capa**: la raíz encontrada es la raíz de la capa actual.

Esto funciona sin configuración adicional porque **solo se puede crear un bilink
sobre un archivo que esté presente en el filesystem local**, lo que implica que
el repositorio de esa capa ya está clonado. Por lo tanto:

- Los endpoints estructurales de una capa siempre referencian archivos del repo
  de esa capa.
- `git log` siempre corre en el root correcto: el de la capa donde vive el bilink.
- Para verificar una cadena completa, se corre `bilinker check` en cada capa
  independientemente.

## Lenguaje de los archivos

El lenguaje (gramática tree-sitter) se determina automáticamente por la extensión
del archivo referenciado:

| Extensión | Lenguaje |
|-----------|----------|
| `.java`   | java     |
| `.rs`     | rust     |
| otros     | text     |

## Invariantes

- La raíz se resuelve una vez por invocación del CLI.
- Todos los paths de archivos en endpoints estructurales son relativos a la raíz de su capa.
- No se requiere configuración explícita del lenguaje: la extensión es suficiente.
- No existe `.bilinker.toml` ni ningún otro archivo de configuración.
