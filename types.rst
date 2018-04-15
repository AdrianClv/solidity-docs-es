.. index:: type

.. _types:

*****
Tipos
*****

Solidity es un lenguaje de tipado estático, que significa que cada tipo de
variable (estado y local) tiene que ser especificada (o al menos conocida -
ver :ref:`type-deduction` abajo) en tiempo de compilación.
Solidity proporciona varios tipos elementales que pueden ser combinados para
crear tipos más complejos.


Además de eso, los tipos pueden interactuar el uno con el otro en expresiones
conteniendo operadores. Para una lista rápida de referencia de los operadores,
ver :ref:`order`.

.. index:: ! value type, ! type;value

Tipos de valor
==============

Los siguientes tipos también son llamados tipos de valor porque las variables
de este tipo serán siempre pasadas como valores, ej. siempre serán copiados cuando
son usados como argumentos de funciones o en asignaciones.

.. index:: ! bool, ! true, ! false

Booleanos
---------

``bool``: Los posibles valores son las constantes ``true`` y ``false``.

Operadores:

*  ``!`` (negación lógica)
*  ``&&`` (conjunción lógica, "y")
*  ``||`` (disyunción lógica, "or")
*  ``==`` (igualdad)
*  ``!=`` (inigualdad)

Los operadores ``||`` y ``&&`` aplican las reglas comunes de corto circuitos. Esto significa que en la expresión ``f(x) || g(y)``, si ``f(x)`` evalúa a ``true``, ``g(y)`` no será evaluado incluso si tuviera efectos secundarios.

.. index:: ! uint, ! int, ! integer

Enteros
-------

``int`` / ``uint``: Enteros con y sin signo de varios tamaños. Las palabras clave ``uint8`` a ``uint256`` en pasos de ``8`` (sin signo de 8 hasta 256 bits) y ``int8`` a ``int256``. ``uint`` y ``int`` son alias para ``uint256`` y ``int256``, respectivamente.

Operadores:

* Comparaciones: ``<=``, ``<``, ``==``, ``!=``, ``>=``, ``>`` (evalúa a ``bool``)
* Operadores bit: ``&``, ``|``, ``^`` (OR exclusivo a nivel de bit), ``~`` (negación a nivel de bit)
* Operadores aritméticos: ``+``, ``-``, ``-`` unario, ``+`` unario, ``*``, ``/``, ``%`` (restante), ``**`` (exponenciales), ``<<`` (desplazamiento a la izquierda), ``>>`` (desplazamiento a la derecha)

La división siempre trunca (está compilada a la opcode DIV de la EVM), pero no trunca si los dos
operadores son :ref:`literales<rational_literals>` (o expresiones literales).

División por cero y módulos con cero arrojan una excepción en tiempo de ejecución.

El resultado de una operación de desplazamiento es del tipo del operador izquierdo. La
expresión ``x << y`` es equivalente a ``x * 2**y`` y ``x >> y`` es
equivalente a ``x / 2**y``. Esto significa que hacer un desplazamiento de números negativos
extiende en signo. Hacer un desplazamiento por un número negativo arroja una excepción en tiempo de ejecución.

.. warning::
    Los resultados producidos por desplazamientos a la derecha de valores negativos de tipos enteros con signo
    son diferentes de los producidos por otros lenguajes de programación. En Solidity, el desplazamiento a la derecha
    mapea la división para que los valores negativos del desplazamiento a la derecha sean redondeados hacia cero (truncado).
    En otros lenguajes de programación el desplazamiento a la derecha de valores negativos funciona como una división
    con redondeo hacia abajo (hacia infinito negativo).

.. index:: address, balance, send, call, callcode, delegatecall, transfer

.. _address:

Address
-------

``address``: Contiene un valor de 20 bytes (tamaño de una dirección de Ethereum). Los tipos address también tienen miembros y sirven como base para todos los contratos.

Operadores:

* ``<=``, ``<``, ``==``, ``!=``, ``>=`` y ``>``

Miembros de Address
^^^^^^^^^^^^^^^^^^^

* ``balance`` y ``transfer``

Para una referencia rápida, ver :ref:`address_related`.

Es posible consultar el monto de una dirección usando la propiedad ``balance``
y de enviar Ether (en unidades de wei) a una dirección usando la función ``transfer``:

::

    address x = 0x123;
    address myAddress = this;
    if (x.balance < 10 && myAddress.balance >= 10) x.transfer(10);

.. note::
    Si ``x`` es una dirección de contrato, su código (específicamente: su función de fallback, si es que está presente) será ejecutada con el llamado ``transfer`` (esta es una limitación de la EVM y no puede ser prevenida). Si esa ejecución agota el gas o falla de cualquier forma, el Ether transferido será revertido y el contrato actual se detendrá con una excepción.

* ``send``

Send es la contrapartida de bajo nivel de ``transfer``. Si la ejecución falla, el contrato actual no se detendrá con una excepción, sino que ``send`` devolverá ``false``.

.. warning::
    Hay algunos peligros en utilizar ``send``: La transferencia falla si la profundidad de la llamada es de 1024
    (esto puede ser forzado por el llamador) y también falla si al recipiente se le acaba el gas. Entonces para
    hacer transferencias de Ether seguras, siempre revisar el valor devuelto por ``send``, usar ``transfer`` o incluso mejor:
    usar un patrón donde el recipiente retira el dinero.

* ``call``, ``callcode`` y ``delegatecall``

Además, para interactuar con contratos que no se adhieren al ABI,
la función ``call`` es prevista que tome un número arbitrario de argumentos de cualquier tipo. Estos argumentos son acolchados a 32 bytes y concatenados. Una excepción es el caso donde el primer argumento es codificado a exactamente 4 bytes. En este caso, no está acolchado para permitir el uso de firmas de función.

::

    address nameReg = 0x72ba7d8e73fe8eb666ea66babc8116a41bfb10e2;
    nameReg.call("register", "MyName");
    nameReg.call(bytes4(keccak256("fun(uint256)")), a);

``call`` devuelve un booleano indicando si la función llamada terminó (``true``) o causó una excepción de la EVM (``false``). No es posible acceder a los datos reales devueltos (para esto necesitaremos saber de antemano el tamaño de codificación).

``delegatecall`` puede ser usado de forma similar: la diferencia es que sólo se usa el código de la dirección dada, todos los demás aspectos (almacenamiento, saldo, ...) salen directamente del contrato actual. El propósito de ``delegatecall`` es usar el código de librería que está almacenado en otro contrato. El usuario tiene que asegurarse de que el layout del almacenamiento en ambos contratos es correcto para usar ``delegatecall``. Antes de homestead, sólo una versión limitada llamada ``callcode`` estaba disponible pero no daba acceso a los valores ``msg.sender`` y ``msg.value`` originales.

Las tres funciones ``call``, ``delegatecall`` y ``callcode`` son funciones de muy bajo nivel y deben usarse sólo como medida de último recurso ya que rompen la seguridad de tipo de Solidity.

La opción ``.gas()`` está disponible en los 3 métodos, mientras que la opción ``.value()`` no se admite para ``delegatecall``.

.. note::
    Todos los contratos heredan los miembros de address, así que es posible consultar el saldo del contrato actual
    usando ``this.balance``.

.. warning::
    Todas estas funciones son funciones de bajo nivel y deben usarse con cuidado.
    Específicamente, cualquier contrato desconocido puede ser malicioso y si se le llama,
    se le da el control a ese contrato, que luego puede llamar de vuelta a tu contrato,
    así que prepárate para cambios a tus variables de estado cuando la llamada retorna el valor.

.. index:: byte array, bytes32


Arrays de bytes de tamaño fijo
------------------------------

``bytes1``, ``bytes2``, ``bytes3``, ..., ``bytes32``. ``byte`` es un alias para ``bytes1``.

Operadores:

* Comparaciones: ``<=``, ``<``, ``==``, ``!=``, ``>=``, ``>`` (evalúa a ``bool``)
* Operadores Bit: ``&``, ``|``, ``^`` (OR exclusivo a nivel de bits), ``~`` (negación a nivel de bits), ``<<`` (desplazamiento a la izquierda), ``>>`` (desplazamiento a la derecha)
* Acceso por índice: Si ``x`` es de tipo ``bytesI``, entonces ``x[k]`` para ``0 <= k < I`` devuelve el byte ``k`` (sólo lectura).

El operador de desplazamiento funciona con cualquier entero como operador derecho (pero
devuelve el tipo del operador izquierdo, que denota el número de bits a desplazarse.
Desplazarse por un número negativo arroja una excepción en tiempo de ejecución.

Miembros:

* ``.length`` devuelve el largo fijo del array byte (sólo lectura).

Arrays de bytes de tamaño dinámico
----------------------------------

``bytes``:
    Array bytes de tamaño dinámico, ver :ref:`arrays`. ¡No un tipo de valor!
``string``:
    Cadena de caracteres UTF-8-codificado de tamaño dinámico, ver :ref:`arrays`. ¡No un tipo de valor!

Como regla general, usa ``bytes`` para data raw byte de tamaño arbitrario y ``string``
para una cadena de caracteres (UTF-8) de tamaño arbitrario. Si puedes limitar el tamaño a un cierto
número de bytes, siempre usa una de ``bytes1`` a ``bytes32`` porque son muchas más baratas.

.. index:: ! ufixed, ! fixed, ! fixed point number

Números de punto fijo
---------------------

**PRÓXIMAMENTE...**

.. index:: address, literal;address

.. _address_literals:

Address literales
-----------------

Literales hexadecimales que pasan el test checksum, por ejemplo
``0xdCad3a6d3569DF655070DEd06cb7A1b2Ccd1D3AF`` es de tipo ``address``.
Literales hexadecimales que están entre 39 y 41 dígitos de largo y
no pasan el test de checksum producen una advertencia y son tratados como
números racionales literales regulares.

.. index:: literal, literal;rational

.. _rational_literals:

Literales racionales y enteros
------------------------------

Literales enteros son formados por una secuencia de números en el rango 0-9.
Son interpretados como decimales. Por ejemplo, ``69`` significa sesenta y nueve.
Literales octales no existen en Solidity y los ceros a la izquierda son inválidos.

Literales de fracciones decimales son formados por un ``.`` con al menos un número en
un lado. Ejemplos incluyen ``1.``, ``.1`` y ``1.3``.

La notación científica está también soportada, donde la base puede tener fracciones, mientras que el exponente no puede.
Ejemplos incluyen ``2e10``, ``-2e10``, ``2e-10``, ``2.5e1``.

Expresiones de números literales retienen precisión arbitraria hasta que son convertidas a un tipo no literal (ej. usándolas
juntas con una expresión no literal).
Esto significa que las computaciones no se desbordan y las divisiones no se truncan en expresiones de números literales.

Por ejemplo, ``(2**800 + 1) - 2**800`` resulta en la constante ``1`` (de tipo ``uint8``)
aunque resultados intermedios ni siquiera serían del tamaño de la palabra. Además, ``.5 * 8`` resulta
en el entero ``4`` (aunque no se hayan usado enteros entremedias).

Si el resultado no es un entero, un tipo apropiado ``ufixed`` o ``fixed`` es usado del cual el número
de bits fraccionales es tan grande como se necesite (aproximando el número racional en el peor de los casos).

En ``var x = 1/4;``, ``x`` recibirá el tipo ``ufixed0x8`` mientras que en ``var x = 1/3`` recibirá
el tipo ``ufixed0x256`` porque ``1/3`` no es finitamente representable en binario y entonces será
aproximado.

Cualquier operador que puede ser aplicado a enteros también puede ser aplicado a una expresión de
número literal con tal que los operadores sean enteros. Si cualquiera de los dos es fraccional, las
operaciones de bit no son permitidas y la exponenciación no es permitida si el exponente es fraccional
(porque eso puede resultar en un número no racional).

.. note::
    Solidity tiene un tipo literal de número para cada número racional.
    Literales enteros y números racionales literales pertenecen a los tipos de números
    literales. Por otra parte, todas las expresiones literales (p.ej. las expresiones que
    contienen sólo números literales y operadores) pertenecen a tipos de números literales.
    Entonces las expresiones de números literales ``1 + 2`` y ``2 + 1`` ambas
    pertenecen al mismo tipo de número literal para el número racional tres.

.. note::
    La mayoría de fracciones decimales finitas como ``5.3743`` no son finitamente representables en binario.
    El tipo correcto para ``5.3743`` es ``ufixed8x248`` porque permite la mejor aproximación del número. Si
    quieres usar el número junto con tipos como ``ufixed`` (ej. ``ufixed128x128``), tienes que
    especificar la precisión buscada de forma explícita: ``x + ufixed(5.3743)``.

.. warning::
    La división de enteros literales se solía truncar en versiones anteriores, pero ahora se convertirá 
    en un número racional, ej. ``5 / 2`` no es igual a ``1``, más bien a ``2.5``.

.. note::
    Expresiones de números literales son convertidas en tipos no literales tan pronto como sean usadas 
    con expresiones no literales. Aunque sabemos que el valor de la expresión asignada a ``b`` 
    en el siguiente ejemplo evalúa a un entero, sigue usando tipos de punto fijo (y no números literales racionales) 
    entremedio y entonces el código no compila.

::

    uint128 a = 1;
    uint128 b = 2.5 + a + 0.5;

.. index:: literal, literal;string, string

String literales
----------------

Los strings literales se escriben con comillas simples o dobles (``"foo"`` or ``'bar'``). No hay ceros implícitos como en C; ``"foo"`` representa tres bytes, no cuatro. Como con literales enteros, su tipo puede variar, pero son implícitamente convertibles a ``bytes1``, ..., ``bytes32``, si caben a ``bytes`` y a ``string``.

Los strings literales soportan caracteres de escape, tales como ``\n``, ``\xNN`` y ``\uNNNN``. ``\xNN`` toma un valor e inserta el byte apropiado, mientras que ``\uNNNN`` toma un codepoint Unicode e inserta una secuencia UTF-8.


.. index:: literal, bytes


Literales hexadecimales
-----------------------

Los literales hexadecimales son prefijos con la palabra clave ``hex`` y son cerrados por comillas simples o dobles (``hex"001122FF"``). Su contenido debe ser una cadena hexadecimal y su valor será la representación binaria de esos valores.

Los literales hexadecimales se comportan como los string literales y tienen las mismas restricciones de convertibilidad.


.. index:: enum

.. _enums:

Enums
-----

Los Enums son una manera para el usuario de crear sus propios tipos en Solidity. Son explícitamente convertibles
a y desde todos los tipos de enteros, pero la conversión implícita no se permite. Las conversiones explícitas
revisan los valores de rangos en tiempo de ejecución y un fallo causa una excepción. Los Enums necesitan al menos un miembro.

::

    pragma solidity ^0.4.16;

    contract test {
        enum ActionChoices { GoLeft, GoRight, GoStraight, SitStill }
        ActionChoices choice;
        ActionChoices constant defaultChoice = ActionChoices.GoStraight;

        function setGoStraight() public {
            choice = ActionChoices.GoStraight;
        }

        // Ya que los tipos enum no son parte del ABI, la firma de "getChoice"
        // será automáticamente cambiada a "getChoice() returns (unit8)"
        // para todo lo externo a Solidity. El tipo entero usado es apenas
        // suficientemente grande para guardar todos los valores enum, p.ej. si
        // tienes más valores, `unit16` será utilizado y así sucesivamente.
        function getChoice() public view returns (ActionChoices) {
            return choice;
        }

        function getDefaultChoice() public pure returns (uint) {
            return uint(defaultChoice);
        }
    }

.. index:: ! function type, ! type; function

.. _function_types:

Tipos de función
----------------

Los tipos función son tipos de función. Variables de tipo función
pueden ser asignados desde funciones y parámetros de funciones de tipo función
pueden ser usadas para pasar funciones y retornar funciones de llamados de funciones.
Los tipos de función, los hay de dos tipos: *internas* y *externas*:

Las funciones internas sólo pueden ser usadas dentro del contrato actual (específicamente,
dentro de la unidad de código actual, que también incluye funciones de librerías internas
y funciones heredadas) porque no pueden ser ejecutadas fuera del
contexto del contrato actual. La llamada a una función interna se realiza
saltando a su label de entrada, tal como cuando se llama a una función interna del
contrato actual.

Las funciones externas están compuestas de una dirección y una firma de función y pueden
ser pasadas y devueltas desde una llamada de función externa.

Los tipos de funciones son notadas como sigue::

    function (<parameter types>) {internal|external} [constant] [payable] [returns (<return types>)]

A diferencia de los tipos de parámetros, los tipos de retorno no pueden estar vacíos - si
el tipo función no debe retornar nada, la parte ``returns (<return types>)``
tiene que ser omitida.

Por defecto, las funciones son de tipo interna, así que la palabra clave ``internal``
puede ser omitida.

Hay dos formas de acceder una función en el contrato actual: o bien directamente
con su nombre, ``f``, o usando ``this.f``. Usando el nombre resultará en una función
interna, y con ``this`` habrá una función externa.

Si una variable de tipo función no es inicializada, llamarla resultará 
en una excepción. Lo mismo ocurre si llamas una función después de usar
``delete`` en ella.

Si funciones externas son usadas fuera del contexto de Solidity, son tratadas
como tipo ``function``, que codifica la dirección seguida por el identificador
de la función junto con un tipo ``bytes24``.

Nótese que las funciones públicas del contrato actual pueden ser usadas tanto
como una función interna como externa. Para usar ``f`` como función interna, sólo
se le llama como ``f``, y si se quiere usar como externa, usar ``this.f``.

Además, las funciones públicas (o externas) también tienen un miembro especial llamado ``selector``,
que devuelve :ref:`ABI function selector <abi_function_selector>`::

    pragma solidity ^0.4.16;

    contract Selector {
      function f() public view returns (bytes4) {
        return this.f.selector;
      }
    }

Ejemplo que muestra como usar tipos de función internas::

    pragma solidity ^0.4.16;

    library ArrayUtils {
      // internal functions can be used in internal library functions because
      // they will be part of the same code context
      function map(uint[] memory self, function (uint) pure returns (uint) f)
        internal
        pure
        returns (uint[] memory r)
      {
        r = new uint[](self.length);
        for (uint i = 0; i < self.length; i++) {
          r[i] = f(self[i]);
        }
      }
      function reduce(
        uint[] memory self,
        function (uint, uint) pure returns (uint) f
      )
        internal
        pure
        returns (uint r)
      {
        r = self[0];
        for (uint i = 1; i < self.length; i++) {
          r = f(r, self[i]);
        }
      }
      function range(uint length) internal pure returns (uint[] memory r) {
        r = new uint[](length);
        for (uint i = 0; i < r.length; i++) {
          r[i] = i;
        }
      }
    }

    contract Pyramid {
      using ArrayUtils for *;
      function pyramid(uint l) public pure returns (uint) {
        return ArrayUtils.range(l).map(square).reduce(sum);
      }
      function square(uint x) internal pure returns (uint) {
        return x * x;
      }
      function sum(uint x, uint y) internal pure returns (uint) {
        return x + y;
      }
    }

Otro ejemplo que usa tipos de función externa::

    pragma solidity ^0.4.21;

    contract Oracle {
      struct Request {
        bytes data;
        function(bytes memory) external callback;
      }
      Request[] requests;
      event NewRequest(uint);
      function query(bytes data, function(bytes memory) external callback) public {
        requests.push(Request(data, callback));
        emit NewRequest(requests.length - 1);
      }
      function reply(uint requestID, bytes response) public {
        // Aquí va la comprobación de que la respuesta proviene de una fuente de confianza
        requests[requestID].callback(response);
      }
    }

    contract OracleUser {
      Oracle constant oracle = Oracle(0x1234567); // contrato conocido
      function buySomething() {
        oracle.query("USD", this.oracleResponse);
      }
      function oracleResponse(bytes response) public {
        require(msg.sender == address(oracle));
        // Usar los datos
      }
    }

.. Nótese::
    Las funciones lambda o inline están planeadas pero no están aún implementadas.

.. index:: ! type;reference, ! reference type, storage, memory, location, array, struct

Tipos de referencia
===================

Tipos complejos, ej. tipos que no siempre caben en 256 bits tienen que ser manejados
con más cuidado que los tipos de valores que ya hemos visto. Ya que copiarlos puede
ser muy caro, tenemos que pensar sobre si queremos que se almacenen en **memory**
(que no es persistente) o en **storage** (donde las variables de estado se guardan).

Ubicación de datos
------------------

Cada tipo complejo, ej. *arrays* y *structs*, tienen anotaciones
adicionales, la "data location", con respecto a si es almacenado
en memoria o en almacenamiento. Dependiendo del contexto, siempre hay un
valor por defecto, pero puede ser reemplazado añadiendo o bien
``storage`` o `memory`` al tipo. Por defecto para tipos parámetros de
función (incluyendo parámetros de retorno) es ``memory``, por defecto para
variables locales es ``storage`` y la ubicación es forzada a ``storage``
para variables de estado (obviamente).

Hay una tercera ubicación de datos, "calldata", un área que no es modificable
ni persistente donde argumentos de función son almacenados. Parámetros de función
(no parámetros de retorno) de funciónes externas son forzados a "calldata" y
se comportan casi como memoria.

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
        uint[] x; // la ubicación de los datos de x es storage

        // la ubicación de datos de memoryArray es memory
        function f(uint[] memoryArray) public {
            x = memoryArray; // funciona, copia el array entero al almacenamiento
            var y = x; // funciona, asigna una referencia, ubicación de datos de y es almacenamiento
            y[7]; // bien, devuelve el octavo elemento
            y.length = 2; // bien, modifica x a través de y
            delete x; // bien, limpia el array, también modifica y
            // Lo siguiente no funciona; debería crear un nuevo array temporal/sin nombre
            // en almacenamiento, pero el almacenamiento es asignado "estáticamente":
            // y = memoryArray;
            // Esto no funciona tampoco, ya que resetearía el apuntador, pero no hay
            // ubicación donde podría apuntar
            // delete y;
            g(x); // llama g, dando referencia a x
            h(x); // llama h y crea una copia independiente y temporal en la memoria
        }

        function g(uint[] storage storageArray) internal {}
        function h(uint[] memoryArray) public {}
    }


Resumen
^^^^^^^

Ubicación de datos forzada:
 - parámetros (no de retorno) de funciones externas: calldata
 - variables de estado: almacenamiento

Ubicación de datos por defecto:
 - parámetros (también de retorno) de funciones: memoria
 - todas otras variables: almacenamiento

.. index:: ! array

.. _arrays:

Arrays
------

Los array pueden tener tamaño fijo en compilación o pueden ser dinámicos.
Para arrays de almacenamiento, el tipo del elemento puede ser arbitrario (ej. también
otros arrays, mapeos o structs). Para arrays de memoria, no puede ser un mapping y
tiene que ser un tipo ABI si es que es un argumento de una función públicamente
visible.

Un array de tamaño fijo ``k`` y elemento tipo ``T`` es escrito como ``T[k]``,
un array de tamaño dinámico como ``T[]``. Como ejemplo, un array de 5 arrays
dinámicos de ``uint`` es ``uint[][]`` (nótese que la notación es invertida
comparada a otros lenguajes). Para acceder al segundo uint en el tercer
array dinámico, se utiliza ``x[2][1]`` (los índices comienzan en 0 y el acceso
funciona de forma opuesta a la declaración. i.e. ``x[2]`` reduce un nivel en el
tipo desde la derecha)

Variables de tipo ``bytes`` y ``string`` son arrays especiales. Un ``bytes`` es similar a ``byte[]``,
pero está junto en el calldata. ``string`` es igual a ``bytes`` pero no permite el acceso
a la longitud o mediante índice (por ahora).

De modo que ``bytes`` siempre será preferible a ``byte[]`` ya que es más barato.

.. note::
    Si quieres acceder a la representación en bytes de un string ``s``, usa
    ``bytes(s).length`` / ``bytes(s)[7] = 'x';``. ¡Ten en cuenta que
    estás accediendo a los bytes a bajo nivel de la representación en UTF-8,
    y no a los caracteres individualmente!

Es posible marcar arrays como ``public`` y dejar que Solidity cree un getter.
El índice numérico se convertirá en un parámetro requerido por el getter.

.. index:: ! array;allocating, new

Asignación de memoria en Arrays
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Crear arrays con longitud variable en memoria se puede hacer usando la palabra clave ``new``.
Al contrario que con los arrays en storage, no es posible redimensionar los arrays en memoria
mediante asignación al miembro ``.length``.

::

    pragma solidity ^0.4.16;

    contract C {
        function f(uint len) public pure {
            uint[] memory a = new uint[](7);
            bytes memory b = new bytes(len);
            // Aquí tenemos a.length == 7 y b.length == len
            a[6] = 8;
        }
    }

.. index:: ! array;literals, !inline;arrays

Array Literales / Arrays en linea
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Los Array literales son arrays que se escriben como una expresión y no están
asignados a una variable al momento.

::

    pragma solidity ^0.4.16;

    contract C {
        function f() {
            g([uint(1), 2, 3]);
        }
        function g(uint[3] _data) public pure {
            // ...
        }
    }

El tipo de array literal es un array de memoria de tamaño fijo del cual el tipo
base es el tipo común de los elementos dados. El tipo de ``[1, 2, 3]`` es
``uint[3] memory``, porque el tipo de cada una de estas constantes es ``uint8``.
Por eso, fue necesario convertir el primer elemento en el ejemplo arriba
a ``uint``. Nótese que actualmente, los arrays de memoria de tamaño fijo no pueden
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
    Los arrays tienen un miembro ``length`` para guardar su número de elementos.
    Arrays dinámicos pueden ser modificados en almacenamiento (no en memoria) cambiando
    el miembro ``.length``. Ésto no ocurre automáticamente cuando se intenta acceder a los elementos fuera de la longitud actual. El tamaño de arrays de memoria es fijo (pero dinámico, ej. puede depender de parámetros en tiempo de ejecución) cuando son creados.
**push**:
    Los arrays de almacenamiento dinámico y ``bytes`` (no ``string``) tienen una función miembro llamada ``push`` que puede ser usada para agregar un elemento al final del array. La función devuelve el nuevo length.

.. warning::
    Aún no es posible usar arrays en funciónes externas.

.. warning::
    Debido a las limitaciones de la EVM, no es posible retornar
    contenido dinámico de las funciónes externas . La función ``f`` en
    ``contract C { function f() returns (uint[]) { ... } }`` devolverá
    algo si es llamado desde web3.js, pero no si se llama desde Solidity.

    La única alternativa por ahora es usar grandes arrays de tamaño estático.


::

    pragma solidity ^0.4.16;

    contract ArrayContract {
        uint[2**20] m_aLotOfIntegers;
        // Nótese que el siguiente no es un par de arrays dinámicos, sino un
        // array dinámico de pares (ej. de arrays de tamaño fijo de length 2).
        bool[2][] m_pairsOfFlags;
        // newPairs es almacenado en memoria - por defecto para argumentos de función

        function setAllFlagPairs(bool[2][] newPairs) public {
            // asignación a un array de almacenamiento reemplaza el array completo
            m_pairsOfFlags = newPairs;
        }

        function setFlagPair(uint index, bool flagA, bool flagB) public {
            // acceso a un index que no existe arrojará una excepción
            m_pairsOfFlags[index][0] = flagA;
            m_pairsOfFlags[index][1] = flagB;
        }

        function changeFlagArraySize(uint newSize) public {
            // si el tamaño nuevo es más pequeño, los elementos eliminados del array serán limpiados
            m_pairsOfFlags.length = newSize;
        }

        function clear() public {
            // éstos limpian los arrays completamente
            delete m_pairsOfFlags;
            delete m_aLotOfIntegers;
            // efecto idéntico aquí
            m_pairsOfFlags.length = 0;
        }

        bytes m_byteData;

        function byteArrays(bytes data) public {
            // byte arrays ("bytes") son diferentes ya que no son almacenados sin padding,
            // pero pueden ser tratados idénticamente a "uint8[]"
            m_byteData = data;
            m_byteData.length += 7;
            m_byteData[3] = 8;
            delete m_byteData[2];
        }

        function addFlag(bool[2] flag) public returns (uint) {
            return m_pairsOfFlags.push(flag);
        }

        function createMemoryArray(uint size) public pure returns (bytes) {
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

        function newCampaign(address beneficiary, uint goal) public returns (uint campaignID) {
            campaignID = numCampaigns++; // campaignID es variable de retorno
            // Crea un nuevo struct y lo guarda en almacenamiento. Dejamos fuera el tipo mapping.
            campaigns[campaignID] = Campaign(beneficiary, goal, 0, 0);
        }

        function contribute(uint campaignID) public payable {
            Campaign c = campaigns[campaignID];
            // Crea un nuevo struct de memoria temporal, inicializado con los valores dados
            // y lo copia al almacenamiento.
            // Nótese que también se puede usar Funder(msg.sender, msg.value) para inicializarlo
            c.funders[c.numFunders++] = Funder({addr: msg.sender, amount: msg.value});
            c.amount += msg.value;
        }

        function checkGoalReached(uint campaignID) public returns (bool reached) {
            Campaign c = campaigns[campaignID];
            if (c.amount < c.fundingGoal)
                return false;
            uint amount = c.amount;
            c.amount = 0;
            c.beneficiary.transfer(amount);
            return true;
        }
    }

El contrato no provee la funcionalidad total de un contrato crowdfunding,
pero contiene los conceptos básicos necesarios para entender structs.
Los tipos struct pueden ser usados dentro de mappings y arrays, y ellos mismos
pueden contener mappings y arrays.

No es posible para un struct contener un miembro de su propio tipo,
aunque el struct puede ser el tipo valor de un miembro mapping.
Esta restricción es necesaria, ya que el tamaño del struct tiene que ser finito.

Nótese como en todas las funciónes, un tipo struct es asignado a la variable local
(de la ubicación por defecto del almacenamiento).
Esto no copia el struct pero guarda una referencia para que las asignaciones
a miembros de la variable local realmente escriban al estado.

Por supuesto, puedes diréctamente acceder a los miembros del struct sin
asignarlos a la variable local, como en
``campaigns[campaignID].amount = 0``.

.. index:: !mapping

Mappings
========

Tipos mapping son declarados como ``mapping(_KeyType => _ValueType)``.
Aquí ``_KeyType`` puede ser casi cualquier tipo excepto mapping, un array de tamaño dinámico, un contrato, un enum y un struct.
``_ValueType`` puede ser cualquier tipo, incluyendo mappings.

Mappings pueden verse como 'tablas hash <https://en.wikipedia.org/wiki/Hash_table>'_ que son virtualmente inicializadas ya que
cada posible clase existe y es mapeada a un valor que su representación byte es
todo ceros: el valor :ref:`por defecto <default-value>` de un tipo. Aunque la similitud termina aquí: los datos clave no son realmente
almacenados en el mapping, sólo su hash ``keccak256`` usado para buscar el valor.

Por esto, los mappings no tienen un length o un concepto de "fijar" clave o valor.

Los mappings sólo son permitidos para variables de estado (o como tipos de referencia
en funciones internas).

Es posible marcar los mappings ``public`` y hacer que Solidity cree un getter.
El ``_KeyType`` será un parámetro requerido para el getter y devolverá ``_ValueType``.

El ``_ValueType`` puede ser un mapping también. El getter tendrá un parámetro
para cada ``_KeyType``, recursivamente.

::

    pragma solidity ^0.4.0;

    contract MappingExample {
        mapping(address => uint) public balances;

        function update(uint newBalance) public {
            balances[msg.sender] = newBalance;
        }
    }

    contract MappingUser {
        function f() public returns (uint) {
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

``delete a`` asigna el valor inicial para el tipo a ``a``. Ej. para enteros, el equivalente es ``a = 0``, pero puede ser usado en arrays, donde se asigna un array dinámico de length cero o un array estático del mismo length con todos los elementos reseteados. Para structs, se asigna a struct con todos los miembros reseteados.

``delete`` no tiene efecto en mappings enteros (ya que las claves de los mappings pueden ser arbitrarias y generalmente desconocidas). Así que si se hace delete a un struct, reseteará todos los miembros que no son mappings y también recursivamente a los miembros al menos que sean mappings. Sin embargo, las claves individuales y lo que pueden mapear pueden ser eliminados.

Es importante notar que ``delete a`` en realidad se comporta como una asignación a ``a``, ej. almacena un nuevo objeto en ``a``.

::

    pragma solidity ^0.4.0;

    contract DeleteExample {
        uint data;
        uint[] dataArray;

        function f() public {
            uint x = data;
            delete x; // setea x to 0, no afecta a los datos
            delete data; // setea data a 0, no afecta a x que aún tiene una copia
            uint[] storage y = dataArray;
            delete dataArray; // esto setea dataArray.length a cero, pero como uint[] es un objecto complejo,
            // también y es afectado que es un alias al objeto de almacenamiento
            // Por otra parte: "delete y" no es válido, ya que asignaciones a variables locales
            // haciendo referencia a objetos de almacenamiento sólo pueden ser hechas de
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
es verdad para asignaciones). En general, una conversión implícita entre tipos de
valores es posible si tiene sentido semanticamente y no hay información
perdida: ``uint8`` es convertible a ``uint16`` y ``int128`` a ``int256``, pero
``int8`` no es convertible a ``uint256`` (porque ``uint256`` no puede contener ``-1``).
Además, enteros sin signo pueden ser convertidos a bytes del mismo tamaño o más grande
pero no vice-versa. Cualquier tipo que puede ser convertido a ``uint160`` puede también
ser convertido a ``address``.

Conversiones explícitas
-----------------------

Si el compilador no permite conversión implícita pero sabes lo que estás haciendo,
una conversión explícita de tipo es a veces posible. Nótese que esto puede darte un
comportamiento inesperado, ¡así que asegúrate de probar que el resultado es el que querías!
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

Por conveniencia, no es siempre necesario especificar explícitamente el tipo de
una variable, el compilador infiere automáticamente el tipo del la primera
expresión a la cual es asignada esa variable::

    uint24 x = 0x123;
    var y = x;

Aquí, el tipo de ``y`` será ``uint24``. No es posible usar ``var`` para parámetros de
función o parámetros de retorno.

.. warning::
    El tipo es deducido sólo de la primera asignación, así que
    el bucle del siguiente snippet es infinito, ya que ``i`` tendrá el tipo
    ``uint8`` y cualquier valor de este tipo es más pequeño que ``2000``.
    ``for (var i = 0; i < 2000; i++) { ... }``
