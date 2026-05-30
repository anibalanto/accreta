# Comando: `stratum pull`

Clona o actualiza las subcapas declaradas en los archivos `.<nombre>.toml` del directorio `.stratum/` actual.

## Uso

```
stratum pull [<nombre>] [--recursive]
```

| Argumento / Flag | Descripción |
|-----------------|-------------|
| `<nombre>` | Nombre de una subcapa específica. Sin este argumento, procesa todas las subcapas del `.stratum/` actual. |
| `--recursive` | Procesa también los `.stratum/` de las subcapas clonadas, en profundidad. |

## Comportamiento

Para cada subcapa con `.toml` encontrado:

1. Si el directorio de la subcapa **no existe**: clona el repo declarado en `repo`
   a la rama `branch`, aplicando `sparse` y `depth` si están presentes.
2. Si el directorio **ya existe**: ejecuta `git fetch` + `git checkout` a la rama
   declarada. No destruye cambios locales — falla si hay conflicto.

## Ejemplos

```bash
# Clonar todas las subcapas del .stratum/ actual
stratum pull

# Clonar solo tech-decisions
stratum pull technical-decisions

# Clonar todas las subcapas en todo el árbol recursivamente
stratum pull --recursive

# Actualizar impl ya clonada
cd .stratum/technical-decisions
stratum pull impl
```

## Salida

```
$ stratum pull --recursive

pulling technical-decisions  git@github.com:org/bilinker-tech-decisions.git  main
  cloned  .stratum/technical-decisions/

pulling impl  git@github.com:org/bilinker-impl.git  main
  cloned  .stratum/technical-decisions/.stratum/impl/
```

Si una subcapa ya está actualizada:

```
pulling technical-decisions  already up to date
```

## Exit codes

| Código | Condición |
|--------|-----------|
| `0` | Todas las subcapas procesadas correctamente |
| `1` | Error de red, autenticación, o conflicto local |
| `2` | `.toml` con formato inválido |
