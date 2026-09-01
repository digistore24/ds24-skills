---
name: ds24-entitlements
language: es
description: Úsala al decidir qué puede usar realmente un cliente que paga — el registro de acceso en el que escriben los eventos de Digistore24, y la única función que pregunta el resto de la app. Cubre por qué el acceso no es la tabla de pedidos ni la tabla de suscripciones, cómo restringir una página o una funcionalidad, los upgrades en los que alguien tiene dos planes a la vez, y un plan en pausa tras un pago fallido. Úsala siempre que el usuario pregunte cómo comprobar si alguien ha pagado, cómo dejar una funcionalidad detrás de un plan, o informe de que un cliente que canceló perdió el acceso demasiado pronto.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# Qué puede usar un cliente que paga

Hay una sola pregunta que hace la app — *¿puede esta persona usar esta cosa?* —
y tiene que tener exactamente una respuesta, en un solo sitio. Cada versión de
esto que sale mal salió mal por preguntarle a otra tabla.

## Paso 0 — ¿ya existe?

Busca una tabla de acceso, de derechos de acceso o de concesiones, o una
comprobación como `hasPlan(...)`. Si algo existe, no construyas una segunda —
contrástala con el Paso 2 y el Paso 3.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el `VERSION` de este pack. Menciona cualquier discrepancia en
una frase y sigue adelante.

## Paso 1 — tres registros, y solo uno responde

| Registro | Responde | Nunca debe decidir el acceso |
|---|---|---|
| **pedido** | si el dinero se movió, cuánto, cuándo | es un registro financiero |
| **suscripción** | lo que Digistore24 cree sobre la facturación | detrás de una suscripción cancelada sigue habiendo un cliente que paga hasta que acaba el periodo pagado |
| **acceso / concesión de acceso** | **si esta persona puede usar este producto** | — |

La fila del medio es la trampa. Alguien cancela el día 3 de un plan anual;
Digistore24 informa de la suscripción como cancelada de inmediato; al cliente le
quedan once meses. Una app que restringe según el estado de la suscripción lo
deja fuera esa misma tarde, y la petición de reembolso está del todo justificada.

**Mantén un registro de acceso aparte.** Forma mínima:

```
access_grants
  member_id      quién
  product_key    qué
  source         'purchase' | 'manual'
  order_id       de qué compra viene (null para manual)
  suspended_at   lo pone on_payment_missed, lo quita on_payment  (reversible)
  ended_at       lo pone refund / chargeback / last_paid_day     (terminal)
  ended_reason   'refund' | 'chargeback' | 'lastPaidDay' | 'revoked'
  access_until   una FECHA de fin, solo para concesiones manuales — null para compras
  unique (member_id, product_key, order_id)
```

La IPN lo mantiene. Nada más escribe en él salvo una concesión manual
deliberada. Qué hace cada evento está en la tabla de eventos de la skill
**`ds24-ipn`** — léela allí antes de escribir nada de esto. Si esa skill no está
instalada, esta no se puede terminar correctamente: instálala también.

## Paso 2 — una sola función, preguntada por cada funcionalidad

```
hasAccess(memberId, productKey) -> boolean
```

Verdadero cuando existe una fila para ese par con `ended_at IS NULL` **y**
`suspended_at IS NULL` **y** (`access_until IS NULL` o `access_until > now`).

Cada restricción de la app la llama. Ninguna página lee la tabla de concesiones
por su cuenta, y ninguna página lee pedidos ni suscripciones para decidir nada.

**Pregúntala por funcionalidad, no una vez por usuario.** Un cambio de plan en
Digistore24 detiene el recobro periódico antiguo y arranca una compra nueva, y
los dos eventos llegan **con días de diferencia, en cualquier orden** — así que
durante un upgrade un cliente tiene *ambos* planes, o brevemente *ninguno*. El
código que toma «su plan» como la primera entrada de una lista muestra el plan
equivocado a todos los clientes que hacen upgrade.

## Paso 3 — tres reglas que no son obvias

**Un pago fallido no debe leerse como el cierre de una cuenta.** Cuando
`suspended_at` está puesto, `hasAccess` dice false con razón — y el cliente ve
un producto que simplemente se ha esfumado. Dale a la UI una segunda pregunta,
solo de visualización («¿está esto en pausa?»), y di *tu acceso está en pausa,
el último pago no salió adelante*. Nunca nada en absoluto.

**`access_until` necesita una zona horaria UTC explícita al mostrarse.** Guarda
el final del día que cubre y muéstralo fijado a UTC — si no, todo el que lo vea
desde una zona por delante de UTC lee el día siguiente. Dale a `null` una frase
de verdad («sin fecha de fin»), nunca una celda vacía.

**Un saldo de prepago no es un derecho de acceso.** `hasAccess` responde false
para un paquete de créditos para siempre, y eso es correcto: un plan es un
derecho, un saldo es una cantidad. Ver **`ds24-tokens`**.

## Paso 4 — concesiones manuales, porque el soporte las necesita

Alguien pagará fuera del sistema, o una compra no se conseguirá atribuir.
Permite que un operador conceda acceso a mano, con `source = 'manual'` y un
motivo por escrito.

Dos límites que vale la pena incorporar desde el principio:

- **Solo las concesiones manuales pueden revocarse a mano.** El acceso comprado
  termina por un evento de Digistore24, nunca por un clic — de lo contrario el
  soporte puede quitar algo que un cliente pagó sin ningún reembolso asociado.
- **Impón eso en la propia escritura**, no solo en la UI que esconde el botón.
  Cualquier handler es un endpoint HTTP por derecho propio.

## Paso 5 — demuéstralo

Ejecuta el verificador de la skill **`ds24-ipn`** con `--probe` apuntando a un
endpoint pequeño respaldado por `hasAccess`. Eso es exactamente lo que prueban
sus comprobaciones de acceso: un reembolso retira el acceso, una cancelación no,
un pago fallido suspende de forma reversible, y un pago reenviado no revive un
pedido reembolsado.

Si el verificador dice `SKIP` en esas, nada de lo anterior ha sido probado —
dilo en vez de informar de ello como hecho.

## Paso 6 — qué viene después

- **`ds24-tokens`** — si el uso se mide en vez de restringirse.
- **`ds24-golive`** — la compra de prueba real.
- **`ds24-compliance`** — lo que ahora guardas sobre las personas, y a qué te
  obliga eso.

Di cuál vas a empezar y empiézala.
