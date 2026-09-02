<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Español** · Original en inglés — [`verification.md`](verification.md) · [Français](verification.fr.md)

# Demostrar que el endpoint IPN es correcto

Razonar sobre un handler de IPN no es probarlo. Este documento dice **qué hay
que demostrar** y **qué no se puede improvisar** — no con qué herramienta.
Monta la comprobación con lo que se ejecute en tu plataforma.

## Lo único que no puedes generar por tu cuenta

`../scripts/vectors.json` contiene pares de entrada y salida congelados para la
firma: los parámetros, la passphrase y el SHA512 exacto, en hexadecimal y en
mayúsculas, que Digistore24 calcula a partir de ellos.

**Tu implementación tiene que reproducirlos todos, byte a byte.**

Todo el documento se sostiene sobre esta regla, y el motivo no es pedantería:

> Si calculas las firmas esperadas con tu propio código, lo único que has
> demostrado es que tu código coincide consigo mismo. El fallo del que protege
> esta regla — firmar con los nombres de campo pasados a mayúsculas — da una
> implementación rigurosamente coherente consigo misma, cuyas propias pruebas
> pasan todas y que rechaza **todos los pagos reales**. Una prueba escrita por el
> mismo autor, desde el mismo malentendido, le da la razón al error.

Por tanto: los vectores vienen de fuera. No los recalcules, no «arregles» el
que falle, no regeneres el archivo. Si un vector falla, es tu firma la que está
mal.

Son los mismos vectores con los que la [Digistore SAAS App Template](https://github.com/digistore24/ds24-appkit)
mide su propia implementación; reproducirlos es, por tanto, coincidir con
código que ya está en producción.

Compruébalos **antes que nada**. Un verificador que firma mal no dice nada del
endpoint.

## Dos formas: elige la que tu plataforma pueda ejecutar

### A — desde fuera, por HTTP

Un programa aparte firma los payloads y los manda por POST al endpoint
desplegado. No necesita más que un runtime con acceso a la red, y prueba el
endpoint exactamente como lo alcanza Digistore24: a través del proxy real, del
enrutado real, del framework real.

**Ya existe:** `../scripts/verify-ipn.mjs` hace todo esto. Solo necesita Node y
conexión a la red, y funciona contra cualquier stack. Úsalo allí donde tengas
una shell: Replit, v0, Manus, Claude Code, Codex o tu propia máquina.

**El inconveniente:** desde fuera, el verificador no ve la base de datos. No
puede saber si un reembolso retiró de verdad el acceso. Para eso está el sondeo
que se describe más abajo.

### B — desde dentro de la app

Una prueba en el propio stack de la app: una prueba de Deno en Supabase y
Lovable Cloud, una de vitest/jest en una app de Node, pytest en Python.
Construye el payload firmado, llama al handler (o hace POST a la URL local) y
después **mira directamente en la base de datos**.

**Úsala donde no haya shell** — el caso que importa es Lovable: allí las skills
llegan con sus archivos incluidos, pero la plataforma los lee como material de
consulta, no los ejecuta. Lo que el agente construya *dentro de la app*, en
cambio, se ejecuta sin problema.

**La ventaja que nadie espera:** no hace falta ningún endpoint de sondeo. La
prueba ya tiene acceso a la base de datos, así que lee el registro de acceso
directamente. El endpoint temporal que necesita la forma A, la forma B se lo
ahorra.

**El inconveniente:** ejercita el handler, no el despliegue. Una firma que pasa
en la prueba y falla en producción — porque un proxy reescribe el cuerpo o
porque la passphrase falta en el entorno desplegado — es justo lo que la forma A
habría detectado. Cuando la app esté en producción, ejecuta también la forma A
una vez, aunque tengas que hacerlo desde tu propia máquina.

## Qué hay que demostrar

Las dos formas comprueban lo mismo. Lo que no esté en esta lista es un extra;
lo que falte de ella es un agujero.

**La firma**

| Caso | Debe |
|---|---|
| cada vector de `vectors.json` | reproducirse exactamente |
| payload firmado correctamente | aceptarse |
| un byte invertido en la firma | rechazarse |
| un valor cambiado después de firmar | rechazarse |
| payload sin `sha_sign` | rechazarse |
| ninguna passphrase configurada | rechazarse — **nunca un atajo** |
| firma calculada sobre nombres de campo en mayúsculas | aceptarse |
| `GET` al endpoint | responder `200` — así es como Digistore24 lo valida |

**El ciclo de vida del acceso** — un id de pedido nuevo para cada caso, para que
no interfieran entre sí

| Caso | Debe |
|---|---|
| `on_payment` | conceder acceso |
| el mismo evento entregado dos veces | no conceder dos veces, no acreditar dos veces |
| `on_refund` | quitar el acceso |
| `on_payment` reenviado *después* de `on_refund` | **no** revivir el acceso |
| `on_payment_missed` | suspender — sin acceso, pero el registro no se termina |
| `on_payment` después de `on_payment_missed` | restaurar — la suspensión queda **levantada** |
| `on_rebill_cancelled` | dejar el acceso **sin cambios** |
| `last_paid_day` | terminar el acceso |

Las dos últimas son la pareja que cuesta dinero cuando se adivina. Mira
`events.es.md`.

## El sondeo — solo para la forma A

Para comprobar desde fuera la mitad «acceso», la app tiene que responder a una
sola pregunta. Construye un endpoint pequeño que reciba un `order_id` y devuelva

```json
{ "access": true, "suspended": false }
```

Tres reglas:

- **Protégelo con un bearer token.** Habla de las compras de otras personas.
- **Bórralo cuando la ejecución esté en verde.** Es un fixture de prueba, no una
  funcionalidad. Un endpoint que sobrevive a la prueba es un endpoint que nadie
  se acuerda de proteger.
- **Solo lee, nunca escribe.**

La forma B no necesita nada de esto.

## El informe

**Di qué quedó sin comprobar.** Una ejecución que se saltó la mitad «acceso»
porque no había sondeo no está en verde: es una firma demostrada y una
semántica sin demostrar. `verify-ipn.mjs` imprime `SKIP` justo por eso y nunca
lo cuenta como aprobado; lo que construyas debería hacer lo mismo.

Y cuenta lo que la ejecución dijo de verdad, no que la ejecutaste.
