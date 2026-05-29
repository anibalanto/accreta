# Especificación: comandos `stratum`

## Propósito

CLI de navegación por el árbol de capas de un proyecto Stratum. Permite obtener
paths hacia capas superiores e inferiores desde la posición actual, y visualizar
el contexto de profundidad en el árbol.

El argumento es un path Stratum — ver [paths.md](paths.md) para la gramática completa.

El argumento de navegación se pasa siempre entre comillas simples porque contiene
caracteres especiales del shell (`>`, `<`):

```bash
cd $(stratum '>tech-decisions>impl')
cd $(stratum '<<')
```

## Sintaxis general

```
stratum '<query>'
```

Una query es una secuencia de operadores `>` y `<` que describe una navegación
relativa en el árbol de capas.

## Operadores

### `>` — bajar una capa

`>name` desciende a la sub-capa `name` dentro del directorio `.stratum/` actual.
Varios `>` encadenados descienden múltiples niveles:

```
'>tech-decisions'              →  .stratum/tech-decisions
'>tech-decisions>impl'         →  .stratum/tech-decisions/.stratum/impl
'>a>b>c'                       →  .stratum/a/.stratum/b/.stratum/c
```

**Validación:** cada segmento del path resultante debe existir en el filesystem.
Si algún segmento no existe, imprime error a stderr y sale con código 1.

### `<` — subir una capa

Cada `<` sube un nivel Stratum (equivale a `../..` en el filesystem, ya que
cada capa ocupa dos directorios: `.stratum/<nombre>/`). Varios `<` encadenados
suben múltiples niveles:

```
'<'    →  ../..
'<<'   →  ../../../..
'<<<'  →  ../../../../../../
```

**Validación:** la cantidad de `<` no puede superar la profundidad actual en el
árbol. Si se intenta subir más niveles de los disponibles, imprime error a stderr
y sale con código 1.

### `>?` — consultar capas disponibles hacia abajo

Lista los nombres de las sub-capas disponibles en `.stratum/` del directorio actual.

```
'>?'   →  impl  tech-decisions
```

Si no existe `.stratum/` en el directorio actual, retorna lista vacía.

### `<?` — consultar profundidad hacia arriba

Indica cuántos niveles Stratum hay por encima de la posición actual.

```
'<?'   →  2
```

## Sin argumentos

Muestra el contexto completo de la posición actual: profundidad y sub-capas.

```
$ stratum
up:   2  (../../../..)
down: impl  tech-decisions
```

```
$ stratum
up:   0
down: stratum  bilinker
```

```
$ stratum
up:   1  (../..)
down: (ninguna)
```

## Ejemplos de uso

```bash
# Navegar a una sub-capa
cd $(stratum '>tech-decisions>impl')

# Volver dos niveles arriba
cd $(stratum '<<')

# Ver qué sub-capas hay disponibles
stratum '>?'

# Ver a cuántos niveles de profundidad estás
stratum '<?'

# Usar el path en un comando
ls $(stratum '>impl')/src
```

## Códigos de salida

| Código | Condición |
|---|---|
| 0 | Operación exitosa. |
| 1 | Path no encontrado (`>`) o profundidad insuficiente (`<`). |
| 2 | Query inválida (sintaxis incorrecta). |

## Propiedades

- **Solo lectura**: no crea ni modifica archivos.
- **Sin dependencia de git**: opera únicamente sobre la estructura de directorios.
- **Composable**: resultado a stdout, errores a stderr.
