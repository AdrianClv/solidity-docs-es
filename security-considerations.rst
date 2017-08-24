.. _security_considerations:

############################
Consideraciones de seguridad
############################

Aunque en general es bastante fácil hacer software que funcione como esperamos,
es mucho más difícil de chequear que nadie lo pueda usar de alguna forma que **no**
fue anticipada.

En solidity, esto es aún más importante ya que se pueden usar los contratos
inteligentes para mover tokens, o posiblemente, cosas aún más valiosas. Además,
cada ejecución de un contrato inteligente es pública, y a ello se suma
que el código fuente muchas veces está disponible.

Por supuesto, siempre se tiene que considerar lo que está en juego:
Puedes comparar un contrato inteligente con un servicio web que está abierto
al público (y por lo tanto, también a gente malintencionada) y quizá
de código abierto.
Si sólo se guarda la lista de compras en ese servicio web, puede que no tengas
que tener mucho cuidado, pero si accedes a tu cuenta bancaria usando ese servicio,
deberíás tener más cuidado.

Esta sección nombrará algunos errores comunes y recomendaciones de seguridad
generales pero no puede, por supuesto, ser una lista completa. Además, recordar
que incluso si tu contrato inteligente está libre de errores, el compilador
o la plataforma puede que los tenga. Una lista de errores de seguridad públicamente
conocidos del compilador puede encontrarse en: :ref:`lista de errores conocidos<known_bugs>`,
la lista también es legible por máquina. Nótese que hay una recompensa
por encontrar errores (bug-bounty) que cubre el generador de código del compilador de Solidity.

Como siempre ocurre con la documentación de código abierto, por favor, ayúdanos a extender
esta sección, ¡especialmente con algunos ejemplos!


***************
Errores Comunes
***************

Información privada y aleatoriedad
==================================

Todo lo que usas en un contrato inteligente es públicamente visible, incluso
variables locales y variables de estado marcadas como ``private``.

Es bastante complejo usar números aleatorios en contratos inteligentes si no
quieres que los mineros puedan hacer trampa.

Reentrada
=========

Cualquier interacción desde un contrato (A) con otro contrato (B) y cualquier
transferencia de Ether, le da el control a ese contrato (B). Esto hace posible
que B vuelva a llamar a A antes de terminar la interacción. Para dar un ejemplo,
el siguiente código contiene un error (esto es sólo un snippet y no un contrato
completo):

::

  pragma solidity ^0.4.0;

  // ESTE CONTRATO CONTIENE UN ERROR - NO USAR
  contract Fund {
      /// Mapping de distribución de ether del contrato.
      mapping(address => uint) shares;
      /// Retira tu parte.
      function withdraw() {
          if (msg.sender.send(shares[msg.sender]))
              shares[msg.sender] = 0;
      }
  }

El problema aquí no es tan grave por el límite de gas de la función
``send``, pero aun así, expone una debilidad: una transferencia de Ether
siempre incluye ejecución de código, así que el receptor puede ser un
contrato que vuelve a llamar a ``withdraw``. Esto le permitiría obtener
múltiples devoluciones, y por lo tanto, vaciar el Ether del contrato.

Para evitar reentradas, puedes usar el orden Comprobaciones-Consecuencias-Interacciones
como detallamos aquí:

::

  pragma solidity ^0.4.11;

  contract Fund {
      /// Mapping de distribución de ether del contrato.
      mapping(address => uint) shares;
      /// Retira tu parte.
      function withdraw() {
          var share = shares[msg.sender];
          shares[msg.sender] = 0;
          msg.sender.transfer(share);
      }
  }

Nótese que las reentradas no sólo son un riesgo al transferir Ether, sino
de cualquier ejecución de una función de otro contrato. Además, también tienes que
considerar situaciones de multi-contrato. Un contrato ejecutado podría modificar el
estado de otro contrato del cual dependes.

Límite de gas y bucles
======================

Los bucles que no tienen un número fijo de iteraciones, por ejemplo, bucles que dependen de valores de almacenamiento, tienen que
usarse con cuidado:
Dado el límite de gas del bloque, las transacciones sólo pueden consumir una cierta cantidad de gas. Ya sea explícitamente,
o por una operación normal, el número de iteraciones en un bucle puede crecer más allá del límite de gas, lo que puede causar que el
contrato se detenga por completo en un cierto punto. Esto no se aplica a funciones ``constant`` que sólo se llaman para
leer información de la blockchain. Pero aun así, estas funciones pueden ser llamadas por otros contratos como parte de operaciones
on-chain y detenerlos a ellos. Por favor, sé explícito con estos casos en la documentación de tus contratos.

Envío y recibo de Ether
=======================

- Ni los contratos ni las "cuentas externas", son actualmente capaces de prevenir que alguien
  les envíe Ether. Los contratos pueden reaccionar y rechazar una transferencia normal, pero hay maneras de mover
  Ether sin provocar la ejecución de código. Una manera es simplemente "minando" a la
  cuenta del contrato y la otra es usando ``selfdestruct(x)``.

- Si un contrato recibe Ether (sin que se llame a ninguna función), se ejecuta la función fallback.
  Si no tiene una funcin fallback, el Ether será rechazado (lanzando una excepción).
  Durante la ejecución de la función fallback, el contrato sólo puede depender del
  "estipendio de gas" (2300 gas) que tiene disponible en ese momento. Este estipendio no es suficiente para acceder
  al almacenamiento de ninguna forma. Para asegurarte de que tu contrato pueda recibir Ether de ese modo, verifica los
  requerimientos de gas de la función fallback (por ejemplo, en la sección de "details" de Remix).

- Hay una manera de enviar más gas al contrato receptor usando ``addr.call.value(x)()``.
  Esto es esencialmente lo mismo que ``addr.transfer(x)``, solo que envía todo el gas restante
  y permite la posibilidad al receptor de hacer acciones más caras (y sólo devuelve un código
  de error y no propaga automáticamente el error). Dentro de estas acciones se incluyen llamar de nuevo al contrato
  emisor u otros cambios de estado que no fueron previstos. Permite más flexibilidad para
  usuarios honestos pero también para los usuarios maliciosos.

- Si quieres enviar Ether usando ``address.transfer``, hay ciertos detalles que hay que saber:

  1. Si el receptor es un contrato, la función fallback será ejecutada, lo cual puede llamar de vuelta al
  contrato que envía Ether.
  2. Enviar Ether puede fallar debido a la profundidad de la llamada subiendo por encima 1024. Ya que el que llama está
     en control total de la profundidad de la llamada, pueden forzar la transferencia para que falle; tened en consideración esta posibilidad o utilizad siempre ``send`` y asegurarse siempre de revisar el valor de retorno. O mejor aún, escribir el contrato con un orden en que el recipiente pueda retirar Ether.
  3. Enviar Ether también puede fallar, porque la ejecución del contrato de recipiente necesita más gas
     que la cantidad asignada dejándolo sin gas (OOG, por sus siglas en inglés "Out of Gas"). Esto ocurre porque
     explícitamente se usó ``require``, ``assert``, ``revert``, ``throw``, o simplemente porque la operación es demasiado cara.
     Si usas ``transfer`` o ``send`` con revisión de la valor de retorno, esto puede proveer una manera para el recipiente
     de bloquear el progreso en el contrato de envío. Pero volviendo a insistir, aquí lo mejor es usar
     un orden de retiro :ref:`"withdraw" en vez de "orden de envío" <withdrawal_pattern>`.

Profundidad de Pila de Llamadas (Callstack)
==================================

Llamadas externas de funciones pueden fallas en cualquier momento porque
exceden la pila de llamadas de 1024. En tales situaciones, Solidity lanza
una excepción. Usuarios maliciosos podrían forzar la pila a un valor alto
antes de interactuar con el contrato.

Notar que ``.send()`` **no** lanza una excepción si la pila está vacía si no
que retorna ``false`` en ese caso. Las funciones de bajo nivel de ``.call()``,
``.callcode()``  ``.delegatecall()`` se comportan de la misma manera.


tx.origin
=========

Nunca usar tx.origin para autorización. Digamos que tienes un contrato de billetera como esta:

::

    pragma solidity ^0.4.11;

    // ESTE CONTRATO CONTIENE UN ERROR - NO USAR
    contract TxUserWallet {
        address owner;

        function TxUserWallet() {
            owner = msg.sender;
        }

        function transferTo(address dest, uint amount) {
            require(tx.origin == owner);
            dest.transfer(amount);
        }
    }

Ahora alguien te engaña para que le envíes Ether a esta billetera de ataque:

::

    pragma solidity ^0.4.0;

    contract TxAttackWallet {
        address owner;

        function TxAttackWallet() {
            owner = msg.sender;
        }

        function() {
            TxUserWallet(msg.sender).transferTo(owner, msg.sender.balance);
        }
    }

Si tu billetera hubiera checkeado ``msg.sender`` para autorización, recibiría la cuenta de la billetera de ataque, en vez de la billetera del 'owner'. Pero al chequear ``tx.origin``, recibe la cuenta original que envió la transacción, quien aún es la cuenta owner. La billetera atacante inmediatamente vacía todos tus fondos.


Detalles Menores
================

- En ``for (var i = 0; i < arrayName.length; i++) { ... }``, el tipo de ``i`` será ``uint8``, porque este es el más pequeño tipo que es requerido para guardar el valor ``0``. Si el vector (array) tiene más de 255 elementos, el bucle no se terminará.
- La palabra reservada ``constant`` para funciones no es actualmente forzada por compiladores.
  Además, no está forzada por la EVM, entonces una función de contrato que "pretende" ser constante,
  puede aún hacer cambios al estado.
- Tipos que no utilizan totalmente los 32 bytes pueden contener "dirty high order bits".
  Esto es especialmente importante si se accede a ``msg.data`` ya que supone un riesgo de maleabilidad:
  Puedes crear transacciones que llaman una función ``f(uint8 x)`` con un argumento raw byte
  de ``0xff000001`` y con ``0x00000001``. Ambos son pasados al contrato y ambos se verán como
  números como ``1``. Pero ``msg.data`` es diferente, así que si se usa ``keccak246(msg.data)`` para
  algo, tendrás resultados diferentes.

***************
Recomendaciones
***************

Restringir la cantidad de Ether
===============================

Restringir la cantidad de Ether (o otros tokens) que pueden ser almacenados
en un contrato inteligente. Si el código fuente, el compilador o la plataforma
tiene un error, estos fondos pueden ser perdidos. Si quieres limitar la pérdida,
limita la cantidad de Ether.


Pequeño y modular
=================

Mantén tus contratos pequeños y fáciles de entender. Separar funcionalidad
no relacionada en otros contratos o en librerías. Recomendaciones generales de
calidad de otras fuentes de código pueden aplicarse: Limitar la cantidad de variables
locales, limitar el largo de la funciones, y más. Documenta tus funciones para que
otros puedan ver cual era la intención del código y para ver si hace algo diferente de
lo que pretendía.

Usa el orden Checks-Effects-Interactions
=========================================

La mayoría de las funciones primero ejecutan algunos chequeos (¿quién ha llamado
la función? ¿los argumentos están en el rango? ¿mandaron suficiente Ether?
¿La cuenta tiene tokens? etc) Estos chequeos deben de hacerse primero.

Como segundo paso, si es que todos los chequeos pasaron, los efectos a las
variables de estado del contrato actual deben hacerse. Interacción con otros
contratos debe hacerse como el último paso en cualquier función.

Algunos primeros contratos retrasaban algunos efectos y esperaban a una función
externa que retorne un estado sin errores. Esto es un error serio ya que se puede
hacer un reingreso, como explicamos arriba.

Notar que, también, llamadas a contratos conocidos pueden a su vez causar llamadas
a otros contratos no conocidos, así que siempre es mejor de aplicar este orden.

Incluir un modo a Prueba de Fallos (Fail-Safe)
==============================================

Aunque hacer que tu sistema sea completamente descentralizado eliminará cualquier intermediario,
puede que sea una buena idea, especialmente para nuevo código, de incluir un sistema
a prueba de fallos:

Puedes agregar una función a tu contrato que se revise a sí mismo como
"¿Se ha filtrado Ether?", "¿Es igual la suma de los tokens al balance de la cuenta?"
o cosas similares. Recordar que no se puede usar mucho gas para eso, así que ayuda con
computaciones off-chain podrán ser necesarias.

Si los chequeos fallan, el contrato automáticamente cambia a modo prueba de fallos, donde,
por ejemplo, se desactivan muchas funciones, da el control a una entidad tercera de confianza
o se convierte en un contrato "devuélveme mi dinero".


*******************
Verificación Formal
*******************

Usando verificación formal, es posible realizar pruebas matemáticas automatizadas
que el código haga una cierta especificación formal.
La especificación aún es formal (como el código fuente), pero usualmente mucho más simple.
Hay un prototipo en Solidity que realiza verificación formal y será mejor documentada pronto.

Notar que la verificación formal en sí mismo, sólo puede ayudarte a entender la diferencia
entre lo que hiciste (la especificación) y cómo lo hiciste (la implementación real). Aún necesitas
chequear si la especificación es lo que querías y que no hayas olvidado efectos inesperados de ello.
