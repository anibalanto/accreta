---
title: Especificar e implementar el endpoint `abstract` y el endpoint repo
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-31T00:00:05Z
parent: h
---

# Especificar e implementar el endpoint `abstract` y el endpoint repo

Spec primero: `concepts/configuration.md` (deja de ser cierto que no hay ningún archivo de configuración), `concepts/bilink.md` y `reference.md` (los dos tipos nuevos), los comandos `get`, `graph` y `chain`, y `scenarios/frontier.yaml`. En Stratum, `sublayer-config.md`.

Después el código, por los bilinks que se rompan. No requiere ningún repo ajeno: se ejercita entre dos repos locales.

**La verificación de versión en la frontera.** El consumidor lee el `.bilink/version` que viene adentro de la ref del proveedor y se niega si no lo entiende, en vez de malinterpretar. Es la razón de fondo del campo: son repos con ciclos de release independientes, así que la divergencia de versiones no es un accidente sino lo normal. Y arrastra algo contraintuitivo que hay que dejar escrito: **un cambio aditivo también sube la versión** — `link.1: abstract` no rompe al que escribe, pero un parser viejo lo lee como un path de capa, en silencio.

## Lo que quedó hecho

**Specs**: `concepts/frontier.md` es donde vive lo que los dos tipos comparten; `reference.md` los describe; `bilink.md` gana los cinco estados y pierde `UNREACHABLE`; `configuration.md` deja caer la frase absoluta; `check`, `get`, `graph` y `chain` ajustados; `scenarios/frontier.yaml` con los 16 del ADR más 2 de la verificación de versión. En Stratum, `sublayer-config.md`.

**Código**: `LinkEndpoint::Repo` y `::Abstract`, los cinco estados, `frontier.rs` —resolución por alias, sparse derivado, profundidad a pedido, verificación de versión—, `bilinker fetch`, y `chain new --from-repo`.

**Verificado entre dos repos locales**, que es lo que la US pide: el proveedor publica y ve su propio drift, el consumidor detecta `CHAIN_DIRTY` tras el fetch, `REJECTED` cuando la punta deja de ser abstracta, `BROKEN` cuando el bilink remoto desaparece, y `REMOTE_UNREACHABLE` sin que `check` haga red. Dos consumidores sobre **un** archivo del proveedor, que no cambia.

## Lo que se descubrió escribiendo el código

**Una capa creada hoy no declaraba su versión de formato.** `chain new` escribía el `.gitignore` de `.bilink/` y no el `version`, así que del otro lado de la frontera una capa recién nacida era indistinguible de una anterior a que el campo existiera — y el consumidor se negaba a leerla, correctamente. Ahora lo escribe quien crea el directorio, en la misma operación: es la misma regla que ya gobernaba el `.gitignore`, y que se olvidó por la misma razón.

**El `+` del refspec no es opcional con un clon superficial.** Sin historia, git no puede probar que el tip nuevo desciende del viejo, así que rechaza como non-fast-forward un avance legítimo. Es seguro ponerlo porque la ref es append-only por diseño.

**`accepted.link` sí lleva el prefijo cruzando la frontera.** Lo escribí al revés en `reference.md`: es una copia opaca de un id ajeno, pero eso ya valía para un endpoint `path`, y la forma es la misma en los dos casos.

**El guard del esquema disparó, que es exactamente lo que tenía que pasar.** Agregar dos tipos de endpoint es aditivo y sube la versión igual — `3.1.0` — porque un parser de `3.0.0` leería `abstract` como un path de capa, en silencio. Y el hash cambió **dos veces**: la segunda al bumpear, porque el esquema lleva su versión adentro y certifica las dos cosas a la vez.

**Y el clon del proveedor no se excluye con un `.gitignore`.** Lo había escrito así —`fetch` agregaba `<alias>/` a `.bilink/.gitignore`— y es la palanca equivocada dos veces: es una escritura versionada para resolver algo que es del índice, y una regla por proveedor donde ya hay una que los cubre a todos. Del lado del proyecto lo cubre el patrón `.bilink/` que `init` puso en `info/exclude`; del lado de la ref, la enumeración que construye el árbol.

Siguiendo eso apareció que **nada en el código impedía que el clon entrara al commit de la ref**: `collect_tracked` recorría `.bilink/` entero. No se filtró en la prueba, pero por cómo git trata un repo anidado, no por diseño — y depender de eso es depender de que el clon esté sano. Ahora la frontera del repo está dicha también ahí, que es la misma regla que frena el recorrido de capas.
