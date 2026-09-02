<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Español** · Original en inglés — [`start.md`](start.md) · [Français](start.fr.md)

# Empieza aquí

**Version 1 · 2026-07-28** — si algo de lo que sigue sale mal, di qué versión
estabas siguiendo: así se puede encontrar la respuesta.

Estás leyendo esto porque alguien te ha pegado un prompt que decía *«lee
https://ds24-skills.com/start.md y síguelo»*. Esa persona quiere que la app que
le estás construyendo **cobre a través de Digistore24**: un checkout, un webhook
que dice qué se ha pagado y un área de pago que se abre cuando el pago está
hecho.

Para exactamente eso existe ya un conjunto terminado de **Agent Skills**:
**ds24-skills**. Son ocho y cubren el algoritmo de firma, la semántica de los
eventos, los modos de fallo y los vectores de prueba congelados que demuestran
el resultado. Nada de eso tienes que averiguarlo tú. Tu tarea es conseguir esas
skills y, después, usarlas.

**Sigue este archivo hasta el final de una rama y, ahí, deja de leerlo.** Todo lo
que viene después está en las skills, y allí está mejor escrito porque se
mantiene al día con ellas. Aquí no hay sitio para nada sobre cómo funciona
Digistore24.

---

## Paso 1 — averigua qué eres

No lo adivines: si tienes dudas, pregunta.

**¿Puedes ejecutar comandos de shell en el sitio donde está esta app?**

- **Sí** — Replit, v0, Codex, Claude Code, Cursor, Manus o una terminal en la
  máquina de alguien. → **Rama A**.
- **No, pero estoy construyendo una app** — Lovable, o cualquier constructor que
  escribe los archivos de la app por ti sin darte una shell. → **Rama B**.
- **No, y aquí tampoco hay ninguna app** — eres un chat sin más: claude.ai,
  ChatGPT, Gemini en un navegador. → **Rama C**.

Antes de seguir, di en una línea a cuál has llegado. Si te equivocas aquí, la
persona seguirá instrucciones que no pueden funcionar.

---

## Rama A — tienes una shell

### A1. Instala las skills

```bash
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

Eso las deja en `.agents/skills/` — donde las busca el agente de Replit — y las
enlaza en `.claude/skills/` para Claude Code. Con ellas vienen los adaptadores,
las referencias, los vectores congelados y el verificador ya listo.

**Conserva el `/skills` del final.** Si lo apuntas al repositorio a secas, el
instalador se queda en el `SKILL.md` del propio pack — una puerta escrita para
plataformas que solo admiten una skill — y es eso lo que instala, en lugar de
las ocho.

Si no dispones del instalador o prefieres no usarlo, la alternativa manual es
copiar los archivos:

```bash
curl -sSL -o /tmp/ds24.zip https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip
unzip -q /tmp/ds24.zip -d /tmp
mkdir -p .agents/skills && cp -r /tmp/ds24-skills-main/skills/* .agents/skills/
```

### A2. Empieza a usarlas

Lee primero `ds24-billing`: es el punto de entrada. Ella mira qué tiene ya el
proyecto, nombra el único paso que toca a continuación y arranca la skill que lo
hace.

**No vuelvas a este archivo.** Este sabe cómo conseguir las skills; las skills
saben todo lo demás.

---

## Rama B — construyes la app, pero no puedes instalar skills

**Dilo en tu primera frase, antes que nada:** estas skills no puedes instalarlas
tú. En Lovable viven en el workspace de la persona, no en el repositorio, y lo
único que las pone ahí es que ella misma haga clic. Fingir lo contrario le hace
perder la tarde.

Después, haz estas tres cosas, en este orden.

### B1. Dile dónde hacer clic

Indica solo los clics de la plataforma en la que estás de verdad. Si no sabes cuál
es, pregunta: no muestres las dos y dejes que adivine.

**Lovable** — *Skills → Add → Import from GitHub*, y pegar esta dirección:

```
https://github.com/digistore24/ds24-skills
```

Esa es toda la instalación, y trae las ocho: nada que descargar, nada que
descomprimir, nada instalado en su máquina. Lovable importa una skill por
dirección, así que lo que llega es el `SKILL.md` del propio pack — la puerta —
con las ocho como archivos incluidos. Ese archivo arranca `ds24-billing` por su
cuenta.

**Manus, desde la interfaz** — también tiene *Skills → + Add → Import from GitHub*,
y sirve la misma dirección. Si no: descargar
`https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip`,
descomprimirlo y, en *Skills → + Add → Upload a skill*, soltar la carpeta
`ds24-skills-main`.

### B2. No la hagas esperar

Mientras ella hace eso, tú puedes leer lo que necesites. El pack es público y
cualquiera de sus archivos se puede descargar:

```
https://raw.githubusercontent.com/digistore24/ds24-skills/main/skills/ds24-billing/SKILL.es.md
```

Empieza por `ds24-billing` y síguela: ella nombrará la siguiente skill, y todas
están en esa misma dirección cambiando el nombre de la carpeta: `ds24-products`,
`ds24-ipn`, `ds24-entitlements`, `ds24-checkout`, `ds24-tokens`, `ds24-golive`,
`ds24-compliance`.

**Dos cosas sobre esta vía, y di las dos en voz alta:**

- Vale **solo para esta conversación**. No se ha instalado nada, así que la
  próxima sesión empieza otra vez de cero. Lo que lo deja fijo es la importación
  de B1, y merece la pena hacerla aunque ya estés trabajando.
- **Si no puedes descargar esa dirección, dilo y párate aquí.** Vuelve a B1,
  espera a que termine la importación y no reconstruyas la facturación de
  memoria. Lo que llevan estas skills es justo la parte en la que lo obvio es lo
  equivocado: un agente que la adivina escribe una integración cuyas propias
  pruebas pasan y que rechaza todos y cada uno de los pagos reales.

### B3. Pasa el testigo

Hecha la importación, la instrucción completa es *«añade la facturación de
Digistore24 a esta app»* — o la skill por su nombre: `ds24-skills` tras la
importación de una sola dirección de arriba, `ds24-billing` si se importaron las
ocho por separado. A partir de ahí, las skills hablan por sí solas.

---

## Rama C — un chat sin ninguna app

Dilo sin rodeos: aquí no tienes ninguna app en la que construir, y estas skills
son instrucciones para un agente que sí la tiene.

Lo útil que puedes hacer es darle el prompt para un programa que la tenga.
Sirve cualquiera de estos: **Lovable, Manus, Replit, v0, Claude Code o Codex**.
Dale esto para que lo copie:

```
Lee https://ds24-skills.com/start.md y síguelo.
Añade la facturación de Digistore24 a esta app.
```

Si lo que quiere en realidad es un SaaS completo, y no una facturación acoplada
a algo que ya tiene, eso es otro producto y le encaja mejor:
**https://ds24-appkit.com** — un SaaS en Next.js terminado, con todo esto ya
incorporado.

---

## Lo que este archivo no es

Es una puerta, y puerta se queda. No describe la firma, los eventos, los
derechos de acceso ni ninguna otra parte de la facturación a través de
Digistore24: todo eso vive en las skills y cambia con ellas. Una copia aquí
estaría equivocada al cabo de un mes, y un texto equivocado en una dirección
fácil de recordar es peor que ninguno.
