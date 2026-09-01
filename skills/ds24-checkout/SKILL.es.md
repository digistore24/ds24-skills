---
name: ds24-checkout
language: es
description: Úsala cuando construyas el botón de compra, la página de precios o el enlace de checkout de un producto de Digistore24 — crear una URL de compra firmada con createBuyUrl, adjuntar el precio como plan de pago, llevar la identidad del comprador hasta la IPN, y la página de agradecimiento. Úsala siempre que el usuario mencione un enlace de compra, un checkout, una página de precios, «¿cómo paga el cliente?» o una compra que llega sin que nadie pueda decir de quién era.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# El enlace de checkout

Un checkout de Digistore24 es una **URL firmada y de vida corta** que creas a
través de la API y a la que envías al comprador. No es un enlace estático con un
id de producto dentro.

## Paso 0 — ¿ya existe?

Busca en el proyecto `createBuyUrl`, `payment_plan` o una página de precios que
ya enlace hacia fuera con Digistore24. Si existe, no la reconstruyas —
contrástala con el Paso 3 y el Paso 4 y corrige solo lo que esté mal.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compárala con el `VERSION` de este pack. Menciona cualquier discrepancia en una
frase y sigue adelante.

## Paso 1 — la llamada

```
POST https://www.digistore24.com/api/call/createBuyUrl/format/json
Header: X-DS-API-KEY: <la clave>
```

Cuerpo (form-encoded), las partes que importan:

```
product_id                              = 512345
valid_until                             = 24h
payment_plan[first_amount]              = 47.00
payment_plan[other_amounts]             = 47.00
payment_plan[currency]                  = EUR
payment_plan[number_of_installments]    = 0        # 0 = suscripción abierta, 1 = pago único
payment_plan[first_billing_interval]    = 1_month  # omitir por completo para un pago único
payment_plan[other_billing_intervals]   = 1_month
```

**El precio se envía aquí, en el momento de la compra — no se guarda en el
producto.** Digistore24 descarta `data[amount]` en el propio producto, y un plan
de pago guardado no puede llevar un cupón, una prueba gratuita, un upgrade ni una
comisión de afiliado por enlace. Lee los números de la única lista de precios de
tu proyecto (ver **`ds24-products`**).

**`product_id` decide también el IDIOMA del formulario de pedido — elígelo por
el idioma del comprador.** Un producto de Digistore24 lleva exactamente un
idioma, y no hay ningún parámetro en esta llamada que lo anule (mira el cuerpo de
arriba: `buyer`, `payment_plan`, `tracking`, `urls`, `placeholders`, `settings`,
`addons` — ningún idioma por ninguna parte). Así que una app multilingüe mantiene
**un id de producto por idioma** y por oferta, y resuelve el idioma del visitante
a uno de ellos justo aquí, antes de la llamada. Si mandas a todo el mundo al
mismo id, la mitad rellena un formulario en el idioma equivocado en el momento en
que se les pide pagar. La skill **`ds24-products`** tiene la forma de la lista de
precios y la regla al completo.

La respuesta es una URL. **Guárdala en caché por oferta** — es válida durante la
ventana de `valid_until`, y crear una nueva en cada visita a la página es un
viaje de ida y vuelta a Digistore24 en el camino de tu página de precios.

⚠️ **Entonces la clave de caché tiene que incluir el idioma**, no solo la clave
de la oferta. Una fila por clave significa que la URL alemana y la inglesa se
expulsan mutuamente en cada visita a la página y, entre medias, la caché sirve la
página de checkout de un idioma al comprador del otro. `"<offerKey>:<language>"`
basta.

🚨 **Y nunca guardes en caché una URL que lleve la identidad de un comprador.**
El Paso 2 mete el id del miembro con sesión iniciada en `tracking[custom]`, y una
caché indexada por la oferta no tiene dimensión de miembro — así que la identidad
del primer comprador con sesión se sirve a todo el que abra esa página después, y
cada uno de *sus* pagos llega atribuido a ese primer miembro. Nada falla mientras
ocurre: la página se renderiza, el checkout se abre, el dinero se mueve.

Así que una página de precios tiene **dos caminos**, y no son dos versiones de
uno solo:

- **Sin sesión → la URL compartida en caché.** Sin identidad dentro, segura para
  todos, sin viaje de ida y vuelta a Digistore24 mientras la página se renderiza.
- **Con sesión → una URL construida en el momento del clic**, con la identidad de
  ese miembro dentro, usada una vez y **nunca escrita en la caché**.

Decide cuál por el **contenido** del campo de tracking, no por si está puesto o
no: un marcador que nombra el *paquete* se puede compartir, uno que nombra a una
*persona* no. Preguntar meramente «¿está puesto el tracking?» convierte cada
tarjeta en una llamada a la API en vivo en cada visita a la página, que es
justamente lo que la caché existía para evitar.

**Y cuando la llamada falla, la página tiene que renderizarse igual.** Que
Digistore24 vaya lento o que una clave sea incorrecta debe producir un botón
desactivado con un motivo («checkout no disponible»), nunca un error lanzado en
la página de precios y nunca un enlace muerto. Devuelve el fallo a quien llama en
lugar de lanzarlo.

**Esa URL todavía no está terminada en un entorno de desarrollo.** Hasta que el
producto tenga la aprobación del marketplace nadie puede comprar a través de ella
en absoluto, y la manera de desbloquear una compra de prueba sin tocar tu
navegador es añadir el parámetro testpay — **añadido al valor devuelto, después
de la caché, y solo allí donde un cliente nunca pueda llegar**. No construyas el
checkout y dejes esto para más tarde: es el paso que decide si puedes demostrar
que el resto funciona. El **Paso 4a** es la receta y las barreras.

## Paso 2 — lleva la identidad del comprador hasta el final

El fallo más común con diferencia en una integración de Digistore24 es un pago
que llega y no se puede emparejar con ninguna cuenta. Alguien ha pagado, la app
no tiene ni idea de quién, y el soporte tiene que hacerlo a mano.

Envía un identificador en el campo de tracking. Digistore24 lo guarda en la
compra y lo devuelve en **cada** evento posterior de ese pedido — la renovación
un año después, el reembolso, el contracargo. Llega en la IPN como `custom`:

```
tracking[custom] = m:<id de miembro>;t:<un token aleatorio corto guardado en ese miembro>
```

**El campo es una única cadena opaca que es enteramente tuya**, así que dale una
estructura que puedas ampliar: pares `key:value` separados por `;`, y un lector
que **ignore las claves que no conoce** en lugar de fallar con ellas. Más
adelante vas a querer llevar un segundo id (qué paquete, qué tipo de compra, una
intención que el comprador expresó en el checkout), y para entonces habrá compras
en vivo que sostienen el valor antiguo. Un id nuevo es entonces un par nuevo; un
segundo *formato* es una migración que no puedes hacer, porque los valores que ya
están en Digistore24 no se pueden reescribir.

**Dos cosas sobre ese token.** Corrobora el id de miembro, de modo que un id
adivinado o editado por sí solo nunca reclama la compra de otra persona — y **no
es una credencial**: nunca autentica una sesión, solo dice «este id no se lo
inventó la persona que escribe la URL». Ambas mitades deben estar presentes y
bien formadas o el valor no nombra a nadie: media identidad no es una identidad
más débil.

En el otro extremo, en el handler de la IPN, atribuye en este orden — y el
orden es una regla de seguridad, no una preferencia:

1. **El identificador de `custom`, con el token coincidiendo → autenticado.** Tu
   app escribió este valor, Digistore24 lo guardó en el servidor, y el comprador
   nunca tuvo una copia que pudiera editar.
2. **Si no, el correo del comprador contra tus cuentas → no autenticado.** Esa
   dirección la escribió en un formulario de Digistore24 quien estuviera pagando,
   y **Digistore24 no verifica que la controle**. Suele ser correcta y nunca es
   una prueba.
3. **Si no, guarda el pedido sin atribuir** y engánchalo cuando esa dirección
   inicie sesión por primera vez.

Dos negativas son lo que hace que el paso 2 sea seguro de tener siquiera:

- 🚨 **Una dirección que coincide con más de una cuenta se rechaza, no se
  resuelve a la primera fila.** Pide como mucho dos coincidencias y trata «dos»
  como *no se puede saber*. La consulta que devuelve una lista y coge `[0]` es la
  forma exacta de este bug, y lo que hace es entregarle a un cliente la compra de
  otro cliente. Sin atribuir es el resultado correcto; adivinar no es un plan de
  reserva.
- **La atribución solo concede — nunca mueve y nunca revoca.** Una coincidencia
  de correo puede enganchar un pedido que todavía no es de nadie. No puede
  reapuntar un pedido que ya está atribuido, y ningún fallo de atribución puede
  terminar un acceso que existe. Esa unidireccionalidad es toda la razón por la
  que un camino no autenticado es tolerable.

Y cualquier cosa que autorice un acto **desatendido** más tarde — cobrar a un
medio de pago guardado, activar una recarga automática (**`ds24-tokens`**) —
acepta solo el camino 1. Una coincidencia del camino 2 es una buena suposición
sobre quién compró algo; no es permiso para cobrar a una tarjeta.

Un pedido sin atribuir es un ticket de soporte. Uno mal atribuido es un cliente
mirando la compra de otra persona, y es el más caro de los dos.

## Paso 3 — una compra sin cuenta también tiene que funcionar

Deja que la gente compre desde la página de precios pública sin iniciar sesión
antes. Así es como llega la mayoría, y forzar una cuenta antes del pago cuesta
ventas. El camino 3 de arriba es lo que lo hace seguro: el pedido espera, y el
primer inicio de sesión desde esa dirección lo reclama.

## Paso 4 — la página de agradecimiento

Digistore24 envía al comprador a una URL tuya después del pago, con el id del
pedido dentro. Dos reglas:

- **Es pública.** El comprador todavía no tiene sesión. No pongas detrás nada que
  dé por hecho que la hay.
- **No concedas acceso desde ella.** Es un navegador llamando a una URL —
  cualquiera puede llamarla. El acceso viene de la IPN, que está firmada. La
  página de agradecimiento dice «gracias, está en camino / así es como inicias
  sesión», nada más.

**Digistore24 solo guarda URLs https públicas.** Una URL de agradecimiento en
`localhost` se rechaza de plano («Please only use secure URLs with https://»). En
una plataforma alojada la URL de tu app ya es pública, así que esto no es
problema; en un portátil hace falta un redirector público o un túnel.

## Paso 4a — pagos de prueba sin aprobación (la clave testpay)

Un producto que todavía no tiene la aprobación del marketplace solo se puede
comprar como **compra de prueba**. Hay dos maneras de desbloquear una, y sirven
para sitios distintos:

- **La cookie de compra de prueba** — se pone una vez en el navegador del
  vendedor (el centro de ayuda de Digistore24 tiene el enlace). Por navegador,
  caduca. La herramienta adecuada en cualquier dominio al que también pueda
  llegar un cliente.
- **El parámetro testpay** — se obtiene vía la API y se añade a la URL de compra,
  de modo que el desbloqueo viaja con el enlace en lugar de vivir en un
  navegador:

  ```
  POST https://www.digistore24.com/api/call/getTestpayKey/format/json
  Header: X-DS-API-KEY: <la clave>
  ```

  Sin documentar, pero real. La respuesta lleva `testpay_key`,
  `get_param_name` y `expires_at`. Añade
  `?<get_param_name>=<testpay_key>` a la URL de compra (el NOMBRE viene de la
  respuesta — nunca lo escribas fijo en el código) y el checkout se abre en modo
  de pago de prueba, esté aprobado o no. Enviar `do_recreate=1` rota la clave: se
  emite una nueva y todas las copias antiguas dejan de funcionar.

Cuatro barreras, todas ellas indispensables:

- **Solo desarrollo/preview — nunca en una URL a la que pueda llegar un
  cliente.** Un checkout que lleve este parámetro acepta «pagos» de prueba: quien
  lo pulse se lleva el producto gratis. Restríngelo por entorno con una lista de
  permitidos (todo lo que no sea claramente desarrollo cuenta como producción y
  se niega), y añádelo en el momento de renderizar o de hacer clic.
- **Nunca dentro de una URL de compra cacheada o compartida.** Si las URLs de
  compra se cachean (Paso 1), cachea la URL limpia y añade el parámetro después
  de la caché — una URL decorada en una caché compartida se sirve a todo el
  mundo.
- **La clave es a nivel de cuenta — trátala como un secreto.** Funciona en TODAS
  las URLs de checkout de esta cuenta de vendedor, incluidas las que están en
  vivo. Mantenla fuera del repo y fuera de la configuración desplegada.
- **Rótala antes de la puesta en producción** (`do_recreate=1`) — ver
  **`ds24-golive`**.

## Paso 5 — demuéstralo

1. Crea una URL de compra y ábrela. La página de checkout tiene que mostrar
   **tu** precio, moneda e intervalo — si muestra otra cosa, el plan de pago no
   ha viajado.
2. Haz una **compra de prueba** — con la cookie de compra de prueba de
   Digistore24 puesta, o en un entorno de desarrollo con el parámetro testpay
   añadido (Paso 4a).
3. Comprueba que la IPN ha llegado y que el pedido ha salido **atribuido a la
   cuenta correcta**. La atribución es la parte que parece bien hasta que deja de
   estarlo. Las compras de prueba llegan con `api_mode=test` en el payload de la
   IPN — procésalas como las de en vivo (ese camino idéntico es lo que demuestra
   la prueba).

## Paso 6 — qué viene después

- **`ds24-ipn`** — el endpoint que recibe lo que produce este checkout.
- **`ds24-entitlements`** — convertir un pedido pagado en «puede usar el
  producto».
- **`ds24-tokens`** — si vendes créditos de prepago en lugar de planes.
- **`ds24-golive`** — la compra de prueba real, de principio a fin.

Di cuál vas a empezar y empiézala.
