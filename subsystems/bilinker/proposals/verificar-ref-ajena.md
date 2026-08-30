# Propuesta: verificar una ref ajena, y reparar la disyunción rota

**Estado:** especificado, no implementado. Vive acá hasta que alguien lo implemente.

Diferido por [ADR-0004](../.stratum/impl/docs/adr/0004-bilinks-en-ref-paralela.md), que decide **dónde viven los bilinks** y deja fuera de esa decisión la superficie de operación que se resuelve mejor con la ref ya andando.

## `verify-ref` — la ref que construyó otro

Las [dos verificaciones previas](../concepts/ref.md#las-dos-verificaciones-previas) protegen lo que bilinker escribe. Una ref traída de un proveedor la construyó otra herramienta, y el consumidor no tiene con qué contrastarla: si su árbol de código no es el que su segundo padre dice, calcula drift contra un árbol fabricado sin forma de enterarse.

Un `verify-ref` recorre la ref chequeando los dos enunciados de la [invariante de fidelidad](../concepts/ref.md#la-invariante-de-fidelidad) — comparaciones de tree oids, sin tree-sitter ni rehashear nada.

**No debe chequear que los bilinks estén en `OK`**, por el mismo motivo por el que la invariante no lo dice: una ref con drift es normal, y exigirlo haría imposible [`track`](../commands/track.md).

## Bilinks sin procedencia

Un `.bilink/` en el árbol que ninguna ref avala, o commiteado en una rama del proyecto, son el mismo problema: **decisiones sin un commit firmado detrás**.

La salida no es descartarlas. Sacarlo con un commit sirve para el caso en que llegaron de un merge de la ref al proyecto, pero destruye trabajo si alguien commiteó bilinks a mano. La salida es **adoptarlas**, con el commit de adopción atribuyéndolas a quien adopta, que es lo único cierto que se puede decir de ellas.

Sin base de merge, toda diferencia es conflicto: ése es el costo de haber perdido la procedencia.

Es también lo que le falta a la verificación de disyunción para poder **reparar** y no sólo abortar. Hoy [`sync`](../commands/sync.md) detecta que el commit del proyecto trae `.bilink/` en su árbol y se para; cómo volver a sacarlo de la rama, y qué hacer con lo que había ahí, es este mismo problema.
