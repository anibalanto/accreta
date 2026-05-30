# Actor

Todo participante en Accreta es un Actor con identidad descentralizada (DID — W3C).

## Tipos

### Actor humano

```
HumanActor {
  did:        "did:near:alice.near"
  name:       string
  public_key: Ed25519PublicKey
}
```

### Actor agente

Un agente es un proceso autónomo (unikernel hermit-rs) operado por un humano. Hereda el nivel de confianza de su operador y sus acciones contribuyen al historial trazable del operador.

```
AgentActor {
  did:           "did:near:pm-agent.alice.near"
  name:          string
  public_key:    Ed25519PublicKey
  model_id:      "claude-sonnet-4-6"
  model_version: string
  capabilities:  ["spec-review", "impact-analysis", "task-generation"]
  operator:      HumanActor.did
}
```

Los DIDs de agentes son sub-cuentas de su operador en NEAR — la cadena de responsabilidad es visible en la identidad misma.

## Niveles de colaboración

```
   ▲    L1: Owner
  ▲ ▲   L2: Trusted Collaborators
 ▲ ▲ ▲  L3: Collaborators
▲ ▲ ▲ ▲ L4: Community
```

| Acción | L1 | L2 | L3 | L4 | Actor |
|--------|:--:|:--:|:--:|:--:|:-----:|
| Definir gobernanza del proyecto | ✓ | | | | humano |
| Veto final sobre consenso | ✓ | | | | humano |
| Promover actor a L2 | ✓ | | | | humano |
| Promover actor a L3 | ✓ | ✓ | | | humano |
| Votar en consenso | ✓ | ✓ | ✓ | | humano |
| Proponer Iteration | ✓ | ✓ | ✓ | | humano + agente |
| Analizar impacto | ✓ | ✓ | ✓ | ✓ | humano + agente |
| Opinar en Discussion | ✓ | ✓ | ✓ | ✓ | humano + agente |

Los agentes heredan el nivel de su operador. Las acciones de gobernanza son exclusivamente humanas.

## Historial trazable

No existe un número de reputación. Cada actor acumula un historial público, firmado y auditable:

```
iterations_proposed: 23  (18 aceptadas, 3 rechazadas, 2 retiradas)
votes_cast:          41
opinions_given:      89
agents_operated:      2
```

Los observadores forman su propio juicio. El historial no se puede resumir — es la reputación.
