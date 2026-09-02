<!-- Copyright (c) 2026 Digistore24 Inc, St. Petersburg, USA — SPDX-License-Identifier: MIT -->

> **Español** · Original en inglés — [`events.md`](events.md) · [Français](events.fr.md)

# Qué hace cada evento de Digistore24 con el acceso

De esta tabla depende toda la integración. Un error en una fila tiene una de
dos consecuencias: dejas sin acceso a un cliente que paga, o sigues sirviendo a
uno al que ya le has devuelto el dinero.

| Evento | Qué significa | Qué hace con el acceso |
|---|---|---|
| `on_payment` | ha llegado dinero | **concede** acceso — y **levanta** la suspensión, si la hay |
| `on_payment_subscription_signup` | el primer pago de una suscripción | **concede** acceso |
| `on_refund` | se ha devuelto el dinero | **termina** el acceso, para siempre |
| `on_chargeback` | el banco ha revertido el cobro | **termina** el acceso, para siempre |
| `on_payment_missed` | ha fallado un recobro periódico (tarjeta caducada…) | **suspende** el acceso — de forma **reversible** |
| `on_rebill_resumed` | el soporte ha reanudado el recobro periódico | **levanta** la suspensión — y nada más |
| `on_rebill_cancelled` | el comprador o el soporte ha detenido el recobro periódico | **nada en absoluto** |
| `last_paid_day` | el periodo pagado se ha agotado | **termina** el acceso. Es la forma normal en que caduca un acceso comprado |
| `connection_test` | Digistore24 comprueba tu endpoint | nada — responde `200` |

## Las dos filas en las que más gente se equivoca

**`on_rebill_cancelled` no hace nada.** Se envía en el instante en que alguien
cancela — y en un plan anual cancelado el primer mes, eso ocurre once meses
antes de que el acceso tenga que terminar. Lo que se detiene es la facturación;
el acceso sigue corriendo hasta agotar lo pagado. Terminarlo aquí es quitarle
al cliente un tiempo que ya ha pagado — y es, con mucha diferencia, la causa
más frecuente de solicitudes de reembolso en una integración de Digistore24.

**El acceso termina en `last_paid_day`, no con la cancelación.** Ese evento
llega cuando el periodo pagado se ha agotado de verdad, normalmente a primera
hora de la mañana. Es la otra mitad de la fila anterior, y las dos solo tienen
sentido juntas: déjalas una al lado de la otra en tu código, para que a nadie
se le ocurra «simplificar» quitando una de ellas.

## Suspender no es cancelar

`on_payment_missed` es un cliente al que le ha caducado la tarjeta, no un
cliente que se ha ido. Retira el acceso de forma **reversible**: márcalo como
suspendido, nunca como terminado. Cuando el pago por fin se cobre, llegará un
`on_payment`, y ese `on_payment` tiene que **levantar** la suspensión.

Dos consecuencias que se pasan por alto con facilidad:

- **Levantar la suspensión no es la misma operación que conceder el acceso.**
  Si tu ruta de «conceder acceso» es un «insertar si no existe», con una fila
  que ya existe no escribe nada — la suspensión sobrevive al pago que venía a
  resolverla, y un cliente que acaba de pagar sigue sin acceso. Trata de forma
  explícita el caso «ya existe y está suspendido».
- **`on_rebill_resumed` no puede crear acceso jamás.** Es un clic del soporte,
  sin ningún pago detrás. Si hay una suspensión, la levanta; si no, no hace
  nada. Tratarlo como un pago regala acceso a cualquiera que haya tenido una
  suscripción alguna vez.

## Terminado es para siempre

Cuando el acceso ha terminado — por reembolso, por contracargo o porque llegó
el último día pagado — **ningún evento posterior puede reabrirlo.** Como las
entregas no llegan en orden (mira `ipn-protocol.es.md`), un `on_payment`
reenviado puede llegar después del `on_refund`, y un «reanudar el recobro
periódico» del soporte puede llegar meses después de la caducidad. Comprueba
primero el *estado* del registro, antes siquiera de mirar el nombre del evento.

Guarda **por qué** terminó (reembolso / contracargo / caducidad). Un
«terminado» a secas no distingue un reembolso de una caducidad normal, y cuando
un cliente escribe al soporte, cada uno de los dos pide la respuesta contraria.

## No decidas a partir de un estado

Es tentador reducir cada evento a un puñado de palabras — `paid`, `cancelled`,
`refunded` — y decidir el acceso a partir de esa palabra. **No lo hagas.** Para
el registro de un pedido, tanto `on_rebill_cancelled` como `last_paid_day`
significan «cancelado»; para el acceso, significan lo contrario el uno del
otro. Esa correspondencia pierde información justo donde perderla cuesta
dinero.

Conserva el **nombre del evento sin transformar** hasta el punto en que se
decide. Si además guardas un estado del pedido para tus propios informes,
derívalo aparte — y nunca hagas pasar por él la decisión de acceso.

## El acceso es un registro aparte

Son tres cosas, y no son la misma:

| Registro | A qué responde | Nunca sirve para |
|---|---|---|
| **pedido** | si se ha movido dinero, cuánto y cuándo | decidir el acceso — es un registro financiero |
| **suscripción** | lo que Digistore24 cree sobre la facturación | decidir el acceso — detrás de una suscripción cancelada sigue habiendo un cliente que paga, hasta `last_paid_day` |
| **acceso / derecho de acceso** | si esta persona puede usar este producto | la contabilidad |

Pregunta siempre al registro de acceso. Un `deleted`/`cancelled` en la
suscripción habla de la *facturación*, y el cliente que canceló ayer sigue
teniendo derecho de acceso hoy.

## Una misma persona puede tener dos planes a la vez

En Digistore24, un cambio de plan detiene el recobro periódico del plan antiguo
y abre una compra nueva. Los dos eventos llegan **con días de diferencia y en
cualquier orden**. Durante un upgrade, por tanto, el cliente tiene los dos
planes — o, durante un momento, ninguno.

Así que la pregunta es «¿tiene esta persona el plan X?», funcionalidad por
funcionalidad. Nunca tomes como «su plan» la primera entrada de una lista: una
app que lo muestre así le enseña el plan equivocado a todos los clientes que
suben de plan.

## Un saldo no es un derecho de acceso

Si vendes créditos de prepago, comprar créditos es comprar una **cantidad**, no
un derecho. La pregunta de acceso responde `false` para esa compra, siempre, y
hace bien. Medir el consumo es otro mecanismo — mira la skill `ds24-tokens`.
