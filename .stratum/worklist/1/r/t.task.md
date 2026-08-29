---
title: Verificar la versión de formato contra el hash del esquema
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
---

# Verificar la versión de formato contra el hash del esquema

`sha256(esquema generado) == <hash registrado para la versión N>`. Cambiar los tipos sin subir la versión falla.

Es lo que convierte `.bilink/version` de una promesa en una propiedad del artefacto. El caso que motiva: [ADR-0005](../../../subsystems/bilinker/.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md) agrega endpoints **sin migración** porque son aditivos, y un parser viejo leería `abstract` como un path de capa sin fallar — exactamente el cambio que uno se olvida de bumpear.
