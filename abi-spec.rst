.. index:: abi, application binary interface

.. _ABI:

**********************************************
Especificación de Application Binary Interface
**********************************************

Diseño básico
=============

La Application Binary Interface (Interfaz Binaria de Aplicación) o ABI es el modo estándar de interactuar con contratos en el ecosistema Ethereum, tanto desde fuera de la blockchain como en interacciones contrato-contrato. Los datos se codifican siguiendo su tipo acorde a esta especificación. La codificación no es autodescriptiva, por lo que requiere de un esquema para ser decodificado.

Asumimos que la Application Binary Interface (ABI) está fuertemente tipada, es conocida en tiempo de compilación y es estática. No se van a proveer mecanismos de introspección. Además, afirmamos que los contratos tendrán las definiciones de la interfaz de cada contrato que vayan a llamar en tiempo de compilación.

Esta especificación no abarca los contratos cuya interfaz sea dinámica o conocida exclusivamente en tiempo de ejecución. Estos casos, de volverse importantes, podrían manejarse adecuadamente como servicios construidos dentro del ecosistema Ethereum.

.. _abi_function_selector:

Función Selector
================

Los primeros cuatro bytes de los datos de una llamada a una función especifican la función a llamar. Se trata de los primeros (los más a la izquierda, los más extremos por orden) cuatro bytes del hash Keccak (SHA-3) de la firma de la función. La firma se define como la expresión canónica del prototipo básico como puede ser el nombre de la función con la lista de parámetros entre paréntesis. Los tipos de parámetros se separan por comas, no por espacios.

Codificación de argumentos
==========================

A partir del quinto byte, prosiguen los argumentos codificados. Esta codificación es también usada en otros sitios, por ejemplo, en los valores de retorno y también en los argumentos de eventos, sin los cuatro bytes especificando la función.

Tipos
=====

Los tipos elementales existentes son:

- ``uint<M>``: enteros sin signo de ``M`` bits, ``0 < M <= 256``, ``M % 8 == 0``. Ejemplos: ``uint32``, ``uint8``, ``uint256``.

- ``int<M>``: enteros con signo complemento a dos de ``M`` bits, ``0 < M <= 256``, ``M % 8 == 0``.

- ``address``: equivalente a ``uint160``, exceptuando la interpretación asumida y la tipología del lenguaje. Para calcular el selector de la funcin, se usa ``address``.

- ``uint``, ``int``: sinónimos de ``uint256``, ``int256`` respectivamente. Para calcular el selector de la función, se debe usar ``uint256`` y ``int256``.

- ``bool``: equivalente a ``uint8`` restringido a los valores 0 y 1. Para calcular el selector de la función, se usa ``bool``

- ``fixed<M>x<N>``: número decimal con signo y formato decimal fijo de ``M`` bits, ``0 < M <= 256``, ``M % 8 ==0``, y ``0 < N <= 80``, que denota el valor ``v`` como ``v / (10 ** N)``.

- ``ufixed<M>x<N>``: variante sin signo de ``fixed<M>x<N>``. Para calcular el selector de la función, hay que usar ``fixed128x19`` y ``ufixed128x19``.

- ``fixed``, ``ufixed``: sinónimos de ``fixed128x19``, ``ufixed128x19`` respectivamente (no para ser usados para computar la función selector).

- ``bytes<M>``: tipo binario de ``M`` bytes, ``0 < M <= 32``.

- ``function``: una dirección (20 bytes) seguida del selector de una función (4 bytes). Codificado identico a ``bytes24``.

El siguiente array, de tipo fijo, existente es:

- ``<type>[M]``: un array de longitud fija de ``M`` elementos, ``M`` > 0, del tipo dado.

Los siguientes tipos de tamaño no fijo existentes son: 

- ``bytes``: secuencia de bytes de tamaño dinámico.

- ``string``: string unicode de tamaño dinámico codificado como UTF-8.

- ``<type>[]``: array de longitud variable del tipo de longitud fija dada.

Los distintos tipos se pueden combinar en una tupla cerrando un número finito no negativo de ellos entre paréntesis, separados por comas:

- ``(T1,T2,...,Tn)``: tupla consistente de los tipos ``T1``, ..., ``Tn``, ``n >= 0``

Es posible formar tuplas de tuplas, arrays de tuplas, etc.

.. note::
    Solidity soporta todos los tipos presentados arriba con los mismos nombres salvo las tuplas. El ABI tipo tupla se usa para codificar ``structs`` de Solidity.

Especificación formal de la codificación
========================================

Vamos a especificar formalmente la codificación, de tal forma que tendrá las siguientes propiedades, que son especialmente útiles si los argumentos son arrays anidados:

Propiedades:

  1. El número de lecturas necesarias para acceder a un valor es, como mucho, equivalente a la máxima profundidad del array. Por ejemplo, cuatro lecturas se requieren para obtener ``a_i[k][l][r]``. En una versión previa de la ABI, el número de lecturas escalaba linealmente con el número total de parámetros dinámicos en el peor caso.

  2. Los datos de una variable o elemento de un array no se intercalan con otros datos y son recolocables. Por ejemplo, sólo usan "addresses" relativos.

Distinguimos entre tipos estáticos y dinámicos. Los estáticos se codifican insitu y los dinámicos se codifican en una posición asignada separadamente después del bloque actual.

**Definición:** Los siguientes tipos se llaman "dinámicos":
* ``bytes``
* ``string``
* ``T[]`` para cada ``T``
* ``T[k]`` para cualquier dinámico ``T`` y todo ``k > 0``
* ``(T1,...,Tk)`` si cualquier ``Ti`` es dinámico para ``1 <= i <= k``

Todo el resto de tipos son "estáticos".

**Definición:** ``len(a)`` es el número de bytes en un string binario ``a``.
El tipo de ``len(a)`` se presume como ``uint256``.

Definimos ``enc``, la codificación actual, como un mapping de valores de tipos de la ABI a string binarios como ``len(enc(X))`` depende del valor de ``X`` si y solo si el tipo de ``X`` es dinámico.

**Definición:** Para cada valor de ABI ``X``, definimos recursivamente ``enc(X)``, dependiendo del tipo de ``X`` siendo

- ``(T1,...,Tk)`` para ``k >= 0`` y cualquier tipo ``T1``, ..., ``Tk``

  ``enc(X) = head(X(1)) ... head(X(k-1)) tail(X(0)) ... tail(X(k-1))``

  donde ``X(i)`` es el ``ith`` componente del valor, y
  ``head`` y ``tail`` son definidos por ``Ti`` siendo un tipo estático como

    ``head(X(i)) = enc(X(i))`` y ``tail(X(i)) = ""`` (el string vacío)

  y como

    ``head(X(i)) = enc(len(head(X(0)) ... head(X(k-1)) tail(X(0)) ... tail(X(i-1))))``
    ``tail(X(i)) = enc(X(i))``

  en otros casos como si, por ejemplo, ``Ti`` es un tipo dinámico.

  Hay que tener en cuenta que en el caso dinámico, ``head(X(i))`` está bien definido ya que las longitudes de las partes de head sólo dependen de los tipos y no de los valores. Su valor es el offset del principio de ``tail(X(i))`` relativo al comienzo de ``enc(X)``.
  
- ``T[k]`` para cada ``T`` y ``k``:

  ``enc(X) = enc((X[0], ..., X[k-1]))``
  
  como ejemplo, es codificado como si fuera un struct anónimo de ``k`` elementos del mismo tipo.
  
- ``T[]`` donde ``X`` tiene ``k`` elementos (``k`` se presume que es del tipo ``uint256``):

  ``enc(X) = enc(k) enc([X[1], ..., X[k]])``

  Otro ejemplo codificado como si fuera un array estático de tamaño ``k``, prefijado con el número de elementos.

- ``bytes``, de longitud ``k`` (que se presume que es del tipo ``uint256``):

  ``enc(X) = enc(k) pad_right(X)``. Por ejemplo, el número de bytes es codificado como un ``uint256`` seguido del valor actual de ``X`` como una secuencia de bytes, seguido por el número mínimo de bytes-cero como que ``len(enc(X))`` es un múltiplo de 32.

- ``string``:

  ``enc(X) = enc(enc_utf8(X))``, en este caso ``X`` se codifica como utf-8 y su valor se interpreta como de tipo ``bytes`` y codificado posteriormente. Hay que tener en cuenta que la longitud usada en la subsecuente codificación es el número de bytes del string codificado como utf-8, no su número de caracteres.

- ``uint<M>``: ``enc(X)`` es el mayor extremo de la codificación de ``X``, rellenado en el lado de orden mayor (izquierda) con bytes cero de tal forma que la longitud acabe siendo de 32 bytes.
- ``address``: como en el caso de ``uint160``
- ``int<M>``: ``enc(X)`` es el complemento a dos de mayor extremo en la codificación de ``X``, rellenado en el lado de mayor orden (izquierda) con ``0xff`` para ``X`` negativo y bytes cero para ``X`` positivo de tal forma que la longitud final sea un múltiplo de 32 bytes.
- ``bool``: como en el caso de ``uint8``, donde ``1`` se usa para ``true`` y ``0`` para ``false``
- ``fixed<M>x<N>``: ``enc(X)`` es ``enc(X * 10**N)`` donde ``X * 10**N`` se interpreta como un ``int256``.
- ``fixed``: como en el caso de ``fixed128x19``
- ``ufixed<M>x<N>``: ``enc(X)`` es ``enc(X * 10**N)`` donde ``X * 10**N`` se interpreta como un ``uint256``.
- ``ufixed``: como en el caso de ``ufixed128x19``
- ``bytes<M>``: ``enc(X)`` es la secuencia de bytes en ``X`` rellenado con bytes cero hasta una longitud de 32.

Resaltar que para cada ``X``, ``len(enc(X))`` es un múltiplo de 32.

Función Selector y codificación de argumentos
=============================================

Siempre, una llamada a la función ``f`` con parámetros ``a_1, ..., a_n`` se codifican como 

  ``function_selector(f) enc((a_1, ..., a_n))``

y los valores de retorno ``v_1, ..., v_k`` de ``f`` son codificados como

  ``enc((v_1, ..., v_k))``

p.ej.: los valores se combinan en struct anónimos y codificados.

Ejemplos
========

Para el siguiente contrato:

::

    pragma solidity ^0.4.16;

    contract Foo {
      function bar(bytes3[2] xy) public {}
      function baz(uint32 x, bool y) public returns (bool r) { r = x > 32 || y; }
      function sam(bytes name, bool z, uint[] data) public {}
    }


Para nuestro ejemplo ``Foo``, si queremos llamar a ``baz`` pasando como parámetros ``69`` y ``true``, emplearíamos 68 bytes en total, que se podrían dividir en las siguientes partes:

- ``0xcdcd77c0``: el ID del método. Se deriva como los 4 primeros bytes del hash Keccak en ASCII de la firma ``baz(uint32,bool)``.
- ``0x0000000000000000000000000000000000000000000000000000000000000045``: el primer parámetro, un uint32 de valor ``69`` rellenado hasta 32 bytes
- ``0x0000000000000000000000000000000000000000000000000000000000000001``: el segundo parámetro - boolean ``true``, rellenado hasta 32 bytes

En total::

    0xcdcd77c000000000000000000000000000000000000000000000000000000000000000450000000000000000000000000000000000000000000000000000000000000001

Devuelve un ``bool`` simple. Si, por ejemplo, devolviese ``false``, su salida sería un array de byte sencillo ``0x0000000000000000000000000000000000000000000000000000000000000000``, un único bool.

Si quisiéramos llamar a ``bar`` con el argumento ``["abc", "def"]``, pasaríamos 68 bytes en total, divido en:

- ``0xfce353f6``: el ID del método. Este se deriva de la firma ``bar(bytes3[2])``.
- ``0x6162630000000000000000000000000000000000000000000000000000000000``: La primera parte del primer parámetro, un valor ``bytes3`` ``"abc"`` (alineado a la izquierda).
- ``0x6465660000000000000000000000000000000000000000000000000000000000``: La segunda parte del primer parámetro, un valor ``bytes3`` ``"def"`` (alineado a la izquierda).

En total::

    0xfce353f661626300000000000000000000000000000000000000000000000000000000006465660000000000000000000000000000000000000000000000000000000000

Si quisiéramos llamar a ``sam`` con los argumentos ``"dave"``, ``true`` y ``[1,2,3]``, pasaríamos 292 bytes en total, dividido en:
- ``0xa5643bf2``: el ID del método. Este se deriva de la firma ``sam(bytes,bool,uint256[])``. Aquí ``uint`` se reemplaza por su representación canónica ``uint256``.
- ``0x0000000000000000000000000000000000000000000000000000000000000060``: La localización de la parte de datos del primer parámetro (tipo dinámico), medido en bytes desde el principio del bloque de argumentos. En este caso, ``0x60``.
- ``0x0000000000000000000000000000000000000000000000000000000000000001``: el segundo parámetro: boolean verdadero.
- ``0x00000000000000000000000000000000000000000000000000000000000000a0``: La localización de la parte de datos del tercer parámetro (tipo dinámico), medido en bytes. En este caso, ``0xa0``.
- ``0x0000000000000000000000000000000000000000000000000000000000000004``: La parte de datos del primer argumento, comienza con la longitud del array de bytes en elementos, en este caso, 4.
- ``0x6461766500000000000000000000000000000000000000000000000000000000``: Los contenidos del primer argumento: el UTF-8 (equivalente a ASCII en este caso) codificación de ``"dave"``, rellenado hasta 32 bytes por la derecha.
- ``0x0000000000000000000000000000000000000000000000000000000000000003``: La parte de datos del tercer argumento, comenzando con la longitud del array en elementos, en este caso, 3.
- ``0x0000000000000000000000000000000000000000000000000000000000000001``: la primera entrada del tercer parámetro.
- ``0x0000000000000000000000000000000000000000000000000000000000000002``: la segunda entrada del tercer parámetro.
- ``0x0000000000000000000000000000000000000000000000000000000000000003``: la tercera entrada del tercer parámetro.

En total::

    0xa5643bf20000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000000000000000000000000000464617665000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000003

Uso de tipos dinámicos
======================

Una llamada a una función con la firma ``f(uint,uint32[],bytes10,bytes)`` con valores ``(0x123, [0x456, 0x789], "1234567890", "Hello, world!")`` se codifica de la siguiente manera:

Obtenemos los primeros cuatro bytes de ``sha3("f(uint256,uint32[],bytes10,bytes)")``, p.ej.: ``0x8be65246``.
Entonces codificamos las cabeceras de los cuatro argumentos. Para los tipos estáticos ``uint256`` y ``bytes10``, estos son los valores que queremos pasar directamente, mientras que para los tipos dinámicos ``uint32[]`` y ``bytes``, usamos el offset en bytes hasta el inicio de su área de datos, contando desde el comienzo de la codificación del valor (p.ej.: sin contar los primeros cuatro bytes que contienen el hash de la firma de la función). Estos son:

 - ``0x0000000000000000000000000000000000000000000000000000000000000123`` (``0x123`` rellenado hasta 32 bytes)
 - ``0x0000000000000000000000000000000000000000000000000000000000000080`` (offset del inicio de la parte de datos del segundo parámetro, 4*32 bytes, exactamente el tamaño de la parte de la cabecera)
 - ``0x3132333435363738393000000000000000000000000000000000000000000000`` (``"1234567890"`` rellenado hasta 32 bytes por la derecha)
 - ``0x00000000000000000000000000000000000000000000000000000000000000e0`` (offset del comienzo de la parte de datos del cuarto parámetro = offset del inicio de la parte de datos del primer parámetro dinámico + tamaño de la parte de datos del primer parámetro dinámico = 4\*32 + 3\*32 (ver abajo))

Después de esto, la parte de datos del primer argumento dinámico, ``[0x456, 0x789]`` sigue así:

 - ``0x0000000000000000000000000000000000000000000000000000000000000002`` (número de elementos del array, 2)
 - ``0x0000000000000000000000000000000000000000000000000000000000000456`` (primer elemento)
 - ``0x0000000000000000000000000000000000000000000000000000000000000789`` (segundo elemento)

Finalmente, codificamos la parte de datos del segundo argumento dinámico, ``"Hello, world!"``:

 - ``0x000000000000000000000000000000000000000000000000000000000000000d`` (número de elementos (bytes en este caso): 13)
 - ``0x48656c6c6f2c20776f726c642100000000000000000000000000000000000000`` (``"Hello, world!"`` rellenado hasta 32 bytes por la derecha)

Todo junto, la codificación es (nueva línea después de la función selector y cada 32-bytes por claridad):

::

    0x8be65246
      0000000000000000000000000000000000000000000000000000000000000123
      0000000000000000000000000000000000000000000000000000000000000080
      3132333435363738393000000000000000000000000000000000000000000000
      00000000000000000000000000000000000000000000000000000000000000e0
      0000000000000000000000000000000000000000000000000000000000000002
      0000000000000000000000000000000000000000000000000000000000000456
      0000000000000000000000000000000000000000000000000000000000000789
      000000000000000000000000000000000000000000000000000000000000000d
      48656c6c6f2c20776f726c642100000000000000000000000000000000000000

Eventos
=======

Los eventos son una abstracción del protocolo de monitorización de eventos de Ethereum. Las entradas de log proveen la dirección del contrato, una cadena de máximo cuatro tópicos y algún dato binario de longitud arbitraria. Los eventos apalancan la función ABI existente para poder interpretarla (junto con una especificación de interfaz) como una estructura apropiada.

Dado un nombre de evento y una serie de parámetros de evento, los separamos en dos sub-series: los que están indexados y los que no. Los indexados, cuyo número podría llegar hasta tres, se usan junto al hash Keccack de la firma del evento para formar los tópicos de la entrada de log. Los no indexados forman el array de bytes del evento.

En efecto, una entrada de log que usa esta ABI se define como:

- ``address``: la dirección del contrato (intrínsecamente provista por Ethereum);
- ``topics[0]``: ``keccak(EVENT_NAME+"("+EVENT_ARGS.map(canonical_type_of).join(",")+")")`` (``canonical_type_of`` es una función que simplemente devuelve el tipo canónico del argumento dado, p.ej.: para ``uint indexed foo``, devolvería ``uint256``). Si el evento se declara como ``anonymous`` no se genera ``topics[0]``;
- ``topics[n]``: ``EVENT_INDEXED_ARGS[n - 1]`` (``EVENT_INDEXED_ARGS`` es la serie de ``EVENT_ARGS`` que están indexados);
- ``data``: ``abi_serialise(EVENT_NON_INDEXED_ARGS)`` (``EVENT_NON_INDEXED_ARGS`` es la serie de ``EVENT_ARGS`` que no están indexados, ``abi_serialise`` es la función de serialización ABI usada para devolver una serie de valores tipificados desde una función, como se detalla abajo).

Para todos los tipos de longitud fija de Solidity, el array ``EVENT_INDEXED_ARGS`` contiene directamente el valor codificado a 32 bytes. Sin embargo, para *tipos de longitud dinámica*, incluyendo ``string``, ``bytes``, y arrays, ``EVENT_INDEXED_ARGS`` contendrá *el hash Keccak* del valor codificado, en lugar del valor codificado directamente. Esto permite a las aplicaciones hacer consultas de forma eficiente sobre tipos de longitud dinámica (poniendo el hash del valor codificado como topic), pero hace que las aplicaciones no puedan decodificar los valores indexados que no han consultado. Para tipos de longitud dinámica, los desarrolladores de aplicaciones se enfrentan a un intercambio entre una búsqueda rápida para valores predeterminados (si el argumento es indexado) y legibilidad de valores arbitrarios (que requiere que los argumentos no sean indexados). Los desarrolladores pueden evitar esto y conseguir tanto eficiencia en la búsqueda como legibilidad definiendo eventos con dos argumentos — uno indexado y otro no —  con el mismo valor.

JSON
====

El formato JSON para la interfaz de un contrato viene dada por un array de descripciones de función y/o evento.
Una descripción de función es un objeto JSON con los siguientes campos:

- ``type``: ``"function"``, ``"constructor"``, o ``"fallback"`` (el :ref:``función sin nombre "default" <fallback-function>``);
- ``name``: nombre de la función;
- ``inputs``: array de objetos, cada uno contiene:
  * ``name``: nombre del parámetro;
  * ``type``: tipo canónico del parámetro.
  * ``components``: usado para tipos tupla (ver abajo).
- ``outputs``: un array de objetos similar a ``inputs``, puede omitirse si la función no devuelve nada;
- ``payable``: ``true`` si la función acepta ether, por defecto a ``false``.
- ``stateMutability``: un string con uno de los siguientes valores: ``pure`` (:ref:`especificada para no leer el estado de la blockchain <pure-functions>`), ``view`` (:ref:`especificada para no modificar el estado de la blockchain <view-functions>`), ``nonpayable`` y ``payable`` (igual ``payable`` arriba).
- ``constant``: ``true`` si la función es ``pure`` o ``view``.


``type`` se puede omitir, dejándolo por defecto a ``"function"``.

La función Constructor y fallback nunca tienen ``name`` o ``outputs``. Fallback tampoco tiene ``inputs``.

Enviar una cantidad de ether no-nula a una función no payable lanzará excepción. No lo hagas.

Una descripción de evento es un objeto JSON con prácticamente los mismos campos:

- ``type``: siempre ``"event"``
- ``name``: nombre del evento;
- ``inputs``: array de objetos, cada uno contiene:
  * ``name``: nombre del parámetro;
  * ``type``: tipo canónico del parámetro (ver abajo).
  * ``components``: usado para tipos tupla (ver abajo).
  * ``indexed``: ``true`` si el campo es parte de los tópicos del log, ``false`` si es parte del segmento de datos del log.
- ``anonymous``: ``true`` si el evento se declaró ``anonymous``.

Por ejemplo, 

::

  pragma solidity ^0.4.0;

  contract Test {
    function Test() public { b = 0x12345678901234567890123456789012; }
    event Event(uint indexed a, bytes32 b)
    event Event2(uint indexed a, bytes32 b)
    function foo(uint a) public { Event(a, b); }
    bytes32 b;
  }

resultaría en el JSON:

.. code:: json

  [{
  "type":"event",
  "inputs": [{"name":"a","type":"uint256","indexed":true},{"name":"b","type":"bytes32","indexed":false}],
  "name":"Event"
  }, {
  "type":"event",
  "inputs": [{"name":"a","type":"uint256","indexed":true},{"name":"b","type":"bytes32","indexed":false}],
  "name":"Event2"
  }, {
  "type":"function",
  "inputs": [{"name":"a","type":"uint256"}],
  "name":"foo",
  "outputs": []
  }]


Manejo de tipos tupla
---------------------

A pesar de que los nombres no son intencionalmente parte de la codificación ABI, tienen mucho sentido para ser incluidos en el JSON y permitir mostrarlo al usuario final. La estructura se anida de la siguiente manera:

Un objeto con miembros ``name``, ``type`` y potencialmente ``components`` describe una variable mecanografiada.
El tipo canónico se determina hasta que se alcanza un tipo de tuple y la descripción de la cadena es ascendente.
a ese punto se almacena en ``type`` prefijo con la palabra ``tuple``, es decir, será ``tuple`` seguido de
una secuencia de ``[]`` y ``[k]`` con enteros "k". Los componentes de la tuple se almacenan en el miembro "components",
que es de tipo array y tiene la misma estructura que el objeto de nivel superior excepto que ``indexed`` no está permitido ahí.


Por ejemplo, el código
::

    pragma solidity ^0.4.19;
    pragma experimental ABIEncoderV2;

    contract Test {
      struct S { uint a; uint[] b; T[] c; }
      struct T { uint x; uint y; }
      function f(S s, T t, uint a) public { }
      function g() public returns (S s, T t, uint a) {}
    }

resultaría en el JSON:

.. code:: json

  [
    {
      "name": "f",
      "type": "function",
      "inputs": [
        {
          "name": "s",
          "type": "tuple",
          "components": [
            {
              "name": "a",
              "type": "uint256"
            },
            {
              "name": "b",
              "type": "uint256[]"
            },
            {
              "name": "c",
              "type": "tuple[]",
              "components": [
                {
                  "name": "x",
                  "type": "uint256"
                },
                {
                  "name": "y",
                  "type": "uint256"
                }
              ]
            }
          ]
        },
        {
          "name": "t",
          "type": "tuple",
          "components": [
            {
              "name": "x",
              "type": "uint256"
            },
            {
              "name": "y",
              "type": "uint256"
            }
          ]
        },
        {
          "name": "a",
          "type": "uint256"
        }
      ],
      "outputs": []
    }
  ]

.. _abi_packed_mode:

Modo compacto no estándar
=========================

Solidity soporta un modo compacto no estándar donde:

- no :ref:``function selector <abi_function_selector>`` está codificado,
- los tipos de tamaño inferior a 32 bytes no están acolchados por ceros ni tienen signos extendidos y
- los tipos dinámicos están codificados in situ y sin longitud.

Un ejemplo de codificación ``int1, bytes1, uint16, string`` con valores ``-1, 0x42, 0x2424, "Hello, world!"`` da lugar a ::

    0xff42242448656c6c6f2c20776f726c6421
      ^^                                 int1(-1)
        ^^                               bytes1(0x42)
          ^^^^                           uint16(0x2424)
              ^^^^^^^^^^^^^^^^^^^^^^^^^^ string("Hello, world!") sin un campo longitud

Específicamente, cada tipo de tamaño estático toma tantos bytes como su gama tenga
y tipos de tamaño dinámico como ``string``, ``bytes`` o ``uint[]`` están codificados sin 
su campo de longitud. Esto significa que la codificación es ambigua en cuanto hay dos tipos de 
elementos dinámicamente dimensionados.
