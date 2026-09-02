<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Español** · Original en inglés — [`README.md`](README.md) · [Français](README.fr.md)

# Digistore24 Skills

**Agent Skills que enseñan a cualquier agente de programación con IA a cobrar a
través de Digistore24 — sea cual sea la app que esté construyendo y sea cual sea
el stack.**

Funciona con **Lovable**, **Manus**, **Replit**, **v0**, **Claude Code**,
**Codex** y con cualquier otra herramienta que lea la convención `SKILL.md`.

Esto no es una plantilla ni una librería. Es la parte de una integración de
pagos que un agente no puede adivinar — el algoritmo de firma, la semántica de
los eventos, los modos de fallo — más **vectores de prueba congelados y una
especificación de lo que hay que demostrar**, para que el resultado se demuestre
en vez de darse por hecho.

---

## Instalación

### La vía corta — deja que tu agente lea las instrucciones

Pega esto en la herramienta con la que estés construyendo:

```
Lee https://ds24-skills.com/start.md y síguelo.
Añade facturación de Digistore24 a esta app.
```

El agente averigua en qué entorno está corriendo y se ocupa del resto: donde
tiene una shell, instala las skills por su cuenta; donde no la tiene, te dice
los dos clics que hacen falta. Todo lo que sigue es lo mismo, hecho a mano.

**Con Lovable y con Manus no necesitas ni git ni un terminal.** Busca tu
plataforma.

### Lovable — pega una sola dirección

*Skills → Add → Import from GitHub*, y pega:

```
https://github.com/digistore24/ds24-skills
```

Eso es toda la instalación, y trae **las ocho** — sin descargar nada, sin
descomprimir nada, sin instalar nada en tu máquina.

Lovable importa una skill por dirección, así que lo que llega es el
[`SKILL.md`](SKILL.es.md) del propio pack: la puerta. Lleva las otras ocho como
archivos incluidos, comprueba si tu copia está al día y arranca `ds24-billing`,
que averigua qué tiene ya tu proyecto. Pide facturación de Digistore24 y el
agente se encarga del resto.

¿Quieres además tener skills sueltas como comandos `/` propios — `/ds24-ipn`
mientras depuras un webhook, por ejemplo? Impórtalas de la misma forma, cada
una con su dirección:

```
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-billing
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-products
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-ipn
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-entitlements
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-checkout
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-tokens
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-golive
https://github.com/digistore24/ds24-skills/tree/main/skills/ds24-compliance
```

### Manus — una dirección, o el ZIP

Manus también tiene *Skills → **+ Add → Import from GitHub***, y ahí funciona la
misma dirección.

Si prefieres no pasar por GitHub:

1. **[Descarga el pack como ZIP](https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip)**
   *(github.com/digistore24/ds24-skills/archive/refs/heads/main.zip)*
2. Descomprímelo. Dentro encontrarás `ds24-skills-main/`: sube esa carpeta
   entera para tener el pack completo de una vez, o solo una de las carpetas de
   `skills/` si quieres únicamente esa skill.
3. En Manus: *Skills* en la barra lateral izquierda → **+ Add → Upload a
   skill**.

Invócalo directamente con `/ds24-skills`, o pide facturación de Digistore24 sin
más.

### Replit, v0, Claude Code, Codex — un solo comando

```bash
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

**La dirección termina en `/skills`, y eso importa.** Si la apuntas al
repositorio en sí, el instalador se queda en el `SKILL.md` del propio pack — la
puerta escrita para Lovable — e instala esa única skill en lugar de las ocho.
Con `/skills` obtienes las ocho, y cada una se carga con su propio disparador,
que es lo que quieres donde hay una línea de comandos.

Las instala en `.agents/skills/` — justo donde mira el Agent de Replit — y las
enlaza en `.claude/skills/` para Claude Code. Con ellas vienen los adaptadores
incluidos, las referencias, los vectores de prueba y el verificador ya listo.

<details>
<summary>¿Prefieres no ejecutar un paquete con npx?</summary>

Es comprensible — el propio instalador lo advierte: las skills se ejecutan con
todos los permisos del agente, así que léelas antes. La vía manual es una copia:

```bash
curl -sSL -o /tmp/ds24.zip https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip
unzip -q /tmp/ds24.zip -d /tmp
mkdir -p .agents/skills && cp -r /tmp/ds24-skills-main/skills/* .agents/skills/
```

Para Claude Code, usa `.claude/skills/` en lugar de `.agents/skills/`.
</details>

---

Después di **«añade facturación de Digistore24 a esta app»** — o invoca el punto
de entrada por su nombre, `ds24-billing`.

---

## Qué contiene

| Skill | Qué hace |
|---|---|
| **`ds24-billing`** | el punto de entrada: averigua qué existe ya y arranca la skill que toca a continuación |
| **`ds24-products`** | la clave de API, crear los productos, registrar la conexión IPN, la aprobación |
| **`ds24-ipn`** | el webhook: firma, eventos, idempotencia — **y cómo demostrarlo** |
| **`ds24-checkout`** | el enlace de compra, el precio como plan de pago, llevar la identidad del comprador |
| **`ds24-entitlements`** | el registro de acceso y la única función que la app consulta |
| **`ds24-tokens`** | créditos de prepago, su consumo, la recarga automática |
| **`ds24-golive`** | la comprobación previa, la compra de prueba real y el reembolso que demuestra la otra mitad |
| **`ds24-compliance`** | el aviso legal, la política de privacidad, la divulgación que exige el Reglamento de IA de la UE (EU AI Act), acceso y supresión |

## Las tres cosas que un agente no puede adivinar

1. **Digistore24 firma con los nombres de campo EN SU FORMA ORIGINAL**
   (`order_id=…`), no en mayúsculas — aunque su propio ejemplo en PHP dé a
   entender lo contrario. Si te equivocas aquí, todas tus pruebas pasan y todos
   los pagos reales se rechazan como «firma inválida».
2. **`on_rebill_cancelled` no toca el acceso.** La facturación se detiene; el
   periodo pagado sigue corriendo. Terminar el acceso en ese momento le quita al
   cliente meses que ya ha pagado. El acceso termina en `last_paid_day`.
3. **Un pago fallido suspende de forma reversible.** Una tarjeta caducada no es
   una baja, y el pago que lo arregla tiene que *levantar* la suspensión — un
   insert-if-absent no lo hace.

## Demostrar que funciona

Un texto no puede garantizar que un agente haya construido bien la comprobación
de la firma, y «probablemente bien» no vale nada en un circuito de pagos. Por
eso el pack incluye una **especificación de lo que hay que demostrar** —
[`verification.es.md`](skills/ds24-ipn/references/verification.es.md) — y el
agente construye la comprobación con lo que corra en su plataforma.

**La parte que no se puede improvisar** son los ocho vectores congelados de
[`vectors.json`](skills/ds24-ipn/scripts/vectors.json). Cualquier implementación
tiene que reproducirlos exactamente, y nadie puede recalcularlos con su propio
código:

> El fallo que atrapan — firmar con los nombres de campo en mayúsculas —
> produce una implementación que concuerda consigo misma a la perfección y
> rechaza **todos los pagos reales**. Una prueba escrita por el mismo autor,
> desde el mismo malentendido, confirma el fallo. Por eso los valores esperados
> vienen de fuera.

Son los mismos vectores contra los que se mide la [Digistore SAAS App
Template](https://github.com/digistore24/ds24-appkit).

**Donde hay una shell** — Replit, v0, Manus, Claude Code, Codex o tu propia
máquina — la skill trae un verificador ya listo. Solo necesita Node y conexión
de red, nada más, así que corre igual contra una Supabase Edge Function en
Lovable Cloud que contra una ruta de Next.js en Replit. Después de
`npx skills add`:

```bash
node .agents/skills/ds24-ipn/scripts/verify-ipn.mjs \
  --url https://your-app.example.com/api/ipn \
  --passphrase "$DIGISTORE_IPN_PASSPHRASE" \
  --probe https://your-app.example.com/api/ds24-selftest --probe-token "$SECRET"
```

**Lovable no tiene shell.** Allí los archivos incluidos viajan con la skill,
pero la plataforma los lee en lugar de ejecutarlos — así que la skill hace que
el agente escriba el equivalente como una prueba *dentro de la app*. Y resulta
incluso más sencillo: una prueba con acceso a la base de datos lee el registro
de acceso directamente y no necesita ningún endpoint de sondeo. O ejecuta el
script de arriba desde tu propia máquina contra la URL desplegada.

En cualquier caso, esto es lo que se demuestra:

| Caso | Debe |
|---|---|
| **los ocho vectores** | **reproducirse exactamente** — es lo primero que se comprueba, antes que nada |
| `on_payment` firmado correctamente | ser aceptado, con el acceso concedido |
| un byte alterado en la firma | ser rechazado |
| sin firma, o sin passphrase | ser rechazado — rechazo por defecto (*fail closed*) |
| firma con claves en mayúsculas | ser aceptada |
| el mismo evento dos veces | no acreditar dos veces |
| `on_refund` | quitar el acceso |
| `on_payment_missed` → `on_payment` | suspender, luego restaurar |
| `on_rebill_cancelled` | dejar el acceso **sin cambios** |
| un pago reenviado después de un reembolso | **no** revivir el acceso |

El `--probe` de arriba es lo que necesita la parte de acceso cuando se comprueba
desde fuera: un endpoint pequeño, protegido por token, que responde
`{"access": true|false}` para un `order_id` y que se borra en cuanto la
ejecución sale en verde. Si lo dejas fuera, esas filas se marcan como `SKIP` —
nunca se cuentan en silencio como superadas.

Para comprobar por separado los módulos de firma incluidos, donde haya una
shell:

```bash
node .agents/skills/ds24-ipn/scripts/check-adapters.mjs
```

## Adaptadores

[`skills/ds24-ipn/adapters/`](skills/ds24-ipn/adapters) contiene dos tipos de
archivo, y la diferencia importa:

**La firma — cópiala tal cual, sin editarla nunca:**

| Runtime | Archivo |
|---|---|
| Node | `signature-node.mjs` |
| Deno · Supabase Edge Functions · **Lovable Cloud** · Cloudflare Workers | `signature-web.mjs` |
| Python | `signature.py` |

**El endpoint — un ejemplo que adaptas:** `next-node.ts`, `deno-edge.ts`,
`express-node.js`, `python-fastapi.py`.

> **En Lovable Cloud / Supabase, despliega la función con `verify_jwt = false`.**
> Digistore24 no envía ningún JWT de Supabase, así que con el valor por defecto
> activado cada IPN recibe un 401 antes de que tu código llegue a ejecutarse, y
> ninguna compra desbloquea nada — sin que nada lo avise.

## Actualizar

**En Lovable y en Manus las skills viven en tu workspace, no en tu repositorio —
así que no se actualizan solas.** Lo que importaste se queda tal cual hasta que
lo vuelvas a importar. Por eso cada skill empieza comparando su propio `VERSION`
con

```
https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION
```

y avisa cuando no coinciden.

En todos los demás sitios, actualizar es el mismo comando que las instaló:

```bash
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

## Qué no es esto

- **No es una app.** Sin autenticación, sin tabla de usuarios, sin UI. Eso lo
  construye tu agente; estas skills se ocupan de que la parte del dinero esté
  bien hecha.
- **La verificación cubre el circuito del dinero**, no si cada página de tu app
  comprueba los permisos.
- **Es preparación, no asesoramiento jurídico.** `ds24-compliance` deja bien lo
  evidente y señala lo que debería revisar un abogado.

Si prefieres partir de un SaaS terminado y en funcionamiento, con todo esto ya
integrado, eso es otro producto: **[ds24-appkit.com](https://ds24-appkit.com)**
— una plantilla SaaS completa en Next.js que amplías con Claude Code.

## Licencia

MIT — ver [`LICENSE`](LICENSE). Úsalo, modifícalo, lanza productos con él y
véndelos. Sin coste y sin pedir permiso a nadie.
