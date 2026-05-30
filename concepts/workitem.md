# WorkItem

Todo en Accreta es un WorkItem — un objeto firmado, content-addressed y respaldado por CRDT. El feed de un proyecto es el grafo causal ordenado de todos sus WorkItems.

```
WorkItem (base)
  id:         UUID
  type:       Spec | Iteration | Discussion | Opinion | Vote | Task | Attestation
  author:     Actor.did
  created_at: Timestamp (Solana PoH anchor)
  signature:  Ed25519(content_hash, author.private_key)
  content:    CRDT document (Loro)
```

## Spec

Documento Markdown vivo. Las specs nunca se reemplazan — solo se extienden por Iterations aceptadas.

## Iteration

Cambio propuesto sobre una Spec. Incluye análisis automático de agentes al crearse, discusión, y votos humanos.

```
status: Proposed | InDiscussion | ConsensusRequested | Accepted | Rejected | Withdrawn
```

## Opinion

Análisis o argumento de un actor (humano o agente) sobre un WorkItem.

- **Analysis**: contexto estático, turno único. `context_refs` describe exactamente
  qué documentos tuvo el agente.
- **Research**: multi-turno, puede incluir tool calls. El log completo de conversación
  se registra como WorkItem — es el artefacto auditable.

## Vote

Solo actores humanos. Cada voto está firmado criptográficamente.

```
verdict: Approve | Reject | Abstain
```

## Task

Unidad de trabajo generada a partir de una Iteration aceptada. Corresponde a un ítem de worklist linkedeado a la spec que lo originó.
