# Tutorial: vincular dos proyectos con bilinker

Cómo un proyecto **publica** una parte de su código para que otro proyecto la referencie, y cómo el que la referencia se entera cuando cambia.

Todos los comandos y salidas de acá están verificados contra un caso real de dos repos. Al final, [el caso concreto](#el-caso-concreto-hsi--retinar).

---

## El modelo, en un minuto

Hay dos roles y son asimétricos:

| | Qué hace | Qué guarda del otro |
|---|---|---|
| **Proveedor** | publica un fragmento de su código | **nada** |
| **Consumidor** | lo referencia desde el suyo | dos hashes y un alias |

El proveedor **no se entera de quién lo consume**, y su costo no crece con la cantidad de consumidores: un fragmento publicado es un archivo, lo consuman uno o veinte.

Y lo que hace esto adoptable: **la rama del proveedor no cambia ni un byte.** Ni un commit, ni un archivo, ni una rama nueva en la lista. Publicar no es una negociación con el equipo, es una decisión de quien publica.

### Dónde viven los bilinks

En `refs/bilink/<rama>`, una ref de git paralela. **No es una rama**: no aparece en `git branch -a`, ni en el listado de la forja, ni en `git log`. Es lo mismo que hace `git notes`.

En el árbol de trabajo sí están, en `.bilink/`, para tenerlos a mano — pero excluidos del índice del proyecto, así que `git status` no los muestra.

---

## Antes de empezar

Un solo requisito, y es de cada persona en su clon:

```bash
bilinker init
```

Escribe dos líneas en `.git/` —la exclusión y el refspec— y nada más. Es idempotente, y **no toca ninguna rama**:

```
exclude: + .bilink/  + .bilink-migrate-*
refspec: + refs/bilink/*:refs/bilink/*  (origin)

refs/bilink/main no existe.
  Correr `bilinker track main` para crearla.
```

Que diga que la ref no existe es correcto: todavía no publicaste nada.

> **Por qué es explícito.** Escribir en `.git/config` es tocar la configuración de tu repo, y eso merece un comando y no un efecto colateral. Bilinker arregla solo lo que es suyo, y pide lo que es tuyo.

---

# Parte A — El proveedor publica

## A1. Elegir el fragmento

Lo que se publica es **un nodo del código**: un método, una clase, una función. No un rango de líneas — la referencia sobrevive a que el archivo se reformatee o el fragmento se mueva.

Conviene que el nodo **se nombre a sí mismo**. Un método con nombre es un buen ancla; un comentario o un `import` no.

## A2. Publicar

```bash
bilinker chain new \
  --tip 'src/main/java/ar/hsi/Permisos.java:4:5' \
  --tip abstract
```

La posición `4:5` es línea y columna de cualquier punto adentro del fragmento — bilinker sube por el AST hasta el nodo con nombre que lo contiene.

```
Created chain: 2e9efd68-2b7a-438e-96b8-4ff917f0529c
```

**Ese UUID es la dirección pública** de lo que publicás. Es lo que los consumidores van a nombrar, y sobrevive a que muevas o cambies el fragmento.

El archivo que quedó:

```yaml
# .bilink/2e9efd68-….yaml
endpoint:
  0:
    link: capture a8490a73c5bf5006397dcda13ce17079
  1:
    link: abstract        # lo aporta quien lo consuma
```

`abstract` es la punta abierta: no es responsabilidad tuya, la completa quien te consuma. Su estado es `OPEN` para siempre y nunca te va a pedir nada.

## A3. Aprobar lo que se publica

```bash
bilinker check .
bilinker accept .
```

```
2e9efd68  (PENDING, OPEN)
  2e9efd68.0  08f907b84a57  e636c28d3908
accepted 1 endpoint(s)
```

Aceptar es decir *"revisé esto y lo apruebo"*. De ahí en adelante, si el fragmento cambia, `check` te lo dice — **incluso sin ningún consumidor**. Es una pregunta tuya sobre tu propio código: *¿lo que publiqué sigue coincidiendo con lo que aprobé?*

Notá que `accept .` **no tocó la punta `abstract`**: no hay nada que aprobar del lado abierto.

## A4. Crear la ref

```bash
bilinker track main
```

```
ningún commit de refs/bilink/* califica: la ref nace desde cero.
commit:  refs/bilink/main @ e755f02
árbol:   4 archivo(s)
```

## A5. Verificar que tu rama no cambió

Éste es el paso que conviene correr delante de quien haya que convencer:

```bash
git log --oneline | wc -l     # los mismos commits que antes
git status --porcelain        # vacío
git branch -a                 # ninguna rama nueva
git ls-files | wc -l          # los mismos archivos que antes
```

Todo igual. Lo único que existe es una ref que hay que pedir por nombre para verla:

```bash
git show-ref | grep bilink
```

## A6. Publicar la ref

```bash
bilinker push
```

```
publicado: refs/bilink/main @ e755f02 → origin
```

**Ninguna interacción con `refs/bilink/*` se hace tipeando git.** La ref está fuera de `refs/heads/`, así que `git push` a secas no la empuja y habría que nombrarla con un refspec — y el refspec lo arma bilinker, que es de quien es la ref.

Publica **la ref y nada más**: qué commits de tu proyecto salen a la luz lo seguís decidiendo vos, con `git push`, cuando quieras.

---

# Parte B — El consumidor referencia

## B1. Declarar al proveedor

Un archivo, una vez:

```toml
# .bilink/.hsi.toml
remote = "git@gitlab…:minsal/hsi.git"
branch = "rc-2.32"
```

`branch` es la rama **del proyecto** del proveedor; bilinker traduce a su ref.

**Éste es el único lugar de tu repo que sabe algo del otro repo.** Ningún bilink va a contener una URL: si el proveedor cambia de host, editás este archivo y nada más.

## B2. Ver de qué colgarte

Antes de vincular nada, mirá qué publica:

```bash
bilinker fetch hsi
bilinker abstracts hsi
```

```
hsi · 3 abstracción(es)

  ca4dbbd9  src/main/java/ar/hsi/Permisos.java
            public boolean puede(String usuario, String operacion) {
                return consultar(usuario, operacion);
            }

  d290a546  src/main/java/ar/hsi/Turnos.java   ← ya lo consumís

  5ed1d14e  src/main/java/ar/hsi/Padron.java
            public int contar(String institucion) {
```

**Muestra el código**, que es lo que decide de cuál colgarse — una lista de UUIDs no alcanza para elegir. Y lo que ya consumís queda marcado, para no colgarte dos veces de lo mismo.

Mirar el catálogo **no trae nada**: el clon recorta el árbol de trabajo, no el object store, así que los fragmentos se leen aunque sus archivos no estén en disco. Podés mirar diez para elegir una sin que te queden nueve archivos ajenos en el working tree.

Del otro lado, el proveedor contesta la misma pregunta con `bilinker abstracts` a secas: *¿qué estoy publicando?*

## B3. Vincular

Con el UUID de la que elegiste:

```bash
bilinker chain new \
  --from-repo 'hsi:2e9efd68-2b7a-438e-96b8-4ff917f0529c' \
  --tip 'src/permisos.ts:1:1'
```

**Un solo `--tip`, el tuyo.** El otro lado es el proveedor, y lo aporta `--from-repo`.

```
Created chain: 2e9efd68-2b7a-438e-96b8-4ff917f0529c
```

El mismo UUID de los dos lados: **eso es lo que los hace encontrarse**, sin que ninguno escriba en el repo del otro.

## B4. Traer el archivo del fragmento

Sí, `fetch` otra vez — y la segunda corrida dice algo:

```bash
bilinker fetch hsi
```

```
hsi: refs/bilink/main · 1 archivo(s) en el sparse
```

La primera vez decía **0 archivos**: no consumías nada todavía. Ahora dice 1, porque te colgaste de uno.

**Ese número se calcula, no se declara**: sale de qué fragmentos referenciás. Si mañana te colgás de otro, el próximo `fetch` lo trae solo; si dejás de consumir uno, deja de traerlo. Un conjunto escrito a mano en el `.toml` habría quedado viejo con el primer vínculo nuevo.

Trae **una sola ref**, que lleva los bilinks del proveedor y el código al que apuntan, coherentes por construcción. El clon queda en `.bilink/hsi/`, no se commitea, y es superficial: sólo lo que hace falta.

## B5. Verificar y aceptar

```bash
bilinker check .
bilinker accept .
```

```
2e9efd68  (PENDING, PENDING)
accepted 2 endpoint(s)
```

Lo que quedó guardado de tu lado:

```yaml
endpoint:
  0:
    link: repo hsi
    accepted:
      link: capture a8490a73…    # ubicación publicada — opaca
      hash: 08f907b84a57…        # contenido publicado — opaco
  1:
    link: capture d9e4574a…      # tu fragmento
    accepted:
      link: capture d9e4574a…
      hash: ff75a8aa4daf…
```

Dos SHA-256 y un alias. **De ahí no se reconstruye nada** del proveedor: ni el path, ni el texto, ni el commit.

---

# Parte C — El día a día

## C1. El proveedor cambia lo que publica

Cambia el código y lo commitea, como siempre. Su `check` se lo dice:

```
2e9efd68  (ALTERED, OPEN)
```

Si el cambio es correcto, lo aprueba:

```bash
bilinker accept .
```

Con eso **la decisión queda registrada en la ref**, con autor y fecha:

```bash
bilinker log
```

## C2. El consumidor se entera — cuando quiere

Antes de traer nada, `check` no ve nada:

```
all clean (1 bilink(s))
```

**`check` nunca hace red.** Corre sobre todos tus bilinks y es barato; no puede andar hablando con servidores. Lo que no trajiste, no se ve.

Después del fetch, sí:

```bash
bilinker fetch hsi
bilinker check .
```

```
2e9efd68  (CHAIN_DIRTY, OK)
```

`CHAIN_DIRTY` = *el proveedor aprobó algo distinto de lo que vos aprobaste*.

## C3. Mirar qué cambió

```bash
bilinker get 2e9efd68.0 --diff
```

```diff
# src/main/java/ar/hsi/Permisos.java  lines 4–6
--- aceptado (e755f026)
+++ actual
@@ -1,3 +1,3 @@
-    public boolean puede(String usuario, String operacion) {
-        return consultar(usuario, operacion);
+    public boolean puede(String usuario, String operacion) {
+        return consultar(usuario, operacion) && habilitado(usuario);
     }
```

Recién acá el clon se profundiza, y sólo para este bilink: **`check` es masivo y barato; ver el diff es puntual y caro.**

## C4. Decidir

Si tu código sigue estando bien con el cambio:

```bash
bilinker accept 2e9efd68.0
```

Si no, arreglás tu código y aceptás los dos lados. Nadie acepta por vos: `apply` propone, `accept` dispone.

---

## Qué puede salir mal, y qué significa

| Estado | Qué pasó | Qué hacer |
|---|---|---|
| `REMOTE_UNREACHABLE` | el repo del proveedor no está clonado | `bilinker fetch <alias>` |
| `CHAIN_DIRTY` | el proveedor aprobó algo distinto | `get --diff`, revisar, `accept` |
| `REJECTED` | la punta del proveedor **dejó de ser `abstract`** | hablar con el proveedor: dejó de publicar eso |
| `BROKEN` | el clon está y el bilink no | el proveedor lo removió — investigar |
| `OPEN` | la punta abierta del proveedor | nada, está sana |

Y uno que no es un estado sino un corte:

```
Error: el proveedor 'hsi' publica formato 4.0.0 y este binario lee 3.1.0.
```

**El consumidor se niega en vez de malinterpretar.** No poder leer los archivos no es drift, y reportar cualquier estado sobre eso sería inventar. Entre proyectos con releases independientes, que las versiones diverjan es lo normal — se actualiza bilinker, o se fija el `.toml` a una rama compatible.

---

## Deshacer

**Del lado del proveedor**, mientras no publicaste la ref:

```bash
git update-ref -d refs/bilink/main
rm -rf .bilink/
```

Tu rama nunca cambió, así que no hay nada más que revertir. Ésa es la propiedad: **no publicar es el estado por defecto, y volver a él es gratis.**

**Del lado del consumidor:**

```bash
bilinker remove <uuid>
rm -rf .bilink/hsi .bilink/.hsi.toml
```

---

## El caso concreto: `hsi` → `retinar`

Lo de arriba, con los valores reales.

### Lo que corre el equipo de `hsi`

```bash
cd <clon de hsi>
git checkout rc-2.32

bilinker init

# El fragmento a publicar: el método de USER_PERMISSIONS.
bilinker chain new \
  --tip '<path del fragmento>:<línea>:<col>' \
  --tip abstract

bilinker check .
bilinker accept .
bilinker track rc-2.32

# La verificación que importa:
git status --porcelain    # vacío
git log --oneline -1      # el mismo commit que antes
git branch -a             # ninguna rama nueva

bilinker push
```

Y pasan el UUID que imprimió `chain new`.

**Lo que hay que decirles, en una línea:** nada de esto toca `rc-2.32`. Un commit menos, un archivo menos, una rama menos que si no hicieran nada — porque no hay ninguno.

### Lo que corre `retinar`

```bash
cd <clon de retinar>
bilinker init

cat > .bilink/.hsi.toml <<'EOF'
remote = "git@gitlab…:minsal/hsi.git"
branch = "rc-2.32"
EOF

# Ver qué publica, y elegir:
bilinker fetch hsi
bilinker abstracts hsi

bilinker chain new \
  --from-repo 'hsi:<el uuid elegido>' \
  --tip 'src/…/permissions.ts:<línea>:<col>'

bilinker fetch hsi
bilinker check .
bilinker accept .
```

Y de ahí en adelante, cada vez que quieras saber si `hsi` cambió lo que publica:

```bash
bilinker fetch hsi && bilinker check .
```

### Qué mirar para saber que salió bien

Del lado de `hsi`: su `rc-2.32` idéntica, y `git show-ref | grep bilink` mostrando la ref.

Del lado de `retinar`: `all clean`, y un `.bilink/<uuid>.yaml` que contiene el alias `hsi` y **ninguna URL**.

Y la prueba real: que alguien en `hsi` cambie el fragmento, lo acepte y pushee la ref — y que en `retinar`, un `fetch` más un `check` reporten `CHAIN_DIRTY`. Ése es el caso que motiva todo esto.
