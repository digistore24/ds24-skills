---
name: ds24-checkout
language: es
description: Úsala para construir el botón de compra, la página de precios o el enlace de checkout de un producto de Digistore24 — crear una URL de compra firmada con createBuyUrl, seleccionar el plan de pago del producto para la forma de pago elegida, llevar la identidad del comprador hasta la IPN y montar la página de agradecimiento. Úsala siempre que el usuario mencione un enlace de compra, el checkout, una página de precios, «¿cómo paga el cliente?» o una compra que llega sin que nadie sepa de quién es.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# El enlace de checkout

Un checkout de Digistore24 es una **URL firmada y de corta duración**: la creas
a través de la API y envías al comprador a ella. No es un enlace estático con el
id del producto incrustado.

## Paso 0 — ¿ya existe?

Busca en el proyecto `createBuyUrl`, `payment_plan` o una página de precios que
ya enlace a Digistore24. Si lo encuentras, no lo reconstruyas: compáralo con el
Paso 3 y el Paso 4 y corrige solo lo que esté mal.

## Paso 0a — ¿está al día esta copia del Skill Pack?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compárala con el `VERSION` de este pack. Si no coinciden, dilo en una frase y
sigue adelante.

## Paso 1 — la llamada

```
POST https://www.digistore24.com/api/call/createBuyUrl/format/json
Header: X-DS-API-KEY: <la clave>
```

Cuerpo (form-encoded), y hay **dos formas** — cuál de las dos envías depende de
si el precio de la oferta ya es un plan de pago sobre el producto (el Paso 3 de
**`ds24-products`** los escribe, uno por cada forma de pago).

**El caso normal — vender a través del plan guardado:**

```
product_id                = 512345
valid_until               = 24h
settings[plan]            = 991      # el plan de pago de ESTA forma de pago
settings[hide_plans]      = Y        # el comprador ya eligió en tu página
payment_plan[template]    = 991      # ver más abajo: se envía para VALIDAR, no para poner precio
```

Ningún importe. Es el plan guardado el que pone el precio de la venta, y eso es
lo que hace que tu página, el formulario de pedido propio del producto y el
cambio de intervalo que hace el propio comprador cobren la misma cifra.

**La excepción — poner el precio aquí.** Tres casos, y solo estos:

| | |
|---|---|
| un **upgrade o un downgrade** (`payment_plan[upgrade_order_id]`) | el precio existe para este comprador frente a una compra que ya tiene; no hay dónde guardarlo |
| una **prueba gratuita** (`payment_plan[test_interval]`) | la misma forma |
| un plan guardado que **ya no coincide con tu lista de precios** | alguien editó un precio y no sincronizó — envía tu propia cifra, no una caducada |

```
product_id                              = 512345
valid_until                             = 24h
payment_plan[first_amount]              = 47.00
payment_plan[other_amounts]             = 47.00
payment_plan[currency]                  = EUR
payment_plan[number_of_installments]    = 0        # 0 = suscripción indefinida, 1 = pago único
payment_plan[first_billing_interval]    = 1_month  # omitir por completo para un pago único
payment_plan[other_billing_intervals]   = 1_month
```

🚨 **Por qué `payment_plan[template]` viaja con la primera forma.** Digistore24
rechaza una plantilla que no pertenezca al producto para el que se la ha
llamado, con `payment_plan_not_found` — así que un id de plan que sobrevivió a
un borrado al otro lado falla **en voz alta**, mientras que `settings[plan]` por
sí solo se ignoraría en silencio y el comprador se encontraría con el plan que el
producto tenga. Como no la acompaña ningún `first_amount`, sus valores se
descartan luego y es el plan guardado el que pone el precio, que es justo lo que
quieres.

⚠️ **Y ese error se lleva por delante la oferta entera, no una sola tarjeta de
precio**: el plan pertenece al producto, así que fallan de golpe todos los
botones de compra de esa oferta, en todos los idiomas. Captura exactamente ese
error, reintenta una vez sin la plantilla y sin su `settings[plan]`, y deja
registrado qué plan y qué producto eran. La venta sale adelante a tu propio
precio; al vendedor se le dice que vuelva a lanzar la sincronización.

🚨 **NO busques `payment_plan_not_found`.** Ese nombre es como se llama el error
dentro de Digistore24: es una CLAVE de mensaje, traducida antes de salir del
servidor. Medido contra una cuenta real el 2026-09-09, lo que llega es un HTTP
404 con prosa en alemán:

```
"Ungültige Bezahlplan-ID: 999999999 - Bezahlplan nicht vorhanden
 oder nicht für das gewählte Produkt.", code 4
```

Busca en su lugar **el id de plan que enviaste, presente en el mensaje**.
Digistore24 lo repite en todos los idiomas y es único de esa llamada. Un
detector escrito contra la clave no encuentra nada, el reintento no se dispara
nunca, y la página que querías proteger se queda a oscuras justo cuando hacía
falta — que es lo único que ningún test tuyo puede enseñarte, porque en él el
error lo lanzas tú.

🚨 **Nunca envíes `payment_plan[template]` a solas esperando que ponga el precio
de la llamada.** Sin un `first_amount`, Digistore24 descarta el `payment_plan`
entero, incluidos los valores que la plantilla había resuelto. Eso es lo correcto
para la primera forma y es fatal si querías la segunda y se te olvidó un campo.

**`product_id` decide también el IDIOMA del formulario de pedido: elígelo según
el idioma del comprador.** Un producto de Digistore24 tiene exactamente un
idioma, y en esta llamada no hay ningún parámetro que lo cambie (repasa el cuerpo
de arriba: `buyer`, `payment_plan`, `tracking`, `urls`, `placeholders`,
`settings`, `addons`; el idioma no aparece por ninguna parte). Por eso una app
multilingüe mantiene **un id de producto por idioma** para cada oferta, y
resuelve el idioma del visitante a uno de ellos justo aquí, antes de la llamada.
Si mandas a todo el mundo al mismo id, uno de cada dos rellena un formulario en
el idioma equivocado justo cuando se le pide pagar. La forma de la lista de
precios y la regla completa están en la skill **`ds24-products`**.

La respuesta es una URL. **Guárdala en caché por oferta**: vale durante la
ventana de `valid_until`, y crear una nueva en cada visita mete un viaje de ida
y vuelta a Digistore24 en el renderizado de tu página de precios.

⚠️ **Entonces la clave de caché tiene que incluir el idioma Y la forma de
pago**, y no solo la clave de la oferta. Con una fila por clave, la URL alemana
y la inglesa — o la mensual y la anual, que llevan valores distintos de
`settings[plan]` — se desalojan mutuamente en cada visita y, entre una y otra, la
caché sirve la página de checkout de un comprador a otro.
`"<offerKey>:<paymentOption>:<language>"`. Quita la parte del medio cuando la
oferta tenga una sola forma de pago, para que nada cambie en una oferta de precio
único.

🚨 **Y nunca guardes en caché una URL que lleve la identidad de un comprador.**
El Paso 2 mete el id del miembro con sesión iniciada en `tracking[custom]`, y
una caché indexada por oferta no distingue entre miembros: la identidad del
primer comprador que abrió la página con sesión se sirve a todos los que la
abran después, y cada pago de *esos* compradores llega atribuido a aquel primer
miembro. Y mientras tanto nada falla: la página se renderiza, el checkout se
abre, el dinero se mueve.

Así que una página de precios tiene **dos caminos**, y no son dos variantes del
mismo:

- **Sin sesión → la URL compartida de la caché.** No lleva identidad, es segura
  para todo el mundo y no hay viaje a Digistore24 mientras la página se
  renderiza.
- **Con sesión → una URL construida en el momento del clic**, con la identidad
  de ese miembro, usada una sola vez y **que nunca se escribe en la caché**.

Cuál de los dos toca lo decide el **contenido** del campo de tracking, no el
hecho de que esté relleno: un marcador que nombra el *paquete* se puede
compartir; uno que nombra a una *persona*, no. Preguntar solo «¿hay tracking?»
convierte cada tarjeta en una llamada real a la API en cada visita, que es
justo lo que la caché existía para evitar.

**Y cuando la llamada falla, la página tiene que renderizarse de todos modos.**
Que Digistore24 vaya lento o que la clave esté mal tiene que dar como resultado
un botón desactivado con su motivo («checkout no disponible»), nunca una
excepción en la página de precios ni un enlace muerto. Devuelve el fallo a quien
llama en lugar de lanzarlo.

**En un entorno de desarrollo esa URL todavía no está lista.** Mientras el
producto no tenga la aprobación del marketplace nadie puede comprar nada a
través de ella, y la forma de desbloquear una compra de prueba sin tocar el
navegador es añadir el parámetro testpay — **añadido al valor que devuelves,
después de la caché y solo donde un cliente no pueda llegar jamás**. No
construyas el checkout dejando esto para más tarde: es el paso que decide si
podrás demostrar que el resto funciona. La receta y las salvaguardas están en el
**Paso 4a**.

## Paso 2 — lleva la identidad del comprador hasta el final

El fallo más frecuente en una integración de Digistore24 es un pago que llega y
no se puede asociar a ninguna cuenta. Alguien ha pagado, la app no sabe quién, y
el soporte tiene que resolverlo a mano.

Envía un identificador en el campo de tracking. Digistore24 lo guarda junto a
la compra y lo devuelve en **todos** los eventos posteriores de ese pedido: la
renovación un año después, el reembolso, el contracargo. En la IPN llega como
`custom`:

```
tracking[custom] = m:<id de miembro>;t:<token aleatorio corto guardado en ese miembro>
```

**El campo es una única cadena opaca y es enteramente tuya**, así que dale una
estructura que puedas ampliar: pares `key:value` separados por `;`, y un lector
que **ignore las claves que no conozca** en lugar de fallar al verlas. Más
adelante querrás llevar un segundo id (qué paquete, qué tipo de compra, una
intención que el comprador expresó en el checkout), y para entonces habrá
compras reales que conservan el valor antiguo. Un id nuevo es entonces solo un
par nuevo; un segundo *formato* es una migración que no puedes hacer, porque los
valores que ya están en Digistore24 no se pueden reescribir.

**Dos cosas sobre ese token.** Corrobora el id de miembro, de modo que un id
adivinado o manipulado nunca reclama por sí solo la compra de otra persona; y
**no es una credencial**: nunca autentica una sesión, solo dice «este id no se
lo inventó quien escribió la URL». Las dos partes tienen que estar presentes y
bien formadas, o el valor no nombra a nadie: media identidad no es una identidad
más débil.

En el otro extremo, en el handler de la IPN, atribuye en este orden — y el
orden es una regla de seguridad, no una preferencia:

1. **El identificador de `custom`, con el token coincidente → autenticado.** Ese
   valor lo escribió tu app, Digistore24 lo guardó en su servidor y el comprador
   nunca tuvo una copia que pudiera editar.
2. **Si no, el correo del comprador contrastado con tus cuentas → no
   autenticado.** Esa dirección la tecleó en un formulario de Digistore24 quien
   estuviera pagando, y **Digistore24 no comprueba que la controle**. Suele ser
   correcta y nunca es una prueba.
3. **Si no, guarda el pedido sin atribuir** y asócialo cuando esa dirección
   inicie sesión por primera vez.

🚨 **«Sin `custom`» no es un diagnóstico, y tomarlo por uno es el error.** Tiene
al menos dos causas que en el log se ven idénticas: un comprador tuyo que no
tenía sesión al hacer clic (no había ningún id de miembro que escribir), y
alguien que nunca pasó por tu app — el producto de Digistore24 tiene un
formulario de pedido propio, expuesto en el marketplace una vez aprobado, y una
compra hecha ahí no lleva nada escrito por ti.

⚠️ **El importe no las distingue**, y una copia anterior de este pack decía que
sí. Desde que el producto lleva tus planes de pago (**`ds24-products`**,
Paso 3), una compra hecha en su propio formulario de pedido paga la misma cifra
que una hecha por tu enlace — que es justo para lo que se escriben, y te cuesta
esta heurística. Trata «sin `custom`» como *sin atribuir*, resuélvelo por correo
electrónico con las dos negativas de más abajo, y nunca por el precio.

Dos negativas son las que hacen que el paso 2 pueda existir sin peligro:

- 🚨 **Una dirección que coincide con más de una cuenta se rechaza; no se
  resuelve con la primera fila.** Pide como máximo dos coincidencias y trata
  «dos» como *no se puede saber*. La consulta que devuelve una lista y se queda
  con `[0]` es exactamente la forma de este bug, y lo que hace es entregar a un
  cliente la compra de otro. Sin atribuir es el resultado correcto; adivinar no
  es una opción de respaldo.
- **La atribución solo concede: nunca mueve y nunca revoca.** Una coincidencia
  de correo puede asociar un pedido que todavía no es de nadie. No puede
  reasignar un pedido ya atribuido, y ningún fallo de atribución puede terminar
  un acceso que existe. Esa unidireccionalidad es lo único que hace tolerable un
  camino no autenticado.

Y todo lo que más adelante autorice un acto **desatendido** — cobrar a un medio
de pago guardado, activar una recarga automática (**`ds24-tokens`**) — acepta
únicamente el camino 1. Una coincidencia por el camino 2 es una buena suposición
sobre quién compró algo; no es un permiso para cobrar a una tarjeta.

Un pedido sin atribuir es un ticket de soporte. Uno mal atribuido es un cliente
viendo la compra de otra persona, y de los dos es el más caro.

## Paso 3 — una compra sin cuenta también tiene que funcionar

Deja que la gente compre desde la página de precios pública sin iniciar sesión
antes. Así llega la mayoría, y obligar a crear una cuenta antes de pagar cuesta
ventas. Lo que lo hace seguro es el camino 3 de arriba: el pedido espera, y el
primer inicio de sesión desde esa dirección lo reclama.

## Paso 4 — la página de agradecimiento

Después del pago, Digistore24 envía al comprador a una URL tuya con el id del
pedido dentro. Dos reglas:

- **Es pública.** El comprador todavía no tiene sesión. No pongas detrás de ella
  nada que presuponga una.
- **No concedas acceso desde ella.** Es un navegador abriendo una URL, y
  cualquiera puede abrirla. El acceso viene de la IPN, que está firmada. La
  página de agradecimiento dice «gracias, está en camino / así inicias sesión»,
  y nada más.

**Digistore24 solo guarda URL públicas con https.** Una URL de
agradecimiento en `localhost` se rechaza de plano
(`Please only use secure URLs with https://`). En una plataforma alojada la URL
de tu app ya es pública, así que no hay problema; en un portátil hace falta un
redirector público o un túnel.

## Paso 4a — pagos de prueba sin aprobación (la clave testpay)

Un producto que todavía no tiene la aprobación del marketplace solo se puede
comprar como **compra de prueba**. Hay dos formas de desbloquearla, y cada una
encaja en un sitio distinto:

- **La cookie de compra de prueba** — se activa una vez en el navegador del
  vendedor (el enlace está en el centro de ayuda de Digistore24). Vale por
  navegador y caduca. Es la herramienta adecuada en cualquier dominio al que
  también pueda llegar un cliente.
- **El parámetro testpay** — se obtiene por la API y se añade a la URL de
  compra, de modo que el desbloqueo viaja con el enlace en lugar de vivir en un
  navegador:

  ```
  POST https://www.digistore24.com/api/call/getTestpayKey/format/json
  Header: X-DS-API-KEY: <la clave>
  ```

  No está documentado, pero es real. La respuesta trae `testpay_key`,
  `get_param_name` y `expires_at`. Añade `?<get_param_name>=<testpay_key>` a la
  URL de compra (el NOMBRE sale de la respuesta; nunca lo escribas fijo en el
  código) y el checkout se abre en modo de pago de prueba, esté aprobado o no.
  **La opción «Testzahlung» (pago de prueba) solo aparece para el vendedor que
  ha iniciado sesión en Digistore24 en ese mismo navegador** — dilo antes de
  que haga clic, o la página responde «nur bei Ihren eigenen Produkten — und
  nur wenn Sie eingeloggt sind» (medido el 2026-09-15). Enviar `do_recreate=1`
  rota la clave: se genera una nueva y todas las copias anteriores dejan de
  funcionar.

Cuatro salvaguardas, todas indispensables:

- **Solo en desarrollo o vista previa; nunca en una URL a la que pueda llegar
  un cliente.** Un checkout con este parámetro acepta «pagos» de prueba: quien haga
  clic se lleva el producto gratis. Restríngelo por entorno con una lista de
  entornos autorizados (todo lo que no sea claramente desarrollo cuenta como
  producción y se rechaza), y añádelo en el momento de renderizar o de hacer
  clic.
- **Nunca en una URL de compra cacheada o compartida.** Si las URL de compra se
  guardan en caché (Paso 1), cachea la URL limpia y añade el parámetro después:
  una URL decorada en una caché compartida se sirve a todo el mundo.
- **La clave es de toda la cuenta; trátala como un secreto.** Funciona en TODAS
  las URL de checkout de esta cuenta de vendedor, las de producción incluidas.
  Mantenla fuera del repositorio y fuera de la configuración desplegada.
- **Rótala antes de la puesta en producción** (`do_recreate=1`); ver
  **`ds24-golive`**.

## Paso 5 — demuéstralo

1. Crea una URL de compra y ábrela. La página de checkout tiene que mostrar
   **tu** precio, moneda e intervalo; si muestra otra cosa, el plan de pago no
   ha llegado.
2. Haz una **compra de prueba**, con la cookie de compra de prueba de
   Digistore24 activada o, en un entorno de desarrollo, con el parámetro testpay
   añadido (Paso 4a).
3. Comprueba que la IPN ha llegado y que el pedido ha quedado **atribuido a la
   cuenta correcta**. La atribución es la parte que parece estar bien hasta que
   deja de estarlo. Las compras de prueba llegan con `api_mode=test` en el
   payload de la IPN; procésalas igual que las reales (que el camino sea
   idéntico es precisamente lo que demuestra la prueba).

## Paso 6 — qué viene después

- **`ds24-ipn`** — el endpoint que recibe lo que produce este checkout.
- **`ds24-entitlements`** — convertir un pedido pagado en «puede usar el
  producto».
- **`ds24-tokens`** — si vendes créditos de prepago en lugar de planes.
- **`ds24-golive`** — la compra de prueba real, de principio a fin.

Di cuál vas a empezar y empiézala.
