#######
Diverso
#######

.. index:: storage, state variable, mapping

***************************************************
Layout de las variables de estado en almacenamiento
***************************************************

Variables de tamaño estáticas (todo menos mapping y tipos arrays de tamaño variable) se disponen contiguamente en almacenamiento empezando de la posición ``0``. Cuando multiples elementos necesitan menos de 32 bytes, son empaquetados en un slot de almacenamiento cuando posible de acuerdo a las reglas:

- El primer elemento en un slot de almacenamiento es almacenado alineado en lower-order.
- Tipos elementales usan sólo usan la cantidad de bytes que se necesita para almacenarlas.
- Si un tipo elemental no cabe en la parte restante de un slot de almacenamiento, es movido al próximo slot de almacenamiento.
- Structs y datos de array siempre comienzan en un nuevo slot y ocupan slots enteros (pero elementos dentro de un struct o array son empacados estrechamente de acuerdo a estas reglas).


.. warning::
    Cuando se usan elementos que son más pequeños que 32 bytes, el uso del gas del contrato puede
    ser más alto. Esto es porque la EVM opera en 32 bytes a la vez. Por lo tanto, si el elemento es
    más pequeño que eso, la EVM usa más operaciones para reducir el tamaño del elemento de 32 bytes
    al tamaño deseado.

    Sólo es benéfico de reducir el tamaño de los argumentos si estás tratando con valores de
    almacenamiento porque el compilador empacará multiples elementos en un slot de almacenamiento,
    y entonces, combina multiples lecturas y escrituras en una sólo operación. Cuando se trata con
    argumentos de función o valores de memoria, no hay beneficio inherente porque el compilador no
    empaca estos valores.

    Finalmente, para permitir la EVM de optimizar esto, asegúrate de ordenar la variables de
    almacenamiento y miembros ``struct`` para que puedan ser empacados estrechamente. Por ejemplo,
    declarando tus variables de almacenamiento en el orden de ``uint128, uint128, uint256`` en vez de
    ``uint128, uint256, uint128``, ya que el primero utilizará sólo dos slots de almacenamiento y éste
    último tres.

Los elementos de structs y arrays son almacenados después de ellos mismos, como si fueran dados explícitamente.

Dado a su tamaño impredecible, tipos de array dinámicos y de mapping usan computación hash Keccak-256
para encontrar la posición de inicio del valor o del dato del array. Estas posiciones de inicio
son siempre slos de full stack.

El mapping o el array dinámico en sí ocupa un slot (sin llenar) en alguna posición ``p`` de acuerdo
a la regla de arriba (o por aplicar esta regla de mappings a mappings o arrays de arrays). Para un
array dinámico, este slot guarda el número de elmentos en el array (los byte arrays y cadenas son
excepciones aquí, mirar abajo). Para un mapping, el slot no es utilizado (pero es necesario para que
dos mappings iguales seguidos usen diferentes distribuciones de hash).
Datos de array son ubicados en ``kecakk256(p)`` y el valor correspondiente a una llave de mapping
``k`` es ubicada en ``kecakk256(k . p)`` donde ``.`` es una concatenación. Si el valor de nuevo es
un tipo no elemental, las posiciones son encontradas agregando un offset de ``keccak256(k . p)``.

``bytes`` y ``string`` almacenan sus datos en el mismo slot donde también el largo es almacenado si son cortos. En particular: si el data es al menos ``31`` bytes de largo, es almacenado en bytes de orden mayor (alineados a la izquierda) y el byte de orden menor almacena ``length * 2``. Si es más largo, el slot principal almacena ``length * 2 + 1`` y los datos son almacenados como siempre en ``keccak256(slot)``.

Entonces para el siguiente sinppet de contrato::

    contract C {
      struct s { uint a; uint b; }
      uint x;
      mapping(uint => mapping(uint => s)) data;
    }

La posición de ``data[4][9].b`` está en ``keccak256(uint256(9) . keccak256(uint256(4) . uint256(1))) + 1``.

.. index: memory layout

*****************
Layout en Memoria
*****************

Solidity reserva tres slots de 256-bits:

- 0 - 64: espacio de scratch para métodos de hash
- 64 - 96: tamaño de memoria actualmente asignada (también conocida como free memory pointer)

El espacio de scratch puede ser usado entre declaraciones (ej. dento de inline assembly).

Solidity siempre emplaza los nuevos objetos en el puntero de memoria libre y la memoria nunca es liberada (esto puede cambiar en el futuro).

.. warning::
  Hay algunas operaciones en SOlidity que necesitan un área temporal de memoria mas grande que 64 bytes y por lo tanto no caben en el espacio scratch. Ellos serán emplazados donde apunta la memoria libre, pero dado su corta vida, el apuntador no es actualizado. La memoria puede o no ser puesta a cero. Por esto, uno no debiera esperar que la memoria libre sea puesta a cero.

.. index: calldata layout

*******************
Layout de Call Data
*******************

Cuando un contrato Solidity es despliegado y cuando es llamado de un
account, el data de input se asume que está en el formato de la
`espcificación ABI <https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI>`_.
La especificación ABI requiere que los argumentos sean acolchados a
multiples de 32 bytes. Las llamadas de función internas usan otra convención.

.. index: variable cleanup

******************************
Internas - Limpiando Variables
******************************

Cuando un valor es mas corto que 256-bit, en algunos casos los bits
restantes tienen que ser limpiados.
El compilador Solidity es diseñado para limpiar estos bits restantes antes
de cualquier operación que pueda ser afectada adversamente por la potencial
basura en los bits restantes.
Por ejemplo, antes de escribir un valor a la memoria los bits restantes tienen
que ser limpiados porque los contenidos de la memoria pueden ser usados para
computar hashes o enviados como data de un llamado de mensaje. De manera similar,
antes de almacenar un valor en el almacenamiento, los bits restantes tienen
que ser limpiados porque si no el valor ilegible puede ser observado.

Por otro lado, no limpiamos los bits si la operación siguiente no es afectada.
Por ejemplo, ya que cualquier valor no-cero es considerado ``true`` por
una instrucción ``JUMPI``, no limpiamos los valores booleanos antes que sean utiliziados
como condición para ``JUMPI``.



In addition to the design principle above, the Solidity compiler
cleans input data when it is loaded onto the stack.

Different types have different rules for cleaning up invalid values:

+---------------+---------------+-------------------+
|Type           |Valid Values   |Invalid Values Mean|
+===============+===============+===================+
|enum of n      |0 until n - 1  |exception          |
|members        |               |                   |
+---------------+---------------+-------------------+
|bool           |0 or 1         |1                  |
+---------------+---------------+-------------------+
|signed integers|sign-extended  |currently silently |
|               |word           |wraps; in the      |
|               |               |future exceptions  |
|               |               |will be thrown     |
|               |               |                   |
|               |               |                   |
+---------------+---------------+-------------------+
|unsigned       |higher bits    |currently silently |
|integers       |zeroed         |wraps; in the      |
|               |               |future exceptions  |
|               |               |will be thrown     |
+---------------+---------------+-------------------+

.. index:: optimizer, common subexpression elimination, constant propagation

*************************
Internals - The Optimizer
*************************

The Solidity optimizer operates on assembly, so it can be and also is used by other languages. It splits the sequence of instructions into basic blocks at JUMPs and JUMPDESTs. Inside these blocks, the instructions are analysed and every modification to the stack, to memory or storage is recorded as an expression which consists of an instruction and a list of arguments which are essentially pointers to other expressions. The main idea is now to find expressions that are always equal (on every input) and combine them into an expression class. The optimizer first tries to find each new expression in a list of already known expressions. If this does not work, the expression is simplified according to rules like ``constant + constant = sum_of_constants`` or ``X * 1 = X``. Since this is done recursively, we can also apply the latter rule if the second factor is a more complex expression where we know that it will always evaluate to one. Modifications to storage and memory locations have to erase knowledge about storage and memory locations which are not known to be different: If we first write to location x and then to location y and both are input variables, the second could overwrite the first, so we actually do not know what is stored at x after we wrote to y. On the other hand, if a simplification of the expression x - y evaluates to a non-zero constant, we know that we can keep our knowledge about what is stored at x.

At the end of this process, we know which expressions have to be on the stack in the end and have a list of modifications to memory and storage. This information is stored together with the basic blocks and is used to link them. Furthermore, knowledge about the stack, storage and memory configuration is forwarded to the next block(s). If we know the targets of all JUMP and JUMPI instructions, we can build a complete control flow graph of the program. If there is only one target we do not know (this can happen as in principle, jump targets can be computed from inputs), we have to erase all knowledge about the input state of a block as it can be the target of the unknown JUMP. If a JUMPI is found whose condition evaluates to a constant, it is transformed to an unconditional jump.

As the last step, the code in each block is completely re-generated. A dependency graph is created from the expressions on the stack at the end of the block and every operation that is not part of this graph is essentially dropped. Now code is generated that applies the modifications to memory and storage in the order they were made in the original code (dropping modifications which were found not to be needed) and finally, generates all values that are required to be on the stack in the correct place.

These steps are applied to each basic block and the newly generated code is used as replacement if it is smaller. If a basic block is split at a JUMPI and during the analysis, the condition evaluates to a constant, the JUMPI is replaced depending on the value of the constant, and thus code like

::

    var x = 7;
    data[7] = 9;
    if (data[x] != x + 2)
      return 2;
    else
      return 1;

is simplified to code which can also be compiled from

::

    data[7] = 9;
    return 1;

even though the instructions contained a jump in the beginning.

.. index:: source mappings

***************
Source Mappings
***************

As part of the AST output, the compiler provides the range of the source
code that is represented by the respective node in the AST. This can be
used for various purposes ranging from static analysis tools that report
errors based on the AST and debugging tools that highlight local variables
and their uses.

Furthermore, the compiler can also generate a mapping from the bytecode
to the range in the source code that generated the instruction. This is again
important for static analysis tools that operate on bytecode level and
for displaying the current position in the source code inside a debugger
or for breakpoint handling.

Both kinds of source mappings use integer indentifiers to refer to source files.
These are regular array indices into a list of source files usually called
``"sourceList"``, which is part of the combined-json and the output of
the json / npm compiler.

The source mappings inside the AST use the following
notation:

``s:l:f``

Where ``s`` is the byte-offset to the start of the range in the source file,
``l`` is the length of the source range in bytes and ``f`` is the source
index mentioned above.

The encoding in the source mapping for the bytecode is more complicated:
It is a list of ``s:l:f:j`` separated by ``;``. Each of these
elements corresponds to an instruction, i.e. you cannot use the byte offset
but have to use the instruction offset (push instructions are longer than a single byte).
The fields ``s``, ``l`` and ``f`` are as above and ``j`` can be either
``i``, ``o`` or ``-`` signifying whether a jump instruction goes into a
function, returns from a function or is a regular jump as part of e.g. a loop.

In order to compress these source mappings especially for bytecode, the
following rules are used:

 - If a field is empty, the value of the preceding element is used.
 - If a ``:`` is missing, all following fields are considered empty.

This means the following source mappings represent the same information:

``1:2:1;1:9:1;2:1:2;2:1:2;2:1:2``

``1:2:1;:9;2::2;;``

*****************
Contract Metadata
*****************

The Solidity compiler automatically generates a JSON file, the
contract metadata, that contains information about the current contract.
It can be used to query the compiler version, the sources used, the ABI
and NatSpec documentation in order to more safely interact with the contract
and to verify its source code.

The compiler appends a Swarm hash of the metadata file to the end of the
bytecode (for details, see below) of each contract, so that you can retrieve
the file in an authenticated way without having to resort to a centralized
data provider.

Of course, you have to publish the metadata file to Swarm (or some other service)
so that others can access it. The file can be output by using ``solc --metadata``
and the file will be called ``ContractName_meta.json``.
It will contain Swarm references to the source code, so you have to upload
all source files and the metadata file.

The metadata file has the following format. The example below is presented in a
human-readable way. Properly formatted metadata should use quotes correctly,
reduce whitespace to a minimum and sort the keys of all objects to arrive at a
unique formatting.
Comments are of course also not permitted and used here only for explanatory purposes.

.. code-block:: none

    {
      // Required: The version of the metadata format
      version: "1",
      // Required: Source code language, basically selects a "sub-version"
      // of the specification
      language: "Solidity",
      // Required: Details about the compiler, contents are specific
      // to the language.
      compiler: {
        // Required for Solidity: Version of the compiler
        version: "0.4.6+commit.2dabbdf0.Emscripten.clang",
        // Optional: Hash of the compiler binary which produced this output
        keccak256: "0x123..."
      },
      // Required: Compilation source files/source units, keys are file names
      sources:
      {
        "myFile.sol": {
          // Required: keccak256 hash of the source file
          "keccak256": "0x123...",
          // Required (unless "content" is used, see below): Sorted URL(s)
          // to the source file, protocol is more or less arbitrary, but a
          // Swarm URL is recommended
          "urls": [ "bzzr://56ab..." ]
        },
        "mortal": {
          // Required: keccak256 hash of the source file
          "keccak256": "0x234...",
          // Required (unless "url" is used): literal contents of the source file
          "content": "contract mortal is owned { function kill() { if (msg.sender == owner) selfdestruct(owner); } }"
        }
      },
      // Required: Compiler settings
      settings:
      {
        // Required for Solidity: Sorted list of remappings
        remappings: [ ":g/dir" ],
        // Optional: Optimizer settings (enabled defaults to false)
        optimizer: {
          enabled: true,
          runs: 500
        },
        // Required for Solidity: File and name of the contract or library this
        // metadata is created for.
        compilationTarget: {
          "myFile.sol": "MyContract"
        },
        // Required for Solidity: Addresses for libraries used
        libraries: {
          "MyLib": "0x123123..."
        }
      },
      // Required: Generated information about the contract.
      output:
      {
        // Required: ABI definition of the contract
        abi: [ ... ],
        // Required: NatSpec user documentation of the contract
        userdoc: [ ... ],
        // Required: NatSpec developer documentation of the contract
        devdoc: [ ... ],
      }
    }

.. note::
    Note the ABI definition above has no fixed order. It can change with compiler versions.

.. note::
    Since the bytecode of the resulting contract contains the metadata hash, any change to
    the metadata will result in a change of the bytecode. Furthermore, since the metadata
    includes a hash of all the sources used, a single whitespace change in any of the source
    codes will result in a different metadata, and subsequently a different bytecode.

Encoding of the Metadata Hash in the Bytecode
=============================================

Because we might support other ways to retrieve the metadata file in the future,
the mapping ``{"bzzr0": <Swarm hash>}`` is stored
[CBOR](https://tools.ietf.org/html/rfc7049)-encoded. Since the beginning of that
encoding is not easy to find, its length is added in a two-byte big-endian
encoding. The current version of the Solidity compiler thus adds the following
to the end of the deployed bytecode::

    0xa1 0x65 'b' 'z' 'z' 'r' '0' 0x58 0x20 <32 bytes swarm hash> 0x00 0x29

So in order to retrieve the data, the end of the deployed bytecode can be checked
to match that pattern and use the Swarm hash to retrieve the file.

Usage for Automatic Interface Generation and NatSpec
====================================================

The metadata is used in the following way: A component that wants to interact
with a contract (e.g. Mist) retrieves the code of the contract, from that
the Swarm hash of a file which is then retrieved.
That file is JSON-decoded into a structure like above.

The component can then use the ABI to automatically generate a rudimentary
user interface for the contract.

Furthermore, Mist can use the userdoc to display a confirmation message to the user
whenever they interact with the contract.

Usage for Source Code Verification
==================================

In order to verify the compilation, sources can be retrieved from Swarm
via the link in the metadata file.
The compiler of the correct version (which is checked to be part of the "official" compilers)
is invoked on that input with the specified settings. The resulting
bytecode is compared to the data of the creation transaction or CREATE opcode data.
This automatically verifies the metadata since its hash is part of the bytecode.
Excess data corresponds to the constructor input data, which should be decoded
according to the interface and presented to the user.


***************
Tips and Tricks
***************

* Use ``delete`` on arrays to delete all its elements.
* Use shorter types for struct elements and sort them such that short types are grouped together. This can lower the gas costs as multiple SSTORE operations might be combined into a single (SSTORE costs 5000 or 20000 gas, so this is what you want to optimise). Use the gas price estimator (with optimiser enabled) to check!
* Make your state variables public - the compiler will create :ref:`getters <visibility-and-getters>` for you automatically.
* If you end up checking conditions on input or state a lot at the beginning of your functions, try using :ref:`modifiers`.
* If your contract has a function called ``send`` but you want to use the built-in send-function, use ``address(contractVariable).send(amount)``.
* Initialise storage structs with a single assignment: ``x = MyStruct({a: 1, b: 2});``

**********
Cheatsheet
**********

.. index:: precedence

.. _order:

Order of Precedence of Operators
================================

The following is the order of precedence for operators, listed in order of evaluation.

+------------+-------------------------------------+--------------------------------------------+
| Precedence | Description                         | Operator                                   |
+============+=====================================+============================================+
| *1*        | Postfix increment and decrement     | ``++``, ``--``                             |
+            +-------------------------------------+--------------------------------------------+
|            | Function-like call                  | ``<func>(<args...>)``                      |
+            +-------------------------------------+--------------------------------------------+
|            | Array subscripting                  | ``<array>[<index>]``                       |
+            +-------------------------------------+--------------------------------------------+
|            | Member access                       | ``<object>.<member>``                      |
+            +-------------------------------------+--------------------------------------------+
|            | Parentheses                         | ``(<statement>)``                          |
+------------+-------------------------------------+--------------------------------------------+
| *2*        | Prefix increment and decrement      | ``++``, ``--``                             |
+            +-------------------------------------+--------------------------------------------+
|            | Unary plus and minus                | ``+``, ``-``                               |
+            +-------------------------------------+--------------------------------------------+
|            | Unary operations                    | ``delete``                                 |
+            +-------------------------------------+--------------------------------------------+
|            | Logical NOT                         | ``!``                                      |
+            +-------------------------------------+--------------------------------------------+
|            | Bitwise NOT                         | ``~``                                      |
+------------+-------------------------------------+--------------------------------------------+
| *3*        | Exponentiation                      | ``**``                                     |
+------------+-------------------------------------+--------------------------------------------+
| *4*        | Multiplication, division and modulo | ``*``, ``/``, ``%``                        |
+------------+-------------------------------------+--------------------------------------------+
| *5*        | Addition and subtraction            | ``+``, ``-``                               |
+------------+-------------------------------------+--------------------------------------------+
| *6*        | Bitwise shift operators             | ``<<``, ``>>``                             |
+------------+-------------------------------------+--------------------------------------------+
| *7*        | Bitwise AND                         | ``&``                                      |
+------------+-------------------------------------+--------------------------------------------+
| *8*        | Bitwise XOR                         | ``^``                                      |
+------------+-------------------------------------+--------------------------------------------+
| *9*        | Bitwise OR                          | ``|``                                      |
+------------+-------------------------------------+--------------------------------------------+
| *10*       | Inequality operators                | ``<``, ``>``, ``<=``, ``>=``               |
+------------+-------------------------------------+--------------------------------------------+
| *11*       | Equality operators                  | ``==``, ``!=``                             |
+------------+-------------------------------------+--------------------------------------------+
| *12*       | Logical AND                         | ``&&``                                     |
+------------+-------------------------------------+--------------------------------------------+
| *13*       | Logical OR                          | ``||``                                     |
+------------+-------------------------------------+--------------------------------------------+
| *14*       | Ternary operator                    | ``<conditional> ? <if-true> : <if-false>`` |
+------------+-------------------------------------+--------------------------------------------+
| *15*       | Assignment operators                | ``=``, ``|=``, ``^=``, ``&=``, ``<<=``,    |
|            |                                     | ``>>=``, ``+=``, ``-=``, ``*=``, ``/=``,   |
|            |                                     | ``%=``                                     |
+------------+-------------------------------------+--------------------------------------------+
| *16*       | Comma operator                      | ``,``                                      |
+------------+-------------------------------------+--------------------------------------------+

.. index:: assert, block, coinbase, difficulty, number, block;number, timestamp, block;timestamp, msg, data, gas, sender, value, now, gas price, origin, revert, require, keccak256, ripemd160, sha256, ecrecover, addmod, mulmod, cryptography, this, super, selfdestruct, balance, send

Global Variables
================

- ``block.blockhash(uint blockNumber) returns (bytes32)``: hash of the given block - only works for 256 most recent blocks
- ``block.coinbase`` (``address``): current block miner's address
- ``block.difficulty`` (``uint``): current block difficulty
- ``block.gaslimit`` (``uint``): current block gaslimit
- ``block.number`` (``uint``): current block number
- ``block.timestamp`` (``uint``): current block timestamp
- ``msg.data`` (``bytes``): complete calldata
- ``msg.gas`` (``uint``): remaining gas
- ``msg.sender`` (``address``): sender of the message (current call)
- ``msg.value`` (``uint``): number of wei sent with the message
- ``now`` (``uint``): current block timestamp (alias for ``block.timestamp``)
- ``tx.gasprice`` (``uint``): gas price of the transaction
- ``tx.origin`` (``address``): sender of the transaction (full call chain)
- ``assert(bool condition)``: abort execution and revert state changes if condition is ``false`` (use for internal error)
- ``require(bool condition)``: abort execution and revert state changes if condition is ``false`` (use for malformed input)
- ``revert()``: abort execution and revert state changes
- ``keccak256(...) returns (bytes32)``: compute the Ethereum-SHA-3 (Keccak-256) hash of the (tightly packed) arguments
- ``sha3(...) returns (bytes32)``: an alias to `keccak256()`
- ``sha256(...) returns (bytes32)``: compute the SHA-256 hash of the (tightly packed) arguments
- ``ripemd160(...) returns (bytes20)``: compute the RIPEMD-160 hash of the (tightly packed) arguments
- ``ecrecover(bytes32 hash, uint8 v, bytes32 r, bytes32 s) returns (address)``: recover address associated with the public key from elliptic curve signature, return zero on error
- ``addmod(uint x, uint y, uint k) returns (uint)``: compute ``(x + y) % k`` where the addition is performed with arbitrary precision and does not wrap around at ``2**256``
- ``mulmod(uint x, uint y, uint k) returns (uint)``: compute ``(x * y) % k`` where the multiplication is performed with arbitrary precision and does not wrap around at ``2**256``
- ``this`` (current contract's type): the current contract, explicitly convertible to ``address``
- ``super``: the contract one level higher in the inheritance hierarchy
- ``selfdestruct(address recipient)``: destroy the current contract, sending its funds to the given address
- ``<address>.balance`` (``uint256``): balance of the :ref:`address` in Wei
- ``<address>.send(uint256 amount) returns (bool)``: send given amount of Wei to :ref:`address`, returns ``false`` on failure
- ``<address>.transfer(uint256 amount)``: send given amount of Wei to :ref:`address`, throws on failure

.. index:: visibility, public, private, external, internal

Function Visibility Specifiers
==============================

::

    function myFunction() <visibility specifier> returns (bool) {
        return true;
    }

- ``public``: visible externally and internally (creates getter function for storage/state variables)
- ``private``: only visible in the current contract
- ``external``: only visible externally (only for functions) - i.e. can only be message-called (via ``this.func``)
- ``internal``: only visible internally


.. index:: modifiers, constant, anonymous, indexed

Modifiers
=========

- ``constant`` for state variables: Disallows assignment (except initialisation), does not occupy storage slot.
- ``constant`` for functions: Disallows modification of state - this is not enforced yet.
- ``anonymous`` for events: Does not store event signature as topic.
- ``indexed`` for event parameters: Stores the parameter as topic.
- ``payable`` for functions: Allows them to receive Ether together with a call.

Reserved Keywords
=================

These keywords are reserved in Solidity. They might become part of the syntax in the future:

``abstract``, ``after``, ``case``, ``catch``, ``default``, ``final``, ``in``, ``inline``, ``interface``, ``let``, ``match``, ``null``,
``of``, ``pure``, ``relocatable``, ``static``, ``switch``, ``try``, ``type``, ``typeof``, ``view``.

Language Grammar
================

.. literalinclude:: grammar.txt
   :language: none
