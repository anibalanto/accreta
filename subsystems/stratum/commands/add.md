# Especificación: `stratum add`

## Propósito

Registra una nueva sub-capa declarando su repositorio git. Crea el archivo
`.<nombre>.toml` en `.stratum/` del directorio actual. No clona el repositorio
— solo deja el mapa para que `stratum pull` lo haga después.

## Sintaxis

```
stratum add <nombre> <remote> [branch]
```

| Argumento | Descripción |
|-----------|-------------|
| `<nombre>` | Nombre de la sub-capa (debe ser un identificador válido de directorio) |
| `<remote>` | URL del repositorio git (ssh o https) |
| `[branch]` | Rama a usar. Default: `main` |

## Ejemplo

```bash
# Parado en bilinker/
stratum add impl git@github.com:anibalanto/bilinker.git main
# → crea .stratum/.impl.toml

# Con branch explícita
stratum add tech-decisions git@github.com:org/td.git develop
```

## Archivo creado

`.stratum/.<nombre>.toml`:

```toml
remote = "git@github.com:anibalanto/bilinker.git"
branch = "main"
```

Si `.stratum/` no existe, se crea automáticamente.

## Errores

| Condición | Mensaje | Código |
|-----------|---------|--------|
| `.stratum/.<nombre>.toml` ya existe | `ya existe — usa --force para sobreescribir` | 1 |
| `<nombre>` vacío o inválido | `nombre de capa inválido` | 2 |

## Propiedades

- **Idempotente con `--force`**: sobreescribe el `.toml` existente.
- **No clona**: solo registra. Ver `stratum pull` para clonar.
- **Crea `.stratum/`** si no existe.
