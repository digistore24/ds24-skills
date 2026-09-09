---
name: ds24-products
language: es
description: Úsala al conectar una app con una cuenta de Digistore24 por primera vez — introducir la clave de API, crear los productos que se van a vender, registrar la conexión IPN (el webhook) y solicitar la aprobación del marketplace. Úsala también siempre que el usuario mencione una clave de API de Digistore24, diga «conectar Digistore», quiera crear productos o planes, registrar una URL de IPN, o pregunte por qué Digistore24 nunca llama a su webhook.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# Conectar la app con Digistore24

En una integración con Digistore24 nada funciona hasta que existen tres cosas
del lado de Digistore24: una clave de API que tu app pueda usar, un producto que
vender y una conexión IPN que apunte a tu endpoint. **Hazlo antes que cualquier
otra cosa**: un handler de IPN al que nadie llama no se puede probar, y un
enlace de checkout a un producto que no existe es un 404.

## Paso 0 — ¿qué existe ya?

Mira antes de preguntar:

- ¿Existe `DIGISTORE_API_KEY` en el entorno o en el almacén de secretos?
- ¿Tiene el proyecto un registro de productos (un archivo JSON o de
  configuración con los planes y sus precios)?
- ¿Está definida `DIGISTORE_IPN_PASSPHRASE`?

Después pregunta al usuario solo lo que falte de verdad. Si están las tres,
salta al Paso 4 y comprueba la conexión en vez de volver a montarla.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el archivo `VERSION` de este pack. Si no coinciden, dilo en una
frase y sigue adelante.

## Paso 1 — la API

```
POST https://www.digistore24.com/api/call/<FUNCTION>/format/json
Header: X-DS-API-KEY: <la clave>
Body:   application/x-www-form-urlencoded
```

**La clave viaja en la cabecera, nunca como parámetro del formulario.** Es un
secreto: va en una variable de entorno o en el almacén de secretos de la
plataforma, nunca en el código ni en nada que llegue al navegador.

La crea el propio usuario en su cuenta de Digistore24, en *Settings → API keys*
(Ajustes → Claves de API). Pídesela, dile dónde guardarla y no intentes sacarla
de una sesión del navegador.

⚠️ **Dile que le dé permiso de ESCRITURA** (Digistore24 lo llama *writable*).
El alcance de una clave se fija al crearla: una de solo lectura lee los
productos sin ningún problema y falla justo en las dos llamadas de las que la
app no puede prescindir, crear los productos y crear una URL de checkout.
Díselo mientras tiene esa pantalla delante — ampliar una clave después
significa crear otra nueva y sustituirla en todas partes.

## Paso 2 — una lista de precios, en tu app

**Guarda los planes en un único archivo del proyecto** y haz que todo lea de
ahí: la página de precios, el checkout, la comprobación del derecho de acceso.
Una entrada por **oferta**, y dentro de ella una entrada por **forma de pago**.

🚨 **Mensual y anual son dos formas de pago de UNA oferta, no dos ofertas.** Es
la decisión de forma con más consecuencias de toda esta skill. Un registro que
las convierte en dos entradas las convierte en dos claves de producto, y a
partir de ahí cada comprobación de acceso de la app tiene que nombrar las dos:
`hasAccess(m, "pro_monthly") || hasAccess(m, "pro_yearly")`, en cada puerta,
para siempre. El que se escribe con una sola clave rechaza en silencio a la
mitad de los compradores, detrás de una página que se renderiza sin problemas.
Mantenlas bajo una única clave y la pregunta sigue siendo una sola pregunta.

En Digistore24 los dos ejes se comportan de forma distinta, y este es el motivo:

| eje | lo que cuesta |
|---|---|
| **idioma** | un PRODUCTO por cada uno — el idioma del formulario de pedido es una propiedad del producto (Paso 3) |
| **forma de pago** | un PLAN DE PAGO por cada una, sobre ese mismo producto |

```
pro:
  name: "Pro"
  paymentOptions:                     # un plan de pago por cada una
    monthly: { priceCents: 3900,  billingInterval: 1_month }
    yearly:  { priceCents: 39000, billingInterval: 12_month }
  productIds:                         # un producto de Digistore24 por idioma
    de: null
    en: null
  payplanIds:                         # los escribe tu sincronización
    de: { monthly: null, yearly: null }
    en: { monthly: null, yearly: null }
```

⚠️ **Lo que COMPRUEBE este archivo tiene que leer el precio de la forma de pago,
no de la oferta.** Una oferta que declara `paymentOptions` no tiene `priceCents`
propio, así que un validador escrito contra la forma anterior le dice a toda
lista de precios correcta que no tiene precio. Medido, en la primera ejecución
real: un aviso que salta con una entrada correcta le enseña al lector a saltarse
los avisos.

### Dónde vive el precio

**Tu archivo es el original. Digistore24 recibe una copia, en forma de planes
de pago.**

`data[amount]` en el producto está obsoleto y se descarta, así que el precio no
se fija nunca en el producto mismo — lo dice la propia API en su aviso: *«crea
un plan de pago en su lugar»*. Para eso están `createPaymentplan` /
`updatePaymentplan`, y tu sincronización escribe uno por cada forma de pago
(Paso 3).

⚠️ **Puede que hayas leído lo contrario en una copia anterior de este pack**,
que decía que había que mantener todos los precios fuera de Digistore24 y
enviarlos con cada llamada a `createBuyUrl`. El razonamiento era: un plan
guardado es un segundo sitio para el precio, y no sabe hacer pruebas gratuitas,
upgrades ni cupones. La mitad de eso era cierto. Lo que se le escapaba es que
**tu enlace de checkout no es la única entrada al producto**:

- el producto tiene un **formulario de pedido propio**, que ningún enlace tuyo
  toca;
- un **afiliado** puede mandar tráfico directamente a él;
- el comprador puede **cambiar su intervalo de facturación** desde dentro de su
  compra (`switch_pay_interval_url`, en cada IPN) — y eso cambia a otro plan
  *guardado*, así que sin ninguno no hay a qué cambiar.

Los tres cobran los planes que cuelguen del producto. Un producto sin ningún
plan tuyo no se queda sin plan: **Digistore24 le asigna uno por defecto** —
unos 27 €, pago único, visto en una cuenta real en septiembre de 2026; mira el
producto del vendedor en lugar de fiarte de esa cifra. En una oferta de
**suscripción**, un pedido así envía exactamente un evento de pago y ninguna
renovación ni cancelación, así que el comprador paga una vez y conserva el
acceso **para siempre**.

Escribir los planes es lo que cierra eso. Es la segunda razón de esta forma; la
primera es que ahora una oferta necesita un producto en vez de dos.

De ahí se siguen dos cosas, y ambas son fáciles de pasar por alto:

- **Dile al vendedor qué va a ver en su backoffice**: sus propios precios, un
  plan por cada forma de pago — **como copia**. Cambia un precio en la app y
  sincroniza; cámbialo allí y la siguiente sincronización lo vuelve a cambiar.
  Un vendedor al que no se le dice esto edita la copia equivocada una vez y
  concluye que la app se olvida de los precios.
- **Una compra hecha en el formulario de pedido propio del producto sigue sin
  llevar `tracking[custom]`**, así que tu handler de IPN tiene que atribuirla
  por otra vía (el Paso 2 de **`ds24-checkout`** explica por qué la ausencia de
  `custom`, por sí sola, no demuestra que haya pasado eso). Lo que ha cambiado
  es que ya no se cobra a un precio distinto del tuyo, y ya no es una
  suscripción que nunca se renueva.

**Si tu app habla más de un idioma, la entrada guarda un id de producto por
idioma**, no uno solo. El motivo está en el Paso 3; acierta con la forma ya
aquí, porque cambiarla después de la primera venta significa productos nuevos
y aprobaciones nuevas.

**Y si la app tiene más de un entorno, mantén un CONJUNTO de productos por
entorno** (dev / prod — staging solo si existe de verdad). Los productos que
creas contra una URL de vista previa o de desarrollo son artículos de prueba:
dales sus propios ids en el registro, márcalos de forma visible en el nombre
del producto (`"Pro [DEV]"` — la API de Digistore24 no tiene campo de etiqueta;
el nombre es lo que una persona ve en el backoffice) y deja limpios los nombres
de los productos en producción. Un conjunto nunca debe apropiarse de los
productos del otro — véase la nota sobre idempotencia más abajo. Un vendedor
que solo sincroniza contra el dominio en producción tiene un único conjunto, y
eso está bien.

## Paso 3 — crea los productos

Dos llamadas por producto, en este orden:

1. `createProduct` / `updateProduct` con el nombre, la descripción y
   **`language`**. Escribe el id de producto que devuelve en tu lista de
   precios, para que la correspondencia quede registrada y no haya que
   deducirla otra vez.
2. `createPaymentplan` / `updatePaymentplan` **una vez por cada forma de
   pago**, con `product_id`, `first_amount`, `currency`,
   `first_billing_interval`, `other_amounts`, `other_billing_intervals`,
   `number_of_installments` (`0` = suscripción indefinida, `1` = pago único) y
   `position`. Escribe también el `paymentplan_id` que devuelve.

   Pon `is_switching_allowed = Y` cuando la oferta tenga más de una forma de
   pago: eso es lo que hace que el propio `switch_pay_interval_url` de
   Digistore24 lleve a alguna parte, y te ahorra construir un flujo de upgrade
   para el cambio más habitual que hace un suscriptor.

⚠️ **Haz las dos juntas, no en dos pasadas.** Un producto que existe sin sus
planes tiene el plan por defecto de ~27 € de Digistore24 y un formulario de
pedido que lo cobra; la ventana en la que eso es cierto debería medir una sola
llamada de la API.

🚨 **Y escribe de paso una MARCA DE PROPIEDAD.** Tanto `createProduct` como
`updateProduct` aceptan `data[note]`, una nota interna libre que ningún
comprador ve. Pon ahí una línea legible por máquina — el id propio de tu app y
el entorno. Sin ella no hay respuesta honesta a «¿este producto lo creamos
NOSOTROS?», y el Paso 3b necesita una. No deduzcas ese id del nombre de la app
(los vendedores cambian de nombre) ni del nombre interno del producto (dos apps
construidas igual chocan en él): genéralo una vez, guárdalo junto a la lista de
precios y no lo regeneres nunca.

🚨 **`note` conserva 47 caracteres y descarta el resto, en silencio.** Medido
contra una cuenta real el 2026-09-09: un valor de 120 caracteres volvió cortado
a mitad de palabra, sin error, sin aviso y sin una palabra al respecto en la
documentación de la API. Así que mantén la marca corta — `miapp:1:<appId>:<env>`
son unos 30 caracteres — y **no metas en ella la clave de producto ni el
idioma**: ya están en `name_intern`, que el listado devuelve junto a la nota.

Una marca un carácter demasiado larga no es una marca más corta. No se parsea,
todos los productos que creó tu app se leen como de otro, y tu paso de limpieza
informa de «nada que retirar» a partir de una comparación que no encontró nada
porque no pudo. Fija la longitud en un test.

### La segunda marca: `data[tag]`

Junto a la nota hay un campo **tag**, y responde a una pregunta más gruesa: no
«qué app hizo esto» sino «¿lo hizo una app siquiera?». Ponle un nombre que
identifique tu herramienta y el vendedor podrá filtrar su backoffice por él.

```
data[tag] = ds24-appkit
```

🚨 **Los tags son una LISTA separada por comas y el campo se escribe entero.**
Así que añadir uno es siempre leer-modificar-escribir:

1. lee el `tag` actual del producto (`getProduct`, o el listado que ya tienes:
   ahí también viene);
2. si el tuyo ya está en la lista, no escribas nada;
3. si no, añádelo detrás de una coma y manda la lista **entera** de vuelta con
   `updateProduct`.

Escribir solo tu propio tag borra todos los que el vendedor puso ahí. En
`createProduct` no hay nada que leer, así que tu tag es el valor completo.

⚠️ **Y ten esto presente sobre `data` en general: se valida contra una lista
blanca estricta.** Una clave que Digistore24 no conoce es un ERROR — la llamada
entera se rechaza con «ungültiger Array-Schlüssel», no se ignora en silencio.
Así que un campo más nuevo que la API de la cuenta rompe todas las llamadas que
lo lleven. Si escribes contra un campo que se está desplegando: mándalo, captura
el rechazo y repite la llamada una vez sin él.

🚨 **Y NO dejes que tu paso de limpieza lea el tag.** Todas las apps
construidas igual llevan el mismo — una retirada decidida sobre él dejaría que
una app borrase los productos de otra. La propiedad se queda en la marca fina
de la nota, la que lleva el id propio de tu app.

De los 47 caracteres se siguen dos reglas, y las dos van de a quién pertenece
ese campo:

- **No escribas nunca encima de un texto que no escribiste tú.** No hay sitio
  para fusionar, así que si la nota contiene otra cosa, déjala y sigue. Ese
  producto se queda sin marcar y tu paso de limpieza no lo tocará — la
  dirección segura.
- **Pero da por tuyo tu propio PREFIJO aunque no se parsee.** El corte puede
  caer dentro de una marca que escribiste tú. Sin esto, una marca que alguna vez
  te salió mal es permanente: se lee como texto del vendedor para siempre y no
  hay forma de repararla. Un marcador que no puedes arreglar es peor que uno que
  no puedes leer.

### Un producto por oferta Y por idioma — aquí es donde la gente se equivoca

*(Y, por decirlo una vez más donde es más fácil invertirlo: un producto por
IDIOMA, un plan de pago por FORMA DE PAGO. El eje del idioma multiplica los
productos; la forma de pago no.)*

**Un producto de Digistore24 lleva exactamente UN idioma, y es el idioma del
FORMULARIO DE PEDIDO que rellena tu comprador**: las etiquetas de los campos,
los botones, los nombres de los métodos de pago, las condiciones de
cancelación. Es `data[language]` en el producto.

**`createBuyUrl` no tiene parámetro de idioma.** Sus argumentos son
`product_id`, `buyer`, `payment_plan`, `tracking`, `valid_until`, `urls`,
`placeholders`, `settings` y `addons`; ahí no hay nada que sobrescriba el idioma
del producto, y tampoco lo hace ningún parámetro de URL. Así que el idioma del
formulario no se decide en el momento del checkout: se decide **eligiendo a
qué producto envías al comprador**.

Por tanto, una app cuya interfaz habla alemán e inglés necesita **dos productos
de Digistore24 por oferta**, uno con `language=de` y otro con `language=en`, y
el checkout elige según el idioma del visitante. Si mandas a todo el mundo al
mismo producto, a la mitad de tus clientes se les pedirán los datos de la
tarjeta en un idioma que no eligieron — y ese es exactamente el punto en el que
se abandona una compra.

Tres consecuencias que conviene dejar escritas en lo que construyas:

- **Pon `data[language]` de forma explícita en cada producto.** Si lo omites,
  Digistore24 recurre al idioma de la sesión de la API — una elección que nadie
  ha tomado a propósito, y la causa habitual de que una tienda alemana muestre
  formularios de pedido en inglés.
- **Cubre todos los idiomas que tenga tu app.** Un idioma sin producto debería
  seguir vendiendo (recurre a otro producto en vez de mostrar un botón muerto),
  pero dilo en la salida de tu sincronización, porque nada más lo dirá nunca:
  la app se renderiza bien, el checkout se abre, la compra se completa.
- **Cada producto de idioma se aprueba por separado**, en el marketplace al que
  pertenece su idioma. Véase la skill **`ds24-golive`**.

Los *textos* de tu producto son otra cuestión. Enviar el mismo nombre y la
misma descripción a los dos productos es un valor por defecto totalmente
razonable; lo que tiene que seguir al comprador es el *formulario* que los
rodea.

Hazlo **idempotente**: si se ejecuta dos veces, la segunda actualiza en vez de
crear un duplicado. Usa como clave tu propia clave de producto **más el
idioma** — y, si mantienes conjuntos separados por entorno, **más el entorno**
(`pro__en__prod`): cada producto necesita su propio identificador estable.
Nunca uses como clave el nombre visible, que es el mismo en ambos idiomas y
cambia con los textos.

**Borrar un producto de tu lista no lo retira de la venta por sí solo.** El
producto que Digistore24 ya conoce se puede seguir comprando — por su propio
formulario de pedido y por cualquier enlace de checkout que exista — hasta que
algo lo retire. De eso trata el Paso 3b.

🚨 **Y por eso el momento de preguntar es ANTES de crear, no después.**
`deleteProduct(product_id)` existe, así que esto no es definitivo como decía una
copia anterior de este pack. Pero solo es limpio mientras el producto **no haya
vendido nunca**: uno que ya ha cobrado dinero solo se puede desactivar, porque
los reembolsos, los contracargos y las cancelaciones de sus compradores siguen
llegando como IPNs que nombran su id y tu handler tiene que seguir
recibiéndolos. Un producto que ha cobrado dinero es una decisión con la que el
vendedor se queda.

Así que la primera vez que tu sincronización vaya a crear algo: **imprime lo
que se crearía, con su nombre, di lo que cuesta y espera un sí.** Después crea. Las ejecuciones siguientes ya tienen los ids
guardados y no crean nada, así que es una pregunta en un único momento, no un
aviso que la gente aprende a cerrar sin leer. Si algunas entradas son
borradores y no ofertas, dale a tu lista un marcador que las deje fuera de la
sincronización, en vez de pedirle al usuario que borre un texto que todavía
quiere.

## Paso 3b — retira lo que ya no se ofrece

Una entrada que sale de la lista de precios deja un producto detrás. Limpiarlos
merece la pena — una cuenta que se va llenando de productos abandonados es la
forma en que un vendedor acaba vendiendo algo de lo que ya se había olvidado —
y es el único paso en el que equivocarse destruye el trabajo de otra persona.

**No actúes nunca sobre otra cosa que tu propia marca de propiedad** (Paso 3).
Ni sobre el nombre del producto, ni sobre el nombre interno, ni sobre la
carpeta: en la cuenta de un vendedor hay productos anteriores a que tu app
existiera, productos de una segunda app construida igual y productos que un
agente de soporte creó a mano. «Seguramente es nuestro» no basta para borrar.

Después, por cada producto que lleve tu marca y ya no esté en la lista de
precios:

1. **Pregunta si tiene ventas** (`listPurchases` con su `product_id`). Si no
   puedes preguntarlo, dalo por «las tiene»: el error barato es un producto
   inactivo de más.
2. **Sin ventas → `deleteProduct`.** Si falla por lo que sea, pasa al punto
   siguiente.
3. **Con ventas, o si el borrado falló → `updateProduct` con
   `data[is_active] = N`.** El producto deja de poder comprarse y se queda en
   la cuenta. Dile al vendedor que su entrada debería quedarse en la lista de
   precios como entrada aparcada, o su id saldrá de tu registro de IPN y los
   reembolsos de sus compradores dejarán de llegar.

🚨 **Y no hagas absolutamente nada si no pudiste leer las notas.** Si el listado
de productos volvió sin ellas, la propiedad no se puede establecer, y «cero
productos que retirar» no es entonces una respuesta: es silencio. Dilo y para.
Demostrar que el recorrido se hizo no es demostrar que la comparación se hizo.

⚠️ **Y asegúrate de que el marcador siga funcionando con la lista de precios
VACÍA.** El sitio natural para un «no hay nada que sincronizar» es antes de
hablar con Digistore24 — y entonces quien retire su ÚLTIMA entrada se queda el
producto en la cuenta sin forma de retirarlo. Medido: es el más fácil de estos
errores, porque todos los tests tienen al menos un producto dentro. «Nada que
sincronizar» y «nada que limpiar» son dos preguntas.

**Siempre detrás de un marcador explícito.** A diferencia de crear, esta no es
una pregunta de la primera ejecución: una clave renombrada o una errata en la
lista de precios borrarían un producto vivo en una sincronización cualquiera.
Enumera lo que se iría, no cambies nada y deja que sea el usuario quien lo pida.

## Paso 4 — registra la conexión IPN

Este es el paso que se olvida, y su síntoma es «la compra funcionó, pero en la
app no pasó nada».

- `ipnSetup` registra el endpoint. Digistore24 **lo valida de inmediato** con
  un `GET` y exige un HTTP `200`; una redirección (301/302) también falla.
- **La URL tiene que ser `https` y pública.** Digistore24 rechaza `http` y
  rechaza `localhost` de plano.
- Digistore24 genera la **passphrase de la IPN** o acepta la tuya. En ambos
  casos tiene que acabar en el entorno de la app como
  `DIGISTORE_IPN_PASSPHRASE`: es el secreto compartido con el que se calcula la
  firma, y sin él toda IPN se rechaza, y con razón.

La llamada acepta estos parámetros, y dos de ellos deciden si los eventos
llegarán alguna vez:

| | |
|---|---|
| `ipn_url` | tu endpoint, https público |
| `name` | el nombre de la conexión en el backoffice |
| `domain_id` | **la identidad de esta conexión** — véase más abajo |
| `product_ids` | qué productos cubre — ids separados por comas, o `all` |
| `sha_passphrase` | la tuya, o `random` para que se genere una y se devuelva |

### `ipnSetup` es también la actualización — lo decide el `domain_id`

No hay una función de actualización aparte. Digistore24 busca la conexión por
**(comerciante, clave de API, `domain_id`)**: con el mismo id se actualiza la
conexión existente; con un id desconocido nace una segunda conexión. Eso es lo
que hace idempotente la llamada, y por eso el id tiene que estar **apuntado**
(en una variable de entorno, en una fila de ajustes) y no deducirse cada vez de
algo que cambia.

**Y tiene que ser único.** Esta es la parte que se salta, y falla sin dejar
rastro. Un valor genérico — `test-local-1`, `local-app`, `myapp`, `production`
— no es un nombre, es una colisión con **otro** proyecto del propio usuario:
los dos no obtienen dos conexiones, sino que se turnan para sobrescribir una.
La segunda configuración redirige en silencio la IPN de la primera app a su
propia URL, y desde ese momento las compras de la primera app no llegan a
ninguna parte. Y las dos ejecuciones terminan diciendo que todo fue bien.

Así que añádele un sufijo aleatorio y guárdalo:

```
test-local-diw2hvnz73
myapp-prod-k7f2m9x1qc
```

La parte legible dice de qué app se trata; el sufijo es lo que lo hace único.
No reutilices nunca uno entre dos apps, y no lo cambies solo porque haya
cambiado la URL: cambiarlo es justamente la forma de acabar teniendo una
segunda conexión duplicada.

### `product_ids` — qué compras notifica esta conexión

Ids de producto de Digistore24 separados por comas: `product_ids=111,222,333`.
El valor por defecto es `all`, la cuenta entera.

**Mejor nombrar los productos concretos.** La cuenta de un vendedor suele
contener más cosas que la app que estás construyendo — un embudo de ventas
antiguo, una segunda app, el lanzamiento de otra persona — y una conexión
limitada a sus propios productos es lo que permite tener conectadas a la vez
dos apps del mismo vendedor.

`all` es aceptable, con una condición que va en el endpoint: **una compra de un
producto que tu app no conoce se ignora, no se adivina.** Regístrala si
quieres, pero no concedas nada por ella. Un endpoint que asigna un producto
desconocido a un plan por defecto está repartiendo acceso por una compra que
nunca fue tuya.

**En una plataforma alojada de creación con IA esta es la parte fácil**, y
merece la pena decírselo al usuario: la URL de vista previa o de producción de
una app de Lovable, Replit, v0 o Manus ya es https pública, así que el endpoint
se puede registrar directamente. En un portátil no: una dirección local
necesita antes un túnel.

## Paso 5 — antes del dinero real: la aprobación

Un producto se puede **comprar en modo de prueba** de inmediato: lo hace el
propio vendedor con la cookie de compra de prueba de Digistore24 puesta — o, en
un entorno de desarrollo, con el parámetro testpay en la URL de compra
(**`ds24-checkout`**, Paso 4a). Así se verifica la cadena entera sin mover
dinero.

Vender al público a través de un **reseller** exige además la **aprobación del
marketplace** (`approval_status=pending`). Solicítala solo cuando la
descripción y la app estén terminadas de verdad: un producto a medio construir
se rechaza, y el segundo intento es más lento.

**Un Direct Seller no tiene paso de aprobación alguno.** Solo los siteowners 1
(Alemania), 2 (EE. UU.), 3 (Reino Unido) y 4 (Irlanda) son resellers y aprueban
productos; un vendedor que vende por su propia cuenta no tiene nada que
solicitar ni nada que esperar. Comprueba con cuál de los dos estás tratando
antes de prometerle al usuario un paso de aprobación — o de construirle un
recordatorio que nunca podrá satisfacer.

Si la aprobación se concedió es algo que se puede leer: los elementos de
`listProducts` / `getProduct` llevan `approval_status_list`, una entrada por
marketplace. La skill **`ds24-golive`** (Paso 4) documenta el campo, su
conjunto de valores y sus trampas — y recorre toda la puesta en producción,
compra de prueba incluida.

## Paso 6 — demuestra la conexión

No des por buena la conexión solo por la respuesta de la API. Comprueba que:

1. `GET <tu URL de IPN>` responde **200** desde la internet pública.
2. El producto aparece en la cuenta de Digistore24 del usuario.
3. `DIGISTORE_IPN_PASSPHRASE` está definida en el entorno de la app, no solo en
   un archivo local que la app desplegada nunca lee.

Después demuestra el endpoint en sí: la skill **`ds24-ipn`** dice qué tiene que
cumplirse y cómo comprobarlo en esta plataforma.

## Paso 7 — `getPurchase`: consulta un pedido tú mismo

Cuando el usuario diga *«lo he comprado y no ha pasado nada»*, no lo mandes a su
backoffice de Digistore24 a que te lea un estado. Pregúntaselo a la API:

```
POST https://www.digistore24.com/api/call/getPurchase/format/json
Header: X-DS-API-KEY: <la clave>
Body:   purchase_id=ABC12345
```

Devuelve la visión que Digistore24 tiene de ese pedido concreto — estado,
producto, comprador, tipo de facturación, próximo pago y los enlaces de gestión
(factura, recibo, detener el recobro periódico, actualizar los datos de pago).
No cambia nada, así que se puede llamar sin riesgo mientras diagnosticas.
`listPurchases` es lo mismo para muchos pedidos, con filtros (por ejemplo, por
el email del comprador).

**Incorpóralo a la app como una pequeña utilidad de administración o de CLI la
primera vez que lo necesites**: convierte una discusión en una consulta. La
respuesta separa la queja en casos que no tienen nada que ver entre sí:

| Lo que dice `getPurchase` | Qué falla en realidad |
|---|---|
| **Id desconocido / sin datos** | no hubo compra, o se hizo en una cuenta de Digistore24 distinta de la de la clave que estás usando. La app está bien |
| **Conoce el pedido y tu app no** | se pagó y no te llegó ninguna IPN. Revisa la conexión: ¿la URL registrada sigue respondiendo?, ¿otro proyecto sobrescribió el `domain_id`?, ¿este producto está dentro de los `product_ids` de la conexión? |
| **Los dos lo conocen, pero falta el acceso** | la IPN llegó y el fallo está en la correspondencia evento→acceso → **`ds24-entitlements`** |

Una IPN rechazada es un cuarto caso y tiene su propia herramienta: la
comprobación de la firma de **`ds24-ipn`**, ejecutada contra el cuerpo en bruto
que llegó.

## Paso 8 — qué viene después

- **`ds24-ipn`** — el endpoint que recibe los eventos (constrúyelo ahora si no
  existe).
- **`ds24-checkout`** — el enlace de compra, eligiendo la forma de pago.
- **`ds24-golive`** — la compra de prueba que demuestra la cadena entera.

Di cuál vas a empezar y empiézala.
