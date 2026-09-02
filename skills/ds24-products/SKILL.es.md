---
name: ds24-products
language: es
description: Úsala al conectar una app con una cuenta de Digistore24 por primera vez — meter la clave de API, crear los productos que se venden, registrar la conexión del webhook IPN y solicitar la aprobación del marketplace. Úsala siempre que el usuario mencione una clave de API de Digistore24, «conectar Digistore», crear productos o planes, registrar una URL de IPN, o pregunte por qué Digistore24 nunca llama a su webhook.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# Conectar la app con Digistore24

Nada de una integración de Digistore24 funciona hasta que existen tres cosas del
lado de Digistore24: una clave de API que tu app pueda usar, un producto que
vender y una conexión IPN que apunte a tu endpoint. **Haz esto antes que nada** —
un handler de IPN al que nadie llama no se puede probar, y un enlace de
checkout para un producto que no existe es un 404.

## Paso 0 — ¿qué existe ya?

Mira antes de preguntar:

- ¿Hay un `DIGISTORE_API_KEY` en el entorno o en el almacén de secretos?
- ¿Hay un registro de productos en el proyecto (un archivo JSON/config que liste
  los planes con sus precios)?
- ¿Está puesta `DIGISTORE_IPN_PASSPHRASE`?

Luego pregúntale al usuario solo lo que falte de verdad. Si las tres están ahí,
ve al Paso 4 y comprueba la conexión en lugar de reconstruirla.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el archivo `VERSION` de este pack. Menciona cualquier
discrepancia en una frase y sigue adelante.

## Paso 1 — la API

```
POST https://www.digistore24.com/api/call/<FUNCTION>/format/json
Header: X-DS-API-KEY: <la clave>
Body:   application/x-www-form-urlencoded
```

**La clave viaja en la cabecera, nunca como parámetro de formulario.** Es un
secreto: variable de entorno o el almacén de secretos de la plataforma, nunca en
el código, nunca en nada que reciba el navegador.

La crea el propio usuario en su cuenta de Digistore24, en *Ajustes → Claves de
API* (*Settings → API keys*). Pídesela, dile dónde ponerla y no intentes
extraerla de una sesión del navegador.

⚠️ **Dile que le dé permiso de ESCRITURA** (Digistore24 lo llama *writable*).
Una clave queda acotada en el momento de crearla, y una de solo lectura lee los
productos perfectamente y luego falla en las dos llamadas sin las que la app no
puede vivir: crear los productos y crear una URL de checkout. Díselo mientras
está en esa pantalla — volver después para ampliar una clave significa crear una
nueva y reemplazarla en todas partes.

## Paso 2 — una lista de precios, en tu app

**Mantén los planes en un solo archivo de tu proyecto** — clave, nombre visible,
precio en céntimos, moneda, intervalo de facturación — y que todo lea de ahí: la
página de precios, el checkout, la comprobación del derecho de acceso.

El precio **no** vive en el producto de Digistore24. La API de Digistore24
descarta `data[amount]` en `createProduct`/`updateProduct` («obsoleto — crea un
plan de pago en su lugar»), y un plan de pago guardado en Digistore24 es fijo:
las pruebas gratuitas, los upgrades, los downgrades, los cupones y las comisiones
de afiliado por enlace solo funcionan cuando el plan viaja con la llamada de
checkout. Así que el precio va a `createBuyUrl` en el momento de la compra —
véase la skill **`ds24-checkout`**.

Un precio, un sitio. Una segunda lista en el código es una lista que se desvía.

🚨 **Pero el producto no se queda sin plan — Digistore24 le da el suyo por
defecto.** (Unos 27 €, pago único, visto en una cuenta real en septiembre de
2026; mira el producto del vendedor en lugar de fiarte de esa cifra. Lo que no
cambia es que *algún* plan hay.) Tu app nunca lo cobra: un plan que viaja con la
llamada a `createBuyUrl` gana siempre sobre el guardado. Lo que **sí** lo cobra
es el **formulario de pedido propio** del producto, que existe desde el momento
en que el producto existe — y tras la aprobación en el marketplace
(**`ds24-golive`**) es algo que encuentran desconocidos.

De ahí se siguen dos cosas, y las dos se pasan por alto con facilidad:

- **Díselo al vendedor antes de que abra su backoffice.** Verá un precio que
  nunca puso, junto a un producto que su app vende por otra cosa. Avisado, es una
  curiosidad; descubierto a solas, parece un fallo, y el arreglo al que recurre es
  una segunda lista de precios.
- **Decide a propósito qué hace tu manejador de IPN con una compra hecha ahí.**
  No lleva `tracking[custom]`, la tarifa es la del plan del producto y no la
  tuya, y si tu oferta es una **suscripción** enviará exactamente un evento de
  pago — nunca una renovación, nunca una cancelación, así que nada de lo que
  cuelgues de esos eventos se disparará para ella. El Paso 2 de
  **`ds24-checkout`** explica por qué la ausencia de `custom` no basta para saber
  que ha ocurrido esto.

**Si tu app habla más de un idioma, la entrada guarda un id de producto por
idioma** — no un id. La razón es el Paso 3; deja la forma correcta aquí, porque
cambiarla después de la primera venta significa productos nuevos y aprobaciones
nuevas:

```
pro:
  name:      "Pro"
  priceCents: 3900
  productIds:            # un producto de Digistore24 por idioma
    de: null
    en: null
```

**Y si la app tiene más de un entorno, mantén un CONJUNTO de productos por
entorno** (dev / prod — staging solo si existe de verdad). Los productos que
creas contra una URL de vista previa o de desarrollo son artículos de prueba:
dales sus propios ids en el registro, márcalos de forma visible en el nombre del
producto (`"Pro [DEV]"` — la API de Digistore24 no tiene campo de etiqueta, el
nombre es lo que ve una persona en el backoffice) y deja limpios los nombres de
los productos en producción. Un conjunto no debe reclamar nunca los productos
del otro — véase la nota sobre idempotencia más abajo. Un vendedor que solo
sincroniza contra el dominio en producción tiene un conjunto, y eso está bien.

## Paso 3 — crea los productos

`createProduct` / `updateProduct` con el nombre, la descripción y **`language`**.
Escribe el id de producto devuelto de vuelta en tu lista de precios, para que la
correspondencia quede registrada y no haya que volver a deducirla.

### Un producto por oferta Y por idioma — este es el que todos fallan

**Un producto de Digistore24 lleva exactamente UN idioma, y ese idioma es el
idioma del FORMULARIO DE PEDIDO que rellena tu comprador** — las etiquetas de los
campos, los botones, los nombres de los métodos de pago, las condiciones de
cancelación. Es `data[language]` en el producto.

**`createBuyUrl` no tiene parámetro de idioma.** Sus argumentos son `product_id`,
`buyer`, `payment_plan`, `tracking`, `valid_until`, `urls`, `placeholders`,
`settings` y `addons` — ahí no hay nada con lo que sobrescribir el idioma del
producto, y ningún parámetro de URL lo hace tampoco. Así que no puedes decidir el
idioma del formulario en el momento del checkout. Lo decides **eligiendo a qué
producto envías al comprador**.

Una app cuya interfaz habla alemán e inglés necesita por tanto **dos productos de
Digistore24 por oferta**, uno con `language=de` y otro con `language=en`, y el
checkout elige según el idioma del visitante. Manda a todo el mundo a un solo
producto y a la mitad de tus clientes se les piden los datos de su tarjeta en un
idioma que no eligieron — que es exactamente donde se abandona una compra.

Tres consecuencias que vale la pena escribir en lo que sea que construyas:

- **Pon `data[language]` explícitamente en cada producto.** Si se omite,
  Digistore24 recurre al idioma de la sesión de la API — la elección deliberada
  de nadie, y la causa habitual de que una tienda alemana muestre formularios de
  pedido en inglés.
- **Cubre todos los idiomas que tenga tu app.** Uno que falte debería seguir
  vendiendo (recurre a otro producto en vez de mostrar un botón muerto) — pero
  dilo en la salida de tu sincronización, porque nada más lo hará nunca: la app
  se renderiza bien, el checkout se abre, la compra se completa.
- **Cada producto de idioma se aprueba por separado**, en el marketplace al que
  pertenece su idioma. Véase la skill **`ds24-golive`**.

El *texto* de tu producto es una cuestión aparte. Enviar el mismo nombre y la
misma descripción a ambos productos es un valor por defecto perfectamente
razonable — el *formulario* que lo rodea es lo que tiene que seguir al comprador.

Hazlo **idempotente**: ejecútalo dos veces y la segunda vez actualiza en lugar de
crear un duplicado. Indéxalo por tu propia clave de producto **más el idioma** —
y, si mantienes conjuntos separados por entorno, **más el entorno**
(`pro__en__prod`) — cada producto necesita su propio identificador estable. Nunca
lo indexes por el nombre visible, que es el mismo para ambos idiomas y cambia con
el texto.

**Borrar un producto de tu lista no lo despublica.** Un producto que Digistore24
ya conoce sigue siendo comprable hasta que el usuario lo desactiva allí. Dilo en
voz alta cuando quites uno.

🚨 **Lo que significa que el momento de preguntar es ANTES de crear, no después.**
No hay ninguna llamada de API que deshaga un `createProduct`. Una vez que tu
sincronización ha corrido, cada entrada que encontró es un producto real en la
cuenta del usuario, y deshacerse de uno es una mano en el backoffice de
Digistore24 — para cada uno, en cada idioma. Una lista de precios que todavía
lleve las entradas que esbozaste mientras diseñabas la oferta las publicará
todas.

Así que la primera vez que tu sincronización fuera a crear algo: **imprime lo que
se crearía, por nombre, dile al usuario que no se puede deshacer y espera un
sí.** Luego crea. Las ejecuciones posteriores ya tienen ids archivados y no crean
nada, así que esto es una pregunta en un momento, no un aviso que la gente
aprende a saltarse a clics. Si algunas entradas son borradores y no ofertas, dale
a tu lista un marcador que las deje fuera de la sincronización en vez de pedirle
al usuario que borre texto que todavía quiere.

## Paso 4 — registra la conexión IPN

Este es el paso que se olvida, y su síntoma es «la compra funcionó pero en la app
no pasó nada».

- `ipnSetup` registra el endpoint. Digistore24 **lo valida inmediatamente** con
  un `GET` e insiste en un HTTP `200` — una redirección (301/302) también falla.
- **La URL tiene que ser `https` pública.** Digistore24 rechaza `http` y rechaza
  `localhost` de plano.
- Digistore24 genera la **passphrase de la IPN** o toma la tuya. Sea como sea,
  tiene que acabar en el entorno de la app como `DIGISTORE_IPN_PASSPHRASE` — es
  el secreto compartido con el que se calcula la firma, y sin él toda IPN se
  rechaza correctamente.

La llamada acepta estos parámetros, y dos de ellos deciden si los eventos llegan
alguna vez:

| | |
|---|---|
| `ipn_url` | tu endpoint, https pública |
| `name` | cómo se llama la conexión en el backoffice |
| `domain_id` | **la identidad de esta conexión** — véase abajo |
| `product_ids` | qué productos cubre — ids separados por comas, o `all` |
| `sha_passphrase` | la tuya propia, o `random` para que se genere una y se devuelva |

### `ipnSetup` es también la actualización — decide el `domain_id`

No hay una función de actualización aparte. Digistore24 busca una conexión por
**(comerciante, clave de API, `domain_id`)**: mismo id → se actualiza la conexión
existente, id desconocido → nace una segunda conexión. Eso es lo que hace la
llamada idempotente, y por eso el id tiene que estar **anotado** (una variable de
entorno, una fila de ajustes) y no volver a deducirse de algo que cambia.

**Y tiene que ser único.** Esta es la parte que se salta, y falla de forma
invisible. Un valor genérico — `test-local-1`, `local-app`, `myapp`, `production`
— no es un nombre, es una colisión con el **propio** otro proyecto del usuario:
los dos no obtienen dos conexiones, se turnan sobrescribiendo una. La segunda
configuración vuelve a apuntar en silencio la IPN de la primera app a su propia
URL, y a partir de ahí las compras de la primera app no llegan a ninguna parte.
Ambas ejecuciones informan de éxito.

Así que ponle una cola aleatoria y guárdalo:

```
test-local-diw2hvnz73
myapp-prod-k7f2m9x1qc
```

La parte legible dice de qué app se trata; la cola es lo que lo hace único. Nunca
reutilices uno entre dos apps, y nunca lo cambies solo porque haya cambiado la
URL — cambiarlo es como consigues una segunda conexión, duplicada.

### `product_ids` — qué compras informa esta conexión

Ids de producto de Digistore24 separados por comas: `product_ids=111,222,333`. El
valor por defecto es `all`, la cuenta entera.

**Es preferible nombrar los productos reales.** La cuenta de un vendedor suele
tener más que la app que estás construyendo — un funnel antiguo, una segunda app,
el lanzamiento de otra persona — y una conexión acotada a sus propios productos
es lo que permite que dos apps del mismo vendedor estén conectadas a la vez.

`all` es aceptable, con una condición que pertenece al endpoint: **una compra de
un producto que tu app no conoce tiene que ignorarse, no adivinarse.** Regístrala
si quieres, no concedas nada por ella. Un endpoint que asigna un producto
desconocido a un plan por defecto reparte acceso por una compra que nunca fue
tuya.

**En una plataforma alojada de creación con IA esta es la parte fácil**, y vale
la pena decírselo al usuario: la URL de vista previa/producción de una app de
Lovable, Replit, v0 o Manus ya es https pública, así que el endpoint se puede
registrar directamente. En un portátil no — una dirección local necesita antes un
túnel.

## Paso 5 — antes del dinero real: la aprobación

Un producto se puede **comprar en pruebas** de inmediato, por el vendedor, con la
cookie de compra de prueba de Digistore24 puesta — o, en un entorno de
desarrollo, con el parámetro testpay en la URL de compra (**`ds24-checkout`**,
Paso 4a). Así es como verificas la cadena entera sin mover dinero.

Vender al público a través de un **reseller** necesita además la **aprobación del
marketplace** (`approval_status=pending`) — pídela solo cuando la descripción y
la app estén realmente terminadas, porque un producto a medio construir se
rechaza y el segundo intento es más lento.

**Un Direct Seller no tiene ningún paso de aprobación.** Solo los siteowners 1
(Alemania), 2 (EE. UU.), 3 (Reino Unido) y 4 (Irlanda) son resellers y aprueban
productos; un vendedor que vende por su propia cuenta no tiene nada que pedir ni
nada que esperar. Comprueba con cuál estás tratando antes de prometerle al
usuario un paso de aprobación — o construir un recordatorio que nunca podrá
satisfacer.

Si se concedió o no se puede leer: los elementos de `listProducts` / `getProduct`
llevan `approval_status_list`, una entrada por marketplace. La skill
**`ds24-golive`** (Paso 4) tiene el campo, su conjunto de valores y sus trampas —
y recorre toda la puesta en producción, incluida la compra de prueba.

## Paso 6 — demuestra la conexión

No informes de éxito solo a partir de una respuesta de la API. Comprueba que:

1. `GET <tu URL de IPN>` responde **200** desde la internet pública.
2. El producto aparece en la cuenta de Digistore24 del usuario.
3. `DIGISTORE_IPN_PASSPHRASE` está puesta en el entorno de la app — no solo en un
   archivo local que la app desplegada nunca lee.

Luego demuestra el endpoint en sí — la skill **`ds24-ipn`** dice qué tiene que
cumplirse y cómo comprobarlo en esta plataforma.

## Paso 7 — `getPurchase`: consulta un pedido tú mismo

Cuando el usuario dice *«lo he comprado y no ha pasado nada»*, no lo mandes a su
backoffice de Digistore24 a leerte un estado en voz alta. Pregúntale a la API:

```
POST https://www.digistore24.com/api/call/getPurchase/format/json
Header: X-DS-API-KEY: <la clave>
Body:   purchase_id=ABC12345
```

Devuelve la propia visión de Digistore24 de ese pedido — estado, producto,
comprador, tipo de facturación, próximo pago y los enlaces de gestión (factura,
recibo, detener el recobro periódico, actualizar los datos de pago). No cambia
nada, así que es seguro llamarla mientras diagnosticas. `listPurchases` es lo
mismo para muchos, filtrado (por ejemplo, por el email del comprador).

**Constrúyelo en la app como un pequeño ayudante de admin/CLI la primera vez que
lo necesites** — convierte una discusión en una consulta. La respuesta clasifica
la queja en casos que no tienen nada que ver entre sí:

| Lo que dice `getPurchase` | Qué falla en realidad |
|---|---|
| **Id desconocido / sin datos** | no hubo compra, o se hizo en una cuenta de Digistore24 distinta de la clave que estás usando. La app está bien |
| **Conoce el pedido, tu app no** | se pagó y no te llegó ninguna IPN. Mira la conexión: ¿la URL registrada sigue respondiendo?, ¿otro proyecto sobrescribió el `domain_id`?, ¿está este producto dentro de los `product_ids` de la conexión? |
| **Ambos lo conocen, pero falta el acceso** | la IPN llegó y el fallo está en la correspondencia evento→acceso → **`ds24-entitlements`** |

Una IPN rechazada es un cuarto caso y tiene su propia herramienta — la
comprobación de la firma en **`ds24-ipn`**, ejecutada contra el cuerpo en bruto
que llegó.

## Paso 8 — qué viene después

- **`ds24-ipn`** — el endpoint que recibe los eventos (constrúyelo ahora si no
  existe).
- **`ds24-checkout`** — el enlace de compra, con el precio adjunto.
- **`ds24-golive`** — la compra de prueba que demuestra la cadena entera.

Di cuál vas a empezar y empiézala.
