# Impact

Impact es la herramienta de análisis de cambios del ecosistema Accreta. Su responsabilidad es detectar qué modificaciones ocurrieron en el código o las especificaciones, calcular qué partes del sistema están afectadas, y abrir hilos de discusión estructurados para que humanos y agentes evalúen si el cambio es coherente con las capas linkedeadas.

## Problema que resuelve

Cuando un archivo cambia — ya sea código, especificación o decisión técnica — ese cambio tiene un alcance que rara vez es visible de forma inmediata. Un método modificado puede romper un invariante documentado en una spec. Una spec actualizada puede dejar código desactualizado. Sin una herramienta que conecte esos puntos, el drift entre capas se acumula silenciosamente.

Impact hace ese trabajo: toma los eventos de drift detectados por bilinker, los enriquece con el historial de git, y genera reportes accionables que alimentan el proceso de gobernanza de Accreta.

## Posición en el ecosistema

```
git          ← fuente de verdad del historial
bilinker     ← detecta drift entre capas linkedeadas
impact       ← analiza el alcance, abre hilos de discusión
accreta       ← gobierna la resolución del cambio
```

Impact no toma decisiones. Recopila, analiza y presenta. La decisión de aprobar o rechazar un cambio pertenece a los actores humanos en Accreta.

## Principios de diseño

- **Basado en hechos**: todo reporte parte de datos concretos — diffs, commits, hashes
- **No invasivo**: no modifica código ni specs, solo lee y reporta
- **Componible**: puede ser invocado manualmente, por hooks de git, o por bilinker watch
- **Agnóstico al lenguaje**: analiza cualquier archivo linkedeado, independientemente del lenguaje
