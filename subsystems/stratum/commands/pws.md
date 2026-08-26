# Especificación: comando `stratum pws`

## Propósito

Imprime el path Stratum del directorio de trabajo actual, expresado desde la raíz externa del proyecto (`*`). Es el equivalente de `pwd` pero en notación Stratum.

## Firma

```
stratum pws
```

Sin argumentos.

## Comportamiento

1. Determina la raíz externa del proyecto (`*`) buscando el ancestro `.git` más lejano.
2. Calcula el path relativo desde esa raíz hasta el directorio actual.
3. Convierte los segmentos `.stratum/<name>` en tokens `>name`.
4. Imprime la expresión Stratum resultante.

## Salida

```
$ cd /home/user/accreta/subsystems/stratum/.stratum/impl/crates/stratum-cli/src
$ stratum pws
*/subsystems/stratum>impl/crates/stratum-cli/src
```

```
$ cd /home/user/accreta/subsystems/stratum
$ stratum pws
*/subsystems/stratum
```

```
$ cd /home/user/accreta
$ stratum pws
*
```

## Usos típicos

```bash
# Ver dónde estás en el árbol Stratum
stratum pws

# Usar en scripts para construir referencias portátiles
CURRENT=$(stratum pws)
echo "Estoy en: $CURRENT"
```

## Códigos de salida

| Código | Condición |
|--------|-----------|
| 0 | Éxito. |
| 1 | No se encontró raíz `.git` en ningún ancestro. |

## Propiedades

- **Solo lectura**: no modifica ningún archivo.
- El path resultante es válido como argumento para `stratum '<expr>'`.
