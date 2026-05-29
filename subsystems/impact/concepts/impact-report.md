# Impact Report

Un Impact Report es el artefacto central que produce impact. Describe de forma
estructurada qué cambió, en qué archivos, qué cadenas de bilinks se vieron
afectadas, y qué commits intercedieron desde el último estado conocido.

## Estructura

```
<uuid>.impact
```

Archivo de texto con frontmatter YAML y cuerpo Markdown.

```yaml
---
id: <uuid>
generated_at: <iso8601-utc>
trigger:
  kind: bilinker_event | git_hook | manual
  file: <path-relativo>
  commit: <sha1>        # presente si kind == git_hook o manual con commit
chains:
  - uuid: <chain-uuid>
    node: <layer-path>  # el nodo de la cadena que referencia el archivo
    endpoint: 0 | 1
    state: ALTERED | DELETED | ...
    file: <path>
    commit_anchor: <sha1>  # commit de la última entrada en accepted.N del bilink
    commits_since:
      - sha: <sha1>
        author: <name>
        date: <iso8601>
        message: <primera línea>
---

## Cambio

<descripción del diff o resumen del cambio>

## Cadenas afectadas

<lista de cadenas con su estado y los commits que intercedieron>
```

## Generación

Un Impact Report se genera cuando:

1. bilinker emite un evento `ALTERED` o `DELETED` para un archivo
2. Un git hook detecta un commit que modifica archivos linkeados
3. El usuario invoca `impact report <file>` manualmente

La generación no modifica ningún archivo fuera de `.impact/reports/`.

## Ciclo de vida

```
generado → abierto en thread → resuelto | descartado
```

Un reporte sin thread asociado es información pasiva. Al abrirse un thread,
el reporte se convierte en el mensaje inicial de ese thread y queda disponible
para discusión en Accreta.

## Invariantes

- `id` es UUID v4, coincide con el nombre del archivo.
- `commit_anchor` es el commit de la última entrada en `accepted.N` del bilink al momento de la detección.
- `commits_since` son exactamente los commits en `git log <commit_anchor>..HEAD -- <file>`.
- Si `commits_since` está vacío, el cambio ocurrió pero no hay commits posteriores
  al anchor (archivo modificado sin commitear).
- Un reporte no se modifica después de ser generado. Si el estado vuelve a cambiar,
  se genera un nuevo reporte.
