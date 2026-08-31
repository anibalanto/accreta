# Propuesta: discutir sobre un commit de la ref

**Estado:** en discusión. Sin diseño. Vive acá para que la pregunta no se pierda.

Una aceptación es un commit firmado, inmutable y con autor. **¿Dónde se discute?** *"Che, ¿por qué aprobaste esto?"* hoy no tiene lugar en el ecosistema.

## Por qué el commit es un buen ancla

Mejor que cualquier otra cosa del sistema, y vale enunciarlo porque es lo que hace viable la pregunta:

| Ancla | Problema |
|---|---|
| un fragmento | se mueve, y con él el capture |
| un bilink por UUID | su contenido cambia con cada aceptación |
| **un commit de la ref** | **inmutable por [invariante 6](../concepts/ref.md#invariantes), y firmado** |

Una discusión colgada de un commit apunta a una decisión que **no puede cambiar debajo suyo**. Y con [el mensaje como comando](../concepts/ref.md#el-mensaje-es-el-comando), el commit ya dice qué se decidió sin abrir un archivo.

## La discusión no puede vivir en la ref

Es la única cosa que esta propuesta ya tiene resuelta, y sale de lo que la ref decidió: **un commit sobre la ref hace una cosa, y las dos que puede hacer son mover código o decidir.** Un comentario no es ninguna, así que agregarlo como commit de la ref rompe la regla; y editarlo reescribiría historia append-only.

Una discusión es mutable por naturaleza y la ref es inmutable por diseño. **Van en artefactos distintos, y el commit es el punto de contacto.**

## Tres formas, y una tiene dueño

### A — Los threads de impact, con un ancla más

Impact ya tiene esto: `.impact/threads/<uuid>/` con `thread.md` y `messages/NNNN.md`, `status: open | resolved | discarded`, autores humanos o agentes. Y su `thread.md` ya ancla a `reports:` y `chains:`.

Lo que falta es una lista más —`commits:`— y nada de git nuevo. **Es la opción más barata y la que respeta los dueños**: discutir es trabajo de impact, y esta propuesta sólo agrega a qué se puede colgar un hilo.

Encaja además con el [ADR-0001 de impact](../../impact/.stratum/impl/docs/adr/0001-orquesta-no-escribe.md): impact orquesta y no escribe formatos ajenos. Un hilo que referencia un commit de la ref no toca `.bilink/` ni parsea nada de bilinker — sólo guarda un hash.

### B — `git notes`

`refs/notes/bilink`, anotaciones colgadas del commit. `ref.md` ya cita a las notes como precedente de vivir [fuera de `refs/heads/`](../concepts/ref.md#fuera-de-refsheads).

**A favor:** el ancla es el hash y no hay nada que mantener sincronizado, viaja con un fetch, y `git log --notes` la muestra al lado de la decisión. Cero artefactos nuevos.

**En contra:** las notes se reescriben en cada edición y **sus merges conflictúan** — es el dolor conocido del mecanismo. Con varias personas anotando concurrentemente hay que definir una estrategia de merge, y aparece la misma pregunta de refspec y protección que `refs/bilink/*`, otra vez.

### C — Un artefacto propio de bilinker

Descartable de entrada: sería inventar un sistema de discusión al lado de uno que ya existe. La frontera de bilinker es *sólo git y tree-sitter*, y un hilo de conversación no es ninguno de los dos.

## Preguntas abiertas

1. **¿Threads de impact o notes?** La primera reusa todo y no toca git; la segunda pone la discusión donde está la decisión, al costo de los merges de notes. La respuesta probable es A, y B como presentación —un `--notes` derivado y no versionado— si alguien quiere verla en el `git log`.
2. **¿Se puede discutir una aceptación *antes* de que exista?** Lo interesante sería objetar antes de aceptar, y para eso el ancla no puede ser el commit — todavía no hay. Ahí el ancla natural es el estado no-OK, que es lo que impact ya reporta. Son dos momentos y quizás dos anclas.
3. **¿Una discusión resuelta cambia algo del bilink?** Probablemente no, y conviene que no: el `accepted` lo escribe `accept` y nadie más. Lo que puede salir de un hilo es *otra* aceptación, con su propio commit.
4. **¿La discusión cruza la frontera entre proyectos?** Un consumidor que trae la ref de un proveedor recibe sus decisiones. ¿Y sus discusiones? Casi seguro no —son de otro equipo— pero hay que decirlo.

## Qué haría falta antes de decidir

Nada de infraestructura. Un caso real: la primera vez que alguien mire un `accepted` y quiera preguntar por qué. Hasta entonces esto es una pregunta bien planteada sin evidencia de urgencia, y está bien que espere.
