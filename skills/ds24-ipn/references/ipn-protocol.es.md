<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Español** · Original en inglés — [`ipn-protocol.md`](ipn-protocol.md) · [Français](ipn-protocol.fr.md)

# La firma IPN de Digistore24

Digistore24 le cuenta a tu app que se ha movido dinero haciendo POST de un
payload codificado como formulario a un endpoint que has registrado. Ese
endpoint está en la internet pública, así que cualquiera puede hacer POST contra
él. **La firma es lo único que separa un pago real de alguien que escribe una
URL en `curl`.** Todo lo demás en este documento se deriva de esa única frase.

Implementación de referencia que publica Digistore24:
<https://www.digistore24.com/download/ipn/examples/ipn/sha_sign.php>

## El algoritmo — seis pasos

Dados los parámetros enviados por POST y tu **passphrase IPN**:

1. **Elimina `sha_sign` y `SHASIGN`** del conjunto de parámetros (compara los
   nombres sin distinguir mayúsculas de minúsculas). Llevan la firma misma y no
   formaban parte de lo que se firmó.
2. **Ordena las claves restantes como cadenas de bytes.** Esto es el
   `ksort($params, SORT_STRING)` de PHP — una comparación simple en orden de
   bytes, *no* una que tenga en cuenta la configuración regional ni que ignore
   mayúsculas y minúsculas. En JavaScript eso es
   `a < b ? -1 : a > b ? 1 : 0`, no `a.localeCompare(b)`.
3. **Omite los valores vacíos.** `undefined`, `null` y `""` no aportan nada — ni
   siquiera su clave. Un campo que llegó vacío debe tratarse como si no hubiera
   llegado en absoluto.
4. **Concatena**, para cada parámetro restante en el orden ordenado:
   `KEY` + `=` + `VALUE` + `PASSPHRASE`. La passphrase va después de *cada*
   par, no una sola vez al final.
5. **Aplica SHA512 al resultado**, codificado en UTF-8, y represéntalo en
   **hexadecimal en mayúsculas**.
6. **Compara** con el `sha_sign` recibido **sin distinguir mayúsculas de
   minúsculas** y en **tiempo constante**.

Ejemplo resuelto — parámetros `{order_id: "ORD-1", product_id: "42"}` con la
passphrase `s3cret-passphrase`. Ordenados, eso es `order_id` antes que
`product_id`, así que la cadena que se hashea es:

```
order_id=ORD-1s3cret-passphraseproduct_id=42s3cret-passphrase
```

No hay separadores entre los pares. La passphrase es lo que termina cada uno.

## La trampa que le cuesta un día a todo el mundo

**Digistore24 firma con los nombres de campo ORIGINALES — `order_id=…`, no
`ORDER_ID=…`.**

El ejemplo oficial en PHP lleva un interruptor `convert_keys_to_uppercase`, y
leerlo de arriba abajo sugiere que pasar a mayúsculas es lo normal. Observado
contra cuentas reales de Digistore24 no lo es: los nombres de campo se firman
exactamente como se enviaron. Una implementación que pasa a mayúsculas sin
condiciones calcula una firma perfectamente válida sobre la entrada equivocada y
rechaza **todas las IPN reales** con «firma inválida» — mientras todas tus
propias pruebas pasan, porque firman de la misma manera equivocada con la que
verifican.

**Así que verifica contra ambas convenciones.** Calcula la firma con las
mayúsculas y minúsculas originales, y si eso no coincide, calcúlala de nuevo con
las claves en mayúsculas. Aceptar cualquiera de las dos no cuesta nada en
seguridad — ambas variantes requieren la passphrase secreta — y le ahorra al
operador tener que acertar con un ajuste de su cuenta de Digistore24 que no
puede ver.

## Falla en cerrado, siempre

Estos no son casos límite. Son la forma que tiene un ataque:

| Situación | Respuesta correcta |
|---|---|
| Ningún `sha_sign` en el payload | **rechazar** |
| Ninguna passphrase configurada de tu lado | **rechazar** |
| Firma presente pero que no coincide | **rechazar** |
| Un evento desconocido con una firma válida | acepta la petición, no cambies nada |

La segunda fila es la que se implementa mal. «Si no hay passphrase configurada,
sáltate la comprobación» es una línea de código que suena razonable y que
convierte tu webhook de pagos en un endpoint de escritura público en el momento
en que una variable de entorno desaparece en un redespliegue. **Una passphrase
ausente es un rechazo, no un atajo.**

## Leer el cuerpo

La firma cubre los bytes que se enviaron. Cualquier cosa que los reescriba la
rompe:

- **Lee el cuerpo en bruto** y luego parséalo tú mismo. Un framework que
  parsea, vuelve a serializar y te entrega un objeto puede haber reordenado o
  recodificado algo.
- **Digistore24 envía `application/x-www-form-urlencoded`**, no JSON.
- **No recortes, no pases a minúsculas ni normalices los valores** antes de
  firmar. Decodifica el porcentaje exactamente una vez, como hace el parseo de
  formularios, y déjalo ahí.
- **UTF-8.** Los nombres de los compradores llevan diéresis, acentos y
  escrituras no latinas. Hashea los bytes UTF-8; un lenguaje que por defecto
  recae en Latin-1 (instalaciones antiguas de Python, algunas configuraciones de
  PHP) producirá en silencio un hash distinto para `Jörg Müller` que el del
  mismo nombre que firmó Digistore24.

## Responder

- **Responde `200` con un cuerpo corto** una vez que hayas procesado el evento.
  Digistore24 **reintenta hasta que recibe un 200**, así que una excepción no
  capturada se convierte en un bucle de reenvío sin fin.
- **Responde a la prueba de conexión.** Digistore24 valida el endpoint cuando lo
  registras, y lo hace con un `GET`. Devuelve `200 OK` para un GET, y para un
  POST cuyo evento sea `connection_test`.
- **Nunca redirijas.** Un `301`/`302` desde tu endpoint IPN falla la validación —
  Digistore24 quiere el endpoint mismo, no un salto.
- **La URL tiene que ser `https` pública.** Digistore24 rechaza `http` y rechaza
  `localhost` de plano. En una plataforma alojada de construcción con IA tu URL
  de vista previa ya es https pública, que es lo único que allí es *más fácil*
  que en un portátil.

## La entrega no tiene orden ni límite

Dos propiedades del transporte a las que tu handler tiene que sobrevivir,
porque ninguna de las dos aparece en las pruebas:

- **Los eventos llegan desordenados.** Un `on_payment` reenviado puede aterrizar
  *después* del `on_refund` que terminó el acceso. Así que la decisión «¿puede
  esta persona usar el producto?» debe tomarse a partir del **estado**, no de
  una marca de tiempo ni del orden de llegada. Una vez que el acceso ha
  terminado, ningún evento posterior puede reabrirlo.
- **El mismo evento llega más de una vez.** Digistore24 reintenta hasta que
  recibe un 200, y un timeout de tu lado después de que el trabajo ya estaba
  hecho sigue contando como un fallo. **Cada escritura que haga tu handler
  tiene que ser idempotente**, con una clave sacada del payload — `order_id` más
  el nombre del evento. `order_id` es el identificador que Digistore24
  garantiza, y está documentado como *«ID único del pedido. Varias
  transacciones del mismo pedido tienen el mismo order-ID»*: el pago, su
  reembolso, un contracargo y cada recobro periódico de una suscripción llegan
  todos llevando ese mismo valor. Eso es lo que lo convierte a la vez en la
  clave de idempotencia y en la clave bajo la que se guarda el acceso mismo — un
  reembolso solo puede revocar lo que un pago concedió si los dos coinciden en
  el identificador. Acreditar un saldo de tokens sin una clave así reparte los
  créditos dos veces.

  ⚠️ **Una IPN no lleva `purchase_id`.** No aparece en ninguna tabla publicada
  de parámetros de IPN, y el mensaje real en `../scripts/vectors.json`
  (`captured-on-payment`, 173 parámetros) no lo contiene. El nombre pertenece a
  la **API** de Digistore24, donde `getPurchase` documenta su `purchase_id` como
  «el order id de Digistore24» — el mismo valor con otro nombre. Indexa tus
  escrituras con él y las estarás indexando sobre `undefined` en todos los
  mensajes que lleguen jamás: o bien todos los pedidos colapsan sobre una única
  clave, o bien nada coincide nunca y el reintento hace el trabajo una segunda
  vez. Ambos fallos parecen un endpoint que funciona hasta que se mueve dinero
  de verdad.

## Qué guardar

Guarda el **payload en bruto** de cada IPN que aceptes, literal, antes de actuar
sobre ella. No cuesta casi nada y es la única forma de responder «¿de verdad
envió eso Digistore24?» semanas después, cuando un cliente discute su acceso y
todas tus tablas derivadas coinciden entre sí.

Guarda el **nombre del evento** tal como llegó. No colapses los eventos en un
estado y luego tomes decisiones a partir del estado — mira `events.es.md`, donde
dos eventos que significan lo contrario sobre el acceso se asignan a la misma
palabra.
