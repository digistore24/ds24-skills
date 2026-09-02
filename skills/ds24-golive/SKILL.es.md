---
name: ds24-golive
language: es
description: Úsala cuando la integración de Digistore24 ya está construida y hay que demostrar que funciona antes de que lleguen clientes reales — la comprobación previa, la compra de prueba con la cookie de prueba de Digistore24, la aprobación del marketplace y las comprobaciones de la puesta en producción sobre el dominio en producción. Úsala siempre que el usuario diga que quiere poner la app en producción, lanzarla, vender de verdad o hacer una compra de prueba, o pregunte si la integración de pagos está lista de verdad.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# La puesta en producción

La integración ya está construida. Ahora toca demostrar que mueve dinero y que
desbloquea el producto — antes de que sea otra persona quien descubra que no lo
hace.

**No saltes directamente a la aprobación.** Un producto aprobado y público con
una IPN rota vende un acceso que nadie recibe, y cada venta así es un reembolso
más una conversación con soporte.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el `VERSION` de este pack. Si no coinciden, dilo en una frase y
sigue adelante.

## Paso 1 — la comprobación previa

Recorre estos puntos y, para cada uno, **di lo que has visto de verdad** — no
pongas una marca de verificación:

| Comprobación | Pasa cuando |
|---|---|
| La app es accesible en su dominio **público con https** | una petición hecha desde fuera obtiene respuesta |
| `GET <domain>/api/ipn` (o la ruta que uses) | responde **200**, sin redirección |
| `DIGISTORE_IPN_PASSPHRASE` está definida **en el entorno desplegado** | no solo en un archivo local |
| `DIGISTORE_API_KEY` está definida en el entorno desplegado | — |
| La conexión IPN en Digistore24 apunta al dominio **en producción** | no a un túnel ni a una URL de vista previa que quedó del desarrollo |
| El producto existe y su precio coincide con tu lista de precios | — |
| Los secretos están en el almacén de secretos de la plataforma | no en el repositorio |

La quinta fila es la que muerde tras un redespliegue: la URL de vista previa
que usaste mientras construías sigue registrada, y cada compra real acaba en
una dirección que ya no responde.

Para volver a apuntarla, llama otra vez a `ipnSetup` con el **mismo
`domain_id`** y la URL de producción: eso actualiza la conexión que ya existe.
Con un `domain_id` nuevo creas una segunda conexión, y la primera, muerta, se
queda donde estaba (**`ds24-products`**, Paso 4). Ya que estás ahí, comprueba
que los `product_ids` de la conexión cubren los productos que de verdad vas a
vender.

## Paso 2 — la firma, una vez más, contra producción

Lo que sirvió para demostrar el endpoint durante el desarrollo, ejecútalo otra
vez, ahora **contra el dominio en producción**. El cómo está en la skill
**`ds24-ipn`**: su referencia de verificación describe dos formas, y este paso
necesita la que entra por HTTP desde fuera. Una prueba dentro de la app ejercita
el handler, no el despliegue — y los fallos que solo aparecen ahora son fallos
de despliegue: un proxy que reescribe el cuerpo, una passphrase que nunca llegó
al entorno desplegado.

Si `ds24-ipn` no está instalada, instálala: sin lo que dice ahí, este paso no se
puede hacer bien.

O sale en verde, **sin ningún salto**, o no está lista. Una ejecución que se
saltó las comprobaciones de acceso demuestra la firma, pero no la semántica —
dilo así de claro en vez de llamarla verde.

Cuando esto pase, borra el endpoint de sondeo.

## Paso 3 — la compra de prueba

Este paso no lo sustituye ninguna otra cosa, porque es el único que ejercita
también el lado de Digistore24.

1. El vendedor pone en su navegador la **cookie de compra de prueba de
   Digistore24**. (El enlace que la activa está en el centro de ayuda de
   Digistore24; vale para un solo navegador y caduca.) En el dominio en
   producción la cookie es la herramienta adecuada — el parámetro testpay del
   desarrollo (**`ds24-checkout`**, Paso 4a) pertenece a direcciones a las que
   ningún cliente puede llegar, así que no lo traigas aquí.
2. Compra el producto por el enlace de compra de la propia app, no por uno que
   hayas montado a mano para la prueba.
3. Fíjate en esto: el checkout muestra **tu** precio y tu intervalo; la página
   de agradecimiento carga; la IPN llega; el pedido queda guardado; **el acceso
   aparece en la app**. Si la IPN no llega, `getPurchase` (**`ds24-products`**,
   Paso 7) te dice si Digistore24 llegó siquiera a registrar la compra — esa es
   la diferencia entre un checkout que falló y una conexión rota, y desde la app
   sola no puedes distinguirlas.
4. Inicia sesión como ese cliente y confirma que lo que pagó se puede usar de
   verdad.

Después, la otra parte — la que la gente se salta y no debería:

5. **Reembolsa la compra de prueba** desde la cuenta de Digistore24.
6. Confirma que el acceso **ha desaparecido** de la app.

Una compra que concede acceso demuestra la mitad de la integración. El reembolso
demuestra la otra mitad: la que te protege a ti.

## Paso 4 — la aprobación

La aprobación es para los productos **en producción**. Si mantuviste un juego
de desarrollo aparte (`ds24-products`, Paso 2), no lo envíes nunca: un producto
«[DEV]» en un marketplace es un rechazo que te has buscado, y las compras de
prueba no necesitan aprobación.

**Antes de nada: ¿la aprobación afecta siquiera a este vendedor?** Solo los
cuatro **resellers** de Digistore24 aprueban productos — Alemania
(`siteowner_id` 1), EE. UU. (2), Reino Unido (3) e Irlanda (4). Cualquier otro
siteowner es un **Direct Seller**: el vendedor vende en su propia cuenta y no
hay paso de aprobación — nada que solicitar, nada que esperar. **Para ellos,
sáltate este paso entero**, y no construyas tampoco un recordatorio al respecto:
un aviso que reclama una aprobación que no puede existir no se apaga nunca, y el
vendedor no puede hacer nada para apagarlo.

Se sabe de dos maneras: por un siteowner configurado fuera de 1–4, o por un
producto cuya `approval_status_list` no tiene ninguna entrada de reseller
*activa* (`is_siteowner_active`). Y ojo: un `approval_status` en una entrada que
no es de reseller no significa nada — leerlo como veredicto es inventarse una
aprobación que nadie concedió.

Todo lo que sigue es para un vendedor que vende a través de un reseller.

Solicita la aprobación del marketplace (`approval_status=pending`) solo cuando
la descripción del producto y la app estén terminadas de verdad: un producto a
medio hacer se rechaza, y el segundo intento tarda más que el primero.

Hasta que llegue la aprobación, las únicas compras posibles son las compras de
prueba del vendedor. Mientras se construye, ese es exactamente el estado en el
que hay que estar.

⚠️ **La aprobación es un LISTADO en el marketplace, así que también es el
momento en que el formulario de pedido propio del producto pasa a ser algo que
encuentran desconocidos.** Y esos desconocidos pagan el plan de pago guardado en
el producto — el que Digistore24 pone por defecto y nadie eligió, no tu precio
(**`ds24-products`**, Paso 2). Esa compra es real: llega a tu IPN, y si tu
handler concede acceso con `on_payment`, lo concede. Pide la aprobación porque
quieres estar en el marketplace, sabiendo que esto forma parte del trato.

**El marketplace al que se envía lo decide el idioma del PRODUCTO**, no el de
la app: un producto en alemán va a Digistore24 GmbH, Alemania (`siteowner_id`
1); cualquier otro, a Digistore24 Inc., EE. UU. (2). Deducir un único
marketplace de un ajuste global de la app es justo el error que hay que evitar
aquí, porque manda sin decir nada tu oferta en inglés al reseller alemán.

**Y una app multilingüe tiene más productos que ofertas.** Un producto de
Digistore24 lleva exactamente un idioma (el del formulario de pedido que ve el
comprador; ver **`ds24-products`**), así que una oferta que se vende en alemán y
en inglés son *dos* productos, enviados a *dos* marketplaces, y cada uno recibe
su propio veredicto.

Ahí está la trampa de este paso: **que esté aprobado en Alemania no dice nada
de su gemelo en inglés.** Una pantalla de estado que informa por oferta y no
por producto enseña una luz verde mientras media tienda no se puede vender, y
el producto en inglés es justo el que nadie se acuerda de enviar. Recorre
productos, no ofertas, y comprueba que todos y cada uno llegan a `approved`.

**Si se concedió es algo que se lee, no que se adivina.** Cada elemento de
`listProducts` / `getProduct` trae `approval_status_list`: una entrada por
marketplace (`reseller_id`), con `approval_status` en uno de estos valores —
`new` (nunca solicitado), `pending`, `approved` o `rejected` —, más
`is_siteowner_active` y los campos con el motivo del rechazo. El campo no
aparece en la documentación oficial de la API (verificado empíricamente en
2026-07), así que léelo a la defensiva: una lista ausente o un valor desconocido
significan «no se puede saber», no un estado.

**Hay dos preguntas distintas, y cada una lee la lista de otra manera:**

| Pregunta | Cómo leer la lista |
|---|---|
| *¿Se puede vender siquiera este producto?* — para una pantalla de estado o un recordatorio | Combina todos los marketplaces en los que la cuenta está **activa**: **aprobado en cualquiera de ellos gana**; si no, pending; si no, rejected; si no, new. Un producto aprobado en Alemania se vende en Alemania, decida lo que decida el reseller de EE. UU. |
| *¿Debo solicitar aquí la aprobación?* — antes de una escritura | Solo la entrada de **ese único** `reseller_id`. Un producto aprobado en Alemania puede tener aún una solicitud legítima que hacer en EE. UU. |

Ignora cualquier entrada cuyo `is_siteowner_active` sea `"N"`: ese marketplace
no puede actuar, así que su veredicto no dice nada — y tratarlo como un estado
real produce un aviso sobre un marketplace que nadie puede usar.

Cuatro reglas para la escritura en sí:

- **`pending` es el único estado que tiene sentido escribir.** `updateProduct`
  acepta los demás, y ahí está la trampa: si escribes `approved` sobre tu propio
  producto, todas las pantallas de estado creerán que se vende, y el
  recordatorio que hayas construido se callará para un producto que ningún
  reseller ha mirado nunca. `new` retira una solicitud que ya estaba en cola.
  `approved` y `rejected` son del reseller.
- **No vuelvas a solicitar un producto que ya está `approved` en el marketplace
  al que escribes.** El reseller solo decide sobre productos `pending`, y no
  está documentado si escribir `pending` encima de una aprobación la anula — no
  es un experimento para hacer en una cuenta en producción.
- **No escribas si no has podido leer.** Si la llamada de estado falló, o el
  producto no aparece en la respuesta, no puedes descartar que ya exista una
  aprobación: niégate y explica por qué, en vez de solicitar a ciegas. Dar por
  buena una lectura que falló (fail-open) es exactamente la forma en que un
  producto aprobado y en venta vuelve a pending.
- **No escribas a un marketplace cuyo `is_siteowner_active` sea `"N"`.** La
  llamada sale bien, nadie allí la va a mirar nunca, y una pantalla de estado
  que filtre los marketplaces inactivos seguirá informando del producto como
  nunca enviado — para siempre, y por mucho que repitas la solicitud no
  cambiará nada.

Un producto `rejected` tiene su motivo en la cuenta de vendedor de Digistore24.
Arréglalo allí **primero**: si lo reenvías sin cambios, lo rechazan otra vez, y
el segundo intento tarda más que el primero.

## Paso 5 — el día que ya está en producción

**Lo primero: rota la clave de compra de prueba** si la integración usó en
algún momento el parámetro testpay durante el desarrollo (**`ds24-checkout`**,
Paso 4a): llama a `getTestpayKey` con `do_recreate=1`. La clave es de toda la
cuenta — cualquier copia que haya circulado mientras construías desbloquea
«compras» de prueba en el checkout de producción para quien todavía la tenga.
Rotarla invalida todas las copias antiguas con una sola llamada.

Dile al usuario estas tres cosas con palabras llanas, porque ninguna es
evidente:

- **Vigila la primera compra real.** No el panel — la app. Si el acceso
  apareció es la única pregunta que importa.
- **Guarda los payloads IPN en bruto.** Con ellos se responde cualquier disputa
  de los próximos meses.
- **Un pago que llega sin atribuir es normal, no un error.** Alguien compró sin
  cuenta, o con otra dirección. Ten preparada una forma de asignarlo a mano
  (ver **`ds24-entitlements`**, concesiones manuales) antes de que te haga
  falta con prisa.

## Paso 6 — qué viene después

- **`ds24-compliance`** — las páginas legales y las obligaciones que activa una
  app de pago en producción dentro de la UE. Hazlo antes de tener clientes
  reales, no después.

Di si toca empezarla ya.
