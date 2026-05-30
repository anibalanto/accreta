# Integración con bilinker

Bilinker provee la infraestructura de linking que certifica la alineación entre capas dentro de Accreta.

## Cadenas como invariantes del sistema

Cada cadena de bilinks representa un invariante: *"este fragmento de spec está implementado por este fragmento de código"*. Una cadena en estado OK global es evidencia técnica de que el invariante se mantiene.

## Drift como disparador de Iterations

Cuando `bilinker check` detecta ALTERED o CHAIN_DIRTY, ese evento puede disparar automáticamente una Iteration propuesta en Accreta sobre la spec afectada. El agente de análisis calcula el blast radius y propone el cambio.

## Aceptación con gobernanza

`bilinker accept` en el contexto de Accreta puede requerir que la aceptación pase por el proceso de consenso antes de ser válida — dependiendo del nivel de gobernanza configurado para el proyecto.
