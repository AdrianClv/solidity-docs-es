.. index:: abi, application binary interface

.. _ABI:

**********************************************
Especificación de Application Binary Interface
**********************************************

Diseño básico
=============

La Application Binary Interface (Interfaz Binaria de Aplicación) o ABI es el modo estándar de interactuar con contratos en el ecosistema Ethereum, tanto desde fuera de la blockchain como en interacciones contrato-contrato. Los datos se codifican siguiendo su tipo acorde a esta especificación.

Asumimos que la Application Binary Interface (ABI) está extremadamente definida, es conocida en tiempo de compilación y es estática. No se van a proveer mecanismos de introspección. Además, afirmamos que los contratos tendrán las definiciones de la interfaz de cada contrato que vayan a llamar en tiempo de compilación.

Esta especificación no abarca los contratos cuya interfaz sea dinámica o conocida exclusivamente en tiempo de ejecución. Estos casos, de volverse importantes, podrían manejarse adecuadamente como servicios construídos dentro del ecosistema Ethereum.

Función Selector
=================

Los primeros cuatro bytes de lo datos de una llamada a una función especifiacan la función a llamar. Se trata de los primeros (los más a la izquierda, los más extremos por orden) cuatro bytes del hash Keccak (SHA-3) de la firma de la función. La firma se define como la expresión canónica del prototipo básico como puede ser el nombre de la función con la lista de parámetros entre paréntesis. Los tipos de parámetros se separan por comas, no por espacios.

Codificación de argumentos
==========================

A partir del quinto byte, prosiguen los argumentos codificados. Esta codificación es también usada en otros sitios, por ejemplo, en los valores de retorno y también en los argumentos de eventos, sin los cuatro bytes especificando la función.

Tipos
=====

Los tipos elementales existentes son:

- `uint<M>`: enteros sin signo de `M` bits, `0 < M <= 256`, `M % 8 == 0`. Ejemplos: `uint32`, `uint8`, `uint256`.

- `int<M>`: enteros con signo de `M` bits, `0 < M <= 256`, `M % 8 == 0`.

- `address`: equivalente a `uint160`, exceptuando la interpretación asumida y la tipología de idioma.

- `uint`, `int`: sinónimos de `uint256`, `int256` respectivamente (no para ser usados con la función selector).

- `bool`: equivalente a `uint8` restringido a los valores 0 y 1.

- `fixed<M>x<N>`: número decimal con signo y formato decimal fijo de `M` bits, `0 < M <= 256`, `M % 8 ==0`, y `0 < N <= 80`, que denota el valor `v` como `v / (10 ** N)`.

- `ufixed<M>x<N>`: variante sin signo de `fixed<M>x<N>`.

- `fixed`, `ufixed`: sinónimos de `fixed128x19`, `ufixed128x19` respectivamente (no para ser usados con la función selector).

- `bytes<M>`: tipo binario de `M` bytes, `0 < M <= 32`.

- `function`: equivalente a `bytes24`: un address, seguido de la función selector

El siguiente array, de tipo fijo, existente es:

- `<type>[M]`: un array de longitud fija del tipo de longitud fija dada.

Los siguientes tipos de tamaño no fijo existentes son: 

- `bytes`: secuancia de bytes de tamaño dinámico.

- `string`: string unicode de tamaño dinámico codificado como UTF-8.

- `<type>[]`: array de longitud variable del tipo de longitud fija dada.

Los distintos tipos se pueden combinar en structs anónimos cerrando un número finito no negativo de ellos entre paréntesis, separados por comas:

- `(T1,T2,...,Tn)`: struct anónimo (tupla ordenada) consistente de los tipos `T1`, ..., `Tn`, `n >= 0`

Es posible formar structs de structs, arrays de structs, etc.


Especificación formal de la codificación
========================================

Vamos a especificar formalmente la codificación, de tal forma que tendrá las siguientes propiedades, que son especialmente útiles si los argumentos son arrays anidados:

Propiedades:

  1. El número de lecturas necesarias para acceder a un valor es como mucho equivalente a la máxima profundidad del array. Por ejemplo, cuatro lecturas se requieren para obtener `a_i[k][l][r]`. En una versión previa de la ABI, el número de lecturas escalaba linearmente con el número total de parámetros dinámicos en el peor caso.

  2. Los datos de una variable o elemento de un array no se intercalan con otros datos y son recolocables. Por ejemplo, sólo usan "addresses" relativos.

Distinguimos entre tipos estáticos y dinámicos. Los estáticos se codifican insitu y los dinámicos se codifican en una posición asignada separadamente después del bloque actual.

**Definición:** Los siguientes tipos se llaman "dinámicos":
* `bytes`
* `string`
* `T[]` para cada `T`
* `T[k]` para cualquier dinámico `T` y todo `k > 0`

Todo el resto de tipos son "estáticos".

**Definición:** `len(a)` es el número de bytes en un string binario `a`.
El tipo de `len(a)` se presume como `uint256`.

Definimos `enc`, la codificación actual, como un mapping de valores de tipos de la ABI a string binarios como `len(enc(X))` depende del valor de `X` si y solo si el tipo de `X` es dinámico.

**Definición:** Para cada valor de ABI `X`, definimos recursivamente `enc(X)`, dependiendo del tipo de `X` siendo

- `(T1,...,Tk)` para `k >= 0` y cualquier tipo `T1`, ..., `Tk`

  `enc(X) = head(X(1)) ... head(X(k-1)) tail(X(0)) ... tail(X(k-1))`

  donde `X(i)` es el `ith` componente del valor, y
  `head` y `tail` son definidos por `Ti` siendo un tipo estático como

    `head(X(i)) = enc(X(i))` y `tail(X(i)) = ""` (el string vacío)

  y como

    `head(X(i)) = enc(len(head(X(0)) ... head(X(k-1)) tail(X(0)) ... tail(X(i-1))))`
    `tail(X(i)) = enc(X(i))`

  de otra manera, como `Ti` es un tipo dinámico.

  Hay que tener en cuenta que en el caso dinámico, `head(X(i))` está bien definido ya que las longitudes de las partes de head sólo dependen de los tipos y no de los valores. Su valor es el offset del principio de `tail(X(i))` relativo al comienzo de `enc(X)`.
  
- `T[k]` para cada `T` y `k`:

  `enc(X) = enc((X[0], ..., X[k-1]))`
  
  como ejemplo, es codificado como si fuera un struct anónimo de `k` elementos del mismo tipo.
  
- `T[]` donde `X` tiene `k` elementos (`k` se presume que es del tipo `uint256`):

  `enc(X) = enc(k) enc([X[1], ..., X[k]])`

  Otro ejemplo codificado como si fuera un array estático de tamaño `k`, prefijado con el número de elementos.

- `bytes`, de longitud `k` (que se presume que es del tipo `uint256`):

  `enc(X) = enc(k) pad_right(X)`. Por ejemplo, el número de bytes es codificado como un `uint256` seguido del valor actual de `X` como una secuancia de bytes, seguido por el número mínimo de bytes-cero como que `len(enc(X))` es un múltiplo de 32.

- `string`:

  `enc(X) = enc(enc_utf8(X))`, en este caso `X` se codifica como utf-8 y su valor se interpreta como de tipo `bytes` y codificado posteriormente. Hay que tener en cuenta que la longitud usada en la subsecuente codificación es el número de bytes del string codificado como utf-8, no su número de caracteres.

- `uint<M>`: `enc(X)` es el mayor extremo de la codificación de `X`, rellenado en el lado de orden mayor (izquierda) con bytes cero de tal forma que la longitud acabe siendo de 32 bytes.
- `address`: como en el caso de `uint160`
- `int<M>`: `enc(X)` es el complemento a dos de mayor extremo en la codificación de `X`, rellenado en el lado de mayor orden (izquierda) con `0xff` para `X` negativo y bytes cero para `X` positivo de tal forma que la longitud final sea un múltiplo de 32 bytes.
- `bool`: como en el caso de `uint8`, donde `1` se usa para `true` y `0` para `false`
- `fixed<M>x<N>`: `enc(X)` es `enc(X * 10**N)` donde `X * 10**N` se interpreta como un `int256`.
- `fixed`: como en el caso de `fixed128x19`
- `ufixed<M>x<N>`: `enc(X)` es `enc(X * 10**N)` donde `X * 10**N` se interpreta como un `uint256`.
- `ufixed`: como en el caso de `ufixed128x19`
- `bytes<M>`: `enc(X)` es la secuencia de bytes en `X` rellenado con bytes cero hasta una longitud de 32.

Resaltar que para cada `X`, `len(enc(X))` es un múltiplo de 32.

Función Selector y codificación de argumentos
=============================================

Siempre, una llamada a la función `f` con parámetros `a_1, ..., a_n` se codifican como 

  `function_selector(f) enc((a_1, ..., a_n))`

y los valores de retorno `v_1, ..., v_k` de `f` son codificados como

  `enc((v_1, ..., v_k))`

p.ej.: los valores se combinan en struct anónimos y codificados.

Ejemplos
========

Para el siguiente contrato:

::

    contract Foo {
      function bar(bytes3[2] xy) {}
      function baz(uint32 x, bool y) returns (bool r) { r = x > 32 || y; }
      function sam(bytes name, bool z, uint[] data) {}
    }


Para nuestro ejemplo `Foo`, si queremos llamar a `baz` pasando como parámetros `69` y `true`, emplearíamos 68 bytes en total, que se podrían dividir en las siguientes partes:

- `0xcdcd77c0`: el ID del método. Se deriva como los 4 primeros bytes del hash Keccak en ASCII de la firma `baz(uint32,bool)`.
- `0x0000000000000000000000000000000000000000000000000000000000000045`: el primer parámetro, un uint32 de valor `69` rellenado hasta 32 bytes
- `0x0000000000000000000000000000000000000000000000000000000000000001`: el segundo parámetro - boolean `true`, rellendo hasta 32 bytes

En total::

    0xcdcd77c000000000000000000000000000000000000000000000000000000000000000450000000000000000000000000000000000000000000000000000000000000001

Devuelve un `bool` simple. Si, por ejemplo, devolviese `false`, su salida sería un array de byte sencillo `0x0000000000000000000000000000000000000000000000000000000000000000`, un único bool.

Si quisiéramos llamar a `bar` con el argumento `["abc", "def"]`, pasaríamos 68 bytes en total, divido en:

- `0xfce353f6`: el ID del método. Este se deriva de la firma `bar(bytes3[2])`.
- `0x6162630000000000000000000000000000000000000000000000000000000000`: La primera parte del primer parámetro, un valor `bytes3` `"abc"` (alineado a la izquierda).
- `0x6465660000000000000000000000000000000000000000000000000000000000`: La segunda parte del primer parámetro, un vaor `bytes3` `"def"` (alineado a la izquierda).

En total::

    0xfce353f661626300000000000000000000000000000000000000000000000000000000006465660000000000000000000000000000000000000000000000000000000000

Si quisiéramos llamar a `sam` con los argumentos `"dave"`, `true` y `[1,2,3]`, pasaríamos 292 bytes en total, dividido en:
- `0xa5643bf2`: el ID del método. Este se deriva de la firma `sam(bytes,bool,uint256[])`. Aquí `uint` se reemplaza por su representación canónica `uint256`.
- `0x0000000000000000000000000000000000000000000000000000000000000060`: La localización de la parte de datos del primer parámetro (tipo dinámico), medido en bytes desde el principio del bloque de argumentos. En este caso, `0x60`.
- `0x0000000000000000000000000000000000000000000000000000000000000001`: el segundo parámetro: boolean verdadero.
- `0x00000000000000000000000000000000000000000000000000000000000000a0`: La localización de la parte de datos del tercer parámetro (tipo dinámico), medido en bytes. En este caso, `0xa0`.
- `0x0000000000000000000000000000000000000000000000000000000000000004`: La parte de datos del primer argumento, comienza con la longitud del array de bytes en elementos, en este caso, 4.
- `0x6461766500000000000000000000000000000000000000000000000000000000`: Los contenidos del primer argumento: el UTF-8 (equivalente a ASCII en este caso) codificación de `"dave"`, rellenado hasta 32 bytes por la derecha.
- `0x0000000000000000000000000000000000000000000000000000000000000003`: La parte de datos del tercer argumento, comenzando con la longitud del array en elementos, en este caso, 3.
- `0x0000000000000000000000000000000000000000000000000000000000000001`: la primera entrada del tercer parámetro.
- `0x0000000000000000000000000000000000000000000000000000000000000002`: la segunda entrada del tercer parámetro.
- `0x0000000000000000000000000000000000000000000000000000000000000003`: la tercera entrada del tercer parámetro.

En total::

    0xa5643bf20000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000000000000000000000000000464617665000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000003

Uso de tipos dinámicos
======================

Una llamada a una función con la firma `f(uint,uint32[],bytes10,bytes)` con valores `(0x123, [0x456, 0x789], "1234567890", "Hello, world!")` se codifica de la siguiente manera:

Obtenemos los primeros cuatro bytes de `sha3("f(uint256,uint32[],bytes10,bytes)")`, p.ej.: `0x8be65246`.
Entonces codificamos las cabeceras de los cuatro argumentos. Para los tipos estáticos `uint256` y `bytes10`, estos son los valores que queremos pasar directamente, miestras que para los tipos dinámicos `uint32[]` y `bytes`, usamos el offset en bytes hasta el inicio de su área de datos, contando desde el comienzo de la codificación del valor (p.ej.: sin contar los primeros cuatro bytes que contienen el hash de la firma de la función). Estos son:

 - `0x0000000000000000000000000000000000000000000000000000000000000123` (`0x123` rellenado hasta 32 bytes)
 - `0x0000000000000000000000000000000000000000000000000000000000000080` (offset del inicio de la parte de datos del seguno parámetro, 4*32 bytes, exactamente el tamaño de la parte de la cabecera)
 - `0x3132333435363738393000000000000000000000000000000000000000000000` (`"1234567890"` rellenado hasta 32 bytes por la derecha)
 - `0x00000000000000000000000000000000000000000000000000000000000000e0` (offset del comienzo de la parte de datos del cuarto parámetro = offset del inicio de la parte de datos del primer parámetro dinámico + tamaño de la parte de datos del primer parámetro dinámico = 4\*32 + 3\*32 (ver abajo))

Después de esto, la parte de datos del primer argumento dinámico, `[0x456, 0x789]` sigue así:

 - `0x0000000000000000000000000000000000000000000000000000000000000002` (número de elementos del array, 2)
 - `0x0000000000000000000000000000000000000000000000000000000000000456` (primer elemento)
 - `0x0000000000000000000000000000000000000000000000000000000000000789` (segundo elemento)

Finalmente, codificamos la parte de datos del segundo argumento dinámico, `"¡Hola mundo!"`:

 - `0x000000000000000000000000000000000000000000000000000000000000000d` (número de elementos (bytes en este caso): 13)
 - `0x48656c6c6f2c20776f726c642100000000000000000000000000000000000000` (`"¡Hola mundo!"` rellenado hasta 32 bytes por la derecha)

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

Dado un nombre de evento y una serie de parámetros de evento, los separamos en dos sub-series: las que están indexadas y las que no. Las indexadas, cuyo número podría llegar hasta tres, se usan junto al hash Keccack de la firma del evento para formar los tópicos de la entrada de log. Los no indexados forman el array de bytes del evento.

En efecto, una entrada de log que usa esta ABI se define como:

- `address`: la dirección del contrato (intrínsicamente provista por Ethereum);
- `topics[0]`: `keccak(EVENT_NAME+"("+EVENT_ARGS.map(canonical_type_of).join(",")+")")` (`canonical_type_of` es una función que simplemente devuelve el tipo canónico del argumento dado, p.ej.: para `uint indexed foo`, devolvería `uint256`). Si el evento se declara como `anonymous` no se genera `topics[0]`;
- `topics[n]`: `EVENT_INDEXED_ARGS[n - 1]` (`EVENT_INDEXED_ARGS` es la serie de `EVENT_ARGS` que están indexados);
- `data`: `abi_serialise(EVENT_NON_INDEXED_ARGS)` (`EVENT_NON_INDEXED_ARGS` es la serie de `EVENT_ARGS` que no están indexados, `abi_serialise` es la función de serialización ABI usada para devolver una serie de valores tipificados desde una función, como se detalla abajo).

JSON
====

El formato JSON para la interfaz de un contrato viene dada por una array de función y/o descripciones de evento. Una descripción de función es un objeto JSON con los siguientes campos:

- `type`: `"function"`, `"constructor"`, o `"fallback"` (the :ref:`unnamed "default" function <fallback-function>`);
- `name`: nombre de la función;
- `inputs`: array de objetos, cada uno contiene:
  * `name`: nombre del parámetro;
  * `type`: tipo canónico del parámetro.
- `outputs`: un array de objet0s similar a `inputs`, puede omotirse si la función no devuelve nada;
- `constant`: `true` si la función es :ref:`specified to not modify blockchain state <constant-functions>`);
- `payable`: `true` si la función acepta ether, por defecto a `false`.

`type` se puede omitir, dejándolo por defecto a `"function"`.

La función Constructor y fallback nunca tienen `name` o `outputs`. Fallback tampoco tiene `inputs`.

Enviar ether no-cero a una función no pagable se disparará. No lo hagas.

Una descripción de evento es un objeto JSON con prácticamente los mismos campos:

- `type`: siempre `"event"`
- `name`: nombre del evento;
- `inputs`: array de objetos, cada uno contiene:
  * `name`: nombre del parámetro;
  * `type`: tipo canónico del parámetro.
  * `indexed`: `true` si el campo es parte de los tópicos del log, `false` si es parte del segmento de datos del log.
- `anonymous`: `true` si el evento se declaró `anonymous`.

Por ejemplo, 

::

  contract Test {
    function Test(){ b = 0x12345678901234567890123456789012; }
    event Event(uint indexed a, bytes32 b)
    event Event2(uint indexed a, bytes32 b)
    function foo(uint a) { Event(a, b); }
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
  "type":"event",
  "inputs": [{"name":"a","type":"uint256","indexed":true},{"name":"b","type":"bytes32","indexed":false}],
  "name":"Event2"
  }, {
  "type":"function",
  "inputs": [{"name":"a","type":"uint256"}],
  "name":"foo",
  "outputs": []
  }]
