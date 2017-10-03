.. index:: abi, application binary interface

.. _ABI:

******************************************
Especificación de Application Binary Interface
******************************************

Diseño básico
============

La Application Binary Interface (Interfaz Binaria de Aplicación) es el modo estándar de interactuar con contratos en el ecosistema Ethereum, tanto desde fuera de la blockchain como en interacciones contrato-contrato. Los datos se codifican siguiendo su tipo acorde a esta especificación.

Asumimos que la Application Binary Interface (ABI) está extremadamente definida, es conocida en tiempo de compilación y es estática. No se van a proveer mecanismos de introspección. Además, afirmamos que los contratos tendrán las definiciones de la interfaz de cada contrato que vayan a llamar en tiempo de compilación.

Esta especificación no abarca los contratos cuya interfaz es dinámica o conocida exclusivamente en tiempo de ejecución. Estos casos, de volverse importantes, podrían manejarse adecuadamente como servicios construídos dentro del ecosistema Ethereum.

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

- `<type>[M]`: un array de longitud fija del tipo de longitud fija dado.

Los siguientes tipos de tamaño no fijo existentes son: 

- `bytes`: secuancia de bytes de tamaño dinámico.

- `string`: string unicode de tamaño dinámico codificado como UTF-8.

- `<type>[]`: array de longitud variable del tipo de longitud fija dado.

Los distintos tipos se pueden combinar en structs anónimos cerrando un número finito no negativo de ellos entre paréntesis, separados por comas:

- `(T1,T2,...,Tn)`: struct anónimo (tupla ordenada) consistente de los tipos `T1`, ..., `Tn`, `n >= 0`

Es posible formar structs de structs, arrays de structs, etc.


Especificación formal de la codificación
========================================

Vamos a especificar formalmente la codificación, de tal forma que tendrá las siguientes propiedades, que son especialmente útiles si los argumentos son arrays anidados:

Propiedades:

  1. El número de lecturas necesarias para acceder a un valor es como mucho equivalente a la máxima profundidad del array. Por ejemplo, cuatro lecturas se requieren para obtener `a_i[k][l][r]`. En una versión previa de la ABI, el número de lecturas escalaba linearmente con el número total de parámetros dinámicos en el peor caso.

  2. Los datos de una variable o elemento de un array no se intercalan con otros datos y son recolocables. Por ejemplo, sólo usa "addresses" relativos.

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

  `enc(X) = enc(k) pad_right(X)`. Por ejemplo, el número de bytes es codificado como un `uint256` seguido del valor actual de `X` como una secuancia de bytes, continuado por el número mínimo de bytes-cero como que `len(enc(X))` es un múltiplo de 32.

- `string`:

  `enc(X) = enc(enc_utf8(X))`, i.e. `X` is utf-8 encoded and this value is interpreted as of `bytes` type and encoded further. Note that the length used in this subsequent encoding is the number of bytes of the utf-8 encoded string, not its number of characters.

- `uint<M>`: `enc(X)` is the big-endian encoding of `X`, padded on the higher-order (left) side with zero-bytes such that the length is a multiple of 32 bytes.
- `address`: as in the `uint160` case
- `int<M>`: `enc(X)` is the big-endian two's complement encoding of `X`, padded on the higher-oder (left) side with `0xff` for negative `X` and with zero bytes for positive `X` such that the length is a multiple of 32 bytes.
- `bool`: as in the `uint8` case, where `1` is used for `true` and `0` for `false`
- `fixed<M>x<N>`: `enc(X)` is `enc(X * 10**N)` where `X * 10**N` is interpreted as a `int256`.
- `fixed`: as in the `fixed128x19` case
- `ufixed<M>x<N>`: `enc(X)` is `enc(X * 10**N)` where `X * 10**N` is interpreted as a `uint256`.
- `ufixed`: as in the `ufixed128x19` case
- `bytes<M>`: `enc(X)` is the sequence of bytes in `X` padded with zero-bytes to a length of 32.

Note that for any `X`, `len(enc(X))` is a multiple of 32.

Function Selector and Argument Encoding
=======================================

All in all, a call to the function `f` with parameters `a_1, ..., a_n` is encoded as

  `function_selector(f) enc((a_1, ..., a_n))`

and the return values `v_1, ..., v_k` of `f` are encoded as

  `enc((v_1, ..., v_k))`

i.e. the values are combined into an anonymous struct and encoded.

Examples
========

Given the contract:

::

    contract Foo {
      function bar(bytes3[2] xy) {}
      function baz(uint32 x, bool y) returns (bool r) { r = x > 32 || y; }
      function sam(bytes name, bool z, uint[] data) {}
    }


Thus for our `Foo` example if we wanted to call `baz` with the parameters `69` and `true`, we would pass 68 bytes total, which can be broken down into:

- `0xcdcd77c0`: the Method ID. This is derived as the first 4 bytes of the Keccak hash of the ASCII form of the signature `baz(uint32,bool)`.
- `0x0000000000000000000000000000000000000000000000000000000000000045`: the first parameter, a uint32 value `69` padded to 32 bytes
- `0x0000000000000000000000000000000000000000000000000000000000000001`: the second parameter - boolean `true`, padded to 32 bytes

In total::

    0xcdcd77c000000000000000000000000000000000000000000000000000000000000000450000000000000000000000000000000000000000000000000000000000000001

It returns a single `bool`. If, for example, it were to return `false`, its output would be the single byte array `0x0000000000000000000000000000000000000000000000000000000000000000`, a single bool.

If we wanted to call `bar` with the argument `["abc", "def"]`, we would pass 68 bytes total, broken down into:

- `0xfce353f6`: the Method ID. This is derived from the signature `bar(bytes3[2])`.
- `0x6162630000000000000000000000000000000000000000000000000000000000`: the first part of the first parameter, a `bytes3` value `"abc"` (left-aligned).
- `0x6465660000000000000000000000000000000000000000000000000000000000`: the second part of the first parameter, a `bytes3` value `"def"` (left-aligned).

In total::

    0xfce353f661626300000000000000000000000000000000000000000000000000000000006465660000000000000000000000000000000000000000000000000000000000

If we wanted to call `sam` with the arguments `"dave"`, `true` and `[1,2,3]`, we would pass 292 bytes total, broken down into:
- `0xa5643bf2`: the Method ID. This is derived from the signature `sam(bytes,bool,uint256[])`. Note that `uint` is replaced with its canonical representation `uint256`.
- `0x0000000000000000000000000000000000000000000000000000000000000060`: the location of the data part of the first parameter (dynamic type), measured in bytes from the start of the arguments block. In this case, `0x60`.
- `0x0000000000000000000000000000000000000000000000000000000000000001`: the second parameter: boolean true.
- `0x00000000000000000000000000000000000000000000000000000000000000a0`: the location of the data part of the third parameter (dynamic type), measured in bytes. In this case, `0xa0`.
- `0x0000000000000000000000000000000000000000000000000000000000000004`: the data part of the first argument, it starts with the length of the byte array in elements, in this case, 4.
- `0x6461766500000000000000000000000000000000000000000000000000000000`: the contents of the first argument: the UTF-8 (equal to ASCII in this case) encoding of `"dave"`, padded on the right to 32 bytes.
- `0x0000000000000000000000000000000000000000000000000000000000000003`: the data part of the third argument, it starts with the length of the array in elements, in this case, 3.
- `0x0000000000000000000000000000000000000000000000000000000000000001`: the first entry of the third parameter.
- `0x0000000000000000000000000000000000000000000000000000000000000002`: the second entry of the third parameter.
- `0x0000000000000000000000000000000000000000000000000000000000000003`: the third entry of the third parameter.

In total::

    0xa5643bf20000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000000000000000000000000000464617665000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000003

Use of Dynamic Types
====================

A call to a function with the signature `f(uint,uint32[],bytes10,bytes)` with values `(0x123, [0x456, 0x789], "1234567890", "Hello, world!")` is encoded in the following way:

We take the first four bytes of `sha3("f(uint256,uint32[],bytes10,bytes)")`, i.e. `0x8be65246`.
Then we encode the head parts of all four arguments. For the static types `uint256` and `bytes10`, these are directly the values we want to pass, whereas for the dynamic types `uint32[]` and `bytes`, we use the offset in bytes to the start of their data area, measured from the start of the value encoding (i.e. not counting the first four bytes containing the hash of the function signature). These are:

 - `0x0000000000000000000000000000000000000000000000000000000000000123` (`0x123` padded to 32 bytes)
 - `0x0000000000000000000000000000000000000000000000000000000000000080` (offset to start of data part of second parameter, 4*32 bytes, exactly the size of the head part)
 - `0x3132333435363738393000000000000000000000000000000000000000000000` (`"1234567890"` padded to 32 bytes on the right)
 - `0x00000000000000000000000000000000000000000000000000000000000000e0` (offset to start of data part of fourth parameter = offset to start of data part of first dynamic parameter + size of data part of first dynamic parameter = 4\*32 + 3\*32 (see below))

After this, the data part of the first dynamic argument, `[0x456, 0x789]` follows:

 - `0x0000000000000000000000000000000000000000000000000000000000000002` (number of elements of the array, 2)
 - `0x0000000000000000000000000000000000000000000000000000000000000456` (first element)
 - `0x0000000000000000000000000000000000000000000000000000000000000789` (second element)

Finally, we encode the data part of the second dynamic argument, `"Hello, world!"`:

 - `0x000000000000000000000000000000000000000000000000000000000000000d` (number of elements (bytes in this case): 13)
 - `0x48656c6c6f2c20776f726c642100000000000000000000000000000000000000` (`"Hello, world!"` padded to 32 bytes on the right)

All together, the encoding is (newline after function selector and each 32-bytes for clarity):

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

Events
======

Events are an abstraction of the Ethereum logging/event-watching protocol. Log entries provide the contract's address, a series of up to four topics and some arbitrary length binary data. Events leverage the existing function ABI in order to interpret this (together with an interface spec) as a properly typed structure.

Given an event name and series of event parameters, we split them into two sub-series: those which are indexed and those which are not. Those which are indexed, which may number up to 3, are used alongside the Keccak hash of the event signature to form the topics of the log entry. Those which as not indexed form the byte array of the event.

In effect, a log entry using this ABI is described as:

- `address`: the address of the contract (intrinsically provided by Ethereum);
- `topics[0]`: `keccak(EVENT_NAME+"("+EVENT_ARGS.map(canonical_type_of).join(",")+")")` (`canonical_type_of` is a function that simply returns the canonical type of a given argument, e.g. for `uint indexed foo`, it would return `uint256`). If the event is declared as `anonymous` the `topics[0]` is not generated;
- `topics[n]`: `EVENT_INDEXED_ARGS[n - 1]` (`EVENT_INDEXED_ARGS` is the series of `EVENT_ARGS` that are indexed);
- `data`: `abi_serialise(EVENT_NON_INDEXED_ARGS)` (`EVENT_NON_INDEXED_ARGS` is the series of `EVENT_ARGS` that are not indexed, `abi_serialise` is the ABI serialisation function used for returning a series of typed values from a function, as described above).

JSON
====

The JSON format for a contract's interface is given by an array of function and/or event descriptions. A function description is a JSON object with the fields:

- `type`: `"function"`, `"constructor"`, or `"fallback"` (the :ref:`unnamed "default" function <fallback-function>`);
- `name`: the name of the function;
- `inputs`: an array of objects, each of which contains:
  * `name`: the name of the parameter;
  * `type`: the canonical type of the parameter.
- `outputs`: an array of objects similar to `inputs`, can be omitted if function doesn't return anything;
- `constant`: `true` if function is :ref:`specified to not modify blockchain state <constant-functions>`);
- `payable`: `true` if function accepts ether, defaults to `false`.

`type` can be omitted, defaulting to `"function"`.

Constructor and fallback function never have `name` or `outputs`. Fallback function doesn't have `inputs` either.

Sending non-zero ether to non-payable function will throw. Don't do it.

An event description is a JSON object with fairly similar fields:

- `type`: always `"event"`
- `name`: the name of the event;
- `inputs`: an array of objects, each of which contains:
  * `name`: the name of the parameter;
  * `type`: the canonical type of the parameter.
  * `indexed`: `true` if the field is part of the log's topics, `false` if it one of the log's data segment.
- `anonymous`: `true` if the event was declared as `anonymous`.

For example, 

::

  contract Test {
    function Test(){ b = 0x12345678901234567890123456789012; }
    event Event(uint indexed a, bytes32 b)
    event Event2(uint indexed a, bytes32 b)
    function foo(uint a) { Event(a, b); }
    bytes32 b;
  }

would result in the JSON:

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
