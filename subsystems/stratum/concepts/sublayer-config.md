# Configuración de subcapas

Cada subcapa de `.stratum/` puede tener un archivo de configuración `.<nombre>.toml` en el directorio `.stratum/` que la contiene. El archivo declara cómo obtener ese repositorio git y cualquier configuración específica de la subcapa.

## Ubicación y nombre

El archivo se llama `.<nombre>.toml` donde `<nombre>` es exactamente el nombre del directorio de la subcapa. Vive junto a la subcapa en el mismo `.stratum/`:

```
bilinker/
  .stratum/
    .technical-decisions.toml   ← config de la subcapa "technical-decisions"
    technical-decisions/         ← la subcapa (presente si fue clonada)
      .stratum/
        .impl.toml               ← config de la subcapa "impl" de tech-decisions
        impl/
```

Si la subcapa no está clonada localmente, el archivo de config sigue presente — es el mapa que permite obtenerla con `stratum pull`.

## Formato

```toml
# .stratum/.technical-decisions.toml

repo   = "git@github.com:org/bilinker-tech-decisions.git"
branch = "main"

# Opcional: sparse checkout — solo estas rutas se clonan
sparse = ["concepts/", "adr/"]

# Opcional: clon superficial para repos grandes
depth = 1
```

### Campos

| Campo | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| `repo` | string | sí | URL del repositorio git (ssh o https) |
| `branch` | string | sí | Rama a clonar |
| `sparse` | array de strings | no | Paths a incluir en sparse checkout. Sin este campo se clona completo. |

### `sparse` se escribe acá, y no en todas partes

Para una **subcapa** el conjunto es una decisión de quien la declara: qué parte de una capa vecina le interesa a este proyecto no se deduce de nada.

Para un **repo ajeno** no: ahí el conjunto sale de los bilinks que cruzan la frontera —qué archivos del proveedor hacen falta es exactamente qué fragmentos se referencian— y declararlo a mano sería meter un valor derivado en un archivo de declaración, que además quedaría viejo con el primer vínculo nuevo. Por eso `.bilink/.{alias}.toml` **no tiene** campo `sparse`. Ver [la frontera](../../bilinker/concepts/frontier.md) § "El sparse se calcula, no se declara".
| `depth` | entero | no | Profundidad del clon. Sin este campo se clona el historial completo. |

## Relación con `stratum pull`

`stratum pull` lee los archivos `.toml` del directorio `.stratum/` actual y clona o actualiza cada subcapa declarada:

```bash
stratum pull                         # todas las subcapas del .stratum/ actual
stratum pull technical-decisions     # solo esa subcapa
stratum pull --recursive             # todas las subcapas en todo el árbol
```

Si la subcapa ya existe localmente, `pull` ejecuta `git fetch` + `git checkout` a la rama declarada. Si no existe, la clona.

## Subcapas opcionales y presencia parcial

Un proyecto puede tener configuradas subcapas que no están clonadas localmente. Esto es normal — se trabaja solo con las capas necesarias:

```bash
# Trabajando solo en spec, sin clonar impl
bilinker/
  .stratum/
    .technical-decisions.toml   ✓ config presente
    .impl.toml                  ✓ config presente
    # technical-decisions/ y impl/ ausentes — no clonadas
```

`bilinker check` y `bilinker graph` reportan **`LAYER_UNREACHABLE`** para los endpoints que apuntan a subcapas declaradas y no presentes, sin fallar.

**Declarada y ausente no es lo mismo que ausente a secas.** Una subcapa con su `.toml` y sin directorio es `LAYER_UNREACHABLE` y se arregla con `stratum pull`; una sin `.toml` ni directorio, pero con aceptación previa, es `LAYER_UNCONFIGURED` y lo que le falta es la declaración. Un solo nombre no distinguía *"me falta traer algo"* de *"algo se rompió"*, que es la diferencia que decide si alguien tiene que mirar. El desglose completo está en [la frontera](../../bilinker/concepts/frontier.md) § "Taxonomía de ausencia".

## Invariantes

1. El nombre del archivo `.toml` coincide exactamente con el nombre del directorio
   de la subcapa.
2. Si existe el directorio de la subcapa, debe existir su `.toml` correspondiente.
3. El `.toml` puede existir sin que exista el directorio (subcapa no clonada).
4. Los campos `repo` y `branch` son siempre requeridos.
