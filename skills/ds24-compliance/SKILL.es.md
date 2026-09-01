---
name: ds24-compliance
language: es
description: Úsala cuando una app que cobra dinero a clientes de la UE necesita poner en orden su lado legal — el aviso legal, la política de privacidad, las condiciones, la divulgación del Reglamento de IA de la UE (EU AI Act) para todo lo que habla con personas como una máquina, si hace falta siquiera un banner de cookies, los registros de consentimiento y el derecho del cliente a sus datos y a la supresión. Úsala siempre que el usuario pregunte por el RGPD (GDPR), un banner de cookies, un aviso legal, el AI Act, la eliminación de cuentas o qué tiene que mostrar antes de vender a clientes reales.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# Qué exige la UE a una app de pago

Esto es preparación, no asesoramiento jurídico. Deja bien resueltas las cosas
obvias y nombra aquellas que debería mirar un abogado. Dilo una vez, al
principio, y luego sé útil.

El detonante no es el tamaño. Es **cobrar dinero a personas de la UE**, que es
justo para lo que sirve una integración con Digistore24.

## Paso 0 — ¿qué hay ya?

Mira antes de escribir nada. Busca en el proyecto un aviso legal, una política
de privacidad, unas condiciones, un banner de cookies, una tabla de
consentimientos.

**Sea lo que sea lo que exista, no lo reemplaces.** Las páginas legales son con
frecuencia la única parte de una app que un abogado ya ha visto, y una política
de privacidad reescrita que se lee mejor y dice otra cosa es peor que una torpe
pero exacta. Lee lo que hay, contrástalo con los pasos siguientes e informa de
las lagunas — cambiar la redacción es decisión del usuario, no tuya.

Un **banner de cookies que ya esté instalado** es lo único que merece
cuestionarse en voz alta: el Paso 3 explica por qué una app como esta
normalmente no necesita ninguno.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el `VERSION` de este pack. Menciona cualquier discrepancia en
una frase y sigue adelante. El texto legal envejece más rápido que el código —
esta es la skill en la que más importa una copia obsoleta.

## Paso 1 — el inventario va primero

No puedes escribir una política de privacidad veraz a base de imaginación.
**Enumera lo que la app almacena realmente sobre las personas**, tabla por
tabla, antes de escribir una palabra de política:

- cuentas: correo, nombre, marcas de tiempo de inicio de sesión
- pedidos: nombre del comprador, dirección, importe — de Digistore24
- payloads de IPN en bruto: todo lo que envió Digistore24, incluidos los datos
  del comprador
- concesiones de acceso y cualquier nota del operador sobre ellas
- filas del libro mayor y sus notas
- logs, y cuánto tiempo se conservan

Para cada una: **por qué** la conservas, **cuánto tiempo** y **quién más la
ve** — Digistore24, el proveedor de correo, el hosting, cualquier proveedor de
IA. Esa lista es el documento a partir del cual se escribe todo lo demás, y hay
que actualizarla cada vez que se añade una tabla.

**Las notas del operador son datos personales.** Una nota que el soporte
escribió *sobre* un cliente queda cubierta por una solicitud de acceso aunque
la app nunca se la muestre. Ocultarla en la interfaz es una decisión sobre el
tono, no una exención.

**Si los miembros pueden verse o alcanzarse entre sí, dos cosas más entran en
la lista**, y ninguna de las dos es obvia a partir de un diagrama de tablas:

- **Contenido que los propios miembros escribieron** — un perfil, una
  publicación en un espacio compartido, un mensaje privado. La app pasa a
  guardar texto que una persona escribió para que otra lo lea. Entra en la
  solicitud de acceso y tiene que ser alcanzable por una solicitud de
  supresión, y la manera honesta de hacer lo segundo es: vaciar las palabras,
  mantener la fila para que la conversación a su alrededor se siga leyendo, y
  decirlo en la política. Una respuesta a un mensaje borrado no debe
  convertirse en la respuesta a nada.
- **⚠️ La participación misma puede ser información de compra.** Una lista de
  miembros de un espacio de pago es una lista de quién lo compró — y para un
  producto de salud, finanzas o coaching eso se acerca a los datos de
  categorías especiales (art. 9 del RGPD [GDPR]). El diseño seguro es no tener
  ningún listado: sin lista de miembros, sin recuento de miembros, sin «quién
  está aquí». Alguien se hace visible al publicar, que es algo que eligió
  hacer. Si aun así construyes una lista de miembros, dilo en la política de
  privacidad y lee antes el art. 9.

**Y si construyes mensajes privados, decide quién puede leerlos antes de
construirlos, no después.** «Solo los dos participantes» es una promesa que el
código tiene que cumplir: cada consulta que lee un mensaje lleva el id del
propio lector, no hay vista de administrador, y una sesión de soporte que puede
iniciar sesión como un miembro tampoco la obtiene — leer el correo de alguien
no cambia nada y no deja rastro, así que no se puede volver responsable
registrándolo. Las excepciones que merece la pena permitir son una solicitud de
acceso del interesado (respondida a mano, para una solicitud que se hizo) y el
propio reporte de un participante, limitado a lo que eligió adjuntar.

## Paso 2 — el aviso legal

Bajo el § 5 de la DDG alemana (y sus equivalentes en otros países), un sitio
comercial necesita un aviso legal alcanzable: nombre, dirección — **una real,
no un apartado de correos** — correo electrónico, teléfono o una vía de
contacto rápida equivalente, y cuando proceda el NIF-IVA (VAT ID) y la
inscripción en el registro mercantil.

Construye la página y **falla ruidosamente mientras esté vacía**. Un aviso
legal de marcador de posición que se publica es peor que ninguno: es
información visiblemente falsa sobre quién vende.

**El aviso legal viaja en dos formas, y ninguna se transfiere a la otra:**

- **En las páginas, un ENLACE en el pie es la respuesta completa** — llamado
  «Impressum» / «Imprint», a un clic de distancia, en todas las páginas. No
  copies el texto del aviso legal en los pies de página: «fácilmente
  reconocible, directamente alcanzable» pide el enlace, y una segunda copia
  insertada es la que se desvía.
- **En los correos que envía la app** — enlaces de inicio de sesión,
  confirmaciones, avisos — **el CONTENIDO del aviso legal va al final del
  propio correo.** Un correo enviado en el curso de la actividad comercial es
  una carta comercial, y su destinatario no tiene ningún pie que clicar; para
  las sociedades inscritas, las reglas sobre cartas (Alemania: § 35a GmbHG,
  § 125a HGB) piden los datos del prestador *en* la carta. Renderízalo como
  líneas de texto plano bajo el pie del correo — y nunca envíes un marcador de
  posición por correo: ningún bloque de aviso legal en el correo hasta que
  exista el aviso legal real. Se permite una excepción, deliberada: un aviso
  puramente de seguridad construido para no llevar nada clicable se queda
  desnudo, porque un aviso legal contiene direcciones web y de correo y los
  clientes las enlazan automáticamente.

## Paso 3 — probablemente ningún banner de cookies, y eso no es pereza

**Una compra no necesita consentimiento.** Se apoya en el art. 6(1)(b) del RGPD
— la ejecución de un contrato — no en un permiso. Y las cosas que una app de
pago pone normalmente en un dispositivo — la sesión, el idioma, el tema, un «no
me muestres esto otra vez» que el propio cliente clicó — están cubiertas sin
preguntar: o son estrictamente necesarias, o son el resultado directo de que
alguien accione un interruptor.

Así que: **no añadas un banner de cookies a una app que no fija cookies no
esenciales.** Pide un permiso que ni necesitas ni usas, y entrena a la gente a
clicar sin leer el que sí importará más adelante.

⚠️ **La regla trata del DISPOSITIVO, no de las cookies.** El § 25 de la TDDDG y
sus equivalentes cubren `localStorage`, `sessionStorage`, IndexedDB y las
huellas digitales de dispositivo exactamente con las mismas palabras — así que
«no usamos cookies, usamos localStorage» no es una exención, es el mismo acto
con otra API. Merece la pena decirlo porque es el atajo que se toma: una
preferencia guardada sin preguntar está bien cuando la fijó el usuario, y una
analítica guardada sin preguntar es una infracción caiga en el almacenamiento
que caiga.

**Mantén una lista escrita de todo lo que tu app pone en un dispositivo**, con
una línea para cada cosa que diga por qué no necesita consentimiento. Es
aquello a partir de lo cual se escribe la política de privacidad, y es lo que
en silencio deja de ser cierto — cada nueva función que recuerda algo es una
entrada nueva, y nadie se da cuenta de la cuarta.

**Cuando algo sí necesita consentimiento de verdad** — una etiqueta de
analítica, un correo de marketing, un widget de terceros incrustado —
entonces:

- declara la **finalidad**, por separado para cada finalidad
- registra **quién consintió, a qué redacción, cuándo** — y guarda una
  **versión de la redacción**, porque cambiar el texto significa que todo el
  mundo consintió a otra cosa. Después **vuelve a leer esa versión**: un
  consentimiento registrado contra una redacción anterior cuenta como ausente y
  la pregunta se vuelve a hacer. Guardar la versión e ignorarla en el punto en
  el que compruebas es el fallo que parece conforme en la tabla y no lo es en
  el producto
- haz que el registro sea **de solo adición (append-only)**. Una retirada es
  una fila nueva, nunca una edición: tienes que ser capaz de *demostrar* el
  consentimiento (art. 7(1)), y una fila que sobrescribiste no demuestra nada
- retirarlo tiene que ser tan fácil como darlo

## Paso 4 — la divulgación de IA es ley, no texto publicitario

**Art. 50(1) del Reglamento de IA de la UE (EU AI Act), aplicable desde el 2 de
agosto de 2026:** un sistema que interactúa con personas debe dejar claro que
están tratando con una máquina, a más tardar en la primera interacción.

Si la app tiene un chat, un asistente, una respuesta generada — cualquier cosa
que hable con una persona como una máquina — **lo dice, de forma visible, en
todos los idiomas que habla la app**. No en las condiciones. Donde ocurre la
conversación.

Escríbelo como una regla y no como algo puntual: *todo lo que aquí hable con
una persona como una máquina lo dice*. Cualquier función de IA que se añada
después la hereda.

**Convierte esa regla en una LISTA, y haz que algo la recorra.** Un único sitio
que nombre cada superficie donde una máquina habla con una persona, y una
comprobación que falle cuando una superficie de la lista no tiene aviso. Una
regla que nadie puede ejecutar es una regla que aguanta hasta que se publica la
segunda función de IA. Dos cosas sobre la lista:

- **Lo que añade una superficie añade su propia entrada.** Una función
  opcional, un plug-in, una segunda área de la app — la lista central no puede
  enumerar algo que no está activado, así que una parte de la app que trae su
  propia IA trae su propia entrada, y la comprobación recorre todas las listas
  que haya. Una superficie que nadie registró es exactamente la que sale a
  producción sin aviso.
- **Y la comprobación lee todos los almacenes de TEXTO que haya, no solo el
  central.** Una función que trae su propia superficie suele traer su propia
  redacción consigo, allá donde esa parte de la app guarde sus textos. Una
  comprobación que recorre ambas listas pero lee solo el archivo de textos
  principal informa de un aviso que falta para una frase que la app está
  mostrando perfectamente — y una comprobación legal que da falsas alarmas es
  una que la gente aprende a saltarse, lo que cuesta más que no tenerla. Así
  que: lo que la app FUSIONE en tiempo de ejecución para decidir qué lee una
  persona, la comprobación lo fusiona también.
- **El aviso no es condicional.** Se renderiza para todo el mundo, todas las
  veces — nunca detrás de una preferencia, un rol, un plan o una marca de «ya
  se lo dijimos una vez». La obligación se adhiere a la interacción, y un
  cliente que vuelve está interactuando de nuevo; un aviso que se puede
  desactivar está a un clic de una app que no divulga nada, y después no puedes
  mostrar qué clientes llegaron a verlo.

**Una IA que lee lo que el usuario PRODUJO debe el aviso antes, y debe una
frase distinta.** Un chat de soporte es fácil: la interacción es una pregunta
que alguien eligió hacer, y «esto es una IA» llega a tiempo. Pero una app que
le pide a su usuario que entregue su borrador, su respuesta, su plan — y luego
hace que un modelo lo lea — ya ha tenido su primera interacción en el momento
en que empieza a teclear. Así que:

- el aviso se puede leer **antes de que escriban**, no una vez que hay una
  transcripción y no una vez que algo ha cargado;
- dice **qué pasa con lo que escriben**, no solo qué es la cosa. *«Una IA lee
  lo que escribes aquí y lo responde»* le dice a alguien a qué está accediendo;
  *«impulsado por IA»* no.

Lo mismo vale para lo que va en la política de privacidad. Una frase como *«no
se envía nada sobre ti a la IA»* es cierta de un chatbot de manual y **falsa**
de cualquier cosa que lea el trabajo propio del usuario — y es falsa en un
documento legal, que es el peor sitio donde equivocarse. Si la app tiene las
dos, delimita las dos: di a cuál no se le envía nada y a cuál se le envía lo
que el usuario aporta.

## Paso 5 — los datos propios del cliente

Dos obligaciones, y ambas son ingeniería ordinaria una vez que tienes el
Paso 1:

**Acceso (art. 15).** Un comando o un botón produce todo lo que se guarda sobre
una persona. Busca por **dirección de correo, no por cuenta** — las personas
con más probabilidad de preguntar son las que nunca tuvieron cuenta, porque una
compra hecha sin iniciar sesión deja su nombre en un pedido sin id de miembro.

Una excepción documentada: los payloads de webhook de terceros en bruto pueden
llevar datos de otra persona y no hay nadie en medio para redactarlos
(art. 15(4)). Déjalos fuera de la exportación *dirigida al cliente* y
consérvalos en la del operador.

**Supresión (art. 17), y lo que no cubre.** Borrar una cuenta no borra todo, y
el diálogo tiene que decirlo:

- **Los pedidos se quedan.** Son registros contables sujetos a un plazo de
  conservación legal. Borrar uno sería la infracción, no el remedio. En su
  lugar, corta el vínculo con la cuenta.
- **Todo lo demás se va**, o se anonimiza.
- **Una suscripción en curso avisa y no bloquea.** Negar la supresión porque
  resulta incómoda es la infracción. Pero una facturación que sigue en
  Digistore24 sin ninguna cuenta detrás merece una frase bien alta — y un
  enlace para cancelar.
- La acción de supresión no toma **ningún id de la petición**: siempre la
  propia cuenta de quien llama.

## Paso 6 — las condiciones y el derecho de desistimiento

Vender a consumidores en la UE significa un derecho de desistimiento, y para el
contenido digital significa pedirle al comprador que acepte la entrega
inmediata — si no, el plazo corre y el acceso ya se ha entregado. Digistore24
se ocupa de buena parte de esto en el checkout como comerciante registrado
(merchant of record); **confirma qué cubre para esta cuenta en lugar de dar por
supuesto lo uno o lo otro**, y di qué confirmaste.

## Paso 6a — dos reglas que no son ley y cuestan más que la mayoría de las que sí lo son

La plataforma a través de la que vendes tiene criterios propios. Incumplir uno
no produce ningún error, ninguna prueba fallida y ningún cliente descontento —
produce un producto rechazado en la aprobación, o una cuenta cerrada tras meses
vendiendo. Nada dentro de la app puede notarlo, y por eso ambas pertenecen a
una comprobación y no a la memoria de alguien.

**1. No prometas cuánto dura el acceso.** Un área de miembros no puede venderse
como vitalicia, permanente, ilimitada o «para todo el tiempo que quieras»; dos
años es lo máximo que puede ofrecerse. La razón es el dinero más que el tono:
una oferta que desaparece a los 24 meses puede obligar al vendedor a reembolsar
el precio entero. Escribe en su lugar lo que es VERDAD — *paga una vez, sin
suscripción* para un pago único, *mientras dure tu plan* para una suscripción.
Una concesión de acceso sin fecha de fin no es una promesa; es la ausencia de
un evento que la termine, y un reembolso sigue siendo uno de esos eventos.

**Compruébalo por RAÍCES, y solo donde la frase nombre el acceso.** Tres cosas
que deciden si la comprobación sirve de algo:

- **Raíces, porque los idiomas flexionan.** La frase que convirtió esto en una
  regla fue *«Einmal kaufen, dauerhaft nutzen»* — no contiene ninguna de las
  palabras prohibidas tal y como las escriben los criterios (*dauerhafter*) y
  las contiene todas tal y como se entienden. Una lista literal de palabras la
  deja pasar.
- **Solo con una palabra de acceso en la misma frase.** *«Unbegrenzt viele
  Notizen»* es una característica y está bien; *«unbegrenzt nutzen»* es la
  promesa rechazada. La diferencia es el sustantivo. Una lista de palabras a
  secas abre con un muro de hallazgos en toda app que sea generosa con algo, y
  una comprobación que abre con un muro es una que alguien desactiva —
  llevándose la regla con ella.
- **Di cuántas dejaste fuera.** Imprime el recuento de frases que llevaban una
  palabra referida a algo que no era el acceso, para que nadie lea el visto
  verde como «esas palabras no aparecen aquí».

**2. Dile al comprador quién le cobró.** El nombre en el extracto bancario es
el de la plataforma de pago, no el del vendedor. Una línea que nadie reconoce
no se convierte en un correo al soporte, se convierte en una llamada al banco —
y un contracargo (chargeback) cuesta la venta, la comisión y una marca en la
cuenta.

Dos propiedades, y la segunda es la que se pasa por alto:

- **Todas las superficies posteriores a la compra, no solo la página de
  agradecimiento.** A un comprador que ya había iniciado sesión se le suele
  mandar directo a lo que pagó y nunca ve la página de agradecimiento. La
  pantalla en la que SÍ aterrice lo dice también.
- **La frase y el lugar donde se muestra son dos cosas distintas que
  comprobar.** Un texto que nadie renderiza es una cadena en un archivo; un
  render sin texto le enseña la clave a un cliente. Comprueba ambas cosas, por
  superficie y por idioma.

## Paso 7 — qué entregar

Deja atrás, en el repositorio:

1. el inventario del Paso 1, como un archivo que se actualiza con cada tabla
   nueva
2. el aviso legal, la política de privacidad y las condiciones como páginas
   reales
3. una nota fechada de qué se comprobó, qué se decidió y qué sigue abierto

Esa última es la diferencia entre «lo pensamos» y poder demostrarlo.

## Paso 8 — qué viene después

Si esto se ejecutó antes del lanzamiento, vuelve a **`ds24-golive`** y termina
la compra de prueba. Si la app ya está en producción, el siguiente paso honesto
es un abogado mirando las páginas que acabas de escribir.
