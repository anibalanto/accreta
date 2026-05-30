# Jerarquía

La jerarquía de worklist es flexible. Cualquier tipo puede existir en el nivel raíz y los niveles intermedios son opcionales.

## Niveles permitidos

| Item | Puede vivir en | Puede contener |
|------|----------------|----------------|
| Epic | raíz | user-stories, tasks |
| User Story | raíz, carpeta de epic | tasks |
| Task | raíz, carpeta de epic, carpeta de user-story | nada |

## Estructura de ejemplo

```
accreta/.stratum/worklist/
  1.epic                        ← epic en raíz
  1/
    2.user-story
    2/
      3.task
      4.task
    5.task                      ← task directo bajo epic
  6.user-story                  ← story en raíz, sin epic
  6/
    7.task
  8.task                        ← task en raíz, sin padre
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
