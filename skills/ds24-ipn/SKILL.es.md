---
name: ds24-ipn
language: es
description: Úsala cuando construyas o arregles el webhook IPN de Digistore24 — el endpoint que recibe los eventos de pago y los convierte en acceso. Cubre la comprobación de la firma SHA512, la correspondencia entre evento y acceso, la idempotencia y un script de verificación que demuestra que el endpoint es correcto. Úsala siempre que el usuario mencione la IPN de Digistore24, un webhook de pago, «firma inválida», compras que no desbloquean nada, reembolsos que no revocan el acceso o una suscripción cancelada que se quedó sin acceso antes de tiempo.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# El endpoint IPN de Digistore24

Aquí se decide quién ha pagado y qué puede usar. Todo lo demás en una
integración con Digistore24 se puede rehacer más adelante; esto tiene que salir
bien a la primera, porque un error aquí o bien deja que un desconocido
desbloquee tu producto, o bien le quita el acceso a quien ya ha pagado.

Lo construyes sobre el stack que el usuario ya tiene. Esta skill no da por
supuesto Next.js, Supabase, Python ni ningún otro: te da el protocolo, un
adaptador para tu runtime y un script que demuestra el resultado.

## Paso 0 — ¿ya existe?

Mira antes de preguntar. Busca en el proyecto `sha_sign`, `on_payment`,
`DIGISTORE_IPN_PASSPHRASE` o una ruta que contenga `ipn`.

- **No aparece nada** → sigue con el Paso 1.
- **Aparece algo** → no lo reconstruyas. Ve al Paso 5 y ejecuta el verificador
  contra lo que hay. Arregla lo que señale, y nada más.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Las skills importadas a un workspace no se actualizan solas. Descarga
`https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION` y
compáralo con el archivo `VERSION` que hay junto a este pack. Si el tuyo es más
antiguo, dilo en una frase y ofrece reimportar el pack — y sigue adelante en
cualquiera de los dos casos. Una copia desactualizada suele seguir siendo
correcta; una copia desactualizada de la que nadie avisa es la vía por la que
vuelve un bug ya corregido.

## Paso 1 — lee el protocolo

Lee ahora **`references/ipn-protocol.es.md`**, entero. Son dos páginas, y
contienen el detalle que le cuesta un día a todo el mundo: Digistore24 firma con
las mayúsculas y minúsculas **originales** de los nombres de campo
(`order_id=…`), no con los nombres pasados a mayúsculas, aunque su propio
ejemplo en PHP sugiera lo contrario. Una implementación que se equivoca aquí
pasa todas sus propias pruebas y rechaza todos los pagos reales.

Luego lee **`references/events.es.md`**. Contiene la tabla que hace corresponder
cada evento con el acceso, incluidas las dos filas que van contra la intuición y
que cuestan dinero cuando alguien las adivina:

- `on_rebill_cancelled` no toca el acceso **en absoluto**.
- `on_payment_missed` **suspende de forma reversible**: es una tarjeta caducada,
  no un cliente que se marcha.

No las leas por encima para escribir luego de memoria. Cada frase está ahí
porque alguien se equivocó en producción.

La tercera referencia, **`references/verification.es.md`**, es para el Paso 5:
lo que hay que demostrar una vez que el endpoint existe. Léela cuando llegues
ahí, no ahora.

## Paso 2 — copia el módulo de firma, no escribas uno

En `adapters/` hay tres implementaciones de la firma. **Copia la que corresponda
a tu runtime, tal cual, y no la edites nunca:**

| Runtime | Archivo |
|---|---|
| Node (runtime Node de Next.js, Express, Nest, Node sin framework) | `adapters/signature-node.mjs` |
| **Deno / Supabase Edge Functions / Lovable Cloud** / Cloudflare Workers / edge de Next.js | `adapters/signature-web.mjs` |
| Python (FastAPI, Django, Flask, sin framework) | `adapters/signature.py` |

Las tres se comprueban contra vectores de prueba congelados, compartidos con la
Digistore SAAS App Template, así que está demostrado que coinciden entre sí y
con una cuenta real de Digistore24. Reescribir una «para que encaje con el
estilo del código» tira esa garantía por la borda a cambio de nada.

Son JavaScript plano con tipos en JSDoc (o Python plano): un proyecto en
TypeScript las importa y conserva la comprobación de tipos completa.

## Paso 3 — construye el endpoint a partir del adaptador correspondiente

Los archivos de endpoint que hay al lado son **ejemplos que adaptas**, no
archivos para copiar a ciegas:

| Stack | Archivo |
|---|---|
| Next.js App Router | `adapters/next-node.ts` |
| **Supabase Edge Function / Lovable Cloud** | `adapters/deno-edge.ts` |
| Express | `adapters/express-node.js` |
| FastAPI | `adapters/python-fastapi.py` |

Sea cual sea el stack, estas cinco propiedades no se negocian:

1. **Lee el cuerpo en bruto y parséalo tú.** Digistore24 envía
   `application/x-www-form-urlencoded`. Un framework que parsea y vuelve a
   serializar puede romper la firma.
2. **Rechaza por defecto (fail closed).** Sin firma → rechazo. Sin passphrase
   configurada → rechazo. «Saltarse la comprobación si falta la passphrase»
   convierte el endpoint en un endpoint público de escritura la primera vez que
   una variable de entorno se pierde en un redespliegue.
3. **Responde `200` a un GET**, y también a `connection_test`. Así es como
   Digistore24 valida el endpoint, y se niega a registrar uno que redirige.
4. **Nunca dejes que una excepción salga del handler.** Digistore24 reintenta
   hasta recibir un 200, así que una excepción se convierte en un bucle de
   reenvío sin fin. Anótala en el log, responde 200 y vuelve a procesar el
   evento a partir del payload en bruto que guardaste.
5. **Guarda el payload en bruto antes de actuar sobre él.** Es el único registro
   que sobrevive a un bug en todo lo que viene después.

**En Lovable Cloud / Supabase hay una sexta**, y saltársela no hace ruido: la
función tiene que desplegarse con **`verify_jwt = false`**. Digistore24 no envía
ningún JWT de Supabase, así que con el valor por defecto activado cada IPN
recibe un 401 antes de que tu código llegue a ejecutarse, y ninguna compra
desbloquea nada — sin ningún error visible en ninguna parte de la app. Ponlo en
`supabase/config.toml`:

```toml
[functions.ds24-ipn]
verify_jwt = false
```

## Paso 4 — las cinco invariantes que no están en el switch

Anótalas en las notas de la propia app, porque en una revisión de código son
invisibles:

- **Cada escritura es idempotente**, con clave `(order_id, event)` — y mediante
  una restricción UNIQUE, no con un `SELECT` seguido de un `INSERT`, que dos
  reenvíos concurrentes atraviesan sin que nada los pare. Digistore24 reintenta
  tras un timeout aunque el trabajo haya salido bien.
- **Terminado es para siempre.** Una vez que el acceso terminó (reembolso,
  contracargo, último día pagado), ningún evento posterior puede reabrirlo. La
  entrega no sigue ningún orden, así que un `on_payment` reenviado puede llegar
  *después* del reembolso. Decide a partir del estado guardado antes de mirar el
  nombre del evento.
- **Un producto que no conoces no concede nada.** La conexión IPN se registra
  con una lista `product_ids`, y `all` — la cuenta entera del vendedor — es una
  configuración normal (ver **`ds24-products`**). Así que a tu endpoint pueden
  llegar, legítimamente, eventos de un embudo de ventas antiguo, de una segunda
  app o del lanzamiento de otra persona. Guarda el payload, responde `200` y no
  concedas nada. Nunca asignes un id de producto desconocido a un plan por
  defecto: eso regala acceso por una compra que nunca fue tuya, y es un error
  que nadie nota hasta que la persona equivocada ya está dentro.
- **Una misma oferta puede tener VARIOS ids de producto — asígnalos todos.**
  Un producto de Digistore24 lleva exactamente un idioma, así que una tienda
  multilingüe vende cada oferta con un producto por idioma
  (**`ds24-products`**), y el payload nombra el que el comprador usó de verdad.
  Si solo asignas el id alemán, todas las compras en inglés caen en la regla
  anterior: un cliente que ha pagado, registrado correctamente, al que no se
  concede nada. Busca el `product_id` del payload entre **todos** los ids de
  todas las ofertas, y resuélvelos a la misma clave de producto.
- **A quién pertenece este pago sigue un ORDEN, y se decide aquí.**
  Primero, el identificador que tu checkout puso en `tracking[custom]`: ese está
  autenticado. Solo después, el correo del comprador, y como suposición **no
  autenticada**: Digistore24 no verifica la dirección que el comprador escribió.
  Y una dirección que coincide con **más de una cuenta se rechaza**, nunca se
  resuelve a favor de la primera fila. La atribución concede y nunca revoca; esa
  es la única razón por la que la vía del correo resulta tolerable. El orden
  completo, sus rechazos y lo que una coincidencia de correo no puede autorizar
  están en el Paso 2 de **`ds24-checkout`** — léelo antes de escribir esta
  parte, porque aquí cada fallo tiene el aspecto de un endpoint que funciona.

## Paso 5 — demuéstralo

No le digas al usuario que funciona. **`references/verification.es.md` dice qué
hay que demostrar** — léelo y construye después la comprobación con lo que esta
plataforma sea capaz de ejecutar.

Dos cosas deciden cómo:

**¿Hay una shell?** Replit, v0, Manus, Claude Code, Codex: sí. Entonces ejecuta
el script que viene con esta skill; solo necesita Node y conexión a la red:

```bash
node scripts/verify-ipn.mjs \
  --url https://<la app>/api/ipn \
  --passphrase "$DIGISTORE_IPN_PASSPHRASE" \
  --probe https://<la app>/api/ds24-selftest --probe-token "$SECRET"
```

**Lovable no tiene.** Allí las skills llevan consigo sus archivos incluidos,
pero la plataforma los lee en vez de ejecutarlos, así que en Lovable este script
es documentación, no una herramienta. Escribe el equivalente como un **test
dentro de la app** (en Lovable Cloud, un test de Deno), que es la forma B de
`verification.es.md`. Y sale incluso más sencillo: un test con acceso a la base
de datos lee el registro de acceso directamente y no necesita ningún endpoint de
sondeo.

**Una regla vale en los dos casos, y es la razón de ser de todo esto:**

> Tu firma tiene que reproducir exactamente cada vector de
> `scripts/vectors.json`. **Nunca calcules los valores esperados con tu propio
> código.** El bug que esto detecta — firmar con los nombres de campo en
> mayúsculas — produce una implementación que coincide consigo misma a la
> perfección y rechaza todos los pagos reales. Una comprobación escrita desde el
> mismo malentendido confirma el bug.

Si construyes tú la comprobación, esa comparación es lo primero que hace.

**Cuenta lo que dijo la ejecución**, incluido lo que no cubrió. Una ejecución
que se saltó la mitad «acceso» es una firma demostrada y una semántica sin
demostrar: dilo así, en vez de darla por verde.

### Cuando no llega ninguna IPN, este script no puede ayudar

Demuestra lo que hace tu endpoint con un payload. Un pago que nunca llegó hasta
él no deja nada que comprobar, y entonces la pregunta es sobre la *conexión*, no
sobre el código. Pregúntale a Digistore24 qué tiene de ese pedido:

```
POST https://www.digistore24.com/api/call/getPurchase/format/json
Header: X-DS-API-KEY: <la clave>
Body:   purchase_id=ABC12345
```

Si allí no consta → no hubo compra (o se hizo en otra cuenta de vendedor). Si
allí consta y en tu app no → la IPN nunca llegó: una URL registrada que ya no
responde, un `domain_id` que otro proyecto sobrescribió, o una lista
`product_ids` en la que este producto no está. Las tres cosas son
**`ds24-products`**, Paso 4 — y las tres fallan sin un mensaje de error en
ninguna parte.

Para comprobar por separado los módulos de firma incluidos, allí donde haya una
shell:

```bash
node scripts/check-adapters.mjs      # los tres runtimes contra los vectores
```

## Paso 6 — qué viene después

El endpoint recibe eventos. Alrededor de él aún tienen que existir tres cosas:

- **`ds24-products`** — consigue la clave de API, crea los productos en
  Digistore24 y registra este endpoint como conexión IPN. Sin eso, nunca te
  llamará nadie. **Empieza por aquí.**
- **`ds24-entitlements`** — el registro de acceso en el que escriben los
  eventos, y la única función que el resto de la app consulta.
- **`ds24-checkout`** — el enlace de compra con el que empieza una compra.

Di cuál vas a empezar, y empiézala.
