---
name: ds24-skills
language: es
description: Úsalo cuando una app tenga que cobrar a través de Digistore24 — un checkout o un botón de compra, un muro de pago, una suscripción, créditos de prepago, el webhook IPN que avisa de qué se ha pagado, la puesta en producción, o una integración de Digistore24 que da problemas (firma inválida, una compra que no desbloqueó nada, un cliente reembolsado que conserva el acceso). Este es el Skill Pack de Digistore24: contiene ocho skills y arranca la que pide la tarea. Úsalo siempre que aparezca Digistore24, y siempre que se hable de facturación, de pagos o de un área de pago con Digistore24 como proveedor.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# El Skill Pack de Digistore24

Digistore24 es el comerciante de registro (merchant of record): gestiona el
checkout, cobra el dinero, se encarga del IVA y de los reembolsos, y le comunica
a tu app lo que ha ocurrido mediante un webhook firmado. Este pack reúne la parte
de todo eso que un agente no puede adivinar: el algoritmo de firma, la semántica
de los eventos, los modos de fallo y unos vectores de prueba congelados que
demuestran el resultado.

**Este archivo es la puerta, no el manual.** Dice qué hay aquí y cómo leerlo.
Nada de cómo funciona Digistore24 tiene sitio en esta página: eso vive en las
ocho skills, y allí está mejor escrito porque se mantiene al día junto con ellas.

## Paso 1 — ¿esta copia está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el archivo `VERSION` que hay junto a este. Las skills importadas
a un workspace no se actualizan solas, así que una copia puede llevar meses de
retraso. Si no coinciden, dilo en una frase, ofrece volver a importarlas y sigue
adelante en cualquiera de los dos casos.

## Paso 2 — lee el punto de entrada y síguelo

```
skills/ds24-billing/SKILL.es.md
```

`ds24-billing` mira qué tiene ya el proyecto, nombra el único paso que toca a
continuación y arranca la skill que lo hace. Léela ahora: no planifiques la
integración desde esta página, y no le hagas al usuario preguntas que ella
responde con solo mirar.

**Estás leyendo la versión en español.** Cada archivo de este pack tiene al lado
una versión `.es.md` y otra `.fr.md` — el punto de entrada de arriba ya es la
española; la francesa es la misma ruta con `.fr.md`, y el original inglés, la
misma ruta sin sufijo (`SKILL.md`). Las tres dicen lo mismo: solo se traduce la
prosa, los comandos y las rutas de scripts son idénticos en todas, y cada enlace
lleva a la versión en el mismo idioma. Decide según el idioma en el que escribe
el usuario, mantenlo durante todo el pack y responde en él.

## Paso 3 — cómo leer cualquiera de las ocho

Son archivos dentro de esta skill. Prueba estas dos vías en orden y, si la
primera no funcionó, di cuál sí:

1. **`skills/<name>/SKILL.es.md`** — la copia que viajó con esta skill.
2. **`https://raw.githubusercontent.com/digistore24/ds24-skills/main/skills/<name>/SKILL.es.md`**
   — el mismo archivo desde GitHub, por si tu plataforma no conservó las
   carpetas incluidas. Los archivos que acompañan a una skill cuelgan de esa
   misma dirección: basta con añadir su entrada `references/…`, `scripts/…` o
   `adapters/…`.

**Si no funciona ninguna de las dos, para y dilo.** No reconstruyas de memoria
una integración de Digistore24. Lo que llevan estas skills es precisamente la
parte en la que lo obvio es lo equivocado: adivinada, el resultado es una
integración que pasa sus propias pruebas y rechaza todos los pagos reales, uno
por uno, con «firma inválida».

## Las ocho

| Skill | Cuándo leerla |
|---|---|
| **`ds24-billing`** | **siempre la primera** — averigua qué hay ya y arranca la que corresponde |
| `ds24-products` | la clave de API, crear productos, registrar la conexión IPN |
| `ds24-ipn` | el webhook: firma, eventos, idempotencia — y cómo demostrarlo |
| `ds24-entitlements` | convertir un pedido pagado en «puede usar el producto» |
| `ds24-checkout` | el enlace de compra, el precio como plan de pago, la página de agradecimiento |
| `ds24-tokens` | los créditos de prepago, su consumo, la recarga automática |
| `ds24-golive` | la comprobación previa, la compra de prueba real y el reembolso que demuestra el resto |
| `ds24-compliance` | el aviso legal, la política de privacidad, la divulgación de IA que pide el Reglamento de IA de la UE (EU AI Act), la supresión |

## Dos cosas que llevar a todas ellas

**El acceso nace de un evento firmado, nunca de un navegador.** Una página de
agradecimiento es una URL que cualquiera puede abrir. Solo la IPN cuya firma has
verificado puede conceder algo.

**Demuéstralo, no lo afirmes.** Toda skill termina con algo que se puede
ejecutar. Los ocho vectores congelados de `skills/ds24-ipn/scripts/vectors.json`
nunca deben recalcularse con tu propio código: el fallo que atrapan produce una
implementación que concuerda consigo misma a la perfección y rechaza todos los
pagos reales, así que una prueba escrita desde el mismo malentendido no hace más
que confirmar el fallo. Reprodúcelos tal cual.

## Donde hay una shell

Este archivo existe para las plataformas que importan las skills de una en una
— Lovable y Manus —, de modo que una sola importación traiga las ocho. Donde hay
línea de comandos, es mejor instalarlas como ocho skills separadas, porque así
cada una se carga con su propio disparador:

```
npx skills add https://github.com/digistore24/ds24-skills/tree/main/skills
```

Esa dirección termina en `/skills` a propósito: si la apuntas al repositorio
entero, el instalador se detiene en este archivo e instala esta única skill en
lugar de las ocho.
