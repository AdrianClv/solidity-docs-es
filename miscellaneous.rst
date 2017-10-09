#######
Diverso
#######

.. index:: storage, state variable, mapping

***************************************************
Layout de las variables de estado en almacenamiento
***************************************************

Variables de tamaño estáticas (todo menos mapping y tipos arrays de tamaño variable) se disponen contiguamente en almacenamiento empezando de la posición ``0``. Cuando múltiples elementos necesitan menos de 32 bytes, son empaquetados en un slot de almacenamiento cuando es posible de acuerdo a las reglas:

- El primer elemento en un slot de almacenamiento es almacenado alineado en lower-order.
- Tipos elementales sólo usan la cantidad de bytes que se necesita para almacenarlos.
- Si un tipo elemental no cabe en la parte restante de un slot de almacenamiento, es movido al próximo slot de almacenamiento.
- Structs y datos de array siempre comienzan en un nuevo slot y ocupan slots enteros (pero elementos dentro de un struct o array son empacados estrechamente de acuerdo a estas reglas).


.. warning::
    Cuando se usan elementos que son más pequeños que 32 bytes, el uso del gas del contrato puede
    ser más alto. Esto es porque la EVM opera en 32 bytes a la vez. Por lo tanto, si el elemento es
    más pequeño que eso, la EVM usa más operaciones para reducir el tamaño del elemento de 32 bytes
    al tamaño deseado.

    Sólo es beneficioso reducir el tamaño de los argumentos si estás tratando con valores de
    almacenamiento porque el compilador empacará múltiples elementos en un slot de almacenamiento,
    y entonces, combina múltiples lecturas y escrituras en una sóla operación. Cuando se trata con
    argumentos de función o valores de memoria, no hay beneficio inherente porque el compilador no
    empaca estos valores.

    Finalmente, para permitir a la EVM optimizar esto, asegúrate de ordenar las variables de
    almacenamiento y miembros ``struct`` para que puedan ser empacados estrechamente. Por ejemplo,
    declarando tus variables de almacenamiento en el orden de ``uint128, uint128, uint256`` en vez de
    ``uint128, uint256, uint128``, ya que el primero utilizará sólo dos slots de almacenamiento y éste
    último tres.

Los elementos de structs y arrays son almacenados después de ellos mismos, como si fueran dados explícitamente.

Dado a su tamaño impredecible, los tipos de array dinámicos y de mapping usan computación hash Keccak-256
para encontrar la posición de inicio del valor o del dato del array. Estas posiciones de inicio
son siempre slots completos.

El mapping o el array dinámico en sí ocupa un slot (sin llenar) en alguna posición ``p`` de acuerdo
a la regla de arriba (o por aplicar esta regla de mappings a mappings o arrays de arrays). Para un
array dinámico, este slot guarda el número de elementos en el array (los byte arrays y cadenas son
excepciones aquí, mirar abajo). Para un mapping, el slot no es utilizado (pero es necesario para que
dos mappings iguales seguidos usen diferentes distribuciones de hash).
Datos de array son ubicados en ``kecakk256(p)`` y el valor correspondiente a una clave de mapping
``k`` es ubicada en ``kecakk256(k . p)`` donde ``.`` es una concatenación. Si el valor de nuevo es
un tipo no elemental, las posiciones son encontradas agregando un offset de ``keccak256(k . p)``.

``bytes`` y ``string`` almacenan sus datos en el mismo slot junto con su longitud si es corta. En particular: si los datos son al menos ``31`` bytes de largo, es almacenado en bytes de orden mayor (alineados a la izquierda) y el byte de orden menor almacena ``length * 2``. Si es más largo, el slot principal almacena ``length * 2 + 1`` y los datos son almacenados como siempre en ``keccak256(slot)``.

Entonces para el siguiente sinppet de contrato::

    contract C {
      struct s { uint a; uint b; }
      uint x;
      mapping(uint => mapping(uint => s)) data;
    }

La posición de ``data[4][9].b`` está en ``keccak256(uint256(9) . keccak256(uint256(4) . uint256(1))) + 1``.

.. index: memory layout

*****************
Layout en memoria
*****************

Solidity reserva tres slots de 256-bits:

- 0 - 64: espacio de scratch para métodos de hash
- 64 - 96: tamaño de memoria actualmente asignada (también conocida como free memory pointer)

El espacio de scratch puede ser usado entre declaraciones (ej. dento de inline assembly).

Solidity siempre emplaza los nuevos objetos en el puntero de memoria libre y la memoria nunca es liberada (esto puede cambiar en el futuro).

.. warning::
  Hay algunas operaciones en Solidity que necesitan un área temporal de memoria mas grande que 64 bytes y por lo tanto no caben en el espacio scratch. Estas operaciones serán emplazadas donde apunta la memoria libre, pero dado su corta vida, el puntero no es actualizado. La memoria puede o no ser puesta a cero. Por esto, uno no debiera esperar que la memoria libre sea puesta a cero.

.. index: calldata layout

*******************
Layout de Call Data
*******************

Cuando un contrato de Solidity es desplegado y cuando es llamado desde una
cuenta, los datos de entrada se asume que están en el formato de la
`especificación ABI <https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI>`_.
La especificación ABI requiere que los argumentos sean ajustados a
múltiplos de 32 bytes. Las llamadas de función internas usan otra convención.

.. index: variable cleanup

******************************
Internas - Limpiando variables
******************************

Cuando un valor es más corto que 256-bit, en algunos casos los bits
restantes tienen que ser limpiados.
El compilador de Solidity está diseñado para limpiar estos bits restantes antes
de cualquier operación que pueda ser afectada adversamente por la potencial
basura en los bits restantes.
Por ejemplo, antes de escribir un valor en la memoria, los bits restantes tienen
que ser limpiados porque los contenidos de la memoria pueden ser usados para
computar hashes o ser enviados como datos en una llamada. De manera similar,
antes de almacenar un valor en el almacenamiento, los bits restantes tienen
que ser limpiados porque si no, el valor ilegible puede ser observado.

Por otro lado, no limpiamos los bits si la operación siguiente no es afectada.
Por ejemplo, ya que cualquier valor no-cero es considerado ``true`` por
una instrucción ``JUMPI``, no limpiamos los valores booleanos antes de que sean utilizados
como condición para ``JUMPI``.

Además de este principio de diseño, el compilador de Solidity
limpia los datos de entrada cuando está cargado en el stack.

Diferentes tipos tienen diferentes reglas para limpiar valores inválidos:

+---------------+---------------+--------------------+
|Tipo           |Valores válidos|Val. inv. significan|
+===============+===============+====================+
|enum de n      |0 hasta n - 1  |excepción           |
|miembros       |               |                    |
+---------------+---------------+--------------------+
|bool           |0 o  1         |1                   |
+---------------+---------------+--------------------+
|enteros con    |palabra        |por ahora envuelve  |
|signo          |extendida por  |silenciosamente;    |
|               |signo          |en el futuro esto   |
|               |               |arrojará            |
|               |               |excepciones         |
|               |               |                    |
+---------------+---------------+--------------------+
|enteros        |altos bits     |por ahora envuelve  |
|sin signos     |a cero         |silenciosamente;    |
|               |               |en el futuro esto   |
|               |               |arrojará            |
|               |               |excepciones         |
+---------------+---------------+--------------------+


.. index:: optimizer, common subexpression elimination, constant propagation

*************************
Internos - El optimizador
*************************

El optimizador de solidity funciona con ensamblador, así que puede y es usado con otros lenguajes. Divide la secuencia de las instrucciones en bloques básicos de JUMPs y JUMPDESTs. Dentro de estos bloques, las instrucciones son analizadas y cada modificación al stack, a la memoria o al almacenamiento son guardadas como una expresión que consiste en una instrucción y una lista de argumentos que son esencialmente punteros a otras expresiones. La idea principal es encontrar expresiones que sean siempre iguales (en cada entrada) y combinarlas a una clase de expresión. El optimizador primero intenta encontrar una nueva expresión en una lista de expresiones conocidas. Si esto no funciona, la expresión es simplificada de acuerdo a reglas como ``constante + constante = suma_de_constantes`` o ``X * 1 = X``. Ya que esto es hecho recursivamente, también podemos aplicar la última regla si el segundo factor es una expresión más compleja donde sabemos que siempre evaluará a uno. Modificaciones al almacenamiento y ubicaciones de memoria tienen que borrar el conocimiento de almacenamiento y ubicaciones de memoria que no son conocidas como diferentes: Si primero escribimos a ubicación `x` y luego a ubicación `y` y ambas son variables de entrada, la segunda puede sobrescribir la primera, entonces no sabemos realmente lo que es almacenado en `x` después de escribir a `y`. Por otro lado, si una simplificación de la expresión `x - y` evalúa a una constante distinta de cero, sabemos que podemos mantener nuestro conocimiento de lo que es almacenado en x.

Al final de este proceso, sabemos qué expresiones tienen que estar al final del stack y tienen una lista de modificaciones a la memoria y almacenamiento. Esta información es almacenada junto con los bloques básicos y es usada para unirlas. Además, información sobre el stack, almacenamiento y configuración de memoria es enviada al (los) próximo(s) bloque(s). Si sabemos los objetivos de cada una de las instrucciones JUMP y JUMPI, podemos construir un gráfico de flujo completo del programa. Si hay un sólo objetivo que no conocemos (esto puede pasar ya que en principio, los objetivos de jumps pueden ser computados de las entradas), tenemos que borrar toda información sobre los estados de entrada de un bloque ya que puede ser el objetivo del JUMP desconocido. Si se encuentra un JUMPI cuya condición evalúa a una constante, es transformado en un jump incondicional.

Como en el último paso, el código en cada bloque es completamente regenerado. Un gráfico de dependencias es creado de la expresión en el stack al final del bloque y cada operación que no es parte de este gráfico es esencialmente olvidada. Ahora se genera código que aplica las modificaciones a la memoria y al almacenamiento en el orden en que fueron hechas en el código original (olvidando modificaciones que fueron encontradas innecesarias) y finalmente, genera todos los valores que son requeridos para estar en el stack en el lugar correcto.

Estos pasos son aplicados a cada bloque básico y el nuevo código generado se usa como reemplazo si es más pequeño. Si un bloque básico es dividido en un JUMPI y durante el análisis la condición evalúa a una constante, el JUMPI es reemplazado dependiendo del valor de la constante, y por lo tanto código como

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
Mappings de fuente
******************

Como parte del AST (Abstract syntax tree) de salida, el compilador provee el rango del código fuente
que es representado por el nodo respecto al AST. Esto puede ser usado
para varios propósitos desde herramientas de análisis estático que reportan
errores basados en el AST y herramientas de debugging que demarcan variables
locales y sus usos.

Además, el compilador también puede generar un mapping del bytecode
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
``l`` es el largo del rango de la fuente en bytes y ``f`` es el índice de fuente
mencionado arriba.

La codificación en el mapping de fuente para el bytecode es más complicada:
Es una lista de ``s:l:f:j`` separada por ``;``. Cada una de estos elementos
corresponde a una instrucción, i.e. no puedes usar el byte-offset
si no que tienes que usar la instrucción offset (las instrucciones push son mas largas
que un sólo byte).
Los campos ``s``, ``l`` y ``f`` son como detallamos arriba y ``j`` puede ser ``i``,
``o`` o ``-`` y significa si una instrucción jump va en la función, devuelve desde
una función o si es un jump regular como parte de un (ej) bucle.

A fin de comprimir estos mappings de fuente especialmente para bytecode, las
siguientes reglas son usadas:

 - Si un campo está vacío, el valor del elemento precedente es usado.
 - Si falta un ``:``, todos los campos siguientes son considerados vacíos.

Esto significa que los siguientes mappings de fuente representan la misma información:

``1:2:1;1:9:1;2:1:2;2:1:2;2:1:2``

``1:2:1;:9;2::2;;``

*********************
Metadata del contrato
*********************

El compilador de Solidity genera automáticamente un archivo JSON, el
metadata del contrato, que contiene información sobre el contrato actual.
Se puede usar para consultar la versión del compilador, las fuentes usadas,
el ABI y documentación NatSpec a fin de interactuar con más seguridad con el
contrato y verificar su código fuente.

El compilador agrega un hash Swarm del archivo metadata al final del bytecode
(para detalles, mirar abajo) de cada contrato, para que se pueda recuperar el
archivo en una manera autentificada sin tener que usar un proveedor de datos
centrales.

Sin embargo, se tiene que publicar el archivo metadata a Swarm (o otro servicio)
para que otros puedan verlo. El archivo puede ser producido usando ``solc --metadata``
y el archivo será llamado ``NombreContrato_meta.json``.
Contendrá referencias Swarm al código fuente, así que tienes que subir
todos los archivos de código fuente y el archivo metadata.

El archivo metadata tiene el formato siguiente. El ejemplo de abajo es presentado de
manera legible por humanos. Los metadatos formateados correctamente deben usar comillas
correctamente, reducir el espacio en blanco a un mínimo y ordenarse de manera diferente.
Los comentarios obviamente tampoco son permitidos y son usados aquí sólo por
razones explicativos.

.. code-block:: none

    {
      // Requerido: La versión del formato de metadata
      version: "1",
      // Requerido: lenguaje de código fuente, settea una "sub-versión"
      // de la especificación
      language: "Solidity",
      // Requerido: Detalles del compilador, los contenidos son específicos
      // al lenguaje
      compiler: {
        // Requerido para Solidity: Version del compilador
        version: "0.4.6+commit.2dabbdf0.Emscripten.clang",
        // Opcional: Hash del compilador binario que produjo este resultado
        keccak256: "0x123..."
      },
      // Requerido: Compilación de archivos fuente/unidades fuente, las claves son
      // nombres de archivos
      sources:
      {
        "myFile.sol": {
          // Requerido: keccak256 hash del archivo fuente
          "keccak256": "0x123...",
          // Requerido (al menos que "content" sea usado, ver abajo): URL(s) ordenadas
          // al archivo fuente, el protocolo es menos arbitrario, pero se recomienda una
          // URL de Swarm
          "urls": [ "bzzr://56ab..." ]
        },
        "mortal": {
          // Requerido: hash keccak256 del archivo fuente
          "keccak256": "0x234...",
          // Requerido (al menos que "url" sea usado): contenidos literales del archivo fuente
          "content": "contract mortal is owned { function kill() { if (msg.sender == owner) selfdestruct(owner); } }"
        }
      },
      // Requerido: Configuración del compilador
      settings:
      {
        // Requerido para Solidity: Lista ordenada de remappeos
        remappings: [ ":g/dir" ],
        // Opcional: Configuración del optimizador (por defecto falso)
        optimizer: {
          enabled: true,
          runs: 500
        },
        // Requerido para Solidity: Archivo y nombre del contrato o librería para
        // la librería para el cual es creado este archivo metadata.
        compilationTarget: {
          "myFile.sol": "MyContract"
        },
        // Requerido para Solidity: Direcciones de las librerías usadas
        libraries: {
          "MyLib": "0x123123..."
        }
      },
      // Requerido: Información generada sobre el contrato.
      output:
      {
        // Requerido: definición ABI del contrato
        abi: [ ... ],
        // Requerido: documentación de usuario del contrato de NatSpec
        userdoc: [ ... ],
        // Requerido: documentación de desarrollador del contrato de NatSpec
        devdoc: [ ... ],
      }
    }

.. note::
    Nótese que la definición ABI de arriba no tiene orden fijo. Puede cambiar en distintas versiones del compilador.

.. note::
    Ya que el bytecode del contrato resultante contiene el hash del metadata, cualquier cambio
    a la metadata resultará en un cambio en el bytecode. Además, ya que la metadata incluye
    un hash de todos los fuentes usados, un simple espacio en blanco en cualquiera de los
    archivos de fuente resultará en un metadata diferente, y posteriormente en bytecode diferente.


Codificación del hash de metadata en el bytecode
================================================

Ya que en el futuro puede que soportemos otras maneras de consultar el archivo metadata,
el mapping ``{"bzzr0": <Swarm hash>}`` es guardado
codificado en [CBOR](https://tools.ietf.org/html/rfc7049). Ya que el principio de esa
codificación no es fácil de encontrar, la longitud se añade en una codificación two-byte big-endian.
La versión actual del compilador de Solidity entonces agrega lo siguiente al final
del bytecode desplegado::

    0xa1 0x65 'b' 'z' 'z' 'r' '0' 0x58 0x20 <32 bytes swarm hash> 0x00 0x29

Para recuperar estos datos, el final del bytecode desplegado puede ser
revisado para ver si coincide con ese patrón y usar el hash de Swarm para recuperar el archivo.


Uso para generar automáticamente la interfaz y NatSpec
======================================================

El metadata es usado de la siguiente forma: un componente que quiere interactuar
con un contrato (ej. Mist) obtiene el código del contrato, a partir de eso el
hash de Swarm de un archivo que luego es recuperado.
Ese archivo es un JSON con la estructura como la de arriba.

El componente puede luego usar el ABI para generar automáticamente una rudimentaria
interfaz de usuario para el contrato.

Además, Mist puede usar el userdoc para mostrar un mensaje de confirmación al usuario
cuando interactúe con el contrato.


Uso para la verificación de código fuente
=========================================

A fin de verificar la compilación, el código fuente pueden ser recuperado de Swarm
desde el enlace en el archivo metadata.
La versión correcta del compilador (que se comprueba que sea parte de los compiladores
"oficiales") es invocada en esa entrada con la configuración específicada. El resultado
bytecode es comparado a los datos de la transacción de creación o a datos del opcode CREATE.
Esto verifica automáticamente la metadata ya que su hash es parte del bytecode.
Datos en exceso corresponden a los datos de entrada del constructor, que deben ser decodificados
de acuerdo a la interfaz y presentados al usuario.


*****************
Trucos y consejos
*****************

* Usar ``delete`` en arrays para borrar sus elementos.
* Usar tipos mas cortos para elementos struct y ordenarlos para que los elementos mas cortos estén agrupados. Esto puede disminuir los costes de gas ya que múltiples operaciones SSTORE puden ser combinadas en una sóla (SSTORE cuesta 5000 o 20000 gas, así que esto es lo que se optimiza). ¡Usa el estimador de precio de gas (con optimizador activado) para probar!
* Hacer las variables de estado públicas - el compilador creará :ref:`getters <visibility-and-getters>` automáticamente.
* Si revisa las condiciones de entrada o de estado muchas veces en el inicio de las funciones, intenta usar :ref:`modifiers`.
* Si tu contrato tiene una función llamada ``send`` pero quieres usar la función interna de envío, usa ``address(contractVariable).send(amount)``.
* Inicia structs de almacenamiento con una sola asignación: ``x = MyStruct({a: 1, b: 2});``

**********
Cheatsheet
**********

.. index:: precedence

.. _order:

Orden de preferencia de operadores
==================================

El siguiente es el orden de precedencia para operadores, listado en orden de evaluación.

+------------+-------------------------------------+--------------------------------------------+
| Precedencia| Descripción                         | Operador                                   |
+============+=====================================+============================================+
| *1*        | Postfix incremento y decremento     | ``++``, ``--``                             |
+            +-------------------------------------+--------------------------------------------+
|            | llamada de tipo función             | ``<func>(<args...>)``                      |
+            +-------------------------------------+--------------------------------------------+
|            | subíndice de array                  | ``<array>[<index>]``                       |
+            +-------------------------------------+--------------------------------------------+
|            | Acceso de miembro                   | ``<object>.<member>``                      |
+            +-------------------------------------+--------------------------------------------+
|            | Paréntesis                          | ``(<statement>)``                          |
+------------+-------------------------------------+--------------------------------------------+
| *2*        | Prefijo incremento y decremento     | ``++``, ``--``                             |
+            +-------------------------------------+--------------------------------------------+
|            | Más y menos unarios                 | ``+``, ``-``                               |
+            +-------------------------------------+--------------------------------------------+
|            | Operaciones unarias                 | ``delete``                                 |
+            +-------------------------------------+--------------------------------------------+
|            | NOT lógico                          | ``!``                                      |
+            +-------------------------------------+--------------------------------------------+
|            | NOT a nivel de bits                 | ``~``                                      |
+------------+-------------------------------------+--------------------------------------------+
| *3*        | Exponenciación                      | ``**``                                     |
+------------+-------------------------------------+--------------------------------------------+
| *4*        | Multiplicación, división and módulo | ``*``, ``/``, ``%``                        |
+------------+-------------------------------------+--------------------------------------------+
| *5*        | Adición and sustracción             | ``+``, ``-``                               |
+------------+-------------------------------------+--------------------------------------------+
| *6*        | Desplazamiento de bits              | ``<<``, ``>>``                             |
+------------+-------------------------------------+--------------------------------------------+
| *7*        | AND a nivel de bits                 | ``&``                                      |
+------------+-------------------------------------+--------------------------------------------+
| *8*        | XOR a nivel de bits                 | ``^``                                      |
+------------+-------------------------------------+--------------------------------------------+
| *9*        | OR a nivel de bits                  | ``|``                                      |
+------------+-------------------------------------+--------------------------------------------+
| *10*       | Operadores de desigualdad           | ``<``, ``>``, ``<=``, ``>=``               |
+------------+-------------------------------------+--------------------------------------------+
| *11*       | Operadores de igualdad              | ``==``, ``!=``                             |
+------------+-------------------------------------+--------------------------------------------+
| *12*       | AND lógico                          | ``&&``                                     |
+------------+-------------------------------------+--------------------------------------------+
| *13*       | OR lógico                           | ``||``                                     |
+------------+-------------------------------------+--------------------------------------------+
| *14*       | Operador ternario                   | ``<conditional> ? <if-true> : <if-false>`` |
+------------+-------------------------------------+--------------------------------------------+
| *15*       | Operador de asignación              | ``=``, ``|=``, ``^=``, ``&=``, ``<<=``,    |
|            |                                     | ``>>=``, ``+=``, ``-=``, ``*=``, ``/=``,   |
|            |                                     | ``%=``                                     |
+------------+-------------------------------------+--------------------------------------------+
| *16*       | Operador de coma                    | ``,``                                      |
+------------+-------------------------------------+--------------------------------------------+

.. index:: assert, block, coinbase, difficulty, number, block;number, timestamp, block;timestamp, msg, data, gas, sender, value, now, gas price, origin, revert, require, keccak256, ripemd160, sha256, ecrecover, addmod, mulmod, cryptography, this, super, selfdestruct, balance, send

Variables globales
==================

- ``block.blockhash(uint blockNumber) returns (bytes32)``: hash del bloque dado - sólo funciona para los últimos 256 bloques
- ``block.coinbase`` (``address``): address del minero del bloque actual
- ``block.difficulty`` (``uint``): dificultad del bloque actual
- ``block.gaslimit`` (``uint``): gaslimit del bloque actual
- ``block.number`` (``uint``): número del bloque actual
- ``block.timestamp`` (``uint``): timestamp del bloque actual
- ``msg.data`` (``bytes``): calldata completa
- ``msg.gas`` (``uint``): gas restante
- ``msg.sender`` (``address``): sender del mensaje (llamada actual)
- ``msg.value`` (``uint``): número de wei enviados con el mensaje
- ``now`` (``uint``): timestamp del bloque actual (alias para ``block.timestamp``)
- ``tx.gasprice`` (``uint``): precio de gas de la transacción
- ``tx.origin`` (``address``): sender de la transacción (cadena de llamada completa)
- ``assert(bool condition)``: abortar ejecución y deshacer cambios de estado si la condición es ``false`` (uso para error interno)
- ``require(bool condition)``: abortar ejecución y deshacer cambios de estado si la condición es ``false`` (uso para entradas erróneas)
- ``revert()``: abortar ejecución y deshacer cambios de estado
- ``keccak256(...) returns (bytes32)``: computar el hash Ethereum-SHA-3 (Keccak-256) de los argumentos (empacados)
- ``sha3(...) returns (bytes32)``: un alias a `keccak256()`
- ``sha256(...) returns (bytes32)``: computar el hash SHA-256 de los argumentos (empacados)
- ``ripemd160(...) returns (bytes20)``: computar el hash RIPEMD-160 de los argumentos (empacados)
- ``ecrecover(bytes32 hash, uint8 v, bytes32 r, bytes32 s) returns (address)``: recuperar address asociada con la llave pública desde la firma de la curva elíptica, devuelve cero en caso de error
- ``addmod(uint x, uint y, uint k) returns (uint)``: computa ``(x + y) % k`` donde la suma es hecha con precisión arbitraria y no envuelve ``2**256``
- ``mulmod(uint x, uint y, uint k) returns (uint)``: computa ``(x * y) % k`` donde la multiplicación es hecha con precisión arbitraria y no envuelve ``2**256``
- ``this`` (tipo del contrato actual): el contrato actual, explícitamente convertible a ``address``
- ``super``: el contrato un nivel más alto en la jerarquía de herencia
- ``selfdestruct(address recipient)``: destruir el contrato actual, enviando sus fondos al address dada
- ``<address>.balance`` (``uint256``): saldo de :ref:`address` en Wei
- ``<address>.send(uint256 amount) returns (bool)``: enviar monto dado de Wei a :ref:`address`, devuelve ``false`` en caso de error
- ``<address>.transfer(uint256 amount)``: enviar monto dado de Wei a :ref:`address`, arroja excepción si falla


.. index:: visibility, public, private, external, internal

Especificadores de visibilidad de función
=========================================

::

    function myFunction() <visibility specifier> returns (bool) {
        return true;
    }

- ``public``: visible externa e internamente (crea función getter para almacenamiento/variables de estado)
- ``private``: sólo visible en el contrato actual
- ``external``: sólo visible desde el exterior (sólo para funciones) - e.j. sólo puede ser llamado mediante mensajes (via ``this.func``)
- ``internal``: sólo visible internamente


.. index:: modifiers, constant, anonymous, indexed

Modificadores
=============

- ``constant`` para variables de estado: no permite asignaciones (excepto inicialización), no ocupa un slot de almacenamiento.
- ``constant`` para funciones: no permite modificación de estado - no está forzado aún.
- ``anonymous`` para eventos: no guarda la firma del evento como topic.
- ``indexed`` para parámetros de eventos: guarda el parámetro como topic.
- ``payable`` para funciones: les permite recibir ether junto a una llamada.


Keywords reservadas
===================


Estas palabras están reservadas en Solidity. Pueden incorporarse a la sintaxis en el futuro:

``abstract``, ``after``, ``case``, ``catch``, ``default``, ``final``, ``in``, ``inline``, ``interface``, ``let``, ``match``, ``null``,
``of``, ``pure``, ``relocatable``, ``static``, ``switch``, ``try``, ``type``, ``typeof``, ``view``.

Gramática del lenguaje
======================

.. literalinclude:: grammar.txt
   :language: none
