s.. index:: type

.. _types:

*****
Tipos
*****

Solidity es un lenguaje de tipado estático, que significa que cada tipo de
variable (estado y local) tiene ser especificada (o al menos conocida -
ver :ref:`type-deduction` abajo) en tiempo de compilación.
Solidity proporciona varios tipos elementales que pueden ser combinados para
crear tipos más complejos.


Además de eso, los tipos pueden interactuar el uno con el otro en expresiones
conteniendo operadores. Para una lista rápida de referencia de los operadores,
ver :ref:`order`.

.. index:: ! value type, ! type;value

Tipos de Valor
==============

Los siguientes tipos también son llamados tipos de valor porque las variables
de este tipo serán siempre pasadas como valores, ej. siempre serán copiados cuando
son usados como argumentos de funciónes o en asignaciones.

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

``int`` / ``uint``: Enteros con y sin signo de varios tamaños. Las palabras clave ``uint8`` a ``uint256`` en pasos de ``8`` (sin signo de 8 hasta 256 bits) y ``int8`` a ``int256``. ``uint`` y ``int`` son aliases para ``uint256`` y ``int256``, respectivamente.

Operadores:

* Comparaciones: ``<=``, ``<``, ``==``, ``!=``, ``>=``, ``>`` (evalúa a ``bool``)
* Operadores bit: ``&``, ``|``, ``^`` (bitwise exclusivo or), ``~`` (negación bitwise)
* Operadores aritméticos: ``+``, ``-``, unary ``-``, unary ``+``, ``*``, ``/``, ``%`` (restante), ``**`` (exponenciales), ``<<`` (shift izquierda), ``>>`` (shift derecha)

La división siempre trunca (está compilado a la opcode DIV de la EVM), pero no trunca si los dos
operadores son :ref:`literales<rational_literals>` (o expresiones literales).

División por cero y modulus con cero arrojan una excepción runtime.

El resultado de una operación shift es el tipo de operador izquierdo. La
expresión ``x << y`` es equivalente a ``x * 2**y`` y ``x >> y`` es
equivalente a ``x / 2**y``. Esto significa que hacer un shift de números negativos
extiende en signo. Haciendo shift por un número negativo arroja una excepción runtime.

.. warning::
    Los resultados producidos por shift derecho de valores negativos de tipos de enteros con signo es diferente de esos producidos
    por otros lenguajes de programación. En Solidity, shift derecho mapea la división para que los valores negativos de shift
    serán redondeados hacia cero (truncado). En otros lenguajes de programación el shift derecho de valores negativos
    funcióna como una división con redondeo hacia abajo (hacia infinito negativo).

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
    hacer transferencia de Ether seguras, siempre revisar el valor devuelto por ``send``, usar ``transfer`` o incluso mejor:
    usa el patrón donde el recipiente retira el dinero.

* ``call``, ``callcode`` and ``delegatecall``

Además, para interfazar con contratos que no adhieren al ABI,
la función ``call`` es prevista que toma un número arbitrario de argumentos de cualquier tipo. Estos argumentos son acolchados a 32 bytes y concatenados. Una excepción es el caso donde el primer argumento es codificado a exactamente 4 bytes. En este caso, no está acolchado para permitir el uso de firmas de función aquí.

::

    address nameReg = 0x72ba7d8e73fe8eb666ea66babc8116a41bfb10e2;
    nameReg.call("register", "MyName");
    nameReg.call(bytes4(keccak256("fun(uint256)")), a);

``call`` devuelve un booleano indicando si la función llamada terminó (``true``) o causó una excepción del EVM (``false``). No es posible acceder a los datos reales devueltos (para esto necesitaremos saber el tamaño de codificación en avance).

En una forma similar, ``delegatecall`` puede ser usado: La diferencia es que solo el código de la dirección dada es usado, todo otros aspectos (almacenamiento, saldo, ...) salen del contrato actual. El propósito de ``delegatecall`` es usar el código de librería que está almacenado en otro contrato. El usuario tiene que asegurarse que el layout del almacenamiento en ambos contratos es correcto para usar delegatecall. Antes de homestead, sólo una versión limitada llamada ``callcode`` estaba disponible que no daba acceso a los valores ``msg.sender`` y ``msg.value`` originales.

Las tres funciónes ``call``, ``delegatecall`` y ``callcode`` son funciónes de muy bajo nivel y deben usarse sólo como medida de último recurso ya que rompen la seguridad de tipo de Solidity.

La opción ``.gas()`` está disponible en los 3 métodos, mientras la opción ``.value()`` no se admite para ``delegatecall``.

.. note::
    Todos los contratos heredan los miembros de address, así que es posible consultar el saldo del contrato actual
    usando ``this.balance``.

.. warning::
    Todas estas funciónes son funciónes de bajo nivel y debe usarse con cuidado.
    Específicamente, cualquier contrato desconocido puede ser malicioso y si se le llama,
    se le da el control a ese contrato que puede, luego llamar de vuelta a tu contrato,
    así que prepárense para cambios a tus variables de estado cuando el llamado retorna.

.. index:: byte array, bytes32


Colleción de byte de tamaño fijo
--------------------------------

``bytes1``, ``bytes2``, ``bytes3``, ..., ``bytes32``. ``byte`` es un alias para ``bytes1``.

Operadores:

* Comparaciones: ``<=``, ``<``, ``==``, ``!=``, ``>=``, ``>`` (evalúa a ``bool``)
* Operadores Bit: ``&``, ``|``, ``^`` (exclusivo bitwise or), ``~`` (negación bitwise), ``<<`` (shift izquierdo), ``>>`` (shift derecho)
* Acceso index: Si ``x`` es de tipo ``bytesI``, entonces ``x[k]`` para ``0 <= k < I`` devuelve el byte ``k`` (lectura sólo).

El operador shift funcióna con cualquier entero como operador derecho (pero
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
número de bytes, siempre usa una de ``bytes1`` a ``bytes32`` porque son muchas más baratas.

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
Literales hexadecimales que están entre 39 y 41 dígitos de largo y
no pasan test de checksum producen una advertencia y son tratados como
números racionales literales regulares.

.. index:: literal, literal;rational

.. _rational_literals:

Literales racionales y enteros
------------------------------

Literales enteros son formados por una secuencia de números en el rango 0-9.
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
    Solidity tiene tipo literal de número para cada número racional.
    Literales enteros y números racionales literales pertenecen a los tipos de números
    literales. Por otra parte, todos las expresiones literales (ej. las expresiones que
    contienen sólo números literales y operadores) pertenecen a tipos de números literales.
    Entonces las expresiones de números literales  ``1 + 2`` y ``2 + 1`` ambas
    pertenecen al mismo tipo de número literal para el número racional tres.

.. note::
    La mayoría de fracciones decimales finitas como ``5.3743`` no son finitamente representable en binario.
    El tipo correcto para ``5.3743`` es ``ufixed8x248`` porque permite la mejor aproximación del número. Si
    quieres usar el número junto con tipos como ``ufixed`` (ej. ``ufixed128x128``), tienes que explícitamente
    especificar la precisión buscada: ``x + ufixed(5.3743)``.

.. warning::
    División en enteros literales usados para truncar en versiones anteriores, pero ahora se convertirá en un número racional, ej. ``5 / 2`` no es igual a ``1``, más bien a ``2.5``.

.. note::
    Expresiones de números literales son convertidas en tipos no literales tan pronto como ellas son usadas con expresiones
    no literales. Aunque sabemos que el valor de la expresión
    asignada a ``b`` en el siguiente ejemplo evalúa a un entero, sigue usando
    tipos de punto fijo (y no números literales racionales) entremedio y entonces
    el código no compila.

::

    uint128 a = 1;
    uint128 b = 2.5 + a + 0.5;

.. index:: literal, literal;string, string

Literales cadenas
-----------------

Las cadenas literales son cerrados con comillas simples o dobles (``"foo"`` or ``'bar'``). No hay ceros implícitos como en C; ``"foo"`` representa tres bytes, no cuatro. Como con lietrales enteros, su tpo puede variar, pero son implícitamente convertibles a ``bytes1``, ..., ``bytes32``, si caben a ``bytes`` y a ``string``.


Las cadenas literales soportan caracteres de escape, tales como ``\n``, ``\xNN`` y ``\uNNNN``. ``\xNN`` toma un valor e inserta el byte apropiado, mientras que ``\uNNNN`` toma un codepoint Unicode e inserta una secuencia UTF-8.


.. index:: literal, bytes


Literales hexadecimales
-----------------------

Los literales hexadecimales son prefijos con la palabra clave ``hex`` y son cerrados por comillas simples o dobles (``hex"001122FF"``). Su contenido debe ser una cadena hexadecimal y su valor será la representación binaria de esos valores.

Los literales hexadecimales se comportan como los literales de cadena y tienen los mismas restricciones de convertibilidad.


.. index:: enum

.. _enums:

Enums
-----

Enums son una manera de hacer tipos creados por usuario en Solidity. Son explícitamente convertibles
a y desde todo tipos de enteros pero la conversión implícita no se permite. Las conversiones explícitas
revisan los valores de rangos en runtime y una falla causa una excepción. Enums necesitan al menos un miembro.

::

    pragma solidity ^0.4.0;

    contract test {
        enum ActionChoices { GoLeft, GoRight, GoStraight, SitStill }
        ActionChoices choice;
        ActionChoices constant defaultChoice = ActionChoices.GoStraight;

        function setGoStraight() {
            choice = ActionChoices.GoStraight;
        }

        // Ya que los tipos enum no son parte del ABI, la firma de "getChoice"
        // automáticamente será cambiada a "getChoice() returns (unit8)"
        // para todo lo externo a Solidity. El tipo entero usado es apenas
        // suficientemente grande para guardar todos los valores enum, ej. si
        // tienes más valores, `unit16` será utilizado y así.
        function getChoice() returns (ActionChoices) {
            return choice;
        }

        function getDefaultChoice() returns (uint) {
            return uint(defaultChoice);
        }
    }

.. index:: ! function type, ! type; function

.. _function_types:

Función
-------

Los tipos función son tipos de función. Variables de tipo función
pueden ser asignados desde funciónes y parámetros de funciónes de tipo función
pueden ser usadas para pasar funciónes y retornar funciónes de llamados de funciónes.
Los tipos de función hay de dos tipos - *internas* y *externas*:

Las funciónes internas sólo pueden ser usadas dentro del contrato actual (específicamente,
dentro de la unidad de code actual, que también incluye funciónes librerías internas
y funciónes heredadas) porque no pueden ser ejecutadas fuera del
contexto del contrato actual. Llamando una función interna se realiza
saltando a su label de entrada, tal como cuando se llama una función interna del
contrato actual.

funciónes externas están compuestas de una dirección y una firma de función y pueden
ser pasadas y devueltas desde una llamada de función externa.

Los tipos de funciónes son notadas como sigue::

    function (<parameter types>) {internal|external} [constant] [payable] [returns (<return types>)]

En contraste a los tipos de parámetros, los tipos de retorno no pueden estar vacíos - si
el tipo función no debe retornar nada, la parte ``returns (<return types>)``
tiene que ser omitida.

Por defecto, las funciónes son de tipo interna, así que la palabra clave ``internal``
puede ser omitida.

Hay dos formas de acceder una función en el contrato actual: o bien directamente
con su nombre, ``f``, o usando ``this.f``. Usando el nombre resultará en una función
interna, y con ``this`` habrá una función externa.

Si una variable de tipo función no es inicializada, llamarla resultará
resultar en una excepción. Lo mismo ocurre si llamas una función después de usar
``delete`` en ella.

Si funciónes externas son usadas fuera del contexto de Solidity, son tratadas
como tipo ``function``, que codifica la dirección seguida por el identificador
de la función junto con un tipo ``bytes24``.

Nótese que las funciónes públicas del contrato actual pueden ser usado tanto
como una función interna y externa. Para usar ``f`` como función interna, sólo
se le llama como ``f``, y si se quiere usar como externa, usar ``this.f``.


Ejemplo que muestra como usar tipos de función internas::

    pragma solidity ^0.4.5;

    library ArrayUtils {
      // las funciónes internas pueden ser usadas en funciónes de librerías
      // internas porque serán parte del mismo contexto de código
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

Otro ejemplo que usa tipos de función externa::

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
        // Aquí se revisa que el respuesta viene de una fuente de confianza
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
        // Usar los datos
      }
    }

Notar que los lambda o funciónes inline están planeadas pero no están aún implementados.

.. index:: ! type;reference, ! reference type, storage, memory, location, array, struct

Tipos de Referencia
===================

Tipos complejos, ej. tipos que no siempre caben en 256 bits tienen que ser manejadas
cn más cuidado que los tipos de valores que ya hemos visto. Ya que copiarlas puede
ser muy caro, tenemos que pensar sobre si queremos que se almacenen en **memory**
(que no es persistente) o en **storage** (donde las variables de estado se guardan).

Ubicación de datos
------------------

Cada tipo complejo, ej. *arrays* y *structs*, tienen anotaciones
adicionales, la "data location", con respecto a si es almacenado
en memoria o en almacenamiento. Dependiendo del contexto, siempre hay un
valor por defecto, pero puede ser remplazada añadiendo o bien
``storage`` o `memory`` al tipo. Por defecto para tipos parámetros de
función (incluyendo parámetros de retorno) es ``memory``, por defecto para
variables locales es ``storage`` y la ubicación es forzada a ``storage``
para variables de estado (obviamente).

Hay una tercera ubicación de datos, "calldata", un área que no es modificable
y no persistente donde argumentos de función son almacenados. Parámetros de función
(no parámetros de retorno) de funciónes externas son forzados a "calldata" y
se comporta casi como memoria.

Las ubicaciones de datos son importantes porque cambian cómo las asignaciones se comportan:
Las asignaciones entre almacenamiento y memoria y también de variables de estado (incluso desde otras
variable de estado) siempre crean una copia independiente.
Asignaciones a almacenamiento variable de almacenamiento local sólo asignan una referencia, y
esta referencia siempre apunta a la variable de estado aunque la referencia cambie
entretanto.
En cambio, asignaciones de la referencia almacenada en memoria a otro tipo de referencia
no crea una copia.

::

    pragma solidity ^0.4.0;

    contract C {
        uint[] x; // the data location of x is storage

        // la ubicacion de datos de memoryArray es memory
        function f(uint[] memoryArray) {
            x = memoryArray; // funcióna, copia el array entero al almacenamiento
            var y = x; // funcióna, asigna una referencia, ubicación de datos de y es almacenamiento
            y[7]; // bien, devuelve el octavo elemento
            y.length = 2; // bien, modifica de x a y
            delete x; // bien, limpia el array, también modifica y
            // Lo siguiente no funcióna; debería crear un nuevo temporal/sin nombre
            // array en almacenamiento, pero almacenamiento es asignado "estáticamente":
            // y = memoryArray;
            // Esto no funcióna tampoco, ya que resetearía el apuntador, pero no hay
            // ubicación donde podría apuntar
            // borrar y;
            g(x); // llama g, dando referencia a x
            h(x); // llama h y y crea una copia independiente y temporal en la memoria
        }

        function g(uint[] storage storageArray) internal {}
        function h(uint[] memoryArray) {}
    }


Resumen
^^^^^^^

Ubicación de datos forzada:
 - parámetros (no de retorno) de funciónes externas: calldata
 - variables de estado: almacenamiento

Ubicación de datos por defecto:
 - parámetros (también de retorno) de funciónes: memoria
 - todas otras variables: almacenamiento

.. index:: ! array

.. _arrays:

Arrays
------

Los array pueden tener tamaño fijo en compilación o pueden ser dinámicos.
Para arrays de almacenamiento, el tipo elemento puede ser arbitrario (ej. también
otros arrays, mapeos o structs). Para arrays de memoria, no puede ser un mapping
tiene que ser un tipo ABI si es que es un argumento de una función públicamente
visible.

Un array de tamaño fijo ``k`` y elemento tipo ``T`` es escrito como ``T[k]``,
un array de tamaño dinámico como ``T[]``. Como ejemplo, un array de 5 arrays
dinámicos de ``uint`` es ``uint[][]`` (nótese que la notación es invertida
cuando comparada a otros lenguajes). Para acceder la segunda uint en el tercer
array dinámico, se utiliza ``x[2][1]`` ()

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

El tipo de array literal es un array de memoria de tamaño fijo de la cual el tipo
base es el tipo común de los elementos dados. El tipo de ``[1, 2, 3]`` es
``uint[3] memory``, porque el tipo de cada de estas constantes es ``uint8``.
Por eso, fue necesario convertir el primer elemento en el ejemplo arriba
a ``uint``. Nótese que actualmente, array de memoria de tamaño fijo no pueden
ser asignados a arrays de memoria de tamaño dinámico, ej. lo siguiente
no es posible:

::

    pragma solidity ^0.4.0;

    contract C {
        function f() {
            // La próxima línea crea un tipo error porque uint[3] memory
            // no puede ser convertido a uint[] memory.
            uint[] x = [uint(1), 3, 4];
    }

Esta restricción está planeada para ser eliminada en el futuro pero actualmente
crea complicaciones por cómo los arrays son pasados en el ABI.

.. index:: ! array;length, length, push, !array;push

Miembros
^^^^^^^^

**length**:
    Arrays tienen un miembro ``length`` para guardar su número de elementos.
    Arrays dinámicos pueden ser modificados en almacenamiento (no en memoria) cambiando
    el miembro ``.length``. Ésto no ocurre automáticamente cuando se intenta acceder los elementos fuera del length actual. El tamaño de arrays de memoria es fijo (pero dinámico, ej. puede depender de parámetros runtime) cuando son creados.
**push**:
    Arrays de almacenamiento dinámico y ``bytes`` (no ``string``) tienen una función miembro llamada ``push`` que puede ser usada para agregar un elemento al final del array. La función devuelve el nuevo length.

.. warning::
    Aún no es posible usar arrays en funciónes externas.

.. warning::
    Dado a las limitaciones de la EVM, no es posible retornar
    contenido dinámico de las funciónes externas . La función ``f`` en
    ``contract C { function f() returns (uint[]) { ... } }`` devolverá
    algo si es llamdo del web3.js, pero no si se llama desde Solidity.

    La única alternativa por ahora es usar grandes arrays de tamaño estático.


::

    pragma solidity ^0.4.0;

    contract ArrayContract {
        uint[2**20] m_aLotOfIntegers;
        // Nótese que el siguiente no es un par de arrays dinámicos, sino
        // array dinámico de pares (ej. de arrays de tamaño fijo de length 2).
        bool[2][] m_pairsOfFlags;
        // newPairs es almacenado en memoria - el defecto para argumentos de función

        function setAllFlagPairs(bool[2][] newPairs) {
            // asignación a un array de almacenamiento reemplaza el array completo
            m_pairsOfFlags = newPairs;
        }

        function setFlagPair(uint index, bool flagA, bool flagB) {
            // acceso a un index que no existe arrojará una excepción
            m_pairsOfFlags[index][0] = flagA;
            m_pairsOfFlags[index][1] = flagB;
        }

        function changeFlagArraySize(uint newSize) {
            // si el tamaño nuevo es más pequeño, los elementos eliminados del array serán limpiados
            m_pairsOfFlags.length = newSize;
        }

        function clear() {
            // éstos limpian los arrays completamente
            delete m_pairsOfFlags;
            delete m_aLotOfIntegers;
            // efecto idéntico aquí
            m_pairsOfFlags.length = 0;
        }

        bytes m_byteData;

        function byteArrays(bytes data) {
            // byte arrays ("bytes") son diferentes ya que no son almacenados sin padding,
            // pero pueden tratados idénticamente a "uint8[]"
            m_byteData = data;
            m_byteData.length += 7;
            m_byteData[3] = 8;
            delete m_byteData[2];
        }

        function addFlag(bool[2] flag) returns (uint) {
            return m_pairsOfFlags.push(flag);
        }

        function createMemoryArray(uint size) returns (bytes) {
            // Arrays de memoria dinámicos son creados usando `new`:
            uint[2][] memory arrayOfPairs = new uint[2][](size);
            // Crear un byte array dinámico:
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

Solidity provee una manera de definir nuevos tipos con structs, que es
mostrado en el siguiente ejemplo:

::

    pragma solidity ^0.4.11;

    contract CrowdFunding {
        // Define un nuevo tipo con dos campos.
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
            campaignID = numCampaigns++; // campaignID es variable de retorno
            // Crea un nuevo sruct y guarda en almacenamiento. Dejamos fuera el tipo mapping.
            campaigns[campaignID] = Campaign(beneficiary, goal, 0, 0);
        }

        function contribute(uint campaignID) payable {
            Campaign c = campaigns[campaignID];
            // Crea un nuevo struct de memoria temporal, inicializado con los valores dados
            // y lo copia al almacenamiento.
            // Nótese que también se puede usar Funder(msg.sender, msg.value) para inicializar
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

El contrato no provee funciónalidad total de un contrato crowdfunding,
peor contiene los conceptos básicos necesarios para entender structs.
Tipos structs pueden ser usados dentro de mappings y arrays y pueden ellos
mismos, contener mappings y arrays.

No es posible para un struct de contener un miembro de su propio tipo,
aunque el struct puede ser el tipo valor de un miembro mapping.
Esta restricción es necesaria, ya que el tamaño del struct tiene que ser finito.

Nótese como en todas las funciónes, un tipo struct es asignado a la variable local
(de la ubicación por defecto del almacenamiento).
Esto no copia el struct pero guarda una referencia para que las asignaciones
a miembros de la variable local realmente escriban al estado.

Por supuesto, puedes diréctamente acceder los miembros del struct sin
asignarlos a la variable local, como en
``campaigns[campaignID].amount = 0``.

.. index:: !mapping

Mappings
========

Tipos mapping son declarados como ``mapping(_KeyType => _ValueType)``.
Aquí ``_KeyType`` puede ser casi cualquier tipo excepto por mapping, un array de tamaño dinámico, un contrato, un enum y un struct.
``_ValueType`` puede ser cualquier tipo, incluyendo mappings.

Mappings pueden verse como 'has tables <https://en.wikipedia.org/wiki/Hash_table>'_ que son virtualmente inicializadas ya que
cada posible clase existe y es mapeada a un valor que su representación byte es
todo ceros: el valor :ref:`por defecto <default-value> de un tipo. Aunque la similitud termina aquí: los datos clave no son realmente
almacenados en el mapping, sólo su hash ``keccak256`` usado para buscar el valor.

Por esto, mappings no tienen un length o un concepto de "fijar" clave o valor.

Mappings sólo son permitidas para variables de estado (o como tipos de referencia
en funciónes internas).

Es posible marcar los mappings ``public`` y hacer que Solidity cree un getter.
El ``_KeyType`` será un parámetro requerido par el getter y devolverá ``_ValueType``.

El ``_ValueType`` puede ser un mapping también. El getter tendrá un parámetro
para cada ``_KeyType``, recursivamente.

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
  Los mappings no son iterables, pero es posible implementar una estructura de datos encima de ellos.
  Por ejemplo, ver `iterable mapping <https://github.com/ethereum/dapp-bin/blob/master/library/iterable_mapping.sol>`_.

.. index:: assignment, ! delete, lvalue

Operadores con LValues
======================

Si ``a`` es un LValue (ej. una variable o algo que puede ser asignado), los siguientes operadores son abreviaturas posibles:

``a += e`` es equivalente a ``a = a + e`` . Los operadores ``-=``, ``*=``, ``/=``, ``%=``, ``a |=``, ``&=`` y ``^=`` son todos definidos de esa manera. ``a++`` y ``a--`` son equivalentes a ``a += 1`` / ``a -= 1`` pero la expresión en sí todavía tiene el valor anterior de ``a``. En contraste, ``--a`` y ``++a`` tienen el mismo efecto en ``a`` pero devuelven el valor después del cambio.

delete
------

``delete a`` asigna el valor inicial para el tipo a ``a``. Ej. para enteros, el equivalente es ``a = 0``, pero puede ser usado en arrays, donde él asigna un array dinámico de length cero o un array estático del mismo length con todos los elementos reseteados. Para structs, se asigna a struct con todos los miembros reseteados.

``delete`` no tiene efecto en mappings enteras (ya que las claves de los mappings pueden ser arbitrarias y generalmente desconocidas). Así que si se hace delete a un struct, reseteará todos los miembros que no son mappings y también recurrirá a los miembros al menos que sean mappings. Sin embargo, las claves individuales y lo que pueden mapear puede ser deleted.

Es importante notar que ``delete a`` en realidad se comporta como una asignación a ``a``, ej. almacena un nuevo objeto en ``a``.

::

    pragma solidity ^0.4.0;

    contract DeleteExample {
        uint data;
        uint[] dataArray;

        function f() {
            uint x = data;
            delete x; // setea x to 0, no afecta los datos
            delete data; // setea data a 0, no afecta x que aún tiene una copia
            uint[] y = dataArray;
            delete dataArray; // esto setea dataArray.length a cero, pero como uint[] es un objecto complejo,
            // también y es afectado que es un alias al objeto de almacenamiento
            // Por otra parte: "delete y" no es válido, ya que asignaciones a variables locales
            // haciendo referencia a objetos de almacenamiento sólo pueden ser hechos de
            // objetos de almacenamiento existentes.
        }
    }

.. index:: ! type;conversion, ! cast

Conversión entre tipos elementales
==================================

Conversiones implícitas
-----------------------

Si un operador es aplicado a diferentes tipos, el compilador intenta
implícitamente convertir uno de los operadores al tipo del otro (lo mismo
es verdad para asignaciones). En general, una conversión implícita entre tipos
valores es posible si es tiene sentido semanticamente y no hay información
perdida: ``uint8`` es convertible a ``uint16`` y ``int128`` a ``int256``, pero
``int8`` no es convertible a ``uint256`` (porque ``uint256`` no puede contener ``-1``).
Además, enteros sin signo pueden ser convertidos a bytes del mismo tamaño o más grande
pero no vice-versa. Cualquier tipo que puede ser convertido a ``uint160`` puede también
ser convertido a ``address``.


Conversiones explícitas
-----------------------

Si el compilador no permite conversión implícita pero sabes lo que estás haciendo,
una conversión explícita de tipo es a veces posible. Nótese que esto puede darte
comportamiento inesperado así que asegúrate de probar que el resultado es lo que quieras!
Este ejemplo es para convertir de un negativo ``int8`` a ``uint``:

::

    int8 y = -3;
    uint x = uint(y);

Al final de este snippet de código, ``x`` tendrá el valor ``0xfffff..fd`` (64
caracteres hex), que es -3 en la representación de 256 bits de los complementos de dos.

Si un tipo es explícitamente convertido a un tipo más pequeño, los bits de orden mayor son
eliminados::

uint32 a = 0x12345678;
uint16 b = uint16(a); // b será 0x5678 ahora


.. index:: ! type;deduction, ! var

.. _type-deduction:

Deducción de tipo
=================

Para conveniencia, no es siempre necesario de explícitamente especificar el tipo de
una variable, el compilador infiere automáticamente el tipo del tipo de la primera
expresión al cual es asignado esa variable::

    uint24 x = 0x123;
    var y = x;

Aquí, el tipo de ``y`` será ``uint24``. Usando ``var`` no es posible por parámetros de
función de parámetros de devolución.

.. warning::
    El tipo es deducido sólo de la primera asignación, así que
    el loop del siguiente snippet es infinito, ya que ``i`` tendrá el tipo
    ``uint8`` y cualquier valor de este tipo es más pequeño que ``2000``.
    ``for (var i = 0; i < 2000; i++) { ... }``
