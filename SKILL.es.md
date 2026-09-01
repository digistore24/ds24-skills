---
name: ds24-skills
language: es
description: Úsalo cuando una app tenga que cobrar dinero a través de Digistore24 — un checkout o un botón de compra, un muro de pago, una suscripción, créditos de prepago, el webhook IPN que dice qué se ha pagado, la puesta en producción, o una integración de Digistore24 que se comporta mal (firma inválida, una compra que no desbloqueó nada, un cliente reembolsado que sigue teniendo acceso). Este es el Skill Pack de Digistore24, que lleva ocho skills y arranca la que necesita la tarea. Úsalo siempre que se nombre Digistore24, y siempre que salgan facturación, pagos o un área de pago y Digistore24 sea el proveedor.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# El Skill Pack de Digistore24

Digistore24 es el comerciante registrado (merchant of record) — gestiona el
checkout, cobra el dinero, se ocupa del IVA y de los reembolsos, y le cuenta a tu
app lo que ha pasado mediante un webhook firmado. Este pack es la parte de todo
eso que un agente no puede adivinar — el algoritmo de firma, la semántica de los
eventos, los modos de fallo y los vectores de prueba congelados que demuestran el
resultado.

**Este archivo es la puerta, no el manual.** Dice qué hay aquí y cómo leerlo.
Nada sobre cómo funciona Digistore24 pertenece a esta página — eso está en las
ocho skills, y allí está mejor escrito porque se mantiene al día con ellas.

## Paso 1 — ¿esta copia está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el archivo `VERSION` que está junto a este. Las skills importadas
a un workspace no se actualizan solas, así que una copia puede tener meses.
Menciona cualquier discrepancia en una frase, ofrece volver a importarlas y sigue
adelante en cualquier caso.

## Paso 2 — lee el punto de entrada y síguelo

```
skills/ds24-billing/SKILL.es.md
```

`ds24-billing` mira qué tiene ya el proyecto, nombra el único paso siguiente y
arranca la skill que lo hace. Léela ahora — no planifiques la integración desde
esta página, y no le hagas al usuario preguntas que ella responde mirando.

**Estás leyendo la versión española.** Cada archivo de este pack tiene al lado
una versión `.es.md` y otra `.fr.md`, y el archivo sin sufijo es el original en
inglés. Las tres dicen lo mismo — solo cambia la prosa; las rutas y los comandos
son idénticos. Elige según el idioma en el que escribe el usuario, quédate en él
durante todo el pack y respóndele en ese idioma.

## Paso 3 — cómo leer cualquiera de las ocho

Son archivos dentro de esta skill. Prueba estas vías en orden, y di cuál funcionó
si no fue la primera:

1. **`skills/<name>/SKILL.es.md`** — la copia que viajó con esta skill.
2. **`https://raw.githubusercontent.com/digistore24/ds24-skills/main/skills/<name>/SKILL.es.md`**
   — el mismo archivo desde GitHub, si tu plataforma no conservó las carpetas
   incluidas. Los archivos que lleva una skill cuelgan de la misma dirección,
   añadiendo su entrada `references/…`, `scripts/…` o `adapters/…`.

**Si no funciona ninguna, para y dilo.** No reconstruyas de memoria una
integración de Digistore24. Lo que llevan estas skills es justamente la parte que
parece obvia y no lo es — si la adivinas, el resultado es una integración cuyas
propias pruebas pasan y cuyos pagos reales se rechazan todos sin excepción como
«firma inválida».

## Las ocho

| Skill | Léela cuando |
|---|---|
| **`ds24-billing`** | **siempre la primera** — averigua qué existe y arranca la correcta |
| `ds24-products` | la clave de API, crear productos, registrar la conexión IPN |
| `ds24-ipn` | el webhook: la firma, los eventos, la idempotencia — y cómo demostrarlo |
| `ds24-entitlements` | convertir un pedido pagado en «puede usar el producto» |
| `ds24-checkout` | el enlace de compra, el precio como plan de pago, la página de agradecimiento |
| `ds24-tokens` | los créditos de prepago, consumirlos, la recarga automática |
| `ds24-golive` | la comprobación previa, la compra de prueba real y el reembolso que demuestra el resto |
| `ds24-compliance` | el aviso legal, la política de privacidad, la divulgación del Reglamento de IA de la UE (EU AI Act), la supresión |

## Dos cosas que llevar a todas ellas

**El acceso nace de un evento firmado, nunca de un navegador.** Una página de
agradecimiento es una URL que puede abrir cualquiera. Solo la IPN, cuya firma has
verificado, puede conceder algo.

**Demuéstralo, no lo informes.** Toda skill termina con algo que se puede
ejecutar. Los ocho vectores congelados de
`skills/ds24-ipn/scripts/vectors.json` no pueden recalcularse nunca con tu propio
código — el fallo que atrapan produce una implementación que concuerda consigo
misma a la perfección y rechaza todos los pagos reales, de modo que una prueba
escrita desde el mismo malentendido confirma el fallo. Reprodúcelos exactamente.

## Donde hay una shell

Este archivo existe para las plataformas que importan una skill cada vez —
Lovable y Manus — para que una sola importación traiga las ocho. Donde hay línea
de comandos, instalarlas como ocho skills separadas es mejor, porque así cada una
se carga con su propio disparador:

```
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

Esa dirección termina en `/skills` a propósito: apuntado al repositorio en sí, el
instalador se detiene en este archivo e instala esta única skill en lugar de las
ocho.
