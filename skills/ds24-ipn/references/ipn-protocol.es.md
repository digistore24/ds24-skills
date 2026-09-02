<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Español** · Original en inglés — [`ipn-protocol.md`](ipn-protocol.md) · [Français](ipn-protocol.fr.md)

# La firma IPN de Digistore24

Cuando se mueve dinero, Digistore24 se lo comunica a tu app con un POST: un
payload codificado como formulario, enviado al endpoint que registraste. Ese
endpoint está en la internet pública, así que cualquiera puede hacerle un POST.
**La firma es lo único que distingue un pago real de alguien que teclea una URL
en `curl`.** Todo lo demás en este documento se deduce de esa única frase.

Implementación de referencia publicada por Digistore24:
<https://www.digistore24.com/download/ipn/examples/ipn/sha_sign.php>

## El algoritmo — seis pasos

A partir de los parámetros recibidos por POST y de tu **passphrase de la IPN**:

1. **Quita `sha_sign` y `SHASIGN`** del conjunto de parámetros (compara los
   nombres sin distinguir mayúsculas de minúsculas). Contienen la firma en sí y
   no formaban parte de lo que se firmó.
2. **Ordena las claves restantes como cadenas de bytes.** Es el
   `ksort($params, SORT_STRING)` de PHP: una comparación byte a byte, *no* una
   que dependa de la configuración regional ni que ignore mayúsculas y
   minúsculas. En JavaScript, eso es `a < b ? -1 : a > b ? 1 : 0`, no
   `a.localeCompare(b)`.
3. **Omite los valores vacíos.** `undefined`, `null` y `""` no aportan nada, ni
   siquiera su clave. Un campo que llegó vacío se trata como si no hubiera
   llegado.
4. **Concatena**, para cada parámetro que queda y en el orden del paso 2:
   `KEY` + `=` + `VALUE` + `PASSPHRASE`. La passphrase va detrás de *cada*
   par, no una sola vez al final.
5. **Calcula el SHA512 del resultado**, codificado en UTF-8, y escríbelo en
   **hexadecimal en mayúsculas**.
6. **Compáralo** con el `sha_sign` recibido, **sin distinguir mayúsculas de
   minúsculas** y en **tiempo constante**.

Ejemplo resuelto: los parámetros `{order_id: "ORD-1", product_id: "42"}` con la
passphrase `s3cret-passphrase`. Ordenados, `order_id` va antes que `product_id`,
así que la cadena que se hashea es:

```
order_id=ORD-1s3cret-passphraseproduct_id=42s3cret-passphrase
```

Entre los pares no hay ningún separador: lo que cierra cada par es la
passphrase.

## La trampa que le cuesta un día a todo el mundo

**Digistore24 firma con los nombres de campo ORIGINALES: `order_id=…`, no
`ORDER_ID=…`.**

El ejemplo oficial en PHP incluye un interruptor `convert_keys_to_uppercase`, y
leído de arriba abajo da a entender que lo normal es pasar las claves a
mayúsculas. Contrastado con cuentas reales de Digistore24, no lo es: los
nombres de campo se firman tal como se enviaron. Una implementación que
convierte a mayúsculas sin condiciones calcula una firma impecable sobre la
entrada equivocada y rechaza **todas las IPN reales** con «firma inválida» —
mientras tus propias pruebas pasan todas, porque firman con el mismo error con
el que verifican.

**Por eso, verifica con las dos convenciones.** Calcula la firma con las
mayúsculas y minúsculas originales y, si no coincide, vuelve a calcularla con
las claves en mayúsculas. Aceptar cualquiera de las dos no resta nada de
seguridad — ambas variantes exigen la passphrase secreta — y le ahorra al
operador tener que reproducir un ajuste de su cuenta de Digistore24 que ni
siquiera puede ver.

## Rechazar por defecto, siempre

Esto no son casos límite: es la forma que toma un ataque, y ante todos ellos se
falla del lado seguro (*fail closed*):

| Situación | Respuesta correcta |
|---|---|
| El payload no trae `sha_sign` | **rechazar** |
| No hay passphrase configurada de tu lado | **rechazar** |
| Hay firma, pero no coincide | **rechazar** |
| Un evento desconocido con firma válida | aceptar la petición, no cambiar nada |

La segunda fila es la que se implementa mal. «Si no hay passphrase configurada,
sáltate la comprobación» suena a línea de código razonable, y convierte tu
webhook de pagos en un endpoint público de escritura en cuanto una variable de
entorno se pierde en un redespliegue. **Una passphrase ausente es un rechazo, no
un atajo.**

## Leer el cuerpo

La firma cubre los bytes tal como se enviaron. Todo lo que los reescriba la
rompe:

- **Lee el cuerpo en bruto** y pársealo tú mismo. Un framework que parsea,
  vuelve a serializar y te entrega un objeto puede haber reordenado o
  recodificado algo por el camino.
- **Digistore24 envía `application/x-www-form-urlencoded`**, no JSON.
- **No recortes, no pases a minúsculas ni normalices los valores** antes de
  firmar. Resuelve la codificación por porcentaje (*percent-decoding*)
  exactamente una vez, igual que hace el parseo de un formulario, y no toques
  nada más.
- **UTF-8.** Los nombres de los compradores traen diéresis, acentos y alfabetos
  no latinos. Hashea los bytes en UTF-8: un lenguaje que por defecto trabaje en
  Latin-1 (instalaciones antiguas de Python, algunas configuraciones de PHP)
  producirá sin avisar un hash distinto para `Jörg Müller` del que firmó
  Digistore24 para ese mismo nombre.

## Responder

- **Responde `200` con un cuerpo breve** una vez procesado el evento.
  Digistore24 **reintenta hasta recibir un 200**, así que una excepción sin
  capturar se convierte en un bucle de reenvío sin fin.
- **Responde a la prueba de conexión.** Digistore24 valida el endpoint cuando lo
  registras, y lo hace con un `GET`. Devuelve `200 OK` a un GET, y también a un
  POST cuyo evento sea `connection_test`.
- **Nunca redirijas.** Un `301` o un `302` desde tu endpoint IPN no supera la
  validación: Digistore24 quiere el endpoint en sí, no un salto intermedio.
- **La URL tiene que ser `https` y pública.** Digistore24 rechaza `http` y
  rechaza `localhost` de plano. En una plataforma de desarrollo con IA alojada
  en la nube, tu URL de vista previa ya es https pública — la única cosa que
  ahí resulta *más fácil* que en un portátil.

## La entrega no tiene orden ni límite

Dos propiedades del transporte que tu handler tiene que aguantar, porque
ninguna de las dos aparece en las pruebas:

- **Los eventos llegan desordenados.** Un `on_payment` reenviado puede aterrizar
  *después* del `on_refund` que terminó el acceso. Por eso la decisión «¿puede
  esta persona usar el producto?» se toma a partir del **estado**, no de una
  marca de tiempo ni del orden de llegada. Una vez terminado el acceso, ningún
  evento posterior puede reabrirlo.
- **El mismo evento llega más de una vez.** Digistore24 reintenta hasta recibir
  un 200, y un timeout de tu lado, aunque el trabajo ya estuviera hecho, cuenta
  como fallo. **Cada escritura de tu handler tiene que ser idempotente**, con
  una clave tomada del payload: `order_id` más el nombre del evento. `order_id`
  es el identificador que Digistore24 garantiza, y su documentación lo describe
  como *«ID único del pedido. Varias transacciones del mismo pedido llevan el
  mismo order-ID»*: el pago, su reembolso, un contracargo y cada recobro
  periódico de una misma suscripción llegan todos con ese mismo valor. Eso es lo
  que lo convierte, a la vez, en clave de idempotencia y en la clave bajo la que
  se guarda el propio acceso — un reembolso solo puede revocar lo que un pago
  concedió si los dos coinciden en el identificador. Abonar créditos en un saldo
  de tokens sin una clave así los reparte dos veces.

  ⚠️ **Una IPN no lleva `purchase_id`.** No figura en ninguna tabla publicada de
  parámetros de IPN, y el mensaje real de `../scripts/vectors.json`
  (`captured-on-payment`, 173 parámetros) no lo contiene. El nombre pertenece a
  la **API** de Digistore24, donde `getPurchase` documenta su `purchase_id` como
  «el id de pedido de Digistore24»: el mismo valor con otro nombre. Si indexas
  tus escrituras por él, las estás indexando por `undefined` en todos los
  mensajes que lleguen jamás: o todos los pedidos caen en la misma clave, o
  nunca coincide nada y el reintento repite el trabajo. Los dos fallos parecen
  un endpoint que funciona hasta que se mueve dinero de verdad.

## Qué guardar

Guarda el **payload en bruto** de cada IPN que aceptes, literal, antes de actuar
sobre ella. No cuesta casi nada y es la única manera de responder «¿de verdad
envió esto Digistore24?» semanas después, cuando un cliente discuta su acceso y
todas tus tablas derivadas coincidan entre sí.

Guarda el **nombre del evento** tal como llegó. No fundas los eventos en un
estado para luego decidir a partir del estado: en `events.es.md` verás dos
eventos que dicen lo contrario sobre el acceso y acaban asignados a la misma
palabra.
