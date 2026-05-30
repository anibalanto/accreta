# Skills

Una skill es una rutina de evaluación semántica que impact ejecuta cuando detecta
drift en un vínculo gobernado por un elemento de impacto.

## Qué hace una skill

1. Recibe el elemento de impacto, el documento de decisión (`link.0`), el bilink
   estructural afectado (`link.1`), y el historial git del cambio.
2. Evalúa si el cambio es coherente con la decisión documentada.
3. Produce una evaluación de riesgo y una recomendación como mensaje `kind: analysis`
   en el thread de impact correspondiente.

Una skill no modifica código ni specs — solo lee y produce análisis.

## Configuración

Las skills se configuran por layer en `.impact.toml` en la raíz de la layer:

```toml
# bilinker/.impact.toml

[[skill]]
kind    = "impact"            # tipo de bilink que activa la skill
trigger = ["ALTERED"]         # estados de link.1 que la disparan
model   = "claude-opus-4-7"   # modelo a usar
prompt  = "skills/adr-review.md"  # prompt template (relativo a la layer)

[[skill]]
kind    = "impact"
trigger = ["CHAIN_DIRTY"]
model   = "claude-opus-4-7"
prompt  = "skills/chain-drift.md"
```

Sin `.impact.toml`, impact aplica las skills por defecto.

## Skills por defecto

| Trigger en `link.1` | Skill por defecto |
|---------------------|-------------------|
| `ALTERED` | Revisa si el cambio contradice el documento de decisión |
| `CHAIN_DIRTY` | Evalúa si el drift puede invalidar decisiones upstream |
| `DELETED` | Alerta que el fragmento gobernado fue eliminado |
| `UNANCHORED` | Informa que el fragmento gobernado ya no matchea su query AST |

## Inputs de una skill

Cada skill recibe en contexto:

| Input | Fuente |
|-------|--------|
| Contenido de `link.0` | Documento de decisión de la layer actual |
| Estado y diff de `link.1` | Bilink estructural gobernado |
| `name.0`, `name.1` | Contexto semántico del elemento de impacto |
| Commits intercedidos | `git log <commit.N>..HEAD -- <archivo>` |
| Blast radius | Lista de nodos afectados por propagación |

## Prompt templates

Los prompts son archivos Markdown que definen el contexto y las instrucciones
para el modelo. Reciben los inputs como variables interpoladas:

```markdown
# ADR Review

El siguiente cambio ocurrió en un fragmento gobernado por esta decisión de arquitectura:

**Decisión**: {{link0_content}}
**Relación**: {{name0}} → {{name1}}
**Estado del vínculo**: {{link1_state}}
**Commits intercedidos**: {{commits_since}}

Evaluá si el cambio es coherente con la decisión. Si hay riesgo, describí qué
invariante podría estar violándose y qué acción recomendás.
```

## Autonomía por layer

Cada layer puede tener su propio `.impact.toml` con skills adaptadas a su
naturaleza. Una layer de ADRs puede tener skills enfocadas en consistencia
arquitectural; una layer de impl puede tener skills enfocadas en correctitud
de tests.

Las skills de una layer corren con solo esa layer clonada localmente — no
requieren el árbol completo de capas.
