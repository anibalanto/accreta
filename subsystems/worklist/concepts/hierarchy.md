# Jerarquía

La jerarquía de worklist es flexible. Cualquier tipo puede estar en la raíz del árbol y los niveles intermedios son opcionales.

**La relación padre-hijo se declara con el campo `parent`**, no con la ubicación del archivo. Todos los ítems viven juntos en `worklist/`; lo que los ordena es el campo. Ver [ítem](item.md) § "Jerarquía" para el porqué.

## Niveles permitidos

| Item | Puede tener de padre | Puede tener de hijo |
|------|----------------------|---------------------|
| Epic | nada | user-stories, tasks |
| User Story | nada, epic | tasks |
| Task | nada, epic, user-story | nada |
| Sprint | nada — vive en `_sprints/` | nada; **referencia** user stories y tasks |

## Estructura de ejemplo

```
accreta/.stratum/worklist/
  1.epic.md                        ← sin parent: raíz del árbol
  2.user-story.md                  ← parent: 1
  3.task.md                        ← parent: 2
  4.task.md                        ← parent: 2
  5.task.md                        ← parent: 1   · task directa bajo la épica
  6.user-story.md                  ← sin parent: story suelta
  7.task.md                        ← parent: 6
  8.task.md                        ← sin parent: task suelta
  _sprints/
    1.sprint.md
```

El árbol no se ve en el `ls`: se deriva leyendo los `parent`. Es el precio de que la dirección de un ítem sea componible y estable, y se paga una vez —lo rinde un comando que lo dibuje— mientras que buscar un ítem por id se paga en cada uso.

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

Los IDs no codifican la jerarquía — un ítem hijo puede tener un ID posterior o anterior al de su padre. La estructura es el campo `parent`, no el número.

## Sprints

Un sprint no contiene: referencia. Y **vive en `_sprints/`, fuera del árbol de descomposición**, porque no participa de él: puede llevarse ítems de épicas distintas, y tiene su propio contador.

### Los directorios llevan `_`

> **Todo directorio dentro de `worklist/` empieza con `_`.**

Los ítems son archivos sueltos en la raíz, así que un directorio nunca es un ítem — es un espacio de nombres para otra cosa, como `_sprints/`. El `_` lo dice desde el nombre: los ids son base-36, `[0-9a-z]`, y un nombre que empieza con `_` **nunca puede ser uno**.

Sin la regla, `sprints/` sería un nombre libre que el contador alcanzaría en el ítem 62.507.780.128 — nunca, pero "nunca" por improbable y no por imposible, y entonces distinguir un directorio de un ítem pasaría a depender de que ese número no llegue.

```
1.epic.md                     ← épica 1
n.user-story.md               ← parent: 1
8.task.md                     ← parent: n
o.task.md                     ← parent: n
_sprints/
  1.sprint.md                 ← sprint 1; referencia a ../n.user-story.md
```

### La regla del ancestro

> **Un ítem entra a un sprint sólo si ninguno de sus ancestros entra.**

Si va una user story, sus tasks van con ella y no se enumeran. Una task se nombra sola cuando no tiene user story arriba — o cuando cuelga directamente de una épica.

La cadena de ancestros se lee siguiendo `parent` hasta que se acaba.

Dos consecuencias, y las dos son deliberadas:

**Una user story no puede atravesar sprints.** Si entra, entra entera. Con lo cual una que no cabe en una iteración deja de ser algo que se parte en el planning y pasa a ser una user story mal dimensionada — el problema se vuelve visible en vez de esconderse en un compromiso parcial.

**Y si una task se quiere sola pero cuelga de una user story**, la pregunta no es cómo sacarla al sprint sino si está bien colgada de esa user story. La regla lleva la discusión a la descomposición, que es donde va.

### El backlog no es un archivo

Un ítem que no está referenciado por ningún sprint está en el backlog **por definición**, y lo lista un comando. Mantenerlo como archivo obligaría a editar dos lugares para mover algo, y los dos podrían divergir.

El cálculo va sobre lo efectivamente referenciado: si las tasks de una user story están en sprints pero la user story no se nombra, la user story no está en ningún sprint. Está bien — el trabajo son las tasks, y la user story es agrupación.

### Épicas

Una épica no entra a un sprint. No lo prohíbe el formato, pero no tiene sentido: si cabe en una iteración, no era una épica.
