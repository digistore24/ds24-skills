<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Español** · Original en inglés — [`events.md`](events.md) · [Français](events.fr.md)

# Qué le hace al acceso cada evento de Digistore24

Esta es la tabla de la que cuelga toda la integración. Equivócate en una fila y
o bien dejas fuera a un cliente que paga, o bien sigues sirviendo a uno al que
ya le devolviste el dinero.

| Evento | Qué significa | Qué le hace al acceso |
|---|---|---|
| `on_payment` | ha llegado dinero | **concede** acceso — y **levanta una suspensión** si la hay |
| `on_payment_subscription_signup` | el primer pago de una suscripción | **concede** acceso |
| `on_refund` | el dinero se ha devuelto | **termina** el acceso, para siempre |
| `on_chargeback` | el banco lo ha recuperado | **termina** el acceso, para siempre |
| `on_payment_missed` | un recobro periódico ha fallado (tarjeta caducada…) | **suspende** el acceso — **reversible** |
| `on_rebill_resumed` | el soporte ha reactivado el recobro periódico | **levanta una suspensión** — y nada más |
| `on_rebill_cancelled` | el comprador o el soporte ha detenido el recobro periódico | **nada en absoluto** |
| `last_paid_day` | el periodo pagado ha terminado | **termina** el acceso. Así es como caduca normalmente el acceso comprado |
| `connection_test` | Digistore24 validando tu endpoint | nada — responde `200` |

## Las dos filas en las que la gente se equivoca

**`on_rebill_cancelled` no hace nada.** Se envía en el momento en que alguien
cancela, que para un plan anual cancelado en el primer mes son once meses antes
de que el acceso deba terminar. La facturación se detiene; el acceso sigue
corriendo hasta que se agota lo pagado. Terminar el acceso aquí le quita al
cliente tiempo que ya ha pagado — y es, con diferencia, la forma más común en
que una integración de Digistore24 produce solicitudes de reembolso.

**El acceso termina en `last_paid_day`, no en la cancelación.** Ese evento llega
cuando el periodo pagado ha terminado de verdad, normalmente a primera hora de
la mañana. Es la contraparte de la fila de arriba, y las dos solo tienen sentido
como pareja — mantenlas juntas en tu código para que nadie «simplifique» una de
ellas y la haga desaparecer.

## La suspensión no es la cancelación

`on_payment_missed` es un cliente cuya tarjeta ha caducado, no un cliente que se
ha ido. Quita el acceso de forma **reversible**: márcalo como suspendido, no lo
marques como terminado. Cuando el pago salga adelante, llegará `on_payment` y
tiene que **levantar** esa suspensión.

Dos consecuencias fáciles de pasar por alto:

- **El levantamiento no es la misma operación que la concesión.** Si tu camino
  de «conceder acceso» es un insertar-si-no-existe, no escribe nada para una
  fila que ya existe — así que la suspensión sobrevive al pago que la
  respondía, y un cliente que acaba de pagar se queda fuera. Trata de forma
  explícita el caso «ya existe y está suspendido».
- **`on_rebill_resumed` nunca puede crear acceso.** Es un clic del soporte sin
  ningún pago detrás. Levanta una suspensión si la hay, y por lo demás no hace
  nada. Tratarlo como un pago reparte acceso gratis a cualquiera que alguna vez
  tuvo una suscripción.

## Terminado es para siempre

Una vez que el acceso ha terminado — reembolso, contracargo o el último día
pagado — **ningún evento posterior puede reabrirlo.** Como la entrega no tiene
orden (mira `ipn-protocol.es.md`), un `on_payment` reenviado puede llegar
después del `on_refund`, y un «reactivar el recobro periódico» del soporte puede
llegar meses después de la caducidad. Protege según el *estado* del registro,
antes siquiera de mirar el nombre del evento.

Registra **por qué** terminó (reembolso / contracargo / caducidad). «Terminado»
a secas no distingue un reembolso de una caducidad normal, y esos dos piden
respuestas opuestas cuando un cliente escribe.

## No decidas a partir de un estado

Es tentador asignar cada evento a un pequeño conjunto de palabras — `paid`,
`cancelled`, `refunded` — y luego decidir el acceso a partir de esa palabra.
**No lo hagas.** `on_rebill_cancelled` y `last_paid_day` significan los dos
«cancelado» para un registro de pedido, y significan lo contrario para el
acceso. La asignación pierde información justo en el lugar donde la pérdida
cuesta dinero.

Conserva el **nombre del evento en bruto** hasta la decisión misma. Si además
guardas un estado del pedido para tus propios informes, derívalo por separado —
nunca hagas pasar la decisión de acceso por él.

## El acceso es un registro propio

Existen tres cosas y no son la misma:

| Registro | Responde | Nunca se usa para |
|---|---|---|
| **pedido** | si se movió dinero, cuánto y cuándo | decidir el acceso — es un registro financiero |
| **suscripción** | lo que Digistore24 cree sobre la facturación | decidir el acceso — detrás de una suscripción cancelada sigue habiendo un cliente que paga hasta `last_paid_day` |
| **acceso / derecho de acceso** | si esta persona puede usar este producto | la contabilidad |

Pregúntale siempre al registro de acceso. `deleted`/`cancelled` en una
suscripción es una afirmación sobre la *facturación*, y el cliente que canceló
ayer sigue teniendo derecho hoy.

## Una misma persona puede tener dos planes a la vez

Un cambio de plan en Digistore24 detiene el recobro periódico antiguo y arranca
una compra nueva. Los dos eventos llegan **con días de diferencia, en cualquier
orden**. Durante un upgrade, por tanto, un cliente tiene los dos planes — o,
brevemente, ninguno.

Así que: pregunta «¿tiene esta persona el plan X?» por cada funcionalidad. Nunca
tomes «su plan» como la primera entrada de una lista; una app que lo pinta así
le muestra el plan equivocado a todos los clientes que hacen un upgrade.

## Un saldo no es un derecho de acceso

Si vendes créditos de prepago, una compra de créditos es una **cantidad**, no un
derecho. La pregunta de acceso responde `false` para ella, siempre, y con razón.
Medir el consumo es un mecanismo aparte — mira la skill `ds24-tokens`.
