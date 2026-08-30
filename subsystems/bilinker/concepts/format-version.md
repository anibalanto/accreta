# Versión del formato

El formato de los archivos de bilinker —`.bilink` y `.capture`— es un crate, `bilink-format`, y **la versión del crate es la versión del formato**.

No se puede cambiar el parseo sin releasear, ni releasear sin bumpear. Con eso, el campo `.bilink/version` deja de ser una promesa y pasa a ser una propiedad del artefacto.

## Se verifica, no se declara

Declarar la versión a mano no cubre el modo de falla que importa: **un cambio aditivo que nadie bumpea.** Agregar un tipo de endpoint no rompe ningún archivo existente, no deja entrada en el ledger de migraciones, y un parser viejo lo leería mal sin fallar — leería `abstract` como un path de capa y seguiría.

Por eso hay un test:

```
sha256(esquema generado)  ==  <hash registrado para la versión N>
```

Cambiar los tipos sin subir la versión **falla el test**. La versión ordinal sirve para comparar y para leer; el hash garantiza que corresponde a lo que dice ser. Es el mismo principio que aplica a los captures: direccionar por contenido para que la identidad no dependa de que alguien se acuerde.

### El registro es de sólo-agregar

Una entrada registra lo que se publicó bajo esa versión. Corregirla en vez de agregar una nueva reescribiría el pasado, y el hash dejaría de certificar el artefacto que alguien ya descargó.

#### Sacar un campo sube el major

`3.0.0` saca el `offset` del capture: un fragmento es un nodo entero. Quitar un campo no es aditivo —un lector de `3.0.0` no entiende un archivo que lo lleve— así que sube el major, y con él se va `DISPLACED`, el único estado que hablaba de un sub-rango.

**Los ids no cambiaron.** El id termina cada campo con un `\0` en vez de unirlos con separadores, así que el campo que desaparece contribuía la cadena vacía y su terminador sigue estando. Es una propiedad del formato del id, no una casualidad: vale para cualquier campo que se saque en el futuro.

#### Un formato que ya no se escribe todavía se lee

`bilink-format-v1` está congelado en el sentido que importa: nadie escribe formato 1 nunca más. Eso no lo deja quieto — **leerlo mejor también cambia el esquema**.

Pasó: el lector no modelaba `kind` ni `name.N`, que estaban en el formato desde siempre. La migración decía preservarlos y no podía, porque nadie se los pasaba. Agregarlos al tipo movió el hash, el guard lo detectó, y el registro ganó `1.1.0` con `1.0.0` intacto.

La regla no distingue entre "el formato cambió" y "lo entendemos mejor": lo que registra es qué esquema se publicó bajo qué número, y los dos casos publican uno distinto. Un comentario que dijera "esta versión nunca va a tener otra entrada" sería una predicción, no una invariante — y las predicciones se corrigen.

### El esquema lleva su versión adentro

El documento publicado incluye el número de versión, así que el hash certifica las dos cosas a la vez: qué tipos describe y bajo qué nombre se publicó. Un esquema no puede circular diciendo ser una versión que no es.

## Qué tiene que aparecer en el esquema

Un esquema que describa de menos no sirve como guarda. El caso concreto: si el tipo de endpoint se publicara como `{"type": "string"}` a secas, **agregar un tipo de endpoint no movería el hash** — justo el cambio aditivo que motiva todo esto.

Por eso los prefijos reconocidos se publican, y salen de la misma tabla que usa el parser. Agregar un tipo obliga a tocar esa tabla, eso cambia el esquema, y el guard lo detecta.

> **Regla general:** lo que discrimina al parsear tiene que ser visible en el esquema. Si el parser distingue por algo que el esquema no menciona, ese algo puede cambiar sin que nada se entere.

## Qué describe el esquema, hoy

**El modelo, todavía no el archivo.** Los `.bilink` y `.capture` de hoy son texto plano `clave: valor` con líneas de continuación, escrito y leído por un parser a mano. Un esquema JSON no puede describir eso.

Lo que el esquema describe es la forma de los datos: qué campos hay, de qué tipo, cuáles son opcionales. Cuando los archivos pasen a YAML serializado con serde —[ADR-0003](../.stratum/impl/docs/adr/0003-formato-captures-y-aceptacion.md)— el esquema pasa a describir el archivo literal, sin que haya que reescribirlo: los tipos son los mismos.

Hasta entonces el guard ya sirve para lo suyo, que es que ningún cambio de formato pase sin versión. Lo que todavía no se puede es que un tercero valide un archivo ajeno contra el esquema.

## Para qué se publica

`schemars` genera el esquema desde los tipos, y se publica como artefacto de la release:

```sh
cargo run -q -p bilink-format --bin schema > bilink-format-<version>.json
```

Ahí está la razón más fuerte de todo esto. Un consumidor de la frontera —[ADR-0005](../.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md)— lee los `.bilink` de otro proyecto. Con el esquema publicado los valida antes de interpretarlos, **sin adoptar bilinker**, con cualquier validador de JSON Schema en cualquier lenguaje. Eso baja el costo de adopción del lado del proveedor, que es lo que hay que minimizar.

## La dirección es Rust → esquema

Los tipos con serde son la fuente y el esquema sale generado. La alternativa —esquema a mano, tipos generados— sacaría la definición del formato de Rust, que es su virtud, pero agrega codegen al build para un beneficio que hoy nadie cobra: ningún otro proyecto implementa su propio lector. Si aparece uno, es el momento de darlo vuelta.

## Invariantes

1. La versión del formato es la versión del crate `bilink-format`. No hay otra copia.
2. El hash registrado para una versión es el sha256 del esquema publicado de esa versión, texto exacto.
3. El registro de hashes es de sólo-agregar.
4. El esquema publica todo lo que el parser usa para discriminar.
5. El crate de formato no resuelve queries, no consulta git y no calcula estados.
