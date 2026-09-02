---
name: ds24-golive
language: es
description: Úsala cuando una integración de Digistore24 ya está construida y hay que demostrarla antes de que lleguen clientes reales — la comprobación previa, la compra de prueba con la cookie de prueba de Digistore24, la aprobación del marketplace y las comprobaciones de puesta en producción en el dominio en producción. Úsala siempre que el usuario diga que quiere poner en producción, lanzar, vender de verdad, hacer una compra de prueba, o pregunte si la integración de pagos está realmente lista.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# La puesta en producción

La integración está construida. Ahora demuestra que mueve dinero y desbloquea el
producto — antes de que alguien que no eres tú descubra que no lo hace.

**No saltes directamente a la aprobación.** Un producto aprobado y público con
una IPN rota vende un acceso que nadie recibe, y cada uno de esos casos es un
reembolso más una conversación con el soporte.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el `VERSION` de este pack. Menciona cualquier discrepancia en una
frase y sigue adelante.

## Paso 1 — la comprobación previa

Recorre estos puntos e **informa de cada uno con lo que has visto realmente**, no
con una marca de verificación:

| Comprobación | Pasa cuando |
|---|---|
| La app es accesible en su dominio **https público** | una petición desde fuera responde |
| `GET <domain>/api/ipn` (o tu ruta) | responde **200**, sin redirección |
| `DIGISTORE_IPN_PASSPHRASE` está definida **en el entorno desplegado** | no solo en un archivo local |
| `DIGISTORE_API_KEY` está definida en el entorno desplegado | — |
| La conexión IPN en Digistore24 apunta al dominio **en producción** | no a un túnel ni a una URL de vista previa del desarrollo |
| El producto existe y su precio coincide con tu lista de precios | — |
| Los secretos están en el almacén de secretos de la plataforma | no en el repositorio |

La quinta fila es la que muerde después de un redespliegue: sigue registrada una
URL de vista previa de cuando construías la cosa, y cada compra real va a una
dirección que ya no responde.

Vuelve a apuntarla llamando otra vez a `ipnSetup` con el **mismo `domain_id`** y
la URL en producción — eso actualiza la conexión. Un `domain_id` nuevo crea una
segunda y deja la primera, muerta, en su sitio (**`ds24-products`**, Paso 4). Ya
que estás, comprueba que los `product_ids` de la conexión cubren los productos
que realmente vas a vender.

## Paso 2 — la firma, una vez más, contra producción

Lo que demostró el endpoint durante el desarrollo, ejecútalo otra vez **contra el
dominio en producción**. La skill **`ds24-ipn`** tiene el cómo — su referencia de
verificación da dos formas, y el paso 2 necesita la que va por HTTP desde fuera:
una prueba dentro de la app ejercita el handler, no el despliegue, y los fallos
que aparecen solo ahora son fallos de despliegue. Un proxy que reescribe el
cuerpo, una passphrase que nunca llegó al entorno desplegado.

Si `ds24-ipn` no está instalada, instálala — este paso no se puede hacer bien sin
lo que dice.

En verde, **sin ningún salto**, o no está lista. Una ejecución con comprobaciones
de acceso saltadas significa que la firma está demostrada y la semántica no —
dilo con claridad en vez de llamarlo verde.

Borra el endpoint de sondeo en cuanto esto pase.

## Paso 3 — la compra de prueba

Este es el paso que no puede sustituirse por ninguna otra cosa, porque es el
único que ejercita también el lado de Digistore24.

1. El vendedor pone la **cookie de compra de prueba de Digistore24** en su
   navegador. (El centro de ayuda de Digistore24 tiene el enlace que la pone; es
   por navegador y caduca.) En el dominio en producción la cookie es la
   herramienta correcta — el parámetro testpay del desarrollo
   (**`ds24-checkout`**, Paso 4a) pertenece a URLs que los clientes nunca pueden
   alcanzar, así que no lo traigas aquí.
2. Compra el producto a través del propio enlace de compra de la app — no a
   través de un enlace que hayas construido a mano para la prueba.
3. Vigila: el checkout muestra **tu** precio y tu intervalo; la página de
   agradecimiento carga; la IPN llega; el pedido se guarda; **el acceso aparece
   en la app**. Si la IPN no llega, `getPurchase` (**`ds24-products`**, Paso 7)
   dice si Digistore24 llegó siquiera a registrar la compra — esa es la
   diferencia entre un checkout fallido y una conexión rota, y no puedes
   distinguirlos desde la app sola.
4. Inicia sesión como ese cliente y confirma que lo pagado es realmente usable.

Luego la otra mitad, la que la gente se salta y no debería:

5. **Reembolsa la compra de prueba** desde la cuenta de Digistore24.
6. Confirma que el acceso **ha desaparecido** en la app.

Una compra que concede acceso demuestra media integración. El reembolso demuestra
la mitad que te protege.

## Paso 4 — la aprobación

La aprobación es para los productos **en producción**. Si mantuviste un conjunto
de desarrollo aparte (`ds24-products`, Paso 2), no lo envíes nunca — un producto
«[DEV]» en un marketplace es un rechazo que has pedido tú, y las compras de
prueba no necesitan aprobación.

**Primero: ¿la aprobación aplica siquiera a este vendedor?** Solo los cuatro
**resellers** de Digistore24 aprueban productos — Alemania (`siteowner_id` 1),
EE. UU. (2), Reino Unido (3), Irlanda (4). Cualquier otro siteowner es un
**Direct Seller**: el vendedor vende en su propia cuenta, y no hay paso de
aprobación, nada que solicitar y nada que esperar. **Sáltate todo este paso para
ellos**, y tampoco construyas un recordatorio al respecto — una insistencia sobre
una aprobación que no puede existir no se resuelve nunca, y el vendedor no puede
hacer nada al respecto.

Dos formas de saberlo: un siteowner configurado fuera de 1–4, o un producto cuya
`approval_status_list` no tiene ninguna entrada de reseller *activa*
(`is_siteowner_active`). Ten en cuenta que un `approval_status` en una entrada
que no es de reseller no significa nada — leerlo como un veredicto inventa una
aprobación que nadie concedió.

Todo lo de abajo es para un vendedor reseller.

Solicita la aprobación del marketplace (`approval_status=pending`) una vez que la
descripción del producto y la app estén de verdad terminadas — un producto a
medio construir se rechaza, y el segundo intento es más lento que el primero.

Hasta la aprobación, las compras de prueba del vendedor son las únicas compras
posibles. Ese es el estado correcto en el que estar mientras se construye.

⚠️ **La aprobación es un LISTADO en el marketplace, así que es también el momento
en que el formulario de pedido propio del producto pasa a ser algo que
encuentran desconocidos.** Entonces pagan el plan de pago guardado del producto
— el de Digistore24 por defecto, que nadie puso, no tu precio
(**`ds24-products`**, Paso 2). Esa compra es real: llega a tu IPN, y si tu
manejador concede acceso con `on_payment`, lo concede. Aprueba porque quieres el
marketplace, y sabiendo que esto viene con él.

**A qué marketplace envías lo decide el idioma del PRODUCTO**, no el de la app:
un producto en alemán va a Digistore24 GmbH, Alemania (`siteowner_id` 1), y
cualquier otro a Digistore24 Inc., EE. UU. (2). Derivar un único marketplace de
un ajuste global de la app es el error que hay que evitar aquí, porque envía en
silencio tu oferta en inglés al reseller alemán.

**Y una app multilingüe tiene más productos que ofertas.** Un producto de
Digistore24 lleva exactamente un idioma — es el idioma del formulario de pedido
del comprador, ver **`ds24-products`** — así que una oferta vendida en alemán y
en inglés son *dos* productos, enviados a *dos* marketplaces, cada uno con su
propio veredicto.

Esa es la trampa de este paso: **aprobado en Alemania no dice nada sobre el
gemelo inglés.** Una pantalla de estado que informa por oferta en lugar de por
producto muestra una luz verde mientras media tienda no se puede vender, y el
producto inglés es el que nadie se acuerda de enviar. Itera sobre productos, no
sobre ofertas, y comprueba que cada uno de ellos llega a `approved`.

**Si se concedió es legible, no adivinable.** Cada elemento de `listProducts` /
`getProduct` lleva `approval_status_list` — una entrada por marketplace
(`reseller_id`) con `approval_status` siendo uno de `new` (nunca solicitado),
`pending`, `approved` o `rejected`, más `is_siteowner_active` y los campos del
motivo de rechazo. El campo no está en la documentación oficial de la API
(verificado empíricamente en 2026-07), así que léelo a la defensiva: una lista
ausente o un valor desconocido significa «no se puede saber», no un estado.

**Hay dos preguntas distintas, y necesitan lecturas distintas:**

| Pregunta | Cómo leer la lista |
|---|---|
| *¿Se puede vender este producto siquiera?* — para una pantalla de estado o un recordatorio | Agrega sobre todos los marketplaces para los que la cuenta está **activa**: **aprobado en cualquiera gana**, si no pending, si no rejected, si no new. Un producto aprobado en Alemania se vende en Alemania decida lo que decida el reseller de EE. UU. |
| *¿Debo solicitar la aprobación aquí?* — antes de una escritura | La entrada de **ese único** `reseller_id`. Un producto aprobado en Alemania puede tener todavía una solicitud legítima que hacer en EE. UU. |

Ignora una entrada cuyo `is_siteowner_active` sea `"N"`: ese marketplace no puede
actuar, así que su veredicto no dice nada — y tratarlo como un estado real
produce un aviso sobre un marketplace que nadie puede usar.

Cuatro reglas para la escritura en sí:

- **`pending` es el único estado que vale la pena escribir.** `updateProduct`
  aceptará los otros, y ahí está la trampa: escribir `approved` sobre tu propio
  producto hace que toda pantalla de estado crea que vende, así que el
  recordatorio que hayas construido se calla para un producto que ningún reseller
  ha mirado jamás. `new` retira una solicitud que ya estaba en cola. `approved` y
  `rejected` pertenecen al reseller.
- **No vuelvas a solicitar un producto que ya está `approved` en el marketplace
  al que escribes.** El lado del reseller decide solo sobre productos `pending`,
  y si escribir `pending` sobre una aprobación la reinicia no está documentado —
  no es un experimento para una cuenta en producción.
- **No escribas cuando no has podido leer.** Si la llamada de estado falló, o el
  producto falta en la respuesta, no puedes descartar una aprobación existente —
  así que niégate y di por qué, en vez de solicitar a ciegas. Fallar en abierto
  aquí es cómo un producto aprobado y vendiendo acaba de vuelta en pending.
- **No escribas a un marketplace cuyo `is_siteowner_active` sea `"N"`.** La
  llamada tiene éxito, nadie de allí lo mirará jamás, y una pantalla de estado
  que filtre los marketplaces inactivos seguirá informando del producto como
  nunca enviado — para siempre, y repetir la solicitud no cambia nada.

Un producto `rejected` nombra su motivo en la cuenta de vendedor de Digistore24.
Arréglalo allí **primero**: reenviarlo sin cambios hace que lo rechacen otra vez,
y el segundo intento es más lento que el primero.

## Paso 5 — el día que está en producción

**Primero, rota la clave de compra de prueba** si la integración usó alguna vez
el parámetro testpay durante el desarrollo (**`ds24-checkout`**, Paso 4a): llama
a `getTestpayKey` con `do_recreate=1`. La clave es a nivel de cuenta — una copia
que circulara mientras construías desbloquearía «compras» de prueba en el
checkout en producción para quien todavía la tenga. Rotarla invalida todas las
copias antiguas en una sola llamada.

Dile estas tres cosas al usuario con palabras llanas, porque ninguna de ellas es
obvia:

- **Vigila la primera compra real.** No el panel — la app. Si el acceso apareció
  es la única pregunta que importa.
- **Guarda los payloads IPN en bruto.** Son con lo que se responde a cualquier
  disputa en los próximos meses.
- **Un pago que llega sin atribuir es normal, no un error.** Alguien compró sin
  cuenta, o con una dirección distinta. Ten una forma de adjuntarlo a mano (ver
  **`ds24-entitlements`**, concesiones manuales) antes de necesitarla con prisa.

## Paso 6 — qué viene después

- **`ds24-compliance`** — las páginas legales y las obligaciones que dispara una
  app en producción, de pago, en la UE. Haz esto antes de tener clientes reales,
  no después.

Di si hay que empezarla.
