# Impact

Impact es la consultora autónoma de impacto de cambios del ecosistema Accreta. Cuando un vínculo entre capas detecta drift, impact encuentra todos los documentos de decisión que gobiernan ese vínculo, evalúa si el cambio es coherente con ellos, y abre hilos de discusión estructurados.

## Problema que resuelve

Cuando un archivo cambia — código, especificación o decisión técnica — ese cambio tiene un alcance que rara vez es visible de forma inmediata. Un método modificado puede contradecir un ADR. Una spec actualizada puede dejar código desactualizado. Sin una herramienta que conecte esos puntos, el drift entre capas se acumula silenciosamente.

Impact hace ese trabajo: parte de los eventos de drift detectados por bilinker, los cruza con los elementos de impacto declarados en el grafo, y produce evaluaciones semánticas accionables.

## Elementos de impacto

Un **elemento de impacto** es un bilink con `kind: impact` que declara que un documento de decisión gobierna un vínculo estructural entre capas:

```
# .bilink/<uuid>.bilink
link.0: docs/adr/design-voting-machine.md
link.1: .bilink/7f3d8e9a-...bilink

kind:   impact
name.0: architecture-decision
name.1: spec-impl-bridge
```

Esto crea una relación ternaria: el documento sabe qué vínculo gobierna. Cuando ese vínculo detecta drift, impact encuentra el documento automáticamente via el índice de backlinks.

Ver [concepts/impact-element.md](concepts/impact-element.md).

## Skills

Las **skills** son rutinas de evaluación configurables que impact ejecuta al detectar drift en un vínculo gobernado. Reciben el documento de decisión, el bilink afectado y el historial git, y producen una evaluación de riesgo.

Son configurables por layer. Sin configuración, se aplican defaults razonables.

Ver [concepts/skills.md](concepts/skills.md).

## Posición en el ecosistema

```
git          ← fuente de verdad del historial
bilinker     ← detecta drift entre capas linkedeadas
impact       ← evalúa coherencia con decisiones declaradas
accreta      ← gobierna la resolución del cambio
```

## Principios de diseño

- **Declarativo**: los elementos de impacto declaran la gobernanza upfront, no post-hoc
- **Basado en hechos**: toda evaluación parte de diffs, commits y hashes concretos
- **No invasivo**: no modifica código ni specs, solo lee y reporta
- **Componible**: invocable manualmente, por hooks de git, o por bilinker watch
- **Autónomo por layer**: cada layer tiene sus propios elementos y skills sin necesitar
  el árbol completo
