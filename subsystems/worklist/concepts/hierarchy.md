# Jerarquía

La jerarquía de worklist es flexible. Cualquier tipo puede existir en el nivel raíz y los niveles intermedios son opcionales.

## Niveles permitidos

| Item | Puede vivir en | Puede contener |
|------|----------------|----------------|
| Epic | raíz | user-stories, tasks |
| User Story | raíz, carpeta de epic | tasks |
| Task | raíz, carpeta de epic, carpeta de user-story | nada |
| Sprint | `_sprints/` | nada en carpetas — **referencia** user stories y tasks |

## Estructura de ejemplo

```
accreta/.stratum/worklist/
  1.epic.md                        ← epic en raíz
  1/
    2.user-story.md
    2/
      3.task.md
      4.task.md
    5.task.md                      ← task directo bajo epic
  6.user-story.md                  ← story en raíz, sin epic
  6/
    7.task.md
  8.task.md                        ← task en raíz, sin padre
```

## Referencia por ID

Los IDs son cortos y directos — se usan completos:

```bash
$ worklist show 1
1  [open]  Implementar hash.N en Rust  .epic

$ worklist show 3
3  [open]  Actualizar struct BiLinkFile  .task
```

Si por alguna razón el ID es ambiguo (no debería ocurrir, pero si el worklist es compartido entre proyectos), el CLI reporta los candidatos:

```bash
$ worklist show 1
Ambiguous ID '1' — matches:
  1   Implementar hash.N en Rust       .epic
  10  Actualizar parser de BiLinkFile  .user-story
Use the full ID.
```

## IDs secuenciales y jerarquía

Los IDs no codifican la jerarquía — un ítem hijo puede tener un ID posterior o anterior al de su padre. La estructura es la carpeta, no el número.

## Sprints

Un sprint no contiene: referencia. Y **vive en `_sprints/`, fuera del árbol de descomposición**, porque no participa de él: puede llevarse ítems de épicas distintas, y los que nombra siguen viviendo donde estaban.

Eso mantiene limpia la lectura de las carpetas. Una carpeta con nombre de id —`1/`— es contención: lo que hay adentro pertenece al ítem `1`. `_sprints/` no es eso: es un espacio de nombres para el otro eje.

### Los directorios que no son ids llevan `_`

> **Un directorio cuyo nombre no es un id de ítem empieza con `_`.**

Los ids son base-36, `[0-9a-z]`, así que un nombre con `_` adelante **nunca puede ser uno**. Sin la regla, `sprints/` es un nombre libre que el contador alcanzaría en el ítem 62.507.780.128 — nunca, pero "nunca" por improbable y no por imposible, y entonces la lectura de una carpeta pasaría a depender de que ese número no llegue.

Vale para cualquier directorio que se agregue después con otro propósito.

```
1.epic.md                     ← épica 1
1/
  n.user-story.md
  n/
    8.task.md
    o.task.md
_sprints/
  1.sprint.md                 ← sprint 1; referencia a ../1/n.user-story.md
```

### La regla del ancestro

> **Un ítem entra a un sprint sólo si ninguno de sus ancestros entra.**

Si va una user story, sus tasks van por contención y no se enumeran. Una task se nombra sola cuando no tiene user story arriba — o cuando cuelga directamente de una épica.

Dos consecuencias, y las dos son deliberadas:

**Una user story no puede atravesar sprints.** Si entra, entra entera. Con lo cual una que no cabe en una iteración deja de ser algo que se parte en el planning y pasa a ser una user story mal dimensionada — el problema se vuelve visible en vez de esconderse en un compromiso parcial.

**Y si una task se quiere sola pero cuelga de una user story**, la pregunta no es cómo sacarla al sprint sino si está bien colgada de esa user story. La regla lleva la discusión a la descomposición, que es donde va.

### El backlog no es un archivo

Un ítem que no está referenciado por ningún sprint está en el backlog **por definición**, y lo lista un comando. Mantenerlo como archivo obligaría a editar dos lugares para mover algo, y los dos podrían divergir.

El cálculo va sobre lo efectivamente referenciado: si las tasks de una user story están en sprints pero la user story no se nombra, la user story no está en ningún sprint. Está bien — el trabajo son las tasks, y la user story es agrupación.

### Épicas

Una épica no entra a un sprint. No lo prohíbe el formato, pero no tiene sentido: si cabe en una iteración, no era una épica.
