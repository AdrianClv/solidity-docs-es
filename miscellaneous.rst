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
account, el data de entrada se asume que está en el formato de la
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

Además de este principio de diseño, el compilador Solidity
limpia los datos de entrada cuando está cargado en el stack.

Diferentes tipos tienen diferentes reglas para limpiar valores inválidos:

+---------------+---------------+--------------------+
|Tipo           |Valores válidos|Val. inv. significan|
+===============+===============+====================+
|enum de n      |0 hasta n - 1  |excepción          |
|miembros       |               |                   |
+---------------+---------------+--------------------+
|bool           |0 o  1         |1                  |
+---------------+---------------+--------------------+
|enteros con    |palabra        |por ahora envuelve |
|signo          |extendida por  |silenciosamente;   |
|               |signo          |en el futuro esto  |
|               |               |arrojará           |
|               |               |excepciones        |
|               |               |                   |
+---------------+---------------+--------------------+
|enteros        |altos bits     |por ahora envuelve |
|sin signos     |zeroed         |silenciosamente;   |
|               |               |en el futuro esto  |
|               |               |arrojará           |
|               |               |excepciones        |
+---------------+---------------+--------------------+


.. index:: optimizer, common subexpression elimination, constant propagation

*************************
Internos - El optimizador
*************************

El optimizador solidity funciona con assembly, así que puede y es usado con otros idiomas. Divide la secuencia de las instrucciones en bloques básicos de JUMPs y JUMPDESTs. Dentro de estos bloques, las instrucciones son analizadas y cada modificación al stack, a la memoria o al almacenamiento son guardadas como una expresión que consiste en una instrucción y una lista de argumentos que son esencialmente apuntadores a otras expresiones. La idea principal es encontrar expreciones que sean siempre iguales (en todo entradas) y combinatlas a una classe de expression. El optimizador primero intenta encontrar una nueva expresión en una lista de expresiones conocidas. Si esto no funciona, la expresión es simplificada de acuerdo a reglas como ``constante + constante = suma_de_constantes`` o ``X * 1 = X``. Ya que esto es hecho recursivamente, también podemos aplicar la última regla si el segundo factor es más una expresión más compleja donde sabemos que siempre evaluará a uno. Modificaciones al almacenamiento y ubicaciones de memoria tienen que borrar el conocimiento de almacenamiento y ubicaciones de memoriaque no son conocidas como diferentes: Si primero escribimos a ubicación `x` y luego a ubicación `y` y ambas son variables de entrada, la segunda puede sobre escribir la primera, entonces no sabemos realmente lo que es almacenadoen `x` después de escribir a `y`. Por otro lado, si una simplificación de la expresión `x -y` evalúa a una constante no cero, sabemos que podemos mantener nuestro conocimiento de lo que es almacenado en x.

En el fin de este proceso, sabemos cuales expresiones tienen que estar en el stack en el fin y tienen una lista de modificaciones a la memoria y almacenamiento. Esta información es almacenada junta con los bloques básicos y es usada para unirlas. Además, información sobre el stack, almacenamiento y configuración de memoria es enviada al (los) próximo(s) bloque(s). Si sabemos los objetivos de cada una de las instrucciones JUMP y JUMPI, podemos construir un gráfico de flujo completo del programa. Si hay un sólo objetivo que no conocemos (esto puede pasar ya que en principio, los objetivos de jumps pueden ser computados de las entradas), tenemos que borrar toda información sobre los estados de entrada de un bloque ya que puede ser el objetivo de del JUMP desconocido. Si un JUMPI es encontrado que la condición evalúa a una constante, es transformada en un jump incondicional.

Como en el último paso, el código en cada bloque es completamente regenerado. Un gráfico de dependencias es creado de la expresión en el stack en el fin del bloque y cada operación que no es parte de este gráfico es esencialmente olvidado. Ahora código es generado que aplica las modificaciones a la memoria y al almacenamiento en el orden que fueron hechas en el código original (olvidandon modificaciones que fueron encontradas innecesarias) y finalmente, genera todos valores que son requeridos para estar en el stack en el lugar correcto.

Estos pasos son aplicados a cada bloque básico y el código nuevo es usado remplazando si es que es más pequeño. Si un bloque básico es dividido en un JUMPI y durante el análisis, la condición evalúa a una constante el JUMPI es reemplazado dependiendo de la valor de la constante, y por lo tanto código como

::

    var x = 7;
    data[7] = 9;
    if (data[x] != x + 2)
      return 2;
    else
      return 1;

es simplificado a código que también puede ser compilado de

::

    data[7] = 9;
    return 1;

aunque las instrucciones contenían un jump en el inicio.

.. index:: source mappings

******************
Mappings de Fuente
******************

Como parte de la output AST, el compilador provee el rango del código fuente
que es representado por el nodo respectio en el AST. Esto puede ser usado
para varios propísitos desde herramientas de análisis estático que reportan
errores basados en el AST y herramientas de debugging que demarcan variables
locales y sus usos.

Además, el compilador puede también generar un mapping del bytecode
al rango en el código fuente que generó la instrucción. Esto es importante
para herramientas de análisis estático que operan a nivel bytecode y
para mostrar la posición actual en el código fuente dentro del debugger
o para manejar los breakpoints.

Ambos tipos de mappings de fuente usan identificadores enteros para referirse a
archivos fuente. Estos son índices de arrays regulares en una lista de archivos
fuente habitualmente llamados ``"sourcelist"``, que es parte del combined-json y
el output del compilador json / npm.

Los mappings de fuente dentro del AST usan la siguiente notación:

``s:l:f``

Donde ``s`` es el byte-offset de el inicio del rango en el archivo fuente,
``l`` es el largo del rango de la fuente een bytes y ``f`` es el índice de fuente
mencionado arriba.

El encodaje en el mapping de fuente para el bytecode es más complicado:
Es una lista de ``s:l:f:j`` separada por ``;``. Cada una de estos elementos
corresponde a una instrucción, i.e. no puedes usar el byte-offset
si no que tienes que usar la instrucción offset (instrucciones push son mas largas
que un sólo byte).
Los campos ``s``, ``l`` y ``f`` son como detallamos arriba y ``j`` puede ser ``i``,
``i`` o ``-`` y significa si una instrucción jump va en la función, devuelve una
función, devuelve desde una función o si es un jump regular como parte de un (ej) loop.

A fin de comprimir estos mappings de fuente espcialmente para bytecode, las
siguientes reglas son usadas:

 - Si un campo está vacío, el valor del elemento precedente es usado.
 - Si un ``:`` falta, todos los campos siguientes son considerados vacíos.

Esto significa que los suguientes mappings de fuente representan la misma información:

``1:2:1;1:9:1;2:1:2;2:1:2;2:1:2``

``1:2:1;:9;2::2;;``

*********************
Metadata del Contrato
*********************

El compilador Solidity genera automáticamente un archivo JSON, el
metadata del contrato, que contiene información sobre el contrato actual.
Se puede usar para consultar la versión del compilador, las fuentes usadas,
el ABI y documentación NatSpec a fin de interactuar con más seguridad con el
contrato y verificar su código fuente.

El compilador agrega un has Swarm del archivo metadata al final del bytecode
(para detalles, mirar abajo) de cada contrato, para que se pueda recuperar el
archivo en una manera autentificada sin tener que usar un proveedor de datos
centrales.

Sin embargo, se tiene que publicar el arhivo metadata a Swarm (o otro servicio)
para que otros puedan verlo. El archivo puede ser producido usando ``solc --metadata``
y el archivo será llamado ``NombreContrato_meta.json``.
Contendrá referencias Swarm al código fuente, así que tienes que upload
todos los archivos código fuente y el archivo metadata.

El archivo metadata tiene el formato siguiente. El ejemplo abajo es presentado de
manera legible por humanos. Metadata formateada correctamente debe usar comillas
correctamente, reducir espacio blanco a un mínimo y ordenarse diferentemente.
Los comantarios obviamente tampoco son permitidos y son usados aquí sólo por
razones explicativos.

.. code-block:: none

    {
      // Requerido: La versoin del formato de metadata
      version: "1",
      // Requerido: lenguaje de código fuente, settea una "sub-versión"
      // de la espcificación
      language: "Solidity",
      // Requerido: Detalles del compilador, los contenidos son específicos
      // al lenguaje
      compiler: {
        // Requerido para Solidity: Version del compilador
        version: "0.4.6+commit.2dabbdf0.Emscripten.clang",
        // Opcional: Hash del compilador binario que produjo este resultado
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
