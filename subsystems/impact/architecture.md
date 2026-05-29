# Arquitectura

## Flujo general

```mermaid
flowchart TD
    E1[bilinker watch] --> EC[Event Collector]
    E2[git hook] --> EC
    E3[invocación manual] --> EC

    EC --> CR[Chain Resolver]
    EC --> GA[Git Analyzer]

    CR -->|chains afectados| RB[Report Builder]
    GA -->|diff + commits| RB

    RB --> IR[Impact Report]
    IR --> TM[Thread Manager]
    TM -->|WorkItem| AC[Accreta]
```

## Componentes internos

### Event Collector
Recibe eventos de múltiples fuentes y los normaliza en un formato uniforme:
`{ file, kind: Modified|Created|Deleted, commit? }`.

### Chain Resolver
Dado un archivo, consulta los `.bilink` files del proyecto para encontrar todos los chains que lo referencian. Devuelve la lista de chains con sus endpoints y el estado almacenado.

### Git Analyzer
Lee el historial de git para obtener los commits que modificaron el archivo desde el último estado conocido. El ancla es el commit de la última entrada en `accepted.N` del bilink — todo lo posterior son los commits que intercedieron en el cambio.

```mermaid
flowchart LR
    A[accepted.N\núltima entrada] --> C[git log commit..HEAD\n-- file]
    C --> R[commits intercedidos\n→ Impact Report]
```

### Report Builder
Combina la salida del Chain Resolver y el Git Analyzer para construir un Impact Report estructurado.

### Thread Manager
Gestiona los hilos de discusión. Cada hilo vive en su propia carpeta bajo `.impact/threads/`.

## Posición en el ecosistema

```mermaid
graph LR
    GIT[git] -->|historial| IM[impact]
    BL[bilinker] -->|eventos ALTERED| IM
    IM -->|Impact Reports| AC[accreta]
    AC -->|Iterations + Votes| GIT
```

## Persistencia

```
proyecto/
  .impact/
    reports/
      <uuid>.impact
    threads/
      <uuid>/
        thread.md            ← metadata: estado, título, chains afectados
        messages/
          0001.md            ← mensaje inicial (el Impact Report)
          0002.md            ← respuesta humana o de agente
          0003.md
```

Todos los archivos son texto plano con frontmatter YAML y cuerpo Markdown — legibles y diffables en git, sin base de datos.
