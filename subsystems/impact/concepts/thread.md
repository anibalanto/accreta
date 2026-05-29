# Thread

Un thread es un hilo de discusión estructurado asociado a uno o más Impact Reports.
Es el espacio donde humanos y agentes evalúan si un cambio es coherente con las
capas linkedeadas y deciden cómo resolverlo.

## Estructura en el filesystem

```
.impact/threads/<uuid>/
  thread.md          ← metadata y estado del thread
  messages/
    0001.md          ← mensaje inicial (el Impact Report)
    0002.md          ← respuesta (humano o agente)
    0003.md
    ...
```

### `thread.md`

```yaml
---
id: <uuid>
title: <string>
status: open | resolved | discarded
created_at: <iso8601-utc>
resolved_at: <iso8601-utc>   # solo si status != open
reports:
  - <impact-report-uuid>
chains:
  - <chain-uuid>
---
```

### Mensajes

Cada mensaje es un archivo Markdown con frontmatter:

```yaml
---
seq: <número>
author: <did | "human">
kind: report | analysis | reply | resolution
created_at: <iso8601-utc>
---

<cuerpo en Markdown>
```

- `seq` define el orden. Los archivos se nombran con `seq` zero-padded a 4 dígitos.
- `kind: report` es siempre el mensaje `0001.md` — el Impact Report que abrió el thread.
- `kind: analysis` es una opinión generada por un agente.
- `kind: reply` es una respuesta humana o de agente al hilo.
- `kind: resolution` cierra el thread con una decisión explícita.

## Ciclo de vida

```
open → resolved   (se tomó una decisión)
     → discarded  (falsa alarma o irrelevante)
```

Un thread `resolved` tiene siempre un mensaje de `kind: resolution` como último mensaje.

## Relación con Impact Reports

Un thread puede agrupar múltiples Impact Reports si describen el mismo evento
o cambios relacionados en la misma cadena. El primer mensaje del thread es
siempre el reporte que lo abrió.

## Relación con Accreta

Un thread de impact puede convertirse en una `Discussion` de Accreta, arrastrando
su historial de mensajes. La resolución del thread puede generar una `Iteration`
propuesta sobre la spec o el ADR afectado.

## Invariantes

- `id` del thread coincide con el nombre del directorio.
- `0001.md` siempre existe y tiene `kind: report`.
- Los números de secuencia son contiguos y sin gaps.
- Un thread no puede pasar de `resolved` o `discarded` de vuelta a `open`.
