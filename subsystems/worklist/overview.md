# Worklist

Worklist registra el trabajo concreto pendiente en el ecosistema Accreta. Cada
ítem representa algo que hay que hacer como consecuencia de un cambio en alguna
capa del sistema — ya sea implementar lo que la spec requiere, o actualizar la
spec porque la implementación evolucionó.

## Problema que resuelve

Cuando algo cambia — una spec, un ADR, el código mismo — ese cambio genera trabajo
en otras capas. Sin un lugar donde registrar ese trabajo linkedeado al origen, el
drift se acumula sin que nadie lo vea.

Worklist es ese lugar: un ítem por cada trabajo pendiente, linkedeado al fragmento
exacto que lo origina.

## Posición en el ecosistema

```
git          ← fuente de verdad del historial
bilinker     ← detecta drift entre capas linkedeadas
impact       ← analiza el alcance, abre hilos de discusión
worklist     ← registra el trabajo concreto pendiente
accreta       ← gobierna la resolución del cambio
```

## Dónde vive

Todos los ítems viven en una única carpeta dentro del proyecto principal,
en un repositorio git central:

```
accreta/
  .stratum/
    worklist/
      1.epic
      2.user-story
      3.task
```

No hay una instancia de worklist por repo ni por capa. Los bilinks resuelven
la conexión con los fragmentos en cualquier repo del ecosistema.

## Direccionalidad libre

Un ítem de worklist no tiene dirección fija. Puede representar trabajo en
cualquier dirección del stack:

```bash
# Parado en la spec: implementar algo
cd accreta/bilinker/specific
worklist new task "implementar bilinker accept" bilink-format.md:104:1

# Parado en impl: actualizar la spec en base al cambio
cd accreta/bilinker/.stratum/impl/src
worklist new task "actualizar spec hash.N" lib.rs:88:1
```

El fragmento que origina el ítem puede estar en cualquier capa — arriba, abajo,
o en la misma. El selector se resuelve desde el directorio actual en la terminal.
