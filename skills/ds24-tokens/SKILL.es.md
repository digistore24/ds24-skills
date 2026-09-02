---
name: ds24-tokens
language: es
description: Úsala cuando la app mide el uso en vez de restringir funcionalidades — créditos de prepago o tokens que se compran a través de Digistore24, se consumen por acción y se recargan automáticamente cobrando un medio de pago guardado con createBillingOnDemand. Úsala siempre que el usuario hable de créditos, tokens, pago por uso, un saldo, «cobrar por petición», recarga automática o una función de IA que tenga que costarle algo al cliente.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# Créditos de prepago

Hay productos en los que la pregunta no es «¿puedes usar esto?» sino «¿cuánto
has usado?». Eso es un **saldo**, y un saldo es un mecanismo distinto del
acceso — no una variante de él.

## Paso 0 — ¿es este siquiera el modelo correcto?

Hazte la pregunta una sola vez y en una frase: ¿el cliente compra **acceso a
una función** (un plan) o **una cantidad de uso** (créditos)? Los planes son
más sencillos, y la mayoría de los productos son planes. Los créditos solo
justifican su complejidad cuando tus propios costes crecen con el uso — el caso
típico es una función de IA.

Las dos cosas pueden convivir; ninguna sustituye a la otra.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el `VERSION` de este pack. Si no coinciden, dilo en una frase y
sigue adelante.

## Paso 1 — un saldo no es un derecho de acceso

`hasAccess(member, creditPackage)` responde **false**, siempre, y hace bien. Un
plan es un derecho; un saldo es una cantidad. Guárdalos en tablas separadas y
no intentes expresar uno en términos del otro.

```
token_accounts   member_id, balance
token_ledger     member_id, delta, reason, note, created_at, order_id
```

**El libro mayor es la verdad; el saldo, la caché.** Cada cambio es una fila.
Un saldo al que solo se llega sumando filas es un saldo que puedes defender
cuando un cliente lo discute; uno que has sobrescrito, no.

**`reason` y `note` son etiquetas, no contenido.** El libro mayor forma parte
de lo que se entrega en una solicitud de acceso del interesado, así que la nota
dice *qué tipo de acción se cobró* — «generación de informe» — y nunca lo que
el cliente escribió. Su prompt, su borrador o su pregunta en una fila del libro
mayor lo convierten en un segundo almacén de datos personales, sin control, y
justo en la única tabla cuyas filas nunca podrás borrar. Véase
**`ds24-compliance`**.

## Paso 2 — comprar créditos

Un paquete de créditos es un producto de Digistore24 como cualquier otro (véase
**`ds24-products`**). Lo que cambia es el tratamiento de la IPN: para un
paquete de créditos, `on_payment` **abona el saldo** en lugar de crear una
concesión de acceso.

Tres cosas que hay que hacer bien:

- **Idempotencia, con el pedido como clave.** Digistore24 reintenta hasta
  recibir un 200, también después de un timeout que llegó tras una escritura
  correcta. Un abono sin clave se abona dos veces.
- **Registra cuántos créditos son en el momento de la compra.** No busques la
  cantidad más tarde en tu lista de precios: la lista cambia, y el cliente
  compró lo que se ofrecía ese día.
- **Un reembolso retira los créditos.** Decide de antemano qué pasa cuando el
  saldo ya se ha consumido: dejarlo en negativo es honesto, negarse es
  defendible, ignorarlo en silencio no es ni lo uno ni lo otro. Deja la decisión
  por escrito.

## Paso 3 — consumir: comprobar, trabajar, cobrar — en ese orden

```
1. CHECK    ¿el saldo es suficiente?     -> si no, rechaza antes de hacer nada
2. WORK     haz la parte cara
3. CHARGE   descuenta, escribe una fila en el libro mayor
```

Si cobras primero, facturas un trabajo que luego falla. Si haces el trabajo sin
una comprobación delante, regalas el resultado: para cuando el descuento falla,
la parte cara ya se ha ejecutado. **Ese es el error que se comete de verdad.**

Cinco reglas en torno a esto:

- **La función de cobro no debe recibir un id de miembro.** La cuenta que se
  cobra es siempre la de quien llama, tomada de la sesión. Un id leído del
  cuerpo de la petición es una forma de vaciar el saldo de otra persona — y un
  parámetro opcional que por defecto toma la sesión no cierra el agujero: solo
  consigue que la llamada equivocada vuelva a compilar. Cobrar en nombre de
  otro es una función *distinta*, con una comprobación de operador al
  principio.
- **El precio es tuyo y se calcula en el servidor.** Si lees el importe de la
  petición, el cliente lo pondrá a cero.
- **Mantén un bloqueo de fila (o una actualización condicional atómica)
  mientras descuentas**, para que dos peticiones concurrentes no puedan dejar
  el saldo por debajo de cero.
- **No es idempotente.** Dos envíos cobran dos veces — no hay ninguna clave con
  la que deduplicar. Desactiva el botón mientras la petición está en curso y
  nunca montes un reintento a ciegas alrededor.
- **Nada de la propia configuración de la app puede rechazar un consumo.** Un
  ajuste que dice qué modelo vende esta app — un flag «credits enabled», un
  modo de precios, un interruptor de funcionalidad — va delante de la *compra*
  y en la interfaz, nunca delante del cobro. Si dejas de vender créditos, todo
  cliente que aún tenga un saldo pagado tiene derecho a gastarlo; un consumo
  condicionado a ese interruptor deja atrapado un dinero que ya has cobrado.

## Paso 4 — recarga automática

Digistore24 puede cobrar un medio de pago guardado sin que el cliente esté
presente: **`createBillingOnDemand`** sobre la compra original.

Solo funciona si la compra guardó los datos de pago, y para un pago único eso
significa enviar `settings[force_rebilling]=Y` en la URL de compra (véase
**`ds24-checkout`**). Decídelo en el momento del checkout — después ya no se
puede añadir.

Cinco límites:

- **El cliente tiene que haber aceptado**, con palabras, que se le vuelva a
  cobrar, y antes del primer cobro automático. Es una autorización de pago, no
  un ajuste.
- **Un solo cobro en curso a la vez.** Marca la cuenta mientras hay una recarga
  pendiente y retira la marca cuando la IPN la confirme; si no, una respuesta
  lenta se convierte en dos cobros.
- **Cuenta los cobros que la IPN nunca confirmó y detente después de dos.**
  Este es el que hace daño, y hace daño *precisamente por* el límite anterior.

  La marca tiene que caducar — un proceso que muere con ella puesta dejaría la
  cuenta congelada para siempre. Pero ahora piensa en la IPN que no llega
  nunca: la tarjeta se cobró, el saldo nunca se abonó, así que el saldo sigue
  por debajo del umbral, y en cuanto la marca caduca el siguiente consumo cobra
  la tarjeta **otra vez**. Y otra. Mientras el saldo del cliente siga bajo — es
  decir, para siempre, porque el abono que lo subiría es justo el que nunca
  llegó.

  Digistore24 permite diez cobros al día por compra, así que su límite no te
  salva: una caducidad de seis horas da cuatro cobros al día y se queda
  cómodamente por debajo.

  **Nada de esto parece un fallo.** Todos los cobros TIENEN ÉXITO. No se lanza
  ningún error, ninguna petición falla, y el ajuste de recarga automática del
  propio cliente sigue marcando «activada». La única anomalía es un abono que
  no llegó, y nadie vigila eso salvo que lo montes tú.

  Así que lleva un contador junto a la marca — cobros desde el último que
  volvió como abono contabilizado —, increméntalo en la misma escritura atómica
  que pone la marca y niégate a cobrar en cuanto llegue a dos. Ponlo a cero
  cuando un abono se contabilice de verdad. Dos y no uno, porque un único cobro
  sin confirmar es el estado normal de cualquier recarga sana mientras la IPN
  está en camino, y Digistore24 tiene derecho a tardar más de lo que dura tu
  caducidad.

  **Pausa, pero no desactives su ajuste.** Nada de lo que el cliente pidió ha
  cambiado — solo tu confianza en que el cobro le llega. Y así la recarga se
  reanuda sola en cuanto se contabiliza un abono. El aviso es para **ti**, no
  para el cliente: su ajuste sigue activado y sigue siendo correcto, así que
  ponlo donde lo vea quien dé soporte a esta app.
- **Una recarga fallida no es un error que haya que esconder.** Dile al cliente
  que se le acabó el saldo y que la recarga no se completó.
- **Nunca recargues en nombre de otra persona.** Si tu app tiene algún modo de
  «actuar como este cliente» para el soporte, desactiva dentro de él el cobro
  automático — no hay nadie presente que pueda autorizar un pago.

## Paso 5 — demuéstralo

1. Compra un paquete de créditos con una compra de prueba (la cookie o, en
   desarrollo, el parámetro testpay — **`ds24-checkout`**, Paso 4a) → el saldo
   se abona **una vez**.
2. Vuelve a enviar la misma IPN a mano → saldo **sin cambios**. (El verificador
   de **`ds24-ipn`** reproduce un evento precisamente por esto, pero no puede
   ver tu saldo — esta la compruebas a mano.)
3. Consume hasta que el saldo no alcance → la acción se rechaza **antes** del
   trabajo caro y no se escribe ninguna fila en el libro mayor.
4. Reembolsa el paquete → los créditos se retiran, tal como decidiste en el
   Paso 2.
5. **Dispara una recarga automática y luego descarta la IPN** — no la
   entregues. Espera a que pase tu caducidad, consume otra vez, vuelve a
   esperar, consume otra vez. La tarjeta tiene que cobrarse **dos veces y nunca
   más**, y algo que puedas leer tiene que decir en qué cuenta ha pasado. Si te
   saltas esta prueba, el bucle del Paso 4 llega a producción: con él dentro,
   todas las demás pruebas pasan, porque cada cobro por separado funciona.

## Paso 6 — qué viene después

- **`ds24-entitlements`** — si hay cosas que se restringen en vez de medirse.
- **`ds24-compliance`** — el libro mayor guarda notas sobre personas, y eso
  tiene consecuencias.
- **`ds24-golive`** — la compra de prueba real.

Di cuál empiezas y empiézala.
