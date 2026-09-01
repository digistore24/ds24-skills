<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Español** · Original en inglés — [`README.md`](README.md) · [Français](README.fr.md)

# Digistore24 Skills

**Agent Skills que enseñan a cualquier agente de programación con IA a cobrar a
través de Digistore24 — en la app que esté construyendo, sobre el stack que sea.**

Funciona con **Lovable**, **Manus**, **Replit**, **v0**, **Claude Code**,
**Codex** y con cualquier otra cosa que lea la convención `SKILL.md`.

Esto no es una plantilla ni una librería. Es la parte de una integración de
pagos que un agente no puede adivinar — el algoritmo de firma, la semántica de
los eventos, los modos de fallo — más **vectores de prueba congelados y una
especificación de lo que hay que demostrar**, para que el resultado se demuestre
en lugar de afirmarse.

---

## Instalación

### La vía corta — deja que tu agente lea las instrucciones

Pega esto en aquello con lo que estés construyendo:

```
Lee https://ds24-skills.com/start.md y síguelo.
Añade facturación de Digistore24 a esta app.
```

Averigua en qué se está ejecutando y sigue a partir de ahí — instalando las
skills por su cuenta donde tiene una shell, y diciéndote los dos clics donde no
la tiene. Todo lo que viene a continuación es lo mismo hecho a mano.

**Ni Lovable ni Manus te piden git ni un terminal.** Elige tu fila.

### Lovable — pega una sola dirección

*Skills → Add → Import from GitHub*, y pega:

```
https://github.com/digistore24/ds24-skills
```

Esa es toda la instalación, y trae **las ocho** — sin descarga, sin
descomprimir, sin nada instalado en tu máquina.

Lovable importa una skill por dirección, así que lo que llega es el propio
[`SKILL.es.md`](SKILL.es.md) del Skill Pack: la puerta. Lleva las otras ocho
como archivos incluidos, comprueba si tu copia está al día y arranca
`ds24-billing`, que averigua qué tiene ya tu proyecto. Pide facturación de
Digistore24 y sigue a partir de ahí.

¿Quieres además las individuales como comandos `/` propios — `/ds24-ipn`
mientras depuras un webhook, por ejemplo? Impórtalas igual, una dirección cada
una:

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

1. **[Descarga el Skill Pack como ZIP](https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip)**
   *(github.com/digistore24/ds24-skills/archive/refs/heads/main.zip)*
2. Descomprímelo. Dentro obtienes `ds24-skills-main/` — suelta esa carpeta
   entera para tener el Skill Pack de una vez, o una sola carpeta de `skills/`
   para quedarte solo con esa.
3. En Manus: *Skills* en la barra lateral izquierda → **+ Add → Upload a
   skill**.

Invócalo directamente con `/ds24-skills`, o simplemente pide facturación de
Digistore24.

### Replit, v0, Claude Code, Codex — un solo comando

```bash
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

**La dirección termina en `/skills`, y eso importa.** Apuntado al repositorio en
sí, el instalador se detiene en el propio `SKILL.md` del Skill Pack — la puerta
escrita para Lovable — e instala esa única skill en lugar de las ocho. Con
`/skills` obtienes las ocho, cada una cargándose con su propio disparador, que
es lo que quieres donde existe una línea de comandos.

Se instala en `.agents/skills/` — que es exactamente donde mira el Agent de
Replit — y las enlaza en `.claude/skills/` para Claude Code. Los adaptadores
incluidos, las referencias, los vectores de prueba y el verificador ya listo
vienen con ello.

<details>
<summary>¿Prefieres no ejecutar un paquete npx?</summary>

Justo — el propio instalador lo dice: las skills se ejecutan con todos los
permisos del agente, así que léelas primero. La vía manual es una copia:

```bash
curl -sSL -o /tmp/ds24.zip https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip
unzip -q /tmp/ds24.zip -d /tmp
mkdir -p .agents/skills && cp -r /tmp/ds24-skills-main/skills/* .agents/skills/
```

Usa `.claude/skills/` en lugar de `.agents/skills/` para Claude Code.
</details>

---

Después di **«añade facturación de Digistore24 a esta app»** — o invoca el punto
de entrada por su nombre, `ds24-billing`.

---

## Qué contiene

| Skill | Qué hace |
|---|---|
| **`ds24-billing`** | el punto de entrada: averigua qué existe ya y arranca la siguiente skill adecuada |
| **`ds24-products`** | la clave de API, crear productos, registrar la conexión IPN, la aprobación |
| **`ds24-ipn`** | el webhook: firma, eventos, idempotencia — **y cómo demostrarlo** |
| **`ds24-checkout`** | el enlace de compra, el precio como plan de pago, llevar la identidad del comprador |
| **`ds24-entitlements`** | el registro de acceso y la única función que pregunta la app |
| **`ds24-tokens`** | créditos de prepago, consumirlos, recarga automática |
| **`ds24-golive`** | la comprobación previa, la compra de prueba real, y el reembolso que demuestra la otra mitad |
| **`ds24-compliance`** | el aviso legal, la política de privacidad, la divulgación del Reglamento de IA de la UE (EU AI Act), acceso y supresión |

## Las tres cosas que un agente no puede adivinar

1. **Digistore24 firma con los nombres de campo EN SU FORMA ORIGINAL**
   (`order_id=…`), no en mayúsculas — aunque su propio ejemplo en PHP sugiera lo
   contrario. Equivócate en esto y todas tus propias pruebas pasan mientras
   todos los pagos reales se rechazan como «firma inválida».
2. **`on_rebill_cancelled` no le hace nada al acceso.** La facturación se
   detiene; el periodo pagado sigue corriendo. Terminar el acceso ahí le quita
   al cliente meses que ya ha pagado. El acceso termina en `last_paid_day`.
3. **Un pago fallido suspende de forma reversible.** Una tarjeta caducada no es
   una baja, y el pago que lo arregla debe *levantar* la suspensión — un
   insert-if-absent no lo hará.

## Demostrar que funciona

El texto no puede garantizar que un agente haya construido bien la comprobación
de la firma, y «probablemente correcto» no vale nada en una vía de pago. Por eso
el Skill Pack incluye una **especificación de lo que hay que demostrar** —
[`verification.es.md`](skills/ds24-ipn/references/verification.es.md) — y el
agente construye la comprobación en lo que sea que corra en su plataforma.

**La parte que no se puede improvisar** son los ocho vectores congelados de
[`vectors.json`](skills/ds24-ipn/scripts/vectors.json). Cualquier implementación
tiene que reproducirlos exactamente, y nadie puede recalcularlos con su propio
código:

> El fallo que atrapan — firmar con los nombres de campo en mayúsculas —
> produce una implementación que concuerda consigo misma a la perfección y
> rechaza **todos los pagos reales**. Una prueba escrita por el mismo autor,
> desde el mismo malentendido, confirma el fallo. Por eso los valores esperados
> vienen de fuera.

Son los mismos vectores con los que se mide la [Digistore SAAS App
Template](https://github.com/digistore24/ds24-appkit).

**Donde hay una shell** — Replit, v0, Manus, Claude Code, Codex, o tu propia
máquina — la skill trae un verificador ya listo. Solo necesita Node y una
conexión de red, nada más, así que corre contra una Supabase Edge Function en
Lovable Cloud igual que contra una ruta de Next.js en Replit. Después de
`npx skills add`:

```bash
node .agents/skills/ds24-ipn/scripts/verify-ipn.mjs \
  --url https://your-app.example.com/api/ipn \
  --passphrase "$DIGISTORE_IPN_PASSPHRASE" \
  --probe https://your-app.example.com/api/ds24-selftest --probe-token "$SECRET"
```

**Lovable no tiene shell.** Allí los archivos incluidos viajan con la skill,
pero la plataforma los lee en lugar de ejecutarlos — así que la skill hace que
el agente escriba el equivalente como una prueba *dentro de la app*. Resulta más
simple: una prueba con acceso a la base de datos lee el registro de acceso
directamente y no necesita endpoint de sondeo. O ejecuta el script de arriba
desde tu propia máquina contra la URL desplegada.

En cualquier caso, esto es lo que se demuestra:

| Caso | Debe |
|---|---|
| **los ocho vectores** | **reproducirse exactamente** — se comprueba primero, antes que nada |
| `on_payment` firmado correctamente | ser aceptado, acceso concedido |
| un byte alterado en la firma | ser rechazado |
| sin firma, o sin passphrase | ser rechazado (fallar en cerrado) |
| firma con claves en mayúsculas | ser aceptada |
| el mismo evento dos veces | no acreditar dos veces |
| `on_refund` | quitar el acceso |
| `on_payment_missed` → `on_payment` | suspender, luego restaurar |
| `on_rebill_cancelled` | dejar el acceso **sin cambios** |
| un pago reenviado después de un reembolso | **no** revivir el acceso |

El `--probe` de arriba es lo que necesita la mitad del acceso cuando se
comprueba desde fuera: un endpoint pequeño, protegido por token, que responde
`{"access": true|false}` para un `order_id`, y que se borra de nuevo cuando la
ejecución sale en verde. Déjalo fuera y esas filas se informan como `SKIP` —
nunca se cuentan en silencio como aprobadas.

Para comprobar por separado los módulos de firma incluidos, donde exista una
shell:

```bash
node .agents/skills/ds24-ipn/scripts/check-adapters.mjs
```

## Adaptadores

[`skills/ds24-ipn/adapters/`](skills/ds24-ipn/adapters) contiene dos tipos de
archivo, y la diferencia importa:

**La firma — cópiala literalmente, nunca la edites:**

| Runtime | Archivo |
|---|---|
| Node | `signature-node.mjs` |
| Deno · Supabase Edge Functions · **Lovable Cloud** · Cloudflare Workers | `signature-web.mjs` |
| Python | `signature.py` |

**El endpoint — un ejemplo que adaptas:** `next-node.ts`, `deno-edge.ts`,
`express-node.js`, `python-fastapi.py`.

> **En Lovable Cloud / Supabase, despliega la función con `verify_jwt = false`.**
> Digistore24 no envía ningún JWT de Supabase, así que con el valor por defecto
> activado cada IPN recibe un 401 antes de que tu código se ejecute y cada
> compra desbloquea, en silencio, nada.

## Actualizar

**En Lovable y Manus las skills viven en tu workspace, no en tu repositorio —
así que no se actualizan solas.** Lo que importaste se queda hasta que vuelvas a
importar. Por eso cada skill empieza comparando su propio `VERSION` contra

```
https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION
```

y dice algo cuando difieren.

En todo lo demás, actualizar es el mismo comando que las instaló:

```bash
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

## Qué no es esto

- **No es una app.** Sin autenticación, sin tabla de usuarios, sin UI. Tu agente
  construye eso; estas skills hacen que la parte del dinero sea correcta.
- **La verificación cubre la vía del dinero**, no si cada página de tu app
  comprueba permisos.
- **Es preparación, no asesoramiento legal.** `ds24-compliance` acierta en lo
  obvio y nombra lo que debería ver un abogado.

Si prefieres partir de un SaaS acabado y funcionando con todo esto ya integrado,
ese es otro producto: **[ds24-appkit.com](https://ds24-appkit.com)** — una
plantilla SaaS completa de Next.js que amplías con Claude Code.

## Licencia

MIT — ver [`LICENSE`](LICENSE). Úsalo, cámbialo, lanza productos con él,
véndelos. Sin coste, sin nadie a quien pedir permiso.
