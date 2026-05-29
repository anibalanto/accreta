# Integración con Accreta

Bilinker es la infraestructura de linking de Accreta. Los bilinks son los artefactos
que certifican la alineación entre capas — una cadena en estado OK global es evidencia
de que spec, decisiones técnicas e implementación están en sincronía.

## Gobernanza del estado

La aceptación de un estado (`bilinker accept`) es un acto de gobernanza: el
desarrollador certifica explícitamente que el cambio en el fragmento es coherente
con la intención del bilink. Accreta puede requerir que ciertas aceptaciones pasen
por un proceso de votación o revisión antes de ser válidas.

## Cadenas como invariantes del sistema

En la visión completa de Accreta, cada cadena representa un invariante del sistema:
*"este fragmento de spec está implementado por este fragmento de código"*. El estado
OK de la cadena es la evidencia técnica de que el invariante se mantiene.
