.. _security_considerations:

#######################
Consideraciones de Seguridad
#######################

Aunque en general es bastante fácil de hacer software que funciona como esperamos,
es mucho mas difícil de chequear que nadie lo puede usar de alguna forma que **no**
fue anticipada.

En solidity, esto es aún más importante ya que se pueden usar los contratos
inteligentes para mover tokens, o posiblemente, cosas aún mas valiosas. Además,
cada ejecución de de un contrato inteligente ocurre en público y, a ello se suma
que el código fuente muchas veces está disponible.

Por supuesto siempre se tiene que considerar lo que está en juego:
Puedes comprarar un contrato inteligente con un servicio web que está abierto
al público (y por lo tanto, a malos intencionados también) y quizá
de código abierto.
Si solo se guarda la lista de compras en ese servicio web, puede que no tengas
que tener mucho cuidado, pero si accedes a tu cuenta bancaria usando ese servicio,
deberíás tener mas cuidado.


Esta sección va nombrar algunos errores comunes y recomendaciones de seguridad
generales pero no puede, por supuesto, ser una lista completa. Además, recordar
que incluso si tu contrato inteligente está libre de errores (bug-free), el complilador
o la plataforma puede que tenga uno. Una lista de errores de seguridad pulicamente
conocidos del compilador puede encontrarse en: :ref:`lista de errores conocidos<known_bugs>`,
la lista también es legible por maquina (machine-readable). Nótese que hay una recompensa
por errores (bug-bounty) que cubre el generador de código del compliador de Solidity.

Como siempre es el caso con documentación de código abierto, por favor ayúdanos a extender
esta sección, especialmente con algunos ejemplos!


***************
Errores Comunes
***************

Información Privada y Aleatoriedad
==================================

Todo lo que usas en un contrato inteligente es públicamente visible, incluso
variables locales y variables de estado maracadas como ``private``.

Usando números aleatorios en contratos inteligentes es bastante difícil si no
quieres que los mineros puedan hacer trampa.


Reingreso (Re-entry)
====================

Cualquier interacción desde un contrato (A) con otro contrato (B) y cualquier
transferencia de Ether le da el control a ese contrato (B). Esto hace posible
que B vuelva a llamar a A antes de terminar la interacción. Para dar un ejemplo,
el siguiente código contiene un error (esto es solo un snippet y no un contrato
completo):

::

  pragma solidity ^0.4.0;

  // ESTE CONTRATO CONTIENE UN ERROR - NO USAR
  contract Fund {
      /// Mapping de shares de ether del contrato.
      mapping(address => uint) shares;
      /// Retira tu share.
      function withdraw() {
          if (msg.sender.send(shares[msg.sender]))
              shares[msg.sender] = 0;
      }
  }

El problema aquí no es tan grave por los límites del gas como parte
de ``send``, pero aún así, expone una debilidad: transferencia de Ether
siempre incluye ejecución de código, así que el recipiente puede ser un
contrato que vuelve a llamar ``withdraw``. Esto lo permitiría obtener
multiples devoluciones y por lo tanto vaciar el Ether del contrato.

Para evitar reingresos, puedes usar el diseño Checks-Effects-Interactions
como detallamos aquí:

::

  pragma solidity ^0.4.11;

  contract Fund {
      /// Mapping de shares de ether del contrato.
      mapping(address => uint) shares;
      /// Retira tu share.
      function withdraw() {
          var share = shares[msg.sender];
          shares[msg.sender] = 0;
          msg.sender.transfer(share);
      }
  }

Nótese que los reingresos no solo son un riesgo de transferencia de Ether, pero
de cualquier ejecución de una función de otro contrato. Además, también tienes que
considerar situaciones de multi-contrato. Un contrato ejecutado podría modificar el
estado de otro contrato del cual dependes.

Límite de Gas y Bucles (Loops)
==============================

Bucles que no tienen un número fijo de iteraciones, por ejemplo, bucles que dependen de valores de almacenamiento, tienen que
usarse con cuidado:
Dado al límite de gas del bloque, las transacciones pueden sólamente consumir una cierta cantidad de gas. Ya sea explícitamente,
o por une operación normal, el número de iteraciones en un loop puede crecer más allá del límite de gas que puede causar el
contrato completo a detenerse a un cierto punto. Esto no se aplica a funciones ``constant`` que son solo llamadas para
leer data de la blockchain. Pero aún así, estas funciones pueden ser llamadas por otros contratos como parte de operaciones
en la cadena (on-chain) y detenerlas a ellas. Por favor ser explícito con estos casos en la documentación de tus contratos.

Enviando y Recibiendo Ether
===========================

- Ni contratos ni cuentas externas ("external accoutns"), son actualmente capaces de prevenir que alguien
  les envíe Ether. Contratos pueden reaccionar y rechazar una transferencia normal, pero hay maneras de mover
  Ether sin crear un mensaje de llamado (message call). Una manera de de simplemente "minando" a la
  cuenta del contrato y la otra es usando ``selfdestruct(x)``.

- Si un contrato recibe Ether (sin que una función sea llamada), la función de respaldo es ejecutada.
  Si no tiene función de respaldo, el Ether será rechazado (lanzando una excepción).
  Durante la ejecución de la función de respaldo, el contrato solo puede depender del
  "estipendio de gas" (2300 gas) que tiene disponible en ese momento. Esto estipendio no es suficiente para acceder
  el almacenimiento de ningua forma. Para asegurarte que tu contrato puede recibir Ether en ese modo, verifica los
  requerimientos de gas de la función de respaldo (por ejemplo en la sección de "details" de Remix).

- Hay una manera de enviar mas gas a contrato receptor usando ``addr.call.value(x)()``.
  Esto es escencialmente lo mismo que ``addr.transfer(x)``, solo que envía todo el gas restante
  y permite la posibilidad al recipiente de hacer acciones mas caras (y solo devuelve un código
  de error y no propaga automáticamente el error). Esto puede incluir volviendo a llamar al contrato
  que envía o otros cambios de estado que nofueron imaginados. Así que permite mas flexibilidad para
  usuarios honestos pero también para los usuarios maliciosos.

- Si quieres envíar Ether usando ``address.transfer``, hay ciertos detalles de los que hay que saber:

  1. Si el recipiente es un contrato, causa que la función de respaldo sea ejecutada lo cual puede, a su vez, llamar de vuelta el contraro que envía Ether.
  2. Enviar Ether puede fallar debido a la profundidad de la llamada (call depth) subiendo por sobre 1024. Ya que el que llama está
     en control total de la profundidad de llamada, pueden forzar la transferencia a fallas; tener en consideración está posibilidad
     o utilizar siempre ``send`` y asegurarse siempre de revisar el valor de retorno. O mejor aún, escrbir el contrato con un diseño en
     que el recipiente pueda retirar Ether.
  3. Enviar Ether también puede fallar, porque la ejecución del contrato de recipiente necesita mas gas
     que la cantidad asignada dejándolo sin gas (OOG, por sus siglas en inglés "Out of Gas"). Esto ocurre porque
     explícitamente se usó ``require``, ``assert``, ``revert``, ``throw``, o simplemente porque la operación es demasiado cara.
     Si usas ``transfer`` o ``send`` con revisión de la valor de retorno, esto puede proveer una manera para el recipiente
     de bloquear el progreso en el contrato de envío. Pero volviendo a insistir, aquí lo mejor es usar
     un diseño de retiro :ref:`"withdraw" en vez de "diseño de envío" <withdrawal_pattern>`.

Profundidad de Pila de Llamadas (Callstack)
==================================

Llamadas externas de funciones pueden fallas en cualquier momento porque
exceden la pila de llamadas de 1024. En tales situaciones, Solidity lanza
una excepción. Usuarios maliciosos podrían forzar la pila a un valor alto
antes de interactuar con el contrato.

Notar que ``.send()`` **no** lanza una excepción si la pila esta vacía si no
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

Si tu billetera hubiera checkeado ``msg.sender`` para autorización, recibiría la cuenta de la billetera de ataque, en vez de la billetera del 'owner'. Pero al checkear ``tx.origin``, recibe la cuenta original que envió la transacción, quien aún es la cuenta owner. La billetera atacante immediatamente vacía todos tus fondos.


Detalles Menores
================

- In ``for (var i = 0; i < arrayName.length; i++) { ... }``, the type of ``i`` will be ``uint8``, because this is the smallest type that is required to hold the value ``0``. If the array has more than 255 elements, the loop will not terminate.
- The ``constant`` keyword for functions is currently not enforced by the compiler.
  Furthermore, it is not enforced by the EVM, so a contract function that "claims"
  to be constant might still cause changes to the state.
- Types that do not occupy the full 32 bytes might contain "dirty higher order bits".
  This is especially important if you access ``msg.data`` - it poses a malleability risk:
  You can craft transactions that call a function ``f(uint8 x)`` with a raw byte argument
  of ``0xff000001`` and with ``0x00000001``. Both are fed to the contract and both will
  look like the number ``1`` as far as ``x`` is concerned, but ``msg.data`` will
  be different, so if you use ``keccak256(msg.data)`` for anything, you will get different results.

***************
Recommendations
***************

Restrict the Amount of Ether
============================

Restrict the amount of Ether (or other tokens) that can be stored in a smart
contract. If your source code, the compiler or the platform has a bug, these
funds may be lost. If you want to limit your loss, limit the amount of Ether.

Keep it Small and Modular
=========================

Keep your contracts small and easily understandable. Single out unrelated
functionality in other contracts or into libraries. General recommendations
about source code quality of course apply: Limit the amount of local variables,
the length of functions and so on. Document your functions so that others
can see what your intention was and whether it is different than what the code does.

Use the Checks-Effects-Interactions Pattern
===========================================

Most functions will first perform some checks (who called the function,
are the arguments in range, did they send enough Ether, does the person
have tokens, etc.). These checks should be done first.

As the second step, if all checks passed, effects to the state variables
of the current contract should be made. Interaction with other contracts
should be the very last step in any function.

Early contracts delayed some effects and waited for external function
calls to return in a non-error state. This is often a serious mistake
because of the re-entrancy problem explained above.

Note that, also, calls to known contracts might in turn cause calls to
unknown contracts, so it is probably better to just always apply this pattern.

Include a Fail-Safe Mode
========================

While making your system fully decentralised will remove any intermediary,
it might be a good idea, especially for new code, to include some kind
of fail-safe mechanism:

You can add a function in your smart contract that performs some
self-checks like "Has any Ether leaked?",
"Is the sum of the tokens equal to the balance of the contract?" or similar things.
Keep in mind that you cannot use too much gas for that, so help through off-chain
computations might be needed there.

If the self-check fails, the contract automatically switches into some kind
of "failsafe" mode, which, for example, disables most of the features, hands over
control to a fixed and trusted third party or just converts the contract into
a simple "give me back my money" contract.


*******************
Formal Verification
*******************

Using formal verification, it is possible to perform an automated mathematical
proof that your source code fulfills a certain formal specification.
The specification is still formal (just as the source code), but usually much
simpler. There is a prototype in Solidity that performs formal verification and
it will be better documented soon.

Note that formal verification itself can only help you understand the
difference between what you did (the specification) and how you did it
(the actual implementation). You still need to check whether the specification
is what you wanted and that you did not miss any unintended effects of it.
