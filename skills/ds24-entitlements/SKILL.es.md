---
name: ds24-entitlements
language: es
description: Úsala para decidir qué puede usar de verdad un cliente que ha pagado — el registro de acceso en el que escriben los eventos de Digistore24 y la única función a la que pregunta el resto de la app. Explica por qué el acceso no vive ni en la tabla de pedidos ni en la de suscripciones, cómo restringir una página o una funcionalidad, qué pasa en un upgrade cuando alguien tiene dos planes a la vez y qué hacer con un plan en pausa tras un pago fallido. Úsala también siempre que el usuario pregunte cómo comprobar si alguien ha pagado o cómo reservar una funcionalidad a un plan, o cuente que un cliente que canceló perdió el acceso antes de tiempo.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# Qué puede usar un cliente que paga

La app hace una sola pregunta — *¿puede esta persona usar esto?* — y esa
pregunta tiene que tener exactamente una respuesta, guardada en un solo sitio.
Cada vez que esto ha salido mal, ha salido mal por lo mismo: alguien se lo
preguntó a otra tabla.

## Paso 0 — ¿ya existe?

Busca una tabla de acceso, de derechos de acceso o de concesiones, o una
comprobación del tipo `hasPlan(...)`. Si ya hay algo, no construyas una
segunda: contrasta la que existe con el Paso 2 y el Paso 3.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el `VERSION` de este pack. Si no coinciden, dilo en una frase y
sigue adelante.

## Paso 1 — tres registros, y solo uno responde

| Registro | Responde a | Por qué nunca debe decidir el acceso |
|---|---|---|
| **pedido** | si se movió dinero, cuánto y cuándo | es un registro financiero |
| **suscripción** | lo que Digistore24 cree sobre la facturación | detrás de una suscripción cancelada sigue habiendo un cliente que ha pagado, hasta que acaba el periodo pagado |
| **acceso / concesión** | **si esta persona puede usar este producto** | — |

La fila del medio es la trampa. Alguien cancela el tercer día de un plan anual;
Digistore24 marca la suscripción como cancelada en el acto; al cliente le quedan
once meses. Una app que restringe según el estado de la suscripción lo deja sin
acceso esa misma tarde, y la petición de reembolso que sigue está plenamente
justificada.

**Guarda un registro de acceso aparte.** Su forma mínima:

```
access_grants
  member_id      quién
  product_key    qué
  source         'purchase' | 'manual'
  order_id       de qué compra viene (null para las manuales)
  suspended_at   lo pone on_payment_missed, lo quita on_payment  (reversible)
  ended_at       lo pone refund / chargeback / last_paid_day     (terminal)
  ended_reason   'refund' | 'chargeback' | 'lastPaidDay' | 'revoked'
  access_until   una DATE de fin, solo para concesiones manuales — null para compras
  unique (member_id, product_key, order_id)
```

La IPN lo mantiene. Nada más escribe en él, salvo una concesión manual hecha a
propósito. Qué evento hace qué está en la tabla de eventos de la skill
**`ds24-ipn`**: léela allí antes de escribir una sola línea de esto. Si esa
skill no está instalada, esta no puede terminarse bien: instálala también.

## Paso 2 — una sola función, preguntada por cada funcionalidad

```
hasAccess(memberId, productKey) -> boolean
```

Devuelve verdadero cuando existe una fila para ese par con `ended_at IS NULL`
**y** `suspended_at IS NULL` **y** (`access_until IS NULL` o
`access_until > now`).

Todas las restricciones de la app la llaman a ella. Ninguna página lee la tabla
de concesiones por su cuenta, y ninguna lee pedidos ni suscripciones para
decidir nada.

**Pregúntaselo por funcionalidad, no una vez por usuario.** Un cambio de plan en
Digistore24 detiene el recobro periódico antiguo y arranca una compra nueva, y
esos dos eventos llegan **con días de diferencia y en cualquier orden**:
mientras dura un upgrade, el cliente tiene *los dos* planes o, por un momento,
*ninguno*. El código que da por «su plan» la primera entrada de una lista muestra
el plan equivocado a todos los clientes que suben de plan.

## Paso 3 — tres reglas que no son obvias

**Un pago fallido no debe leerse como el cierre de la cuenta.** Con
`suspended_at` puesto, `hasAccess` responde false, y hace bien — pero el cliente
ve un producto que, sin más, ha desaparecido. Dale a la UI una segunda pregunta,
solo para mostrar («¿está en pausa?»), y dile *tu acceso está en pausa: el
último pago no se pudo cobrar*. Nunca lo dejes sin explicación.

**`access_until` necesita una zona horaria UTC explícita al mostrarse.** Guarda
el final del día que cubre y muéstralo anclado a UTC; de lo contrario, quien lo
mire desde una zona horaria por delante de UTC verá el día siguiente. Dale a
`null` una frase de verdad («sin fecha de fin»), nunca una celda vacía.

**Un saldo de prepago no es un derecho de acceso.** Para un paquete de créditos,
`hasAccess` responde false siempre, y es lo correcto: un plan es un derecho; un
saldo, una cantidad. Ver **`ds24-tokens`**.

## Paso 4 — concesiones manuales, porque el soporte las necesita

Alguien acabará pagando fuera del sistema, o habrá una compra que no se pueda
atribuir. Permite que un operador conceda acceso a mano, con `source = 'manual'`
y un motivo por escrito.

Dos límites que conviene incorporar desde el primer día:

- **Solo las concesiones manuales pueden revocarse a mano.** El acceso comprado
  termina por un evento de Digistore24, nunca por un clic; si no, el soporte
  puede quitarle a un cliente algo que pagó, sin que haya un reembolso de por
  medio.
- **Impón esa regla en la propia escritura**, no solo en la UI que oculta el
  botón. Cualquier handler es, por sí mismo, un endpoint HTTP.

## Paso 5 — demuéstralo

Ejecuta el verificador de la skill **`ds24-ipn`** con `--probe` apuntando a un
endpoint pequeño que se apoye en `hasAccess`. Sus comprobaciones de acceso
prueban exactamente esto: un reembolso retira el acceso, una cancelación no, un
pago fallido suspende de forma reversible, y un pago reenviado no revive un
pedido reembolsado.

Si el verificador responde `SKIP` en esas comprobaciones, nada de lo anterior
se ha probado: dilo así, en vez de darlo por hecho.

## Paso 6 — qué viene después

- **`ds24-tokens`** — si el uso se mide en vez de restringirse.
- **`ds24-golive`** — la compra de prueba real.
- **`ds24-compliance`** — lo que ahora guardas sobre las personas, y a qué te
  obliga eso.

Di cuál vas a empezar y empiézala.
