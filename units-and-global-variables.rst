********************************************
Unidades y variables disponibles globalmente
********************************************

.. index:: wei, finney, szabo, ether

Unidades de Ether
=================

Un numero literal puede tomar un sufijo como el ``wei``, el ``finney``, el ``szabo`` o el ``ether`` para convertirlo entre las subdenominaciones del Ether. Se asume que un numero sin sufijo para representar la moneda Ether está expresado en Wei, por ejemplo, ``2 ether == 2000 finney`` devuelve ``true``.

.. index:: time, seconds, minutes, hours, days, weeks, years

Unidades de tiempo
==================

Sufijos como ``seconds``, ``minutes``, ``hours``, ``days``, ``weeks`` y ``years`` utilizados después de numeros literales pueden usarse para convertir unidades de tiempo donde los segundos son la unidad de base. Las equivalencias son las siguientes:

 * ``1 == 1 seconds``
 * ``1 minutes == 60 seconds``
 * ``1 hours == 60 minutes``
 * ``1 days == 24 hours``
 * ``1 weeks == 7 days``
 * ``1 years == 365 days``

Ojo si utilizan estas unidades para realizar cálculos de calendario, porque no todos los años tienen 365 días y no todos los días tienen 24 horas, por culpa de los `_segundos intercalares <https://es.wikipedia.org/wiki/Segundo_intercalar>`_. Debido a que los segundos intercalares no son predecibles, la librería del calendario extacto tiene que estar actualizada por un oráculo externo.

Estos sufijos no pueden aplicarse a variables. Si desea interpretar algunas variables de entrada, como por ejemplo días, puede hacerlo de siguiente forma::

    function f(uint start, uint daysAfter) {
        if (now >= start + daysAfter * 1 days) { ... }
    }

Variables y funciones especiales
================================

Existen variables y funciones especiales de ámbito global que siempre están disponibles y que se usan principalmente para proporcionar información sobre la blockchain.

.. index:: block, coinbase, difficulty, number, block;number, timestamp, block;timestamp, msg, data, gas, sender, value, now, gas price, origin

Bloque y propiedades de las transacciones
-----------------------------------------

- ``block.blockhash(uint blockNumber) returns (bytes32)``: el hash de un bloque dado - sólo funciona para los 256 bloques más recientes, excluyendo el actual
- ``block.coinbase`` (``address``): devuelve la dirección del minero que está procesando el bloque actual
- ``block.difficulty`` (``uint``): devuelve la dificultad del bloque actual
- ``block.gaslimit`` (``uint``): devuelve el límite de gas del bloque actual
- ``block.number`` (``uint``): devuelve el número del bloque actual
- ``block.timestamp`` (``uint``): devuelve el timestamp del bloque actual en forma de segundos siguiendo el tiempo universal de Unix (Unix epoch)
- ``msg.data`` (``bytes``): datos enviados en la transacción (calldata)
- ``msg.gas`` (``uint``): devuelve el gas que queda
- ``msg.sender`` (``address``): devuelve remitente de la llamada actual
- ``msg.sig`` (``bytes4``): devuelve los primeros cuatro bytes de los datos enviados en la transacción (i.e. el identificador de la función)
- ``msg.value`` (``uint``): devuelve el numero de Wei enviado con la llamada
- ``now`` (``uint``): devuelve el timestamp del bloque actual (es un alias de ``block.timestamp``)
- ``tx.gasprice`` (``uint``): devuelve el precio del gas de la transacción
- ``tx.origin`` (``address``): devuelve el emisor original de la transacción

.. note::
    Los valores de todos los elementos de ``msg``, incluido ``msg.sender`` y ``msg.value`` pueden cambiar para cada llamada a una función **externa**. Incluyendo las llamadas a funciones de una librería.
    
    Si desea implementar restricciones de acceso para funciones de una librería utilizando ``msg.sender``, tiene que proporcionar manualmente el valor de ``msg.sender`` como argumento.
    
.. note::
    Los hashes de los bloques no están disponibles para todos los bloques por motivos de escalabilidad. Sólo se puede acceder a los hashes de los 256 bloques más recientes. El valor del hash para bloques más antiguos será cero.

.. index:: assert, revert, keccak256, ripemd160, sha256, ecrecover, addmod, mulmod, cryptography, this, super, selfdestruct, balance, send

Funciones matemáticas y criptográficas
--------------------------------------

``assert(bool condition)``: lanza excepción si la condición no está satisfecha.
``addmod(uint x, uint y, uint k) returns (uint)``: computa ``(x + y) % k`` donde la suma se realiza con una precisión arbitraria y no se desborda en ``2**256``.
``mulmod(uint x, uint y, uint k) returns (uint)``: computa ``(x * y) % k`` donde la multiplicación se realiza con una precisión arbitraria y no se desborda en ``2**256``.
``keccak256(...) returns (bytes32)``: computa el hash de Ethereum-SHA-3 (Keccak-256) de la unión (compactada) de los argumentos.
``sha3(...) returns (bytes32)``: equivalente a ``keccak256()``.
``sha256(...) returns (bytes32)``: computa el hash de SHA-256 de la unión (compactada) de los argumentos.
``ripemd160(...) returns (bytes20)``: computa el hash de RIPEMD-160 de la unión (compactada) de los argumentos.
``ecrecover(bytes32 hash, uint8 v, bytes32 r, bytes32 s) returns (address)``: recupera la dirección asociada a la clave pública de la firma de tipo curva elíptica o devuelve cero si hay un error (`ejempo de uso <https://ethereum.stackexchange.com/q/1777/222>`_).
``revert()``: aborta la ejecución y revierte los cambios de estado a como estaban.

Más arriba, "compactado" significa que los argumentos están concatenados sin relleno (padding).
Esto significa que las siguientes llamadas son todas identicas::

    keccak256("ab", "c")
    keccak256("abc")
    keccak256(0x616263)
    keccak256(6382179)
    keccak256(97, 98, 99)

Si hiciera falta relleno (padding), se pueden usar las conversiones explícitas de tipo: ``keccak256("\x00\x12")`` es lo mismo que ``keccak256(uint16(0x12))``.

Ten en cuenta que las constantes se compactarán usando el mínimo numero de bytes requeridos para almacenarlas.
Eso significa por ejemplo que ``keccak256(0) == keccak256(uint8(0))`` y ``keccak256(0x12345678) == keccak256(uint32(0x12345678))``.

Podría pasar que le falte gas para llamar a las funciones ``sha256``, ``ripemd160`` or ``ecrecover`` en *blockchain privadas*. La razón de ser así se debe a que esas funciones están implementadas como contratos precompilados y estos contratos sólo existen depués de recibir el primer mensaje (a pesar de que el contrato está hardcodeado). Los mensajes que se envían a contratos que todavía no existen son más caros y por lo tanto su ejecución puede llevar a un error por una falta de gas (Out-of-Gas error). Una solución a este problema consiste por ejemplo en mandar 1 Wei a todos los contratos antes de empezar a usarlos. Note que esto no llevará a un error en la red oficial o en la red de testeo.

.. _address_related:

En relación a las direcciones
-----------------------------

``<address>.balance`` (``uint256``): balance en Wei de la :ref:`dirección <address>`.
``<address>.transfer(uint256 amount)``: envía el importe deseado en Wei a la :ref:`dirección <address>` o lanza excepción si falla.
``<address>.send(uint256 amount) returns (bool)``: envía el importe deseado en Wei a la :ref:`dirección <address>` o devuelve ``false`` si falla.
``<address>.call(...) returns (bool)``: crea una instrucción de tipo ``CALL`` a bajo nivel o devuelve ``false`` si falla.
``<address>.callcode(...) returns (bool)``: crea una instrucción de tipo ``CALLCODE`` a bajo nivel o devuelve ``false`` si falla.
``<address>.delegatecall(...) returns (bool)``: crea una instrucción de tipo ``DELEGATECALL`` a bajo nivel o devuelve ``false`` si falla.

Para más información, véase la sección :ref:`dirección <address>`.

.. warning::
    Existe un peligro a la hora de usar ``send``: la transferencia falla si la profundidad de la pila de llamadas es de 1024 (esto siempre lo puede forzar el que hace la llamada); también falla si el destinatario se queda sin gas. Entonces, para asegurarse de hacer transferencias seguras en Ether, fíjese siempre en el valor devuelto por ``send``, use ``transfer`` en lugar de ``send``, o mejor aún, use un patrón donde es el destinatario quien retira los fondos.

.. index:: this, selfdestruct

En relación a los contratos
---------------------------

``this`` (el tipo del contrato actual): el contrato actual, explicitamente convertible en :ref:`address`.
``selfdestruct(address recipient)``: destruye el contrato actual y envía los fondos que tiene a una :ref:`dirección <address>` especificada.

Además, todas las funciones del contrato actual se pueden llamar directamente, incluida la función actual.
