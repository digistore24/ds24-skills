<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Español** · Original en inglés — [`verification.md`](verification.md) · [Français](verification.fr.md)

# Demostrar que el endpoint IPN es correcto

Un handler de IPN sobre el que solo se ha razonado no ha sido probado. Este
documento dice **qué hay que demostrar** y **qué no se puede improvisar** — no
qué herramienta usar. Construye la comprobación con lo que sea que se ejecute en
tu plataforma.

## Lo único que no puedes generar tú mismo

`../scripts/vectors.json` contiene pares congelados de entrada/salida para la
firma: parámetros, passphrase y el SHA512 exacto, en hexadecimal en mayúsculas,
que Digistore24 produce para ellos.

**Tu implementación tiene que reproducirlos todos, byte a byte.**

Esta es la regla que sostiene todo el documento, y la razón no es la pedantería:

> Si calculas las firmas esperadas con tu propio código, has demostrado que tu
> código está de acuerdo consigo mismo. El fallo del que esto protege — firmar
> con los nombres de campo en mayúsculas — produce una implementación
> perfectamente coherente consigo misma cuyas propias pruebas pasan todas y que
> rechaza **todos los pagos reales**. Una prueba escrita por el mismo autor,
> desde el mismo malentendido, está de acuerdo con el error.

Así que: los vectores vienen de fuera. No los recalcules, no «arregles» uno que
falla, no regeneres el archivo. Un vector que falla significa que tu firma está
mal.

Son los mismos vectores contra los que la [Digistore SAAS App
Template](https://github.com/digistore24/ds24-appkit) mide su propia
implementación, así que reproducirlos significa estar de acuerdo con código que
está en producción.

Compruébalos **primero**. Un verificador cuya firma está mal no dice nada sobre
un endpoint.

## Dos formas, elige la que tu plataforma pueda ejecutar

### A — desde fuera, por HTTP

Un programa aparte firma payloads y los envía por POST al endpoint desplegado.
Necesita un runtime con acceso a la red y nada más, y prueba el endpoint
exactamente como llega a él Digistore24 — a través del proxy real, el
enrutamiento real, el framework real.

**Ya hecho:** `../scripts/verify-ipn.mjs` hace todo esto. Necesita Node y una
conexión de red, nada más, y funciona contra cualquier stack. Úsalo dondequiera
que tengas una shell — Replit, v0, Manus, Claude Code, Codex o tu propia
máquina.

**El inconveniente:** desde fuera, el verificador no puede ver la base de datos.
Si un reembolso ha quitado de verdad el acceso le resulta invisible. Para eso
está el sondeo de más abajo.

### B — desde dentro de la app

Una prueba en el stack propio de la app — una prueba de Deno en Supabase y
Lovable Cloud, una prueba de vitest/jest en una app de Node, pytest en Python.
Construye el payload firmado, llama al handler (o hace POST a la URL local) y
luego **mira directamente la base de datos**.

**Usa esto donde no haya shell** — Lovable es el caso que importa: allí las
skills viajan con sus archivos adjuntos, pero la plataforma los lee como
material de referencia en lugar de ejecutarlos. Cualquier cosa que el agente
construya *dentro de la app* se ejecuta sin problema.

**La ventaja que nadie espera:** ningún endpoint de sondeo. La prueba ya tiene
acceso a la base de datos, así que lee el registro de acceso directamente. La
forma A necesita un endpoint temporal que la forma B no necesita.

**El inconveniente:** ejercita el handler, no el despliegue. Una firma que
funciona en una prueba y falla en producción — porque un proxy reescribe el
cuerpo, o porque falta la passphrase en el entorno desplegado — es exactamente
lo que la forma A habría cazado. Cuando la app esté en producción, ejecuta
también la forma A una vez, aunque tengas que hacerlo desde tu propia máquina.

## Qué hay que demostrar

Las dos formas comprueban las mismas cosas. Todo lo que no esté en esta lista es
un extra; todo lo que falte de ella es un agujero.

**La firma**

| Caso | Debe |
|---|---|
| cada vector de `vectors.json` | reproducirse exactamente |
| payload firmado correctamente | ser aceptado |
| un byte cambiado en la firma | ser rechazado |
| un valor cambiado después de firmar | ser rechazado |
| ningún `sha_sign` en el payload | ser rechazado |
| ninguna passphrase configurada | ser rechazado — **nunca un atajo** |
| firma sobre nombres de campo en mayúsculas | ser aceptada |
| `GET` sobre el endpoint | responder `200` — Digistore24 lo valida así |

**El ciclo de vida del acceso** — un order id nuevo por caso, para que no puedan
interferir

| Caso | Debe |
|---|---|
| `on_payment` | conceder acceso |
| el mismo evento entregado dos veces | no conceder dos veces, no acreditar dos veces |
| `on_refund` | quitar el acceso |
| `on_payment` reenviado *después* de `on_refund` | **no** revivir el acceso |
| `on_payment_missed` | suspender — acceso fuera, registro no terminado |
| `on_payment` después de `on_payment_missed` | restaurar — la suspensión queda **levantada** |
| `on_rebill_cancelled` | dejar el acceso **sin cambios** |
| `last_paid_day` | terminar el acceso |

Las dos últimas son la pareja que cuesta dinero cuando se adivina. Mira
`events.es.md`.

## El sondeo — solo la forma A

Para comprobar la mitad del acceso desde fuera, la app tiene que responder a una
pregunta. Construye un endpoint pequeño que tome un `order_id` y devuelva

```json
{ "access": true, "suspended": false }
```

Tres reglas:

- **Protégelo con un bearer token.** Informa sobre las compras de otras
  personas.
- **Bórralo cuando la ejecución esté en verde.** Es un fixture de prueba, no una
  funcionalidad. Un endpoint que sobrevive a la prueba es un endpoint que nadie
  se acuerda de asegurar.
- **Lee, nunca escribe.**

La forma B no necesita nada de esto.

## Informar

**Di lo que no se comprobó.** Una ejecución que se saltó la mitad del acceso
porque no existía ningún sondeo no es una ejecución en verde — es una firma
demostrada y una semántica sin demostrar. `verify-ipn.mjs` imprime `SKIP`
exactamente por esta razón y nunca lo cuenta como un aprobado; lo que tú
construyas debería hacer lo mismo.

Y informa de lo que dijo realmente la ejecución, no de que la has ejecutado.
