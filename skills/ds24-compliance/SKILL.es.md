---
name: ds24-compliance
language: es
description: Úsala cuando una app que cobra a clientes de la UE necesite tener en regla su parte legal — el aviso legal, la política de privacidad, las condiciones, la divulgación exigida por el Reglamento de IA de la UE (EU AI Act) para todo lo que hable con personas como máquina, si hace falta o no un banner de cookies, el registro de consentimientos y el derecho del cliente a sus datos y a su supresión. Úsala siempre que el usuario pregunte por el RGPD (GDPR), un banner de cookies, un aviso legal, el AI Act, la eliminación de cuentas o qué tiene que mostrar antes de vender a clientes reales.
---

> **Español** · Original en inglés — [`SKILL.md`](SKILL.md) · [Français](SKILL.fr.md)

# Qué exige la UE a una app de pago

Esto es preparación, no asesoramiento jurídico: resuelve bien lo evidente y
señala lo que debe revisar un abogado. Dilo una vez, al principio, y a partir
de ahí sé útil.

El disparador no es el tamaño de la app. Es **cobrar a personas de la UE** — y
para eso, precisamente, sirve una integración con Digistore24.

## Paso 0 — ¿qué hay ya?

Antes de escribir nada, mira. Busca en el proyecto un aviso legal, una política
de privacidad, unas condiciones, un banner de cookies, una tabla de
consentimientos.

**Lo que ya exista no se reemplaza.** Las páginas legales suelen ser la única
parte de la app que un abogado ya ha revisado, y una política de privacidad
reescrita que se lee mejor pero dice otra cosa es peor que una torpe y exacta.
Lee lo que hay, contrástalo con los pasos siguientes e informa de las lagunas:
cambiar la redacción lo decide el usuario, no tú.

Lo único que sí merece cuestionarse en voz alta es un **banner de cookies ya
instalado**: el Paso 3 explica por qué una app como esta normalmente no
necesita ninguno.

## Paso 0a — ¿esta copia del Skill Pack está al día?

Descarga `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
y compáralo con el `VERSION` de este pack. Si no coinciden, dilo en una frase y
sigue. Los textos legales envejecen más deprisa que el código: de todas las
skills, esta es en la que más pesa una copia anticuada.

## Paso 1 — el inventario va primero

Una política de privacidad veraz no se escribe de imaginación. Antes de
redactar una sola línea, **haz la lista de lo que la app guarda de verdad sobre
las personas**, tabla por tabla:

- cuentas: correo, nombre, fecha y hora de cada inicio de sesión
- pedidos: nombre del comprador, dirección, importe — vienen de Digistore24
- payloads de IPN en bruto: todo lo que envió Digistore24, datos del comprador
  incluidos
- concesiones de acceso y las notas que el operador haya dejado en ellas
- filas del libro mayor y sus notas
- logs, y cuánto tiempo se conservan

De cada una: **por qué** la guardas, **cuánto tiempo** y **quién más la ve** —
Digistore24, el proveedor de correo, el hosting, cualquier proveedor de IA. Esa
lista es el documento del que sale todo lo demás, y hay que actualizarla cada
vez que se añade una tabla.

**Las notas del operador son datos personales.** Lo que el soporte escribió
*sobre* un cliente entra en una solicitud de acceso aunque la app nunca se lo
muestre. Ocultarlo en la interfaz es una decisión de tono, no una exención.

**Si los miembros pueden verse o contactar entre sí, la lista crece en dos
puntos más**, y ninguno de los dos salta a la vista en un diagrama de tablas:

- **Lo que los propios miembros escribieron** — un perfil, una publicación en
  un espacio compartido, un mensaje privado. La app guarda ahora texto que una
  persona escribió para que otra lo lea. Entra en la solicitud de acceso, y una
  solicitud de supresión tiene que poder alcanzarlo; la manera honesta de hacer
  lo segundo es vaciar el texto, conservar la fila para que la conversación que
  lo rodea siga teniendo sentido, y decirlo así en la política. Una respuesta a
  un mensaje borrado no debe quedarse contestando a la nada.
- **⚠️ La propia participación puede ser información de compra.** La lista de
  miembros de un espacio de pago es la lista de quién lo compró — y en un
  producto de salud, finanzas o coaching eso roza los datos de categorías
  especiales (art. 9 del RGPD [GDPR]). El diseño seguro es no tener ningún
  listado: ni lista de miembros, ni recuento, ni «quién está aquí». Alguien se
  hace visible al publicar, y eso es algo que decidió hacer. Si aun así
  construyes una lista de miembros, dilo en la política de privacidad y lee
  antes el art. 9.

**Y si construyes mensajes privados, decide quién puede leerlos antes de
construirlos, no después.** «Solo los dos participantes» es una promesa que el
código tiene que cumplir: toda consulta que lea un mensaje lleva el id del
propio lector, no existe una vista de administrador, y una sesión de soporte
capaz de entrar como un miembro tampoco la tiene — leer el correo de otra
persona no cambia nada ni deja rastro, así que no hay log que sirva para pedir
cuentas por ello. Las únicas excepciones que vale la pena dejar abiertas son una
solicitud de acceso del interesado (respondida a mano, y solo ante una
solicitud real) y la denuncia del propio participante, acotada a lo que esa
persona decidió adjuntar.

## Paso 2 — el aviso legal

Según el § 5 de la DDG alemana (y sus equivalentes en otros países), un sitio
comercial necesita un aviso legal accesible: nombre, dirección (**una real, no
un apartado de correos**), correo electrónico, teléfono u otra vía de contacto
igual de rápida y, cuando proceda, el NIF-IVA (VAT ID) y la inscripción en el
registro mercantil.

Construye la página y **haz que falle de forma bien visible mientras esté
vacía**. Un aviso legal provisional que llega a producción es peor que ninguno:
es información visiblemente falsa sobre quién está vendiendo.

**El aviso legal viaja de dos formas, y ninguna vale por la otra:**

- **En las páginas, un ENLACE en el pie es la respuesta completa** — con el
  texto «Impressum» / «Imprint», a un clic, en todas las páginas. No copies el
  texto del aviso legal en los pies de página: «fácilmente reconocible y
  directamente accesible» pide el enlace, y la segunda copia incrustada es la
  que se desvía.
- **En los correos que envía la app** — enlaces de inicio de sesión,
  confirmaciones, avisos — **el CONTENIDO del aviso legal va al final del
  propio correo.** Un correo enviado en el curso del negocio es una carta
  comercial, y quien la recibe no tiene ningún pie de página en el que hacer
  clic; para las sociedades inscritas, las normas sobre cartas comerciales (en
  Alemania: § 35a GmbHG, § 125a HGB) piden los datos del prestador *dentro* de
  la carta. Ponlo como líneas de texto plano bajo el pie del correo — y nunca
  envíes un texto provisional: ningún bloque de aviso legal en el correo hasta
  que exista el aviso legal real. Hay una única excepción, y es deliberada: un
  aviso puramente de seguridad, construido para no llevar nada en lo que hacer
  clic, se queda sin él, porque un aviso legal contiene direcciones web y de
  correo, y los clientes de correo las convierten en enlaces por su cuenta.

## Paso 3 — probablemente ningún banner de cookies, y no es por pereza

**Una compra no necesita consentimiento.** Se apoya en el art. 6(1)(b) del RGPD
— la ejecución de un contrato —, no en un permiso. Y lo que una app de pago
suele dejar en un dispositivo — la sesión, el idioma, el tema, un «no volver a
mostrar esto» que el propio cliente pulsó — está cubierto sin preguntar: o es
estrictamente necesario, o es la consecuencia directa de que alguien accionó
un interruptor.

Así que: **no pongas un banner de cookies en una app que no coloca cookies no
esenciales.** Pide un permiso que ni necesitas ni usas, y acostumbra a la gente
a pulsar sin leer justo el que sí importará más adelante.

⚠️ **La regla habla del DISPOSITIVO, no de las cookies.** El § 25 de la TDDDG y
sus equivalentes cubren `localStorage`, `sessionStorage`, IndexedDB y las
huellas de dispositivo con exactamente las mismas palabras — así que «no usamos
cookies, usamos localStorage» no es una exención: es el mismo acto con otra
API. Conviene decirlo porque es el atajo que se acaba tomando: una preferencia
guardada sin preguntar está bien si la fijó el usuario; una analítica guardada
sin preguntar es una infracción, caiga en el almacén que caiga.

**Lleva por escrito una única lista de todo lo que tu app deja en un
dispositivo**, con una línea por entrada que diga por qué no necesita
consentimiento. De ahí sale la política de privacidad, y es lo que deja de ser
cierto sin que nadie lo note: cada nueva función que recuerda algo es una
entrada nueva, y la cuarta ya no la ve nadie.

**Cuando algo sí necesita consentimiento de verdad** — una etiqueta de
analítica, un correo de marketing, un widget de terceros incrustado —,
entonces:

- declara la **finalidad**, y por separado para cada finalidad
- registra **quién consintió, a qué redacción y cuándo** — y guarda una
  **versión de la redacción**, porque cambiar el texto significa que todo el
  mundo consintió a otra cosa. Y luego **léela al comprobar el
  consentimiento**: un consentimiento registrado sobre una redacción anterior
  cuenta como inexistente y la pregunta se hace de nuevo. Guardar la versión e
  ignorarla justo donde compruebas es el fallo que parece conforme en la tabla
  y no lo es en el producto
- el registro es **de solo adición (append-only)**. Una retirada es una fila
  nueva, nunca una edición: tienes que poder *demostrar* el consentimiento
  (art. 7(1)), y una fila sobrescrita no demuestra nada
- retirar el consentimiento tiene que ser tan fácil como darlo

## Paso 4 — la divulgación de IA es ley, no texto publicitario

**Art. 50(1) del Reglamento de IA de la UE (EU AI Act), aplicable desde el 2 de
agosto de 2026:** un sistema que interactúa con personas debe dejarles claro
que están tratando con una máquina, como muy tarde en la primera interacción.

Si la app tiene un chat, un asistente, una respuesta generada — cualquier cosa
que hable con una persona como máquina —, **lo dice, de forma visible y en
todos los idiomas que habla la app**. No en las condiciones: donde ocurre la
conversación.

Formúlalo como regla, no como caso aislado: *todo lo que aquí hable con una
persona como máquina lo dice*. La siguiente función de IA que se añada la
hereda.

**Convierte esa regla en una LISTA, y que algo la recorra.** Un único sitio que
nombre cada superficie en la que una máquina habla con una persona, y una
comprobación que falle cuando una superficie de la lista no tiene aviso. Una
regla que nadie puede ejecutar aguanta hasta que sale la segunda función de IA.
Dos cosas sobre la lista:

- **Quien añade una superficie añade su propia entrada.** Una función opcional,
  un plug-in, una segunda área de la app: la lista central no puede enumerar lo
  que no está activado, así que la parte de la app que trae su propia IA trae
  también su propia entrada, y la comprobación recorre todas las listas que
  haya. La superficie que nadie registró es justo la que sale a producción sin
  aviso.
- **Y la comprobación lee todos los almacenes de TEXTO que haya, no solo el
  central.** Una función que trae su propia superficie suele traer también su
  propia redacción, allí donde esa parte de la app guarde sus textos. Una
  comprobación que recorre las dos listas pero solo lee el archivo de textos
  principal denuncia como ausente un aviso que la app muestra sin ningún
  problema — y una comprobación legal que da falsas alarmas es una que la gente
  aprende a saltarse, lo que sale más caro que no tenerla. Por tanto: lo que la
  app FUSIONA en tiempo de ejecución para decidir qué lee una persona, la
  comprobación lo fusiona también.
- **El aviso no es condicional.** Se muestra a todo el mundo, todas las veces —
  nunca detrás de una preferencia, un rol, un plan ni una marca de «ya se lo
  dijimos una vez». La obligación va ligada a la interacción, y un cliente que
  vuelve está interactuando de nuevo; un aviso que se puede desactivar está a
  un clic de una app que no divulga nada, y después no podrás demostrar qué
  clientes llegaron a verlo.

**Una IA que lee lo que el usuario PRODUJO debe el aviso antes, y debe una
frase distinta.** Un chat de soporte es fácil: la interacción es una pregunta
que alguien decidió hacer, y «esto es una IA» llega a tiempo. Pero una app que
pide al usuario su borrador, su respuesta, su plan — y luego se los da a leer a
un modelo — ya tuvo su primera interacción en el momento en que empezó a
teclear. Por tanto:

- el aviso se lee **antes de que escriban**, no cuando ya hay una transcripción
  ni cuando algo ha terminado de cargar;
- y dice **qué pasa con lo que escriben**, no solo qué es la cosa. *«Una IA lee
  lo que escribes aquí y lo responde»* le dice a alguien qué está aceptando;
  *«con tecnología de IA»* no.

Lo mismo vale para la política de privacidad. Una frase como *«no se envía a la
IA nada sobre ti»* es cierta para un chatbot que responde a partir de un manual
y **falsa** para cualquier cosa que lea el trabajo del propio usuario — y falsa
en un documento legal, que es el peor sitio para equivocarse. Si la app tiene
las dos, acota cada una: di a cuál no se le envía nada y a cuál se le envía lo
que el usuario aporta.

## Paso 5 — los datos del propio cliente

Dos obligaciones, y con el Paso 1 hecho las dos son ingeniería corriente:

**Acceso (art. 15).** Un comando o un botón produce todo lo que se guarda sobre
una persona. Busca por **dirección de correo, no por cuenta**: quienes más
probablemente lo pidan son los que nunca tuvieron cuenta, porque una compra
hecha sin iniciar sesión deja su nombre en un pedido sin id de miembro.

Una excepción, documentada: los payloads en bruto de webhooks de terceros
pueden contener datos de otra persona, y no hay nadie en medio que los tache
(art. 15(4)). Déjalos fuera de la exportación *para el cliente* y consérvalos
en la del operador.

**Supresión (art. 17), y lo que no cubre.** Borrar una cuenta no lo borra todo,
y el diálogo tiene que decirlo:

- **Los pedidos se quedan.** Son registros contables sujetos a un plazo legal
  de conservación. Borrar uno sería la infracción, no el remedio. Lo que se
  hace es cortar el vínculo con la cuenta.
- **Todo lo demás se borra**, o se anonimiza.
- **Una suscripción en curso avisa, pero no bloquea.** Negar la supresión
  porque resulta incómoda es la infracción. Pero una facturación que sigue
  corriendo en Digistore24 sin ninguna cuenta detrás merece una frase bien
  clara — y un enlace para cancelar.
- La acción de borrado **no toma ningún id de la petición**: siempre la cuenta
  de quien la ejecuta.

## Paso 6 — las condiciones y el derecho de desistimiento

Vender a consumidores en la UE implica un derecho de desistimiento, y con
contenido digital implica pedir al comprador que acepte la entrega inmediata —
si no, el plazo sigue corriendo y el acceso ya está entregado. Digistore24
resuelve buena parte de esto en el checkout como comerciante de registro
(merchant of record); **confirma qué cubre para esta cuenta en lugar de darlo
por hecho en un sentido o en otro**, y di qué confirmaste.

## Paso 6a — dos reglas que no son ley y cuestan más que la mayoría de las que sí lo son

La plataforma a través de la que vendes tiene criterios propios. Incumplir uno
no produce ningún error, ninguna prueba en rojo ni ningún cliente descontento:
produce un producto rechazado en la aprobación, o una cuenta cerrada tras meses
vendiendo. Nada dentro de la app puede notarlo, y por eso las dos reglas van en
una comprobación y no en la memoria de nadie.

**1. No prometas cuánto dura el acceso.** Un área de miembros no puede venderse
como vitalicia, permanente, ilimitada o «para todo el tiempo que quieras»; dos
años es lo máximo que se puede ofrecer. La razón es el dinero, no el tono: una
oferta que desaparece a los 24 meses puede obligar al vendedor a devolver el
precio entero. Escribe en su lugar lo que es VERDAD: *pagas una vez, sin
suscripción* para un pago único, *mientras dure tu plan* para una suscripción.
Una concesión de acceso sin fecha de fin no es una promesa; es la ausencia de
un evento que le ponga fin, y un reembolso sigue siendo un evento de esos.

**Compruébalo por RAÍCES, y solo donde la frase hable de acceso.** Tres cosas
que deciden si la comprobación sirve de algo:

- **Raíces, porque los idiomas flexionan.** La frase que convirtió esto en
  regla fue *«Einmal kaufen, dauerhaft nutzen»*: no contiene ninguna de las
  palabras prohibidas tal como las escriben los criterios (*dauerhafter*), y
  las contiene todas tal como se entienden. Una lista literal de palabras la
  deja pasar.
- **Solo con una palabra de acceso en la misma frase.** *«Unbegrenzt viele
  Notizen»* es una característica y está bien; *«unbegrenzt nutzen»* es la
  promesa que se rechaza. La diferencia está en el sustantivo. Una lista de
  palabras a secas abre con un muro de hallazgos en cualquier app que sea
  generosa con algo, y una comprobación que abre con un muro es una que alguien
  acaba desactivando — y se lleva la regla con ella.
- **Di cuántas dejaste fuera.** Imprime el número de frases que llevaban una de
  esas palabras referida a algo que no era el acceso, para que nadie lea el
  visto verde como «esas palabras no aparecen aquí».

**2. Dile al comprador quién le ha cobrado.** En el extracto bancario aparece
el nombre de la plataforma de pago, no el del vendedor. Una línea que nadie
reconoce no acaba en un correo al soporte: acaba en una llamada al banco — y un
contracargo (chargeback) cuesta la venta, la comisión y una marca en la cuenta.

Dos propiedades, y la segunda es la que se pasa por alto:

- **Todas las superficies posteriores a la compra, no solo la página de
  agradecimiento.** A un comprador que ya tenía la sesión iniciada se le suele
  mandar directo a lo que pagó, y la página de agradecimiento no la ve nunca.
  La pantalla en la que SÍ aterriza lo dice también.
- **La frase y el sitio donde se muestra son dos cosas distintas, y se
  comprueban por separado.** Un texto que nadie renderiza es una cadena en un
  archivo; un render sin texto le enseña la clave al cliente. Comprueba ambas,
  superficie por superficie e idioma por idioma.

## Paso 7 — qué entregar

Deja en el repositorio:

1. el inventario del Paso 1, como un archivo que se actualiza con cada tabla
   nueva
2. el aviso legal, la política de privacidad y las condiciones como páginas
   reales
3. una nota fechada con lo que se comprobó, lo que se decidió y lo que sigue
   abierto

Esa última es la diferencia entre «lo pensamos» y poder demostrarlo.

## Paso 8 — qué viene después

Si esto se ha ejecutado antes del lanzamiento, vuelve a **`ds24-golive`** y
termina la compra de prueba. Si la app ya está en producción, el siguiente
paso, con toda franqueza, es que un abogado revise las páginas que acabas de
escribir.
