# Especificación: Raíz del proyecto

## Resolución de la raíz

bilinker determina la raíz del proyecto buscando desde el directorio de trabajo hacia arriba (directory walk). Se detiene en el primer directorio que contenga alguno de los siguientes marcadores:

1. `.bilink/` — directorio de bilinks del proyecto (marcador primario)
2. `.git/` — raíz de repositorio git (marcador secundario)

Si ninguno se encuentra, se usa el directorio de trabajo actual como raíz. Esto permite usar bilinker en proyectos nuevos sin ningún paso de inicialización.

No existe ningún archivo de configuración **de la herramienta**. Dentro de `.bilink/` sí hay tres archivos que el propio bilinker escribe y lee —[`version`](format-version.md), que dice qué formato son estos archivos; [`cache/state`](cache.md); y [`head`](ref.md#bilinkhead--de-dónde-salió-el-árbol), que dice de qué commit de la ref salió este árbol— pero no son configuración: nadie los edita a mano y todos salen de un comando.

## Lo que sí es por clon

Dos líneas en `.git/`, y las pone [`init`](../commands/init.md):

| Dónde | Qué | Por qué ahí |
|---|---|---|
| `.git/info/exclude` | `.bilink/` y `.bilink-migrate-*` | `.gitignore` está versionado, y agregarlo modificaría la rama del proyecto |
| `.git/config` | el refspec de `refs/bilink/*` | sin él, `git fetch` trae la rama al día y deja los bilinks viejos |

No son configuración de bilinker sino del repo del usuario, y por eso se piden en vez de escribirse solas: **bilinker arregla solo lo que es suyo, y pide lo que es del repo del usuario.** Sin `init` ningún comando corre.

**Que el clon esté puesto a punto se detecta pidiendo las dos**, y cada una cubre un caso que la otra no. El refspec es la que no puede estar por accidente —un `.bilink/` en el árbol puede venir de antes del corte, y el exclude lo pudo escribir alguien a mano— pero **no existe en un repo sin remoto**, y ahí pedirla sola dejaría al repo sin forma de estar nunca inicializado: todo comando se negaría para siempre. Un repo local sin origen usa la ref igual, sólo que nunca la empuja.

Que sean por clon y no por repo es lo que las obliga a estar acá: no viajan con un `git clone`, así que la primera cosa que hace quien clona es correr `init`.

## Lo que el proyecto sí declara

[La frontera](frontier.md) trae un archivo que antes no existía: `.bilink/.{alias}.toml`, uno por proveedor.

```toml
# .bilink/.hsi.toml
remote = "git@gitlab…:minsal/hsi.git"
branch = "rc-2.32"
```

**No es configuración de bilinker: es una declaración del proyecto sobre de qué depende**, que es contenido — igual que un `.stratum/.impl.toml`. Lo que sigue siendo cierto es lo que importaba de la frase original: la raíz se descubre caminando hacia arriba, el lenguaje se infiere de la extensión, y nadie tiene que configurar la herramienta para que corra.

Que exista es lo que permite que **el `.bilink` no contenga ninguna URL**. Toda la identidad del proveedor queda concentrada en un archivo por proveedor, y no repartida en N bilinks.

## Uso con múltiples capas y repositorios

Cada capa puede ser un repositorio git independiente. bilinker siempre opera en el contexto de **una sola capa**: la raíz encontrada es la raíz de la capa actual.

Esto funciona sin configuración adicional porque **solo se puede crear un bilink sobre un archivo que esté presente en el filesystem local**, lo que implica que el repositorio de esa capa ya está clonado. Por lo tanto:

- Los endpoints estructurales de una capa siempre referencian archivos del repo
  de esa capa.
- `git log` siempre corre en el root correcto: el de la capa donde vive el bilink.
- Para verificar una cadena completa, se corre `bilinker check` en cada capa
  independientemente.

## Lenguaje de los archivos

El lenguaje (gramática tree-sitter) se determina automáticamente por la extensión del archivo referenciado. La tabla completa está en [`commands/capture.md`](../commands/capture.md) § "Lenguajes soportados"; una extensión sin gramática se trata como texto plano, y ahí no hay `hash_ast` ni `RESTYLED`.

## Invariantes

- La raíz se resuelve una vez por invocación del CLI.
- Todos los paths de archivos en endpoints estructurales son relativos a la raíz de su capa.
- No se requiere configuración explícita del lenguaje: la extensión es suficiente.
- No existe `.bilinker.toml` ni ningún otro archivo que configure la herramienta. Los `.bilink/.{alias}.toml` de la frontera declaran de qué depende el proyecto, que es otra cosa.
- Ningún archivo de bilinker se escribe a mano: todos salen de un comando.
- Lo único que bilinker escribe fuera de `.bilink/` son las dos líneas por clon de `init`, en `.git/`. Ninguna rama del proyecto cambia.
