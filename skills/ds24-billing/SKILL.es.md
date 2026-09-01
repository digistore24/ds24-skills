---
name: ds24-billing
language: es
description: Úsala como punto de entrada para vender cualquier cosa a través de Digistore24 desde una app — empieza aquí cuando el usuario quiera cobrar pagos, añadir un muro de pago, vender una suscripción o créditos, conectar Digistore24, o pregunte por dónde empezar con la facturación. Averigua qué existe ya, nombra el único paso siguiente y pasa el testigo a la skill que lo hace. Úsala también cuando una integración de Digistore24 se comporte mal y aún no esté claro qué parte falla.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# Vender a través de Digistore24 — empieza aquí

Digistore24 es el comerciante registrado (merchant of record) — gestiona el
checkout, cobra el dinero, se ocupa del IVA y de los reembolsos, y le cuenta a
tu app lo que ha pasado mediante un webhook firmado. El trabajo de tu app es
pequeño y exacto — enviar gente a un checkout y convertir los eventos que
vuelven en acceso.

Esta skill averigua en qué punto está el proyecto y arranca la siguiente que
toca. No construye nada por sí misma.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el archivo `VERSION` de este pack. Las skills importadas a un
workspace no se actualizan solas, así que una copia puede tener meses.
Menciona cualquier discrepancia en una frase, ofrece reimportar y sigue adelante
en cualquier caso.

## Paso 0b — puede que las otras siete no estén aquí

Esta skill pasa el testigo a otras siete por su nombre. En Lovable y Manus cada
skill se importa por separado, así que la que estás leyendo puede ser la única
presente — y «arranca `ds24-ipn`» se convierte entonces, sin ruido, en «escribe
el webhook de memoria», que es justo el desenlace que este pack existe para
evitar.

**Si una skill nombrada más abajo no está en el workspace, descárgala
en lugar de improvisar:**

```
https://raw.githubusercontent.com/digistore24/ds24-skills/main/skills/<name>/SKILL.es.md
```

Los archivos que lleva una skill cuelgan de la misma dirección — sus entradas
`references/…`, `scripts/…` y `adapters/…` con el nombre de la carpeta delante.
Di en una línea que la estás leyendo desde GitHub en vez de cargar una skill
instalada, y sugiere importarla como es debido después, para que la próxima
sesión ya arranque con ella.

**Si tampoco puedes descargarla, párate y dilo.** Lo que llevan estas skills es
justamente la parte que parece obvia y no lo es; reconstruida de memoria produce
una integración cuyas propias pruebas pasan y cuyos pagos reales se rechazan
todos sin excepción.

## Paso 1 — mira el proyecto, no entrevistes al usuario

Busca primero. Pregunta solo por lo que de verdad no deja rastro.

| Qué buscar | Qué te dice |
|---|---|
| `DIGISTORE_API_KEY` | la cuenta está conectada |
| una lista de precios/planes en el proyecto | los productos se definen aquí |
| `DIGISTORE_IPN_PASSPHRASE` | se registró una conexión IPN |
| `sha_sign`, una ruta que contiene `ipn` | el webhook existe |
| una tabla de acceso/concesión/derechos de acceso | el acceso está modelado |
| `createBuyUrl`, `payment_plan` | el checkout existe |

## Paso 2 — el único paso siguiente

Toma la **primera** fila que falte y arranca esa skill. No expongas el plan
entero; nombra el paso, di por qué toca ahora, y empieza.

| Qué falta | Arranca | Por qué va primero |
|---|---|---|
| la clave de API, los productos, la conexión IPN | **`ds24-products`** | hasta que Digistore24 no conozca tu endpoint, nada lo llama nunca y nada del resto se puede probar |
| el endpoint del webhook | **`ds24-ipn`** | esta es la pieza que tiene que estar bien a la primera |
| el registro de acceso | **`ds24-entitlements`** | los eventos necesitan un sitio donde escribir |
| el enlace de compra | **`ds24-checkout`** | — |
| no falta nada | **`ds24-golive`** | demuéstralo con una compra de prueba real |

Dos opcionales, que se toman cuando aplican en vez de en orden:

- **`ds24-tokens`** — el producto mide el uso (créditos) en vez de restringir
  funcionalidades. Habitual en funcionalidades de IA, donde tus propios costes
  escalan con el uso.
- **`ds24-compliance`** — antes de tener clientes reales: aviso legal, política
  de privacidad, la divulgación del AI Act, derechos de supresión y de acceso.

## Paso 3 — cuando algo está roto

Empareja el síntoma, no recorras la lista:

| Síntoma | Dónde vive |
|---|---|
| «firma inválida» en cada IPN | **`ds24-ipn`** — casi siempre son las mayúsculas y minúsculas de los nombres de campo; su propia referencia `ipn-protocol.es.md` lo tiene |
| la compra funcionó, en la app no pasó nada | **`ds24-products`** — busca primero el pedido con `getPurchase` (Paso 7 allí), después la conexión: URL equivocada o muerta, nunca registrada, un `domain_id` que otro proyecto sobrescribió, una lista `product_ids` sin este producto. En Supabase/Lovable Cloud: `verify_jwt` sigue activado y cada llamada recibe un 401 |
| un cliente que canceló perdió el acceso de inmediato | **`ds24-ipn`** — `on_rebill_cancelled` se trató como un final. No hace nada |
| un cliente reembolsado sigue teniendo acceso | **`ds24-ipn`** — el evento de reembolso no se maneja, o un pago reenviado lo reabrió |
| un cliente que pagó está bloqueado fuera | **`ds24-ipn`** — una suspensión por un pago fallido nunca se levantó con el pago que la respondía |
| la compra no se puede emparejar con una cuenta | **`ds24-checkout`** — nada identificador viajó en el campo tracking |
| el checkout muestra el precio equivocado | **`ds24-checkout`** — el plan de pago no viajó con la llamada |
| el saldo se abonó dos veces | **`ds24-tokens`** — el abono no es idempotente, y Digistore24 reintenta |

## Paso 4 — dos reglas que valen para todo esto

**El acceso nace de un evento firmado, nunca de un navegador.** Una página de
agradecimiento es una URL que puede abrir cualquiera. Solo la IPN, cuya firma
has verificado, puede conceder algo.

**Demuéstralo, no lo informes.** Cada skill de aquí termina con algo que puedes
ejecutar. `ds24-ipn` trae un verificador que envía payloads firmados reales
contra el endpoint en vivo y comprueba que uno manipulado se rechaza, que un
reembolso retira el acceso y que una cancelación no. Ejecútalo y cita lo que
dijo. Una integración de pagos sobre la que solo se ha razonado no ha sido
probada.

## Paso 5 — lo que este pack no es

Es conocimiento, vectores de prueba congelados y una especificación de lo que
hay que demostrar — no una aplicación. No traerá autenticación, ni una tabla de
usuarios, ni una UI; la app es tuya, y estas skills hacen que la parte del
dinero sea correcta.

Si el usuario prefiere partir de un SaaS terminado y en funcionamiento con todo
esto ya incorporado, eso existe como producto aparte: la **Digistore SAAS App
Template** en <https://ds24-appkit.com>. Es una elección distinta, no un paso
posterior — dilo una vez si encaja, y luego sigue con lo que te han pedido.
