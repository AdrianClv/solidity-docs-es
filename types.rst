.. index:: type

.. _types:

*****
Tipos
*****

Solidity es un lenguaje de tipado estátio, que significa que cada tipo de
variable (estado y local) tiene ser especificada (o al menos conocida -
ver :ref:`type-deduction` abajo) en tiempo de compilación.
Solidity proporciona varios tipos elementales que pueden ser combinados para
crear tipos mas complejos.

Además, tipos puedes interactuar con
In addition, types can interact with each other in expressions containing
operators. For a quick reference of the various operators, see :ref:`order`.

.. index:: ! value type, ! type;value

Tipos de Valor
==============

Los siguientes tipos también son llamados tipos de valor porque las variables
de este tipo serán siempre pasadas como valores, ej. siempre serán copiados cuando
son usados como argumentos de funciones o en asignaciones.

.. index:: ! bool, ! true, ! false

Booleanos
---------

``bool``: Los posibles valores son las constantes ``true`` and ``false``.

Operadores:

*  ``!`` (negación lógica)
*  ``&&`` (conjunción lógica, "y")
*  ``||`` (disyunción lógica, "or")
*  ``==`` (igualdad)
*  ``!=`` (inigualdad)

Los operadores ``||`` y ``&&`` aplican las reglas comunes de corto circuitos. Esto significa que en la expresión c, si ``f(x)`` evalúa a ``true``, ``g(y)`` no será evaluado incluso si tuviera efectos segundarios.

.. index:: ! uint, ! int, ! integer

Enteros
-------

``int`` / ``uint``: Enteros con y sin signo de varios tamaños. Las palabras clave ``uint8`` a ``uint256`` en pasos de ``8`` (sin signo de 8 hasta 256 bits) y ``int8`` a ``int256``. ``uint`` y ``int`` son aliases para ``uint256`` y ``int256``, respectivamete.

Operadores:

* Comparaciones: ``<=``, ``<``, ``==``, ``!=``, ``>=``, ``>`` (evalúa a ``bool``)
* Operadores bit: ``&``, ``|``, ``^`` (bitwise exclusivo or), ``~`` (negación bitwise)
* Operadores aritméticos: ``+``, ``-``, unary ``-``, unary ``+``, ``*``, ``/``, ``%`` (restante), ``**`` (exponenciales), ``<<`` (shift izquiera), ``>>`` (shift derecha)

La división siempre trunca (está compilado a la opcode DIV de la EVM), pero no trunca si los dos
operadores son :ref:`literales<rational_literals>` (o expresiones literales).

División por cero y modulus con cero arrojan una excepcion runtime.

El resultado de una operación shift es el tipo de operador izquierdo. La
expresión ``x << y`` es equivalente a ``x * 2**y`` y ``x >> y`` es
equivalente a ``x / 2**y``. Esto significa que hacer un shift de números negativos
extiende en signo. Haciendo shift por un número negativo arroja una excepción runtime.

.. warning::
    Los resultados producidos por shift derecho de valores negativos de tipos de enteros con signo es diferente de esos producidos
    por otros lenguajes de programación. En Solidity, shift derecho mapea la división para que los valores negativos de shift
    serán redondeados hacia cero (truncado). En otros lenguajes de programacion el shift derecho de valores negativos
    funciona como una división con redondeo hacia abajo (hacia infinito negativo).

.. index:: address, balance, send, call, callcode, delegatecall, transfer

.. _address:

Address
-------

``address``: Contiene un valor de 20 byte (tamaño de una dirección Ethereum). Tipos de address también miembros y sirven como base para todos los contratos.

Operadores:

* ``<=``, ``<``, ``==``, ``!=``, ``>=`` and ``>``

Miembros de Address
^^^^^^^^^^^^^^^^^^^

* ``balance`` and ``transfer``

Para una referencia rápida, ver :ref:`address_related`.

Es posible consultar el monto de una dirección usando la propiedad ``balance``
y de enviar Ether (en unidades de wei) a una dirección usando la función ``transfer``:

::

    address x = 0x123;
    address myAddress = this;
    if (x.balance < 10 && myAddress.balance >= 10) x.transfer(10);

.. note::
    Si ``x`` es una dirección de contrato, su código (específicamente: su función de fallback, si es que está presente) será ejecutada con el llamado ``transfer`` (esta es la limitación de la EVM y no puede ser prevenida). Si esa ejecución acaba el gas o falla de cualquier forma, el Ether transferido será revertido y el contrato actual se detendrá con una excepción.

* ``send``

Send es la contrapartida de bajo nivel de ``transfer``. Si la ejecución falla, el contrato actual no se detendrá con una excepción, pero ``send`` devuelve ``false``.

.. warning::
    Hay algunos peligros en utiliza ``send``: La transferencia falla si la profundidad de la llamada es de 1024
    (esto puede ser fozado por el llamador) y también falla si el recipiente se le acaba el gas. Entonces para
    hacer transferencia de Ether seguras, siempre revisar el valor devuleto por ``send``, usar ``transfer`` o incluso mejor:
    usa el patrón donde el recipiente retira el dinero.

* ``call``, ``callcode`` and ``delegatecall``

Además, para interfazar con contratos que no adhieren al ABI,
la función ``call`` es provedida que toma un número arbitrario de argumentos de cualquier tipo. Estos argumentos son alcochados a 32 bytes y concatenados. Una excepción es el caso donde el primer argumento es codificado a exactamente 4 bytes. En este caso, no está acolchado para permitir el uso de firmas de función aquí.

::

    address nameReg = 0x72ba7d8e73fe8eb666ea66babc8116a41bfb10e2;
    nameReg.call("register", "MyName");
    nameReg.call(bytes4(keccak256("fun(uint256)")), a);

``call`` devuelve un booleano indicando si la función llamada terminó (``true``) o causó una excepción del EVM (``false``). No es posible acceder a los datos reales devueltos (para esto necesitaremos saber el tamaño de codificación en avance).

En una forma similar, ``delegatecall`` puede ser usado: La diferencia es que solo el código de la dirección dada es usado, todo otros aspectos (almacenamiento, saldo, ...) salen del contrato actual. El propósito de ``delegatecall`` es usar el código de librería que está almacenado en otro contrato. El usuario tiene que asegurarse que el layout del almacenamiento en ambos contratos es correcto para usar delegatecall. Antes de homestead, sólo una versión limitada llamada ``callcode`` estaba disponible que no daba acceso a los valores ``msg.sender`` y ``msg.value`` originales.

Las tres funciones ``call``, ``delegatecall`` y ``callcode`` son funciones de muy bajo nivel y deben usarse sólo como medida de último recurso ya que rompen la seguridad de tipo de Solidity.

La opción ``.gas()`` está disponible en los 3 métodos, mientras la opción ``.value()`` no se admite para ``delegatecall``.

.. note::
    Todos los contratos heredan los miembros de address, así que es posible consultar el saldo del contrato actual
    usando ``this.balance``.

.. warning::
    Todas estas funciones son funciones de bajo nivel y debe usarse con cuidado.
    Específicamente, cualquier contrato desconocido puede ser malicioso y si se le llama,
    se le da el controll a ese contrato que puede, luego llamar de vuelta a tu contrato,
    así que prepárense para cambios a tus variables de estado cuando el llamado retorna.

.. index:: byte array, bytes32


Colleción de byte de tamaño fijo
--------------------------------

``bytes1``, ``bytes2``, ``bytes3``, ..., ``bytes32``. ``byte`` es un alias para ``bytes1``.

Operadores:

* Comparaciones: ``<=``, ``<``, ``==``, ``!=``, ``>=``, ``>`` (evalúa a ``bool``)
* Operadores Bit: ``&``, ``|``, ``^`` (exclusivo bitwise or), ``~`` (negación bitwise), ``<<`` (shift izquierdo), ``>>`` (shift derecho)
* Acceso index: Si ``x`` es de tipo ``bytesI``, entonces ``x[k]`` para ``0 <= k < I`` devuelve el byte ``k`` (lectura sólo).

El operador shift funciona con cualquier entero como operador derecho (pero
devuelve el tipo del operador izquierdo, que denota el número de bits a desplazarse.
Desplazarse por un número negativo arroja una excepción runtime.

Miembros:

* ``.length`` devuelve el largo fijo del array byte (lectura sólo).

Array byte de tamaño dinámico
-----------------------------

``bytes``:
    Array byte de tamaño dinámico, ver :ref:`arrays`. No un tipo de valor!
``string``:
    Cadena de caracteres UTF-8-codificado de tamaño dinámico, ver :ref:`arrays`. No un tipo de valor!

Como regla general, usa ``bytes`` para data raw byte de tamaño arbitrario y ``string``
para una cadena de caracteres (UTF-8) de tamaño arbitrario. Si puedes limitar el tamaño a un cierto
número de bytes, siempre usa una de ``bytes1`` a ``bytes32`` porque son muchas mas baratas.

.. index:: ! ufixed, ! fixed, ! fixed point number

Números de punto fijo
---------------------

**PRÓXIMAMENTE...**

.. index:: address, literal;address

.. _address_literals:

Address LIterales
-----------------

Literales hexadecimales que pasan el test checksum, por ejemplo
``0xdCad3a6d3569DF655070DEd06cb7A1b2Ccd1D3AF`` es de tipo ``address``.
Literales hexaecimales que estan entre 39 y 41 dígitos de largo y
no pasan test de checksum producen una advetencia y son tratados como
números racionales literales regulares.

.. index:: literal, literal;rational

.. _rational_literals:

Literales racionales y enteros
------------------------------

Literales enteros son formados por una sequencia de números en el rango 0-9.
Son interpretados como decimales. Por ejemplo, ``69`` significa sesenta y nueve.
Literales octales no existen en Solidity y ceros a la izquierda son inválidos.

Literales de fracciones decimales son formados por un ``.`` con al menos un número en
un lado. Ejemplos incluyen ``1.``, ``.1`` y ``1.3``.

La notación científica está también soportada, donde la base puede tener fracciones, mientras el exponente no puede.
Ejemplos incluyen ``2e10``, ``-2e10``, ``2e-10``, ``2.5e1``.

Expresiones de números literales retienen precisión arbitraria hasta que son convertidas a un tipo no literal (ej. usándolas
juntas con una expresión no literal).
Esto significa que las computaciones no se desbordan y las divisiones no se truncan
en expresiones de números literales.

Por ejemplo, ``(2**800 + 1) - 2**800`` resulta en la constante ``1`` (de tipo ``uint8``)
aunque resultados intermedios ni siquiera serían del tamaño de la palabra. Además, ``.5 * 8`` resulta
en el entero ``4`` (aunque no enteros fueron usados entremedio).

Si el resultado no es un entero,
un tipo apropiado ``ufixed`` o ``fixed`` es usado del cual el número de bits fraccionales es tan grande
como se necesite (aproximando el número racional en el peor de los casos).

En ``var x = 1/4;``, ``x`` recibirá el tipo ``ufixed0x8`` mientras que en ``var x = 1/3`` recibirá
el tipo ``ufixed0x256`` porque ``1/3`` no es finitamente representable en binario y entonces será
aproximado.

Cualquier operador que puede ser aplicado a enteros también puede ser aplicado a una expresión de
número literal con tal que los operadores sea enteros. Si cualquiera de los dos es fraccional, las
operaciones de bit no son permitidas y la exponenciación no es permitida si el exponente es fraccional
(porque eso puede resultar en un número no racional).

.. note::
    Solidity tiene tipo literal de número prar cada número racional.
    Literales enteros y números racionales literales pertenecen a los tipos de numeros
    literales. Por otra parte, todos las expreciones litereales (ej. las epresiones que
    contienen sólo números literales y operadores) pertenecen a tipos de numeros literales.
    Entonces las expresiones de números literales  ``1 + 2`` y ``2 + 1`` ambas
    pertenecen al mismo tipo de numero literal para el número racional tres.

.. note::
    La mayoría de fracciones decimales finitas como ``5.3743`` no son finitamente representable en binario.
    El tipo correcto para ``5.3743`` es ``ufixed8x248`` porque permite la mejor aproximación del número. Si
    quieres usar el número junto con tipos como ``ufixed`` (ej. ``ufixed128x128``), tienes que explicitamente
    espcificar la precisión buscada: ``x + ufixed(5.3743)``.

.. warning::
    División en enteros literales usados para truncar en versiones anteriores, pero ahora se convertirá en un número racional, ej. ``5 / 2`` no es igual a ``1``, mas bien a ``2.5``.

.. note::
    Expresiones de números literales son convertidas en tipos no literales tan pronto como ellas son usadas con expresiones
    no literales. Aunque sabemos que el valor de la expresión
    asignada a ``b`` en el siguiente ejemplo evalúa a un entero, sigue usando
    tipos de punto fijo (y no numeros literales racionales) entremedio y entonces
    el código no compila.

::

    uint128 a = 1;
    uint128 b = 2.5 + a + 0.5;

.. index:: literal, literal;string, string

Cadena de carateres de literales
--------------------------------

String literals are written with either double or single-quotes (``"foo"`` or ``'bar'``).  They do not imply trailing zeroes as in C; ``"foo"`` represents three bytes not four.  As with integer literals, their type can vary, but they are implicitly convertible to ``bytes1``, ..., ``bytes32``, if they fit, to ``bytes`` and to ``string``.

String literals support escape characters, such as ``\n``, ``\xNN`` and ``\uNNNN``. ``\xNN`` takes a hex value and inserts the appropriate byte, while ``\uNNNN`` takes a Unicode codepoint and inserts an UTF-8 sequence.

.. index:: literal, bytes

Hexadecimal Literals
--------------------

Hexademical Literals are prefixed with the keyword ``hex`` and are enclosed in double or single-quotes (``hex"001122FF"``). Their content must be a hexadecimal string and their value will be the binary representation of those values.

Hexademical Literals behave like String Literals and have the same convertibility restrictions.

.. index:: enum

.. _enums:

Enums
-----

Enums are one way to create a user-defined type in Solidity. They are explicitly convertible
to and from all integer types but implicit conversion is not allowed.  The explicit conversions
check the value ranges at runtime and a failure causes an exception.  Enums needs at least one member.

::

    pragma solidity ^0.4.0;

    contract test {
        enum ActionChoices { GoLeft, GoRight, GoStraight, SitStill }
        ActionChoices choice;
        ActionChoices constant defaultChoice = ActionChoices.GoStraight;

        function setGoStraight() {
            choice = ActionChoices.GoStraight;
        }

        // Since enum types are not part of the ABI, the signature of "getChoice"
        // will automatically be changed to "getChoice() returns (uint8)"
        // for all matters external to Solidity. The integer type used is just
        // large enough to hold all enum values, i.e. if you have more values,
        // `uint16` will be used and so on.
        function getChoice() returns (ActionChoices) {
            return choice;
        }

        function getDefaultChoice() returns (uint) {
            return uint(defaultChoice);
        }
    }

.. index:: ! function type, ! type; function

.. _function_types:

Function Types
--------------

Function types are the types of functions. Variables of function type
can be assigned from functions and function parameters of function type
can be used to pass functions to and return functions from function calls.
Function types come in two flavours - *internal* and *external* functions:

Internal functions can only be used inside the current contract (more specifically,
inside the current code unit, which also includes internal library functions
and inherited functions) because they cannot be executed outside of the
context of the current contract. Calling an internal function is realized
by jumping to its entry label, just like when calling a function of the current
contract internally.

External functions consist of an address and a function signature and they can
be passed via and returned from external function calls.

Function types are notated as follows::

    function (<parameter types>) {internal|external} [constant] [payable] [returns (<return types>)]

In contrast to the parameter types, the return types cannot be empty - if the
function type should not return anything, the whole ``returns (<return types>)``
part has to be omitted.

By default, function types are internal, so the ``internal`` keyword can be
omitted.

There are two ways to access a function in the current contract: Either directly
by its name, ``f``, or using ``this.f``. The former will result in an internal
function, the latter in an external function.

If a function type variable is not initialized, calling it will result
in an exception. The same happens if you call a function after using ``delete``
on it.

If external function types are used outside of the context of Solidity,
they are treated as the ``function`` type, which encodes the address
followed by the function identifier together in a single ``bytes24`` type.

Note that public functions of the current contract can be used both as an
internal and as an external function. To use ``f`` as an internal function,
just use ``f``, if you want to use its external form, use ``this.f``.

Example that shows how to use internal function types::

    pragma solidity ^0.4.5;

    library ArrayUtils {
      // internal functions can be used in internal library functions because
      // they will be part of the same code context
      function map(uint[] memory self, function (uint) returns (uint) f)
        internal
        returns (uint[] memory r)
      {
        r = new uint[](self.length);
        for (uint i = 0; i < self.length; i++) {
          r[i] = f(self[i]);
        }
      }
      function reduce(
        uint[] memory self,
        function (uint x, uint y) returns (uint) f
      )
        internal
        returns (uint r)
      {
        r = self[0];
        for (uint i = 1; i < self.length; i++) {
          r = f(r, self[i]);
        }
      }
      function range(uint length) internal returns (uint[] memory r) {
        r = new uint[](length);
        for (uint i = 0; i < r.length; i++) {
          r[i] = i;
        }
      }
    }

    contract Pyramid {
      using ArrayUtils for *;
      function pyramid(uint l) returns (uint) {
        return ArrayUtils.range(l).map(square).reduce(sum);
      }
      function square(uint x) internal returns (uint) {
        return x * x;
      }
      function sum(uint x, uint y) internal returns (uint) {
        return x + y;
      }
    }

Another example that uses external function types::

    pragma solidity ^0.4.11;

    contract Oracle {
      struct Request {
        bytes data;
        function(bytes memory) external callback;
      }
      Request[] requests;
      event NewRequest(uint);
      function query(bytes data, function(bytes memory) external callback) {
        requests.push(Request(data, callback));
        NewRequest(requests.length - 1);
      }
      function reply(uint requestID, bytes response) {
        // Here goes the check that the reply comes from a trusted source
        requests[requestID].callback(response);
      }
    }

    contract OracleUser {
      Oracle constant oracle = Oracle(0x1234567); // known contract
      function buySomething() {
        oracle.query("USD", this.oracleResponse);
      }
      function oracleResponse(bytes response) {
        require(msg.sender == address(oracle));
        // Use the data
      }
    }

Note that lambda or inline functions are planned but not yet supported.

.. index:: ! type;reference, ! reference type, storage, memory, location, array, struct

Reference Types
==================

Complex types, i.e. types which do not always fit into 256 bits have to be handled
more carefully than the value-types we have already seen. Since copying
them can be quite expensive, we have to think about whether we want them to be
stored in **memory** (which is not persisting) or **storage** (where the state
variables are held).

Data location
-------------

Every complex type, i.e. *arrays* and *structs*, has an additional
annotation, the "data location", about whether it is stored in memory or in storage. Depending on the
context, there is always a default, but it can be overridden by appending
either ``storage`` or ``memory`` to the type. The default for function parameters (including return parameters) is ``memory``, the default for local variables is ``storage`` and the location is forced
to ``storage`` for state variables (obviously).

There is also a third data location, "calldata", which is a non-modifyable
non-persistent area where function arguments are stored. Function parameters
(not return parameters) of external functions are forced to "calldata" and
it behaves mostly like memory.

Data locations are important because they change how assignments behave:
Assignments between storage and memory and also to a state variable (even from other state variables)
always create an independent copy.
Assignments to local storage variables only assign a reference though, and
this reference always points to the state variable even if the latter is changed
in the meantime.
On the other hand, assignments from a memory stored reference type to another
memory-stored reference type does not create a copy.

::

    pragma solidity ^0.4.0;

    contract C {
        uint[] x; // the data location of x is storage

        // the data location of memoryArray is memory
        function f(uint[] memoryArray) {
            x = memoryArray; // works, copies the whole array to storage
            var y = x; // works, assigns a pointer, data location of y is storage
            y[7]; // fine, returns the 8th element
            y.length = 2; // fine, modifies x through y
            delete x; // fine, clears the array, also modifies y
            // The following does not work; it would need to create a new temporary /
            // unnamed array in storage, but storage is "statically" allocated:
            // y = memoryArray;
            // This does not work either, since it would "reset" the pointer, but there
            // is no sensible location it could point to.
            // delete y;
            g(x); // calls g, handing over a reference to x
            h(x); // calls h and creates an independent, temporary copy in memory
        }

        function g(uint[] storage storageArray) internal {}
        function h(uint[] memoryArray) {}
    }

Summary
^^^^^^^

Forced data location:
 - parameters (not return) of external functions: calldata
 - state variables: storage

Default data location:
 - parameters (also return) of functions: memory
 - all other local variables: storage

.. index:: ! array

.. _arrays:

Arrays
------

Arrays can have a compile-time fixed size or they can be dynamic.
For storage arrays, the element type can be arbitrary (i.e. also other
arrays, mappings or structs). For memory arrays, it cannot be a mapping and
has to be an ABI type if it is an argument of a publicly-visible function.

An array of fixed size ``k`` and element type ``T`` is written as ``T[k]``,
an array of dynamic size as ``T[]``. As an example, an array of 5 dynamic
arrays of ``uint`` is ``uint[][5]`` (note that the notation is reversed when
compared to some other languages). To access the second uint in the
third dynamic array, you use ``x[2][1]`` (indices are zero-based and
access works in the opposite way of the declaration, i.e. ``x[2]``
shaves off one level in the type from the right).

Variables of type ``bytes`` and ``string`` are special arrays. A ``bytes`` is similar to ``byte[]``,
but it is packed tightly in calldata. ``string`` is equal to ``bytes`` but does not allow
length or index access (for now).

So ``bytes`` should always be preferred over ``byte[]`` because it is cheaper.

.. note::
    If you want to access the byte-representation of a string ``s``, use
    ``bytes(s).length`` / ``bytes(s)[7] = 'x';``. Keep in mind
    that you are accessing the low-level bytes of the UTF-8 representation,
    and not the individual characters!

It is possible to mark arrays ``public`` and have Solidity create a getter.
The numeric index will become a required parameter for the getter.

.. index:: ! array;allocating, new

Allocating Memory Arrays
^^^^^^^^^^^^^^^^^^^^^^^^

Creating arrays with variable length in memory can be done using the ``new`` keyword.
As opposed to storage arrays, it is **not** possible to resize memory arrays by assigning to
the ``.length`` member.

::

    pragma solidity ^0.4.0;

    contract C {
        function f(uint len) {
            uint[] memory a = new uint[](7);
            bytes memory b = new bytes(len);
            // Here we have a.length == 7 and b.length == len
            a[6] = 8;
        }
    }

.. index:: ! array;literals, !inline;arrays

Array Literals / Inline Arrays
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Array literals are arrays that are written as an expression and are not
assigned to a variable right away.

::

    pragma solidity ^0.4.0;

    contract C {
        function f() {
            g([uint(1), 2, 3]);
        }
        function g(uint[3] _data) {
            // ...
        }
    }

The type of an array literal is a memory array of fixed size whose base
type is the common type of the given elements. The type of ``[1, 2, 3]`` is
``uint8[3] memory``, because the type of each of these constants is ``uint8``.
Because of that, it was necessary to convert the first element in the example
above to ``uint``. Note that currently, fixed size memory arrays cannot
be assigned to dynamically-sized memory arrays, i.e. the following is not
possible:

::

    pragma solidity ^0.4.0;

    contract C {
        function f() {
            // The next line creates a type error because uint[3] memory
            // cannot be converted to uint[] memory.
            uint[] x = [uint(1), 3, 4];
    }

It is planned to remove this restriction in the future but currently creates
some complications because of how arrays are passed in the ABI.

.. index:: ! array;length, length, push, !array;push

Members
^^^^^^^

**length**:
    Arrays have a ``length`` member to hold their number of elements.
    Dynamic arrays can be resized in storage (not in memory) by changing the
    ``.length`` member. This does not happen automatically when attempting to access elements outside the current length. The size of memory arrays is fixed (but dynamic, i.e. it can depend on runtime parameters) once they are created.
**push**:
     Dynamic storage arrays and ``bytes`` (not ``string``) have a member function called ``push`` that can be used to append an element at the end of the array. The function returns the new length.

.. warning::
    It is not yet possible to use arrays of arrays in external functions.

.. warning::
    Due to limitations of the EVM, it is not possible to return
    dynamic content from external function calls. The function ``f`` in
    ``contract C { function f() returns (uint[]) { ... } }`` will return
    something if called from web3.js, but not if called from Solidity.

    The only workaround for now is to use large statically-sized arrays.


::

    pragma solidity ^0.4.0;

    contract ArrayContract {
        uint[2**20] m_aLotOfIntegers;
        // Note that the following is not a pair of dynamic arrays but a
        // dynamic array of pairs (i.e. of fixed size arrays of length two).
        bool[2][] m_pairsOfFlags;
        // newPairs is stored in memory - the default for function arguments

        function setAllFlagPairs(bool[2][] newPairs) {
            // assignment to a storage array replaces the complete array
            m_pairsOfFlags = newPairs;
        }

        function setFlagPair(uint index, bool flagA, bool flagB) {
            // access to a non-existing index will throw an exception
            m_pairsOfFlags[index][0] = flagA;
            m_pairsOfFlags[index][1] = flagB;
        }

        function changeFlagArraySize(uint newSize) {
            // if the new size is smaller, removed array elements will be cleared
            m_pairsOfFlags.length = newSize;
        }

        function clear() {
            // these clear the arrays completely
            delete m_pairsOfFlags;
            delete m_aLotOfIntegers;
            // identical effect here
            m_pairsOfFlags.length = 0;
        }

        bytes m_byteData;

        function byteArrays(bytes data) {
            // byte arrays ("bytes") are different as they are stored without padding,
            // but can be treated identical to "uint8[]"
            m_byteData = data;
            m_byteData.length += 7;
            m_byteData[3] = 8;
            delete m_byteData[2];
        }

        function addFlag(bool[2] flag) returns (uint) {
            return m_pairsOfFlags.push(flag);
        }

        function createMemoryArray(uint size) returns (bytes) {
            // Dynamic memory arrays are created using `new`:
            uint[2][] memory arrayOfPairs = new uint[2][](size);
            // Create a dynamic byte array:
            bytes memory b = new bytes(200);
            for (uint i = 0; i < b.length; i++)
                b[i] = byte(i);
            return b;
        }
    }


.. index:: ! struct, ! type;struct

.. _structs:

Structs
-------

Solidity provides a way to define new types in the form of structs, which is
shown in the following example:

::

    pragma solidity ^0.4.11;

    contract CrowdFunding {
        // Defines a new type with two fields.
        struct Funder {
            address addr;
            uint amount;
        }

        struct Campaign {
            address beneficiary;
            uint fundingGoal;
            uint numFunders;
            uint amount;
            mapping (uint => Funder) funders;
        }

        uint numCampaigns;
        mapping (uint => Campaign) campaigns;

        function newCampaign(address beneficiary, uint goal) returns (uint campaignID) {
            campaignID = numCampaigns++; // campaignID is return variable
            // Creates new struct and saves in storage. We leave out the mapping type.
            campaigns[campaignID] = Campaign(beneficiary, goal, 0, 0);
        }

        function contribute(uint campaignID) payable {
            Campaign c = campaigns[campaignID];
            // Creates a new temporary memory struct, initialised with the given values
            // and copies it over to storage.
            // Note that you can also use Funder(msg.sender, msg.value) to initialise.
            c.funders[c.numFunders++] = Funder({addr: msg.sender, amount: msg.value});
            c.amount += msg.value;
        }

        function checkGoalReached(uint campaignID) returns (bool reached) {
            Campaign c = campaigns[campaignID];
            if (c.amount < c.fundingGoal)
                return false;
            uint amount = c.amount;
            c.amount = 0;
            c.beneficiary.transfer(amount);
            return true;
        }
    }

The contract does not provide the full functionality of a crowdfunding
contract, but it contains the basic concepts necessary to understand structs.
Struct types can be used inside mappings and arrays and they can itself
contain mappings and arrays.

It is not possible for a struct to contain a member of its own type,
although the struct itself can be the value type of a mapping member.
This restriction is necessary, as the size of the struct has to be finite.

Note how in all the functions, a struct type is assigned to a local variable
(of the default storage data location).
This does not copy the struct but only stores a reference so that assignments to
members of the local variable actually write to the state.

Of course, you can also directly access the members of the struct without
assigning it to a local variable, as in
``campaigns[campaignID].amount = 0``.

.. index:: !mapping

Mappings
========

Mapping types are declared as ``mapping(_KeyType => _ValueType)``.
Here ``_KeyType`` can be almost any type except for a mapping, a dynamically sized array, a contract, an enum and a struct.
``_ValueType`` can actually be any type, including mappings.

Mappings can be seen as `hash tables <https://en.wikipedia.org/wiki/Hash_table>`_ which are virtually initialized such that
every possible key exists and is mapped to a value whose byte-representation is
all zeros: a type's :ref:`default value <default-value>`. The similarity ends here, though: The key data is not actually stored
in a mapping, only its ``keccak256`` hash used to look up the value.

Because of this, mappings do not have a length or a concept of a key or value being "set".

Mappings are only allowed for state variables (or as storage reference types
in internal functions).

It is possible to mark mappings ``public`` and have Solidity create a getter.
The ``_KeyType`` will become a required parameter for the getter and it will
return ``_ValueType``.

The ``_ValueType`` can be a mapping too. The getter will have one parameter
for each ``_KeyType``, recursively.

::

    pragma solidity ^0.4.0;

    contract MappingExample {
        mapping(address => uint) public balances;

        function update(uint newBalance) {
            balances[msg.sender] = newBalance;
        }
    }

    contract MappingUser {
        function f() returns (uint) {
            return MappingExample(<address>).balances(this);
        }
    }


.. note::
  Mappings are not iterable, but it is possible to implement a data structure on top of them.
  For an example, see `iterable mapping <https://github.com/ethereum/dapp-bin/blob/master/library/iterable_mapping.sol>`_.

.. index:: assignment, ! delete, lvalue

Operators Involving LValues
===========================

If ``a`` is an LValue (i.e. a variable or something that can be assigned to), the following operators are available as shorthands:

``a += e`` is equivalent to ``a = a + e``. The operators ``-=``, ``*=``, ``/=``, ``%=``, ``a |=``, ``&=`` and ``^=`` are defined accordingly. ``a++`` and ``a--`` are equivalent to ``a += 1`` / ``a -= 1`` but the expression itself still has the previous value of ``a``. In contrast, ``--a`` and ``++a`` have the same effect on ``a`` but return the value after the change.

delete
------

``delete a`` assigns the initial value for the type to ``a``. I.e. for integers it is equivalent to ``a = 0``, but it can also be used on arrays, where it assigns a dynamic array of length zero or a static array of the same length with all elements reset. For structs, it assigns a struct with all members reset.

``delete`` has no effect on whole mappings (as the keys of mappings may be arbitrary and are generally unknown). So if you delete a struct, it will reset all members that are not mappings and also recurse into the members unless they are mappings. However, individual keys and what they map to can be deleted.

It is important to note that ``delete a`` really behaves like an assignment to ``a``, i.e. it stores a new object in ``a``.

::

    pragma solidity ^0.4.0;

    contract DeleteExample {
        uint data;
        uint[] dataArray;

        function f() {
            uint x = data;
            delete x; // sets x to 0, does not affect data
            delete data; // sets data to 0, does not affect x which still holds a copy
            uint[] y = dataArray;
            delete dataArray; // this sets dataArray.length to zero, but as uint[] is a complex object, also
            // y is affected which is an alias to the storage object
            // On the other hand: "delete y" is not valid, as assignments to local variables
            // referencing storage objects can only be made from existing storage objects.
        }
    }

.. index:: ! type;conversion, ! cast

Conversions between Elementary Types
====================================

Implicit Conversions
--------------------

If an operator is applied to different types, the compiler tries to
implicitly convert one of the operands to the type of the other (the same is
true for assignments). In general, an implicit conversion between value-types
is possible if it
makes sense semantically and no information is lost: ``uint8`` is convertible to
``uint16`` and ``int128`` to ``int256``, but ``int8`` is not convertible to ``uint256``
(because ``uint256`` cannot hold e.g. ``-1``).
Furthermore, unsigned integers can be converted to bytes of the same or larger
size, but not vice-versa. Any type that can be converted to ``uint160`` can also
be converted to ``address``.

Explicit Conversions
--------------------

If the compiler does not allow implicit conversion but you know what you are
doing, an explicit type conversion is sometimes possible. Note that this may
give you some unexpected behaviour so be sure to test to ensure that the
result is what you want! Take the following example where you are converting
a negative ``int8`` to a ``uint``:

::

    int8 y = -3;
    uint x = uint(y);

At the end of this code snippet, ``x`` will have the value ``0xfffff..fd`` (64 hex
characters), which is -3 in the two's complement representation of 256 bits.

If a type is explicitly converted to a smaller type, higher-order bits are
cut off::

    uint32 a = 0x12345678;
    uint16 b = uint16(a); // b will be 0x5678 now

.. index:: ! type;deduction, ! var

.. _type-deduction:

Type Deduction
==============

For convenience, it is not always necessary to explicitly specify the type of a
variable, the compiler automatically infers it from the type of the first
expression that is assigned to the variable::

    uint24 x = 0x123;
    var y = x;

Here, the type of ``y`` will be ``uint24``. Using ``var`` is not possible for function
parameters or return parameters.

.. warning::
    The type is only deduced from the first assignment, so
    the loop in the following snippet is infinite, as ``i`` will have the type
    ``uint8`` and any value of this type is smaller than ``2000``.
    ``for (var i = 0; i < 2000; i++) { ... }``
