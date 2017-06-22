********************************************
Unidades y Variables Disponibles Globalmente
********************************************

.. index:: wei, finney, szabo, ether

Unidades de Ether
=================

Un numero literal puede tomar un sufijo como el ``wei``, el ``finney``, el ``szabo`` o el ``ether`` para convertirlo entre las subdenominaciones del Ether. Se asume que un numero sin sufijo para representar la moneda Ether está expresado en Wei, por ejemplo, ``2 ether == 2000 finney`` devolverá ``true``.

.. index:: tiempo, segundos, minutos, horas, días, semanas, años

Unidades de Tiempo
==================

Sufijos como ``seconds``, ``minutes``, ``hours``, ``days``, ``weeks`` y ``years`` utilizados después de numeros literales pueden usarse para convertir unidades de tiempo donde los segundos son la unidad de base. Se consideran las unidades de la forma ingénua siguiente:

 * ``1 == 1 seconds``
 * ``1 minutes == 60 seconds``
 * ``1 hours == 60 minutes``
 * ``1 days == 24 hours``
 * ``1 weeks == 7 days``
 * ``1 years == 365 days``

Ójo si utiliza esta unidades para realizar calculos de calendario, porque no todos los años tienen 365 días y no todos los días tienen 24 horas, por culpa de los `_segundos intercalares <https://es.wikipedia.org/wiki/Segundo_intercalar>`_. Debido a que los segundos intercalares no son predecibles, la librería del calendario extacto tiene que estar actualizada por un oráculo externo.

Estos sufijos no pueden aplicarse a variables. Si desea interpretar algunas variables de entrada, como por ejemplo días, puede hacerlo de siguiente forma::

    function f(uint start, uint daysAfter) {
        if (now >= start + daysAfter * 1 days) { ... }
    }

Variables y Funciones Especiales
================================

Existen variables y funciones especiales que siempre están disponibles en el espacio de nombre global y que se usan principalmente para proporcionar información sobre la blockchain.

.. index:: bloque, coinbase, dificultad, número, bloque;número, sello de tiempo, bloque;sello de tiempo, msg, dato, gas, remitente, valor, now, precio del gas, orígen


Bloque y Propiedades de las Transacciones
-----------------------------------------

- ``block.blockhash(uint blockNumber) returns (bytes32)``: el hash de un bloque dado - sólo funciona para los 256 bloques más recientes, excluyendo el actual
- ``block.coinbase`` (``address``): devuelve la dirección del minero que está procesando el bloque actual
- ``block.difficulty`` (``uint``): devuelve la dificultad del bloque actual
- ``block.gaslimit`` (``uint``): devuelve el límite de gas del bloque actual
- ``block.number`` (``uint``): devuelve el numero del bloque actual
- ``block.timestamp`` (``uint``): devuelve el sello de tiempo (timestamp) del bloque actual en forma de segundos desde (unix epoch???)
- ``msg.data`` (``bytes``): ??? (complete calldata)
- ``msg.gas`` (``uint``): devuelve el gas que queda
- ``msg.sender`` (``address``): devuelve remitente del mensaje (de la llamada actual)
- ``msg.sig`` (``bytes4``): devuelve los primeros cuatro bytes de la ??? (calldata) (p.ej. la función identifier)
- ``msg.value`` (``uint``): devuelve el numero de Wei enviado con el mensaje
- ``now`` (``uint``): devuelve el sello de tiempo del bloque actual (es un alias de ``block.timestamp``)
- ``tx.gasprice`` (``uint``): devuelve el precio del gas de la transacción
- ``tx.origin`` (``address``): devuelve el remitente de la transacción (??? (full call chain))

.. note::
    Los valores de todos los (???miembros) de ``msg``, incluido ``msg.sender`` y ``msg.value`` pueden cambiar para cada llamada a una función **externa**. Eso incluye las llamadas a funciones de una librería también.
    
    Si desea implementar restricciones de acceso para funciones de una librería utilizando ``msg.sender``, tiene que proporcionar manualmente el valor de ``msg.sender`` como argumento.
    
.. note::
    Los hashes de los bloques no están disponibles para todos los bloques por motivos de escalabilidad. Sólo se puede acceder a los hashes de los 256 bloques más recientes. El valor del hash para bloques más antiguos será cero.

.. index:: assert, revert, keccak256, ripemd160, sha256, ecrecover, addmod, mulmod, cryptography, this, super, selfdestruct, balance, send

Mathematical and Cryptographic Functions
----------------------------------------

``assert(bool condition)``:
    throws if the condition is not met.
``addmod(uint x, uint y, uint k) returns (uint)``:
    compute ``(x + y) % k`` where the addition is performed with arbitrary precision and does not wrap around at ``2**256``.
``mulmod(uint x, uint y, uint k) returns (uint)``:
    compute ``(x * y) % k`` where the multiplication is performed with arbitrary precision and does not wrap around at ``2**256``.
``keccak256(...) returns (bytes32)``:
    compute the Ethereum-SHA-3 (Keccak-256) hash of the (tightly packed) arguments
``sha3(...) returns (bytes32)``:
    alias to ``keccak256()``
``sha256(...) returns (bytes32)``:
    compute the SHA-256 hash of the (tightly packed) arguments
``ripemd160(...) returns (bytes20)``:
    compute RIPEMD-160 hash of the (tightly packed) arguments
``ecrecover(bytes32 hash, uint8 v, bytes32 r, bytes32 s) returns (address)``:
    recover the address associated with the public key from elliptic curve signature or return zero on error
    (`example usage <https://ethereum.stackexchange.com/q/1777/222>`_)
``revert()``:
    abort execution and revert state changes

In the above, "tightly packed" means that the arguments are concatenated without padding.
This means that the following are all identical::

    keccak256("ab", "c")
    keccak256("abc")
    keccak256(0x616263)
    keccak256(6382179)
    keccak256(97, 98, 99)

If padding is needed, explicit type conversions can be used: ``keccak256("\x00\x12")`` is the
same as ``keccak256(uint16(0x12))``.

Note that constants will be packed using the minimum number of bytes required to store them.
This means that, for example, ``keccak256(0) == keccak256(uint8(0))`` and
``keccak256(0x12345678) == keccak256(uint32(0x12345678))``.

It might be that you run into Out-of-Gas for ``sha256``, ``ripemd160`` or ``ecrecover`` on a *private blockchain*. The reason for this is that those are implemented as so-called precompiled contracts and these contracts only really exist after they received the first message (although their contract code is hardcoded). Messages to non-existing contracts are more expensive and thus the execution runs into an Out-of-Gas error. A workaround for this problem is to first send e.g. 1 Wei to each of the contracts before you use them in your actual contracts. This is not an issue on the official or test net.

.. _address_related:

Address Related
---------------

``<address>.balance`` (``uint256``):
    balance of the :ref:`address` in Wei
``<address>.transfer(uint256 amount)``:
    send given amount of Wei to :ref:`address`, throws on failure
``<address>.send(uint256 amount) returns (bool)``:
    send given amount of Wei to :ref:`address`, returns ``false`` on failure
``<address>.call(...) returns (bool)``:
    issue low-level ``CALL``, returns ``false`` on failure
``<address>.callcode(...) returns (bool)``:
    issue low-level ``CALLCODE``, returns ``false`` on failure
``<address>.delegatecall(...) returns (bool)``:
    issue low-level ``DELEGATECALL``, returns ``false`` on failure

For more information, see the section on :ref:`address`.

.. warning::
    There are some dangers in using ``send``: The transfer fails if the call stack depth is at 1024
    (this can always be forced by the caller) and it also fails if the recipient runs out of gas. So in order
    to make safe Ether transfers, always check the return value of ``send``, use ``transfer`` or even better:
    Use a pattern where the recipient withdraws the money.

.. index:: this, selfdestruct

Contract Related
----------------

``this`` (current contract's type):
    the current contract, explicitly convertible to :ref:`address`

``selfdestruct(address recipient)``:
    destroy the current contract, sending its funds to the given :ref:`address`

Furthermore, all functions of the current contract are callable directly including the current function.

