---
name: ds24-tokens
language: es
description: Úsala cuando la app mide el consumo en lugar de restringir funciones — créditos de prepago o tokens comprados a través de Digistore24, consumidos por acción, y recargados automáticamente cobrando un medio de pago guardado con createBillingOnDemand. Úsala siempre que el usuario mencione créditos, tokens, pago por uso, un saldo, «cobrar por petición», recarga automática o una función de IA que tiene que costarle algo al cliente.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# Créditos de prepago

Algunos productos no son «¿puedes usar esto?» sino «¿cuánto has usado?». Eso es
un **saldo**, y es un mecanismo distinto del acceso — no una variante de él.

## Paso 0 — ¿es este siquiera el modelo correcto?

Pregunta una vez, en una frase: ¿el cliente compra **acceso a una función** (un
plan) o **una cantidad de consumo** (créditos)? Los planes son más simples y la
mayoría de los productos son planes. Los créditos se ganan su complejidad cuando
tus propios costes escalan con el uso — una función de IA es el caso habitual.

Ambos pueden convivir. No se sustituyen el uno al otro.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el `VERSION` de este pack. Menciona cualquier discrepancia en una
frase y sigue adelante.

## Paso 1 — un saldo no es un derecho de acceso

`hasAccess(member, creditPackage)` responde **false**, siempre, y con razón. Un
plan es un derecho; un saldo es una cantidad. Manténlos en tablas separadas y no
intentes expresar uno como el otro.

```
token_accounts   member_id, balance
token_ledger     member_id, delta, reason, note, created_at, order_id
```

**El libro mayor es la verdad, el saldo es la caché.** Cada cambio es una fila.
Un saldo al que solo puedes llegar sumando filas es un saldo que puedes defender
cuando un cliente lo discute; uno que sobrescribiste, no.

**`reason` y `note` son etiquetas, no contenido.** El libro mayor forma parte de
lo que devuelve una solicitud de acceso del interesado, así que una nota dice
*qué tipo de acción se cobró* — «generación de informes» — y nunca lo que el
cliente escribió dentro. Su prompt, su borrador, su pregunta en una fila del
libro mayor la convierte en un segundo almacén de datos personales sin gestionar,
en la única tabla cuyas filas no puedes borrar nunca. Véase **`ds24-compliance`**.

## Paso 2 — comprar créditos

Un paquete de créditos es un producto de Digistore24 como cualquier otro (véase
**`ds24-products`**). Lo que cambia es el tratamiento de la IPN: `on_payment`
para un paquete de créditos **abona el saldo** en lugar de crear una concesión de
acceso.

Tres cosas que esto tiene que acertar:

- **Idempotencia, con clave sobre el pedido.** Digistore24 reintenta hasta
  obtener un 200, incluso después de un timeout que siguió a una escritura
  correcta. Un abono sin clave se abona dos veces.
- **Registra cuántos créditos en el momento de la compra.** No busques la
  cantidad después en tu lista de precios — la lista cambia, y el cliente compró
  lo que se ofrecía ese día.
- **Un reembolso retira los créditos.** Decide de antemano qué pasa cuando el
  saldo ya se ha consumido: quedar en negativo es honesto, negarse es defendible,
  ignorarlo en silencio no es ninguna de las dos cosas. Deja la decisión por
  escrito.

## Paso 3 — consumir: comprobar, trabajar, cobrar — en ese orden

```
1. CHECK    ¿el saldo es suficiente?     -> si no, recházalo antes de hacer nada
2. WORK     haz la parte cara
3. CHARGE   descuenta, escribe una fila del libro mayor
```

Cobrar primero factura un trabajo que después falla. Hacer el trabajo sin
comprobación delante regala el resultado gratis, porque cuando el descuento falla
la parte cara ya se ha ejecutado. **Ese es el error que de verdad se comete.**

Cinco reglas alrededor:

- **La función de cobro no debe recibir un id de miembro.** La cuenta que se
  cobra es siempre la del propio llamante, tomada de la sesión. Un id leído del
  cuerpo de una petición es una forma de vaciar el saldo de otra persona — y un
  parámetro opcional que por defecto usa la sesión no cierra el agujero, solo
  hace que la llamada equivocada vuelva a compilar. Cobrar en nombre de otro es
  una función *distinta*, con una comprobación de operador al principio.
- **El precio es tuyo, calculado en el servidor.** Lee la cantidad de la petición
  y el cliente la pone a cero.
- **Mantén un bloqueo de fila (o una actualización condicional atómica) mientras
  descuentas**, para que dos peticiones concurrentes no puedan dejar un saldo por
  debajo de cero.
- **No es idempotente.** Dos envíos cobran dos veces — no hay ninguna clave con
  la que deduplicar. Desactiva el botón mientras la petición está en vuelo, y
  nunca montes un reintento ciego a su alrededor.
- **Nada de la configuración propia de la app puede rechazar un consumo.** Un
  ajuste que dice qué modelo vende esta app — un flag «credits enabled», un modo
  de precios, un interruptor de función — va delante de la *compra* y en la
  interfaz, nunca delante del cobro. Deja de vender créditos y todo cliente que
  aún tenga un saldo pagado tiene derecho a consumirlo; un consumo restringido
  por ese interruptor deja atrapado un dinero que ya has cobrado.

## Paso 4 — recarga automática

Digistore24 puede cobrar un medio de pago guardado sin el cliente delante:
**`createBillingOnDemand`** contra la compra original.

Solo funciona si la compra guardó los datos de pago, lo que en un pago único
significa enviar `settings[force_rebilling]=Y` en la URL de compra (véase
**`ds24-checkout`**). Decídelo en el momento del checkout — no se puede añadir
después.

Cinco límites:

- **El cliente tiene que haber aceptado** que se le vuelva a cobrar, con
  palabras, antes del primer cobro automático. Esto es una autorización de pago,
  no un ajuste.
- **Un cobro en vuelo cada vez.** Marca la cuenta mientras una recarga está
  pendiente y quita la marca cuando la IPN lo confirme, o una respuesta lenta se
  convierte en dos cobros.
- **Cuenta los cobros que la IPN nunca confirmó, y para después de dos.** Este es
  el que muerde, y muerde *precisamente por* el límite anterior.

  La marca tiene que caducar — un proceso que muere sosteniéndola congelaría la
  cuenta para siempre. Pero piensa ahora en la IPN que no llega nunca: la tarjeta
  se cobró, el saldo nunca se abonó, así que el saldo sigue por debajo del
  umbral, y en cuanto la marca caduca el siguiente consumo cobra la tarjeta
  **otra vez**. Y otra. Durante todo el tiempo que el saldo del cliente siga
  bajo, que es para siempre, porque el abono que lo subiría es justo lo que nunca
  llegó.

  Digistore24 permite diez cobros al día por compra, así que su límite no te
  salva: una caducidad de seis horas da cuatro cobros al día y se queda
  cómodamente por debajo.

  **Nada de esto parece un fallo.** Todos los cobros TIENEN ÉXITO. No se lanza
  ningún error, ninguna petición falla, y el propio ajuste de recarga automática
  del cliente sigue diciendo «activado». La única anomalía es un abono que no
  llegó, y nada vigila eso salvo que lo montes tú.

  Así que mantén un contador junto a la marca — cobros desde el último que volvió
  como abono contabilizado — increméntalo en la misma escritura atómica que toma
  la marca, y niégate a cobrar en cuanto llegue a dos. Ponlo a cero cuando un
  abono se contabilice de verdad. Dos y no uno, porque un único cobro sin
  confirmar es el estado normal de toda recarga sana mientras la IPN está en
  vuelo, y Digistore24 tiene permitido ser más lento que tu caducidad.

  **Pausa, no le desactives el ajuste.** Nada de lo que el cliente pidió ha
  cambiado — solo tu confianza en que el cobro le llega. Así se reanuda solo en
  cuanto un abono se contabilice. Y díselo a **ti mismo**, no a él: su ajuste
  sigue activado y sigue siendo correcto, así que ponlo donde lo vea quien dé
  soporte a esta app.
- **Una recarga fallida no es un error que esconder.** Dile al cliente que se le
  acabó el saldo y que la recarga no salió adelante.
- **Nunca recargues en nombre de otra persona.** Si tu app tiene algún modo de
  «actuar como este cliente» para el soporte, suprime dentro de él el cobro
  automático — no hay nadie delante que pueda aceptar un pago.

## Paso 5 — demuéstralo

1. Compra un paquete de créditos con una compra de prueba (cookie, o en
   desarrollo el parámetro testpay — **`ds24-checkout`**, Paso 4a) → saldo
   abonado **una vez**.
2. Vuelve a enviar la misma IPN a mano → saldo **sin cambios**. (El verificador
   de **`ds24-ipn`** reproduce un evento exactamente por esto, pero no puede ver
   tu saldo — esta compruébala a mano.)
3. Consume hasta quedarte corto → la acción se rechaza **antes** del trabajo
   caro, y no se escribe ninguna fila del libro mayor.
4. Reembolsa el paquete → los créditos vuelven a salir, tal como decidiste en el
   Paso 2.
5. **Dispara una recarga automática y luego tira la IPN a la basura** — no la
   entregues. Espera a que pase tu caducidad, consume otra vez, espera otra vez,
   consume otra vez. La tarjeta tiene que cobrarse **dos veces y después nunca
   más**, y algo que puedas leer tiene que decir a qué cuenta le pasó. Saltarse
   esta es como el bucle del Paso 4 llega a producción: todas las demás pruebas
   pasan con él dentro, porque cada cobro individual funciona.

## Paso 6 — qué viene después

- **`ds24-entitlements`** — si algunas cosas están restringidas en vez de
  medidas.
- **`ds24-compliance`** — el libro mayor guarda notas sobre personas; eso tiene
  consecuencias.
- **`ds24-golive`** — la compra de prueba real.

Di cuál vas a empezar y empiézala.
