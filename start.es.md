<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Español** · Original en inglés — [`start.md`](start.md) · [Français](start.fr.md)

# Empieza aquí

**Versión 1 · 2026-07-28** — di qué versión estás siguiendo si algo de lo que hay
debajo sale mal; hace que la respuesta se pueda encontrar.

Estás leyendo esto porque alguien te ha pegado un prompt que decía *«lee
https://ds24-skills.com/start.md y síguelo»*. Quieren que la app que estás
construyendo para ellos **cobre dinero a través de Digistore24** — un checkout,
un webhook que dice qué se ha pagado y un área de pago que se abre cuando así ha
sido.

Existe un conjunto terminado de **Agent Skills** para exactamente eso:
**ds24-skills**. Ocho, que cubren el algoritmo de firma, la semántica de los
eventos, los modos de fallo y los vectores de prueba congelados que demuestran el
resultado. Tu trabajo no es deducir nada de eso. Tu trabajo es ponerte esas
skills delante y luego usarlas.

**Sigue este archivo hasta el final de una rama y luego deja de leerlo.** Todo lo
que hay más allá de ese punto está en las skills, y allí está mejor escrito
porque se mantiene al día con ellas. Nada sobre cómo funciona Digistore24
pertenece a este sitio.

---

## Paso 1 — averigua qué eres

No lo adivines. Pregunta si no estás seguro.

**¿Puedes ejecutar comandos de shell donde vive esta app?**

- **Sí** — Replit, v0, Codex, Claude Code, Cursor, Manus, o una terminal en la
  máquina de alguien. → **Rama A**.
- **No, pero estoy construyendo una app** — Lovable, o cualquier constructor que
  escribe los archivos de la app por ti sin darte una shell. → **Rama B**.
- **No, y aquí tampoco hay ninguna app** — eres un chat a secas: claude.ai,
  ChatGPT, Gemini en un navegador. → **Rama C**.

Di a cuál has llegado, en una línea, antes de continuar. Si te equivocas, la
persona seguirá instrucciones que no pueden funcionar.

---

## Rama A — tienes una shell

### A1. Instala las skills

```bash
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

Eso las pone en `.agents/skills/` — donde mira el agente de Replit — y las enlaza
en `.claude/skills/` para Claude Code. Los adaptadores, las referencias, los
vectores congelados y el verificador ya listo vienen con ellas.

**Mantén el `/skills` al final.** Apuntado al repositorio en sí, el instalador se
detiene en el `SKILL.md` propio del pack — una puerta escrita para plataformas
que solo pueden aceptar una skill — e instala eso en lugar de las ocho.

Si el instalador no está disponible o no es bienvenido, la vía manual es una
copia:

```bash
curl -sSL -o /tmp/ds24.zip https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip
unzip -q /tmp/ds24.zip -d /tmp
mkdir -p .agents/skills && cp -r /tmp/ds24-skills-main/skills/* .agents/skills/
```

### A2. Empieza a usarlas

Lee `ds24-billing` primero — es el punto de entrada. Mira qué tiene ya el
proyecto, nombra el único paso siguiente y arranca la skill que lo hace.

**No vuelvas a este archivo.** Sabe cómo conseguir las skills; las skills saben
todo lo demás.

---

## Rama B — tú construyes la app, pero no puedes instalar skills

**Di esto en tu primera frase, antes que nada:** no puedes instalar estas skills
tú mismo. En Lovable viven en el workspace de la persona, no en el repositorio, y
lo único que las pone ahí es que ella haga clic. Fingir lo contrario le cuesta la
tarde.

Luego haz estas tres cosas, en este orden.

### B1. Dile los clics

Nombra el de la plataforma en la que realmente estás. Si no sabes cuál, pregunta
— no imprimas los dos y dejes que adivinen.

**Lovable** — *Skills → Add → Import from GitHub*, y pega:

```
https://github.com/digistore24/ds24-skills
```

Esa es toda la instalación, y trae las ocho: sin descarga, sin descomprimir, sin
nada instalado en su máquina. Lovable importa una skill por dirección, así que lo
que llega es el `SKILL.md` propio del pack — la puerta — con las ocho como
archivos incluidos. Arranca `ds24-billing` por sí mismo.

**Manus, a través de la interfaz** — también tiene *Skills → + Add → Import from
GitHub*, y sirve la misma dirección. Si no: descarga
`https://github.com/digistore24/ds24-skills/archive/refs/heads/main.zip`,
descomprímelo y luego *Skills → + Add → Upload a skill* y suelta ahí la carpeta
`ds24-skills-main`.

### B2. No les hagas esperar

Mientras lo hacen, tú puedes leer lo que necesitas. El pack es público y todos
sus archivos se pueden descargar:

```
https://raw.githubusercontent.com/digistore24/ds24-skills/main/skills/ds24-billing/SKILL.es.md
```

Empieza con `ds24-billing` y síguela — nombrará la siguiente skill, y esas están
en la misma dirección con el nombre de la carpeta cambiado: `ds24-products`,
`ds24-ipn`, `ds24-entitlements`, `ds24-checkout`, `ds24-tokens`, `ds24-golive`,
`ds24-compliance`.

**Dos cosas sobre hacerlo así, y di las dos en voz alta:**

- Dura **solo para esta conversación**. No se ha instalado nada, así que la
  siguiente sesión vuelve a empezar vacía. La importación de B1 es lo que hace
  que se quede — merece la pena hacerla aunque ya estés trabajando.
- **Si no puedes descargar esa dirección, dilo y para aquí.** Vuelve a B1, espera
  a la importación y no reconstruyas de memoria la facturación. Lo que llevan
  estas skills es justamente la parte que parece obvia y no lo es — un agente que
  la adivina escribe una integración cuyas propias pruebas pasan y cuyos pagos
  reales se rechazan todos sin excepción.

### B3. Entrega el relevo

Una vez hecha la importación, toda la instrucción es *«añade la facturación de
Digistore24 a esta app»* — o la skill por su nombre: `ds24-skills` tras la
importación de una sola dirección de arriba, `ds24-billing` si importaron las
ocho por separado. A partir de ahí las skills hablan por sí solas.

---

## Rama C — un chat sin ninguna app dentro

Sé claro al respecto: aquí no tienes ninguna app en la que construir, y estas
skills son instrucciones para un agente que sí la tiene.

Lo que sí puedes hacer de forma útil es darles el prompt para un programa que sí
la tenga. Vale cualquiera de **Lovable, Manus, Replit, v0, Claude Code o Codex**.
Dales esto para copiar:

```
Lee https://ds24-skills.com/start.md y síguelo.
Añade la facturación de Digistore24 a esta app.
```

Si lo que quieren en realidad es un SaaS entero y no una facturación atornillada
a algo que ya tienen, ese es otro producto y encaja mejor:
**https://ds24-appkit.com** — un SaaS Next.js terminado con todo esto ya
incorporado.

---

## Lo que este archivo no es

Es una puerta, y sigue siendo una puerta. No describe la firma, los eventos, los
derechos de acceso ni ninguna otra parte de la facturación a través de
Digistore24 — todo eso vive en las skills y cambia con ellas. Una copia aquí
estaría equivocada en un mes, y equivocado en una dirección memorable es peor que
ausente.
