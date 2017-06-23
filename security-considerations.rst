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
Si sólo se guarda la lista de compras en ese servicio web, puede que no tengas
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
el siguiente código contiene un error (esto es sólo un snippet y no un contrato
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

Nótese que reingreso no sólo es un riesgo de transferencia de Ether, pero
de cualquier ejecución de una función de otro contrato. Además, también tienes que
considerar situaciones de multi-contrato. Un contrato ejecutado podría modificar el
estado de otro contrato del cual dependes.

Límite de Gas y Bucles (Loops)
==============================

Bucles que no tienen un número fijo de iteraciones, por ejemplo, bucles que dependen de valores de almacenamiento, tienen que
usarse con cuidado:
Dado al límite de gas del bloque, las transacciones pueden sólamente consumir una cierta cantidad de gas. Ya sea explícitamente,
o por une operación normal, el número de iteraciones en un loop puede crecer más allá del límite de gas que puede causar el
contrato completo a detenerse a un cierto punto. Esto no se aplica a funciones ``constant`` que son sólo llamadas para
leer data de la blockchain. Pero aún así, estas funciones pueden ser llamadas por otros contratos como parte de operaciones
en la cadena (on-chain) y detenerlas a ellas. Por favor ser explícito con estos casos en la documentación de tus contratos.

Enviando y Recibiendo Ether
===========================

- Ni contratos ni cuentas externas ("external accoutns"), son actualmente capaces de prevenir que alguien
  les envíe Ether. Contratos pueden reaccionar y rechazar una transferencia normal, pero hay maneras de mover
  Ether sin crear un mensaje de llamado (message call). Una manera de de simplemente "minando" a la
  dirección del contrato y la otra es usando ``selfdestruct(x)``.

- Si un contrato recibe Ether (sin que una función sea llamada), la función de respaldo es ejecutada.
  Si no tiene función de respaldo, el Ether será rechazado (lanzando una excepción).
  Durante la ejecución de la función de respaldo, el contrato solo puede depender del
  "estipendio de gas" (2300 gas) que tiene disponible en ese momento. Esto estipendio no es suficiente para acceder
  el almacenimiento de ningua forma. Para asegurarte que tu contrato puede recibir Ether en ese modo, verifica los
  requerimientos de gas de la función de respaldo (por ejemplo en la sección de "details" de Remix).

# TODO: continue from here.

- There is a way to forward more gas to the receiving contract using
  ``addr.call.value(x)()``. This is essentially the same as ``addr.transfer(x)``,
  only that it forwards all remaining gas and opens up the ability for the
  recipient to perform more expensive actions (and it only returns a failure code
  and does not automatically propagate the error). This might include calling back
  into the sending contract or other state changes you might not have thought of.
  So it allows for great flexibility for honest users but also for malicious actors.

- If you want to send Ether using ``address.transfer``, there are certain details to be aware of:

  1. If the recipient is a contract, it causes its fallback function to be executed which can, in turn, call back the sending contract.
  2. Sending Ether can fail due to the call depth going above 1024. Since the caller is in total control of the call
     depth, they can force the transfer to fail; take this possibility into account or use ``send`` and make sure to always check its return value. Better yet,
     write your contract using a pattern where the recipient can withdraw Ether instead.
  3. Sending Ether can also fail because the execution of the recipient contract
     requires more than the allotted amount of gas (explicitly by using ``require``,
     ``assert``, ``revert``, ``throw`` or
     because the operation is just too expensive) - it "runs out of gas" (OOG).
     If you use ``transfer`` or ``send`` with a return value check, this might provide a
     means for the recipient to block progress in the sending contract. Again, the best practice here is to use
     a :ref:`"withdraw" pattern instead of a "send" pattern <withdrawal_pattern>`.

Callstack Depth
===============

External function calls can fail any time because they exceed the maximum
call stack of 1024. In such situations, Solidity throws an exception.
Malicious actors might be able to force the call stack to a high value
before they interact with your contract.

Note that ``.send()`` does **not** throw an exception if the call stack is
depleted but rather returns ``false`` in that case. The low-level functions
``.call()``, ``.callcode()`` and ``.delegatecall()`` behave in the same way.

tx.origin
=========

Never use tx.origin for authorization. Let's say you have a wallet contract like this:

::

    pragma solidity ^0.4.11;

    // THIS CONTRACT CONTAINS A BUG - DO NOT USE
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

Now someone tricks you into sending ether to the address of this attack wallet:

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

If your wallet had checked ``msg.sender`` for authorization, it would get the address of the attack wallet, instead of the owner address. But by checking ``tx.origin``, it gets the original address that kicked off the transaction, which is still the owner address. The attack wallet instantly drains all your funds.


Minor Details
=============

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
