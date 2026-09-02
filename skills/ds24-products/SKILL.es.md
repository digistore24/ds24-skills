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

**Guarda los planes en un único archivo del proyecto** — clave, nombre visible,
precio en céntimos, moneda, intervalo de facturación — y haz que todo lea de
ahí: la página de precios, el checkout, la comprobación del derecho de acceso.

El precio **no** vive en el producto de Digistore24. Su API descarta
`data[amount]` en `createProduct`/`updateProduct` («obsoleto — crea un plan de
pago en su lugar»), y un plan de pago guardado en Digistore24 es fijo: las
pruebas gratuitas, los upgrades, los downgrades, los cupones y las comisiones
de afiliado por enlace solo funcionan si el plan viaja con la llamada de
checkout. Por eso el precio se entrega a `createBuyUrl` en el momento de la
compra — lo explica la skill **`ds24-checkout`**.

Un precio, un solo sitio. Una segunda lista en el código es una lista que
tarde o temprano se desvía.

🚨 **Pero el producto no se queda sin plan: Digistore24 le asigna uno por
defecto.** (Unos 27 €, pago único, visto en una cuenta real en septiembre de
2026; mira el producto del vendedor en lugar de fiarte de esa cifra. Lo que no
cambia es que *algún* plan hay.) Tu app nunca lo cobra: el plan que viaja con
la llamada a `createBuyUrl` gana siempre al guardado. Lo que *sí* lo cobra es el
**formulario de pedido propio** del producto, que existe desde que existe el
producto — y que, tras la aprobación del marketplace (**`ds24-golive`**), es
algo que encuentran desconocidos.

De ahí se siguen dos cosas, y ambas son fáciles de pasar por alto:

- **Avisa al vendedor antes de que abra su backoffice.** Verá un precio que
  nunca fijó, junto a un producto que su app vende por otro importe. Si se lo
  dices antes, es una curiosidad; si lo descubre por su cuenta, parece un
  fallo, y el arreglo que se le ocurrirá es una segunda lista de precios.
- **Decide de forma deliberada qué hace tu handler de IPN con una compra hecha
  por esa vía.** No lleva `tracking[custom]`, su precio es el del plan del
  producto y no el tuyo, y si tu oferta es una **suscripción** enviará
  exactamente un evento de pago — ninguna renovación, ninguna cancelación, así
  que nada de lo que cuelgues de esos eventos se disparará nunca para ella. El
  Paso 2 de **`ds24-checkout`** explica por qué la ausencia de `custom`, por sí
  sola, no basta para saber que ha pasado esto.

**Si tu app habla más de un idioma, la entrada guarda un id de producto por
idioma**, no uno solo. El motivo está en el Paso 3; acierta con la forma ya
aquí, porque cambiarla después de la primera venta significa productos nuevos
y aprobaciones nuevas:

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
dales sus propios ids en el registro, márcalos de forma visible en el nombre
del producto (`"Pro [DEV]"` — la API de Digistore24 no tiene campo de etiqueta;
el nombre es lo que una persona ve en el backoffice) y deja limpios los nombres
de los productos en producción. Un conjunto nunca debe apropiarse de los
productos del otro — véase la nota sobre idempotencia más abajo. Un vendedor
que solo sincroniza contra el dominio en producción tiene un único conjunto, y
eso está bien.

## Paso 3 — crea los productos

`createProduct` / `updateProduct` con el nombre, la descripción y **`language`**.
Escribe el id de producto que devuelve en tu lista de precios, para que la
correspondencia quede registrada y no haya que deducirla otra vez.

### Un producto por oferta Y por idioma — aquí es donde la gente se equivoca

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

**Borrar un producto de tu lista no lo retira de la venta.** Un producto que
Digistore24 ya conoce se puede seguir comprando hasta que el usuario lo
desactive allí. Dilo en voz alta cuando quites uno.

🚨 **Lo que significa que el momento de preguntar es ANTES de crear, no
después.** No existe ninguna llamada de la API que deshaga un `createProduct`.
Una vez ejecutada la sincronización, cada entrada que encontró es un producto
real en la cuenta del usuario, y quitar uno es trabajo manual en el backoffice
de Digistore24 — producto por producto, idioma por idioma. Una lista de precios
que todavía arrastre las entradas que esbozaste mientras diseñabas la oferta
las publicará todas.

Así que la primera vez que tu sincronización vaya a crear algo: **imprime lo
que se crearía, con su nombre, avisa al usuario de que no se puede deshacer y
espera un sí.** Después crea. Las ejecuciones siguientes ya tienen los ids
guardados y no crean nada, así que es una pregunta en un único momento, no un
aviso que la gente aprende a cerrar sin leer. Si algunas entradas son
borradores y no ofertas, dale a tu lista un marcador que las deje fuera de la
sincronización, en vez de pedirle al usuario que borre un texto que todavía
quiere.

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
- **`ds24-checkout`** — el enlace de compra, con el precio incorporado.
- **`ds24-golive`** — la compra de prueba que demuestra la cadena entera.

Di cuál vas a empezar y empiézala.
