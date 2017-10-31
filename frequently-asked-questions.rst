####################
Preguntas frequentes
####################

Esta lista fue originalmente compilada por `fivedogit  <mailto:fivedogit@gmail.com>`_.


*****************
Preguntas Básicas
*****************

Ejemplos de contratos
=====================

Hay algunos `ejemplos de contratos <https://github.com/fivedogit/solidity-baby-steps/tree/master/contracts/>`_ por fivedogit
y debe haber un `test contract <https://github.com/ethereum/solidity/blob/develop/test/libsolidity/SolidityEndToEndTest.cpp>`_ para cada funcionalidad de Solidity.


Crear y publicar el contrato mas simple posible
===============================================

Un contrato bastante simple es el `greeter <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/05_greeter.sol>`_

¿Es posible hacer algo en un bloque específico? (ej. publicar un contrato o ejecutar una transaction)
=====================================================================================================

Las transacciones no están garantizadas a ejecutarse en el próximo bloque o en cualquier
bloque futuro, ya que depende de los mineros de incluir transacciones y no del
remitente de la transacción. Esto se aplica a llamadas de funciones/transacciones y trasacciones
de creación de contratos.

Si quieres programar llamadas de contrato a futuro, puedes usar el
`despertador (alarm clock) <http://www.ethereum-alarm-clock.com/>`_.


¿Qué es la "payload" de la transacción?
=======================================

Esto es sólamente el data bytecode enviado junto con la solicitud.

¿Hay un decompilador disponible?
================================

No hay un decompilador en Solidity. Esto es en principio posible
hasta un punto, pero por ejemplo los nombres de variables serán
perdidas y un gran esfuerzo será necesario para replicar el código
de fuente original.

Bytecode puede ser decompilado a opcoses, un servicio que es provisto
por varios exploradores de blockchain.

Los contratos en la blockchain deben tener su código fuente
original publicado si serán utilizados por terceros.

Crear un contrato que puede ser detenido y devolver los fondos
==============================================================

Primero, una advertencia: Detener contratos suena como una buena idea, porque "limpiar"
siempre es bueno, pero como se ve arriba, no se limpia realmente. Además,
si algo de Ether es enviado a contratos eliminados, el Ether será perdido para
siempre.

Si quieres desactivar tus contratos, es mejor "inhabilitarlos" cambiando algunos estados
internos que hace que todas las funciones arrojen excepciones. Esto hará que sea imposible
de usar el contrato y todo ether enviado será devuelto automáticamente.

Ahora para responder la pregunta: Dento de un constructor, ``msg.sender`` es el
creador. Guárdalo. Luego ``selfdestruct(creator);`` para matar y devolver los fondos.

`ejemplo <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/05_greeter.sol>`_

Nótese que si importas ``import "mortal"`` arriba del contrato y declaras
``contract AlgunContrato is mortal { ...`` y compilas con un compilador que ya lo
tiene (que incluye `Remix <https://remix.ethereum.org/>`), luego
``kill()`` es ejecutado por ti. Una vez que un contrato es "mortal", se puede
``contractname.kill.sendTransaction({from:eth.coinbase})``, igual que en los
ejemplos.


Guardar Ether en un contrato
============================

El truco es de crear un contrato con ``{from:someaddress, value: web3.toWei(3,"ether")...}``

Ver `endowment_retriever.sol <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/30_endowment_retriever.sol>`_.

Usar una función no-constante (req ``sendTransation``) para incrementar una variable en un contrato
===================================================================================================

Ver `value_incrementer.sol <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/20_value_incrementer.sol>`_.

Obtener que un contrato te devuelva los fondos (sin usar ``selfdestruct(...)``).
================================================================================

Este ejemplo demuestra como envíar fondos de un contrato a una address.

Ver `endowment_retriever <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/30_endowment_retriever.sol>`_.

¿Puedes devolver un array o un ``string`` desde una llamada de función solidity?
================================================================================

Si. Ver `array_receiver_and_returner.sol <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/60_array_receiver_and_returner.sol>`_.

Lo que sí es problemático es devolver cualquier data de tamaño variable (ej. un
array de tamaño variable como ``uint[]``) desde una función **llamada desde Solidity**.
Esto es una limitación de la EVM y será resuelto en la próxima versión del protocolo.

Devolver data de tamaño variable está bien cuando es parte de una transaction o llamada externa.

¿Cómo representas ``double``/``float``` en Solidity?
====================================================

Esto no es aún posible.

Es posible de iniciar un array in-line (ej. ``string[] myarray = ["a", "b"];``)
===============================================================================

Si. Sin embargo debiera notarse que esto sólo funciona con arrays de tamaño estático. Puedes incluso crear un array en memoria en línea en la declaración de devolución. Cool, ¿no?

Example::

    contract C {
        function f() returns (uint8[5]) {
            string[4] memory adaArr = ["Esto", "es", "un", "array"];
            return ([1, 2, 3, 4, 5]);
        }
    }

Son de confianza los timestamps (``now``, ``block.timestamp``)
==============================================================

Esto depende por lo que te refieres con "de confianza".
En general, son entregados por los mineros y por lo tanto son vulnerables.

Al menos que haya un problema grave en la blockchain o en tu ordenador,
puedes hacer las siguientes suposiciones:

Publicas una transacción en un tiempo X, esta transacción contiene el
mismo código que llama ``now`` y es incluída en un bloque cuyo timestamp
es Y y este bloque es incluído en la cadena canónica (publicado) en un tiempo Z.

El valor de ``now`` será idéntico a Y y X <= Y <= Z.

Nunca usa ``now`` o ``block.hash`` como una fuente aleatoria, a menos que
sepas lo que estás haciendo.


¿Puede una función de contrato devolver un ``struct``?
======================================================

Si, pero sólo en llamadas de funciones ``internal``.

Si devuelvo un ``enum``, Sólo me dan valores enteros en web3.js. ¿Cómo obtengo los valores nombrados?
=====================================================================================================

Enums no son soportados por la ABI, sólo son soportados por Solidity.
Tienes que hacer el mapping tu mismo por ahora, aunque puede que proporcionemos
ayuda mas adelante.


¿Cuál es el significado de ``function() { ... }`` dentro de los contratos Solidity? ¿Cómo es posible que una función no tenga nombre?
======================================================================================================================================

Esta función es llamada "callback function" y es
llamada cuando alguien sólo envía Ether al contrato sin proveer data
o si alguien se equivocó e intentó llamar una función que no existe.

El funcionamiento por defecto (si no hay función fallback explícita) en
estas situaciones es arrojar una excepción.

Si el contrato debiera recibir Ether con transferencias simples, debes
implementar una función callback como

``function() payable { }``

Otro uso de la función callback es por ejemplo registrar que tu contrato
recibió ether usando un evento.

*Attention*: Si implementas la función fallback, cuida que use lo menos gas
posible, porque ``send()`` sólo suministrará una cantidad limitada.


¿Es posible pasar argumentos a la función fallback?
===================================================

La función fallback no puede tomar parámetros.

Bajo ciertas circunstancias, puedes enviar data. Si cuidad que ninguna
de las otras funciones es llamada, puedes acceder a la data usando
``msg.data``.

¿Pueden las variables de estado ser iniciadas in-line?
======================================================

Si, esto es posible para todos los tipos (incluso para structs). Sin embargo,
para arrays debe notarse que se le deben declarar como arrays de memoria estática.

Ejemplos::

    contract C {
        struct S {
            uint a;
            uint b;
        }

        S public x = S(1, 2);
        string name = "Ada";
        string[4] memory adaArr = ["Esto", "es", "un", "array"];
    }


    contract D {
        C c = new C();
    }

¿Cómo funcionan los structs?
============================

Ver `struct_and_for_loop_tester.sol <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/65_struct_and_for_loop_tester.sol>`_.

¿Cómo funcionan los for loop?
=============================

Muy similarmente a Javascript. Aunque esto es un punto al cual debe hacerse atención:

Si usas ``for (var i = 0; i < a.length; i ++) { a[i] = i; }``, entonces
el tipo de ``i`` será inferido sólo de ``0``, o sea, un tipo ``uint8``.
Esto significa que si ``a`` tiene más de ``255`` elementos, tu loop no terminará
ya que ``i`` sólo contendrá valores hasta ``255``.

Mejor usar ``for (uint i = 0; i < a.length...``

Ver `struct_and_for_loop_tester.sol <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/65_struct_and_for_loop_tester.sol>`_.

¿Qué set de caracteres usa Solidity?
====================================

Solidity es agnostico de set de caracteres con respecto a strings en el código fuente,
aunque UTF-8 es recomendado. Los identificadores (variables, funciones, ...) Solo pueden
usar ASCII.

¿Cuáles son algunos ejemplos de manipulación de strings básicos (``substring``, ``indexOf``,, ``charAt``, etc)?
===============================================================================================================

Hay algunas funciones de utilidad de string en `stringUtils.sol <https://github.com/ethereum/dapp-bin/blob/master/library/stringUtils.sol>`_
que serán extendidas en el futuro. Además, Arachnid ha escrito `solidity-stringutils <https://github.com/Arachnid/solidity-stringutils>`_.

Por ahora si quieres modificar un string, (incluso cuando sólo quieres saber su largo),
debes siempre convertirlo en un ``bytes`` primero::


    contract C {
        string s;

        function append(byte c) {
            bytes(s).push(c);
        }

        function set(uint i, byte c) {
            bytes(s)[i] = c;
        }
    }


¿Puedo concatenar dos strings?
==============================

Tienes que hacerlo manualmente por ahora.

Por qué la función de bajo nivel ``.call()`` es menos favorable que instanciando un contrato con ua vaiable (``ContractBb;``) y ejecutando sus funcioens (``b.doSomething();``)?
==========================================================================================================================================================================================

TODO: 
Si usar reales funciones, el compilador le dirá si los tipos de
los argumentos no concuerdan, si la función no existe
o no es visible y hará el empaquetamiento de los argumentos
por tí.


Ver `ping.sol <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/45_ping.sol>`_ y
`pong.sol <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/45_pong.sol>`_.

¿El gas inutilizado es automaticamente devuelto?
================================================

Si y es immediato, ej. hecho como parte de la transacción.

Cuando se devuelve un valor de tipo ``unint``, ¿es posible devolver un `undefined`` o un valor ``null``?
========================================================================================================

Esto no es posible, porque todos los tipos usan el rango de valores totales.

Tienes la opción de ``arrojar`` un error, que también revirtirá la transacción
completa, que puede que sea una buena idea si obtuviste una situación inesperada.

Si no quieres deolver un error, puedes devolver un par::

    contract C {
        uint[] counters;

        function getCounter(uint index)
            returns (uint counter, bool error) {
                if (index >= counters.length)
                    return (0, true);
                else
                    return (counters[index], false);
        }

        function checkCounter(uint index) {
            var (counter, error) = getCounter(index);
            if (error) {
                ...
            } else {
                ...
            }
        }
    }

¿Los comentarios son incluídos en los contratos publicados y incrementan el gas
===============================================================================

No. Todo lo que no sea utilizado para la ejecución es eliminado durante la compilación.
Esto incluye, entre otras cosas, comentarios, nombres de variable y nombres de tipos.

¿Qué pasa si envías ether junto con una llamada de función a un contrato?
=========================================================================

Se agrega al balance total del contrato, igual que cuando mandas ether creando un contrato.
Sólo puedes envíar ether junto con una función que tiene modificador ``payable``,
si no, una excepción es levantada.

¿Es posible obtener una respuesta tx para una transacción ejecutada contrato-a-contrato?
========================================================================================

No, una llamada de función de un contrato a otro no crea su propia transacción,
tienes que mirar en la transacción general. Eso también es la razón por la que
varios exploradores de bloques no muestran Ether envíado entre contratos correctamente.


¿Cuál es la palabra clave ``memory`` y qué hace?
================================================

La Máquina Virtual Ethereum tiene tres áreas donde puede guardar cosas.

La primera es "storage", donde todas las variables de estado del contrato existen.
Cada contrato tiene su propio storage y es persistente entre llamdas de función
y bastate caro usarlo.

La segunda es "memory", esto es usado para guardar valores temporales. Es
borrado entre llamdas de función (externas) y es más barato usar.

La tercera es en el stack, que es usado para guardar pequeñas variables locales.
Es casi gratis para usar, pero sólo puede guardar una cantidad limitada de valores.

Para casi todos los tipos, no puedes especificar donde deben ser gurdadas, porque
son copiadas cada vez que se usan.

Los tipos donde la storage-location es importante son structs
y arrays. Si, por ejemplo, pasaras estas variables en llamadas de función, su
data no es copiada si puede quedar en memoria o quedar en storage.
Esto significa que puedes modificar su contenido en la función llamada
y estas modificaciones aún serán visibles al llamador.

Hay valores por defecto para el storage location dependiendo de que tipo
de variable le concierne:

* variables de estado siempre están en storage
* argumentos de función siempre están en memoria
* variable locales siempre referencian storage

Example::

    contract C {
        uint[] data1;
        uint[] data2;

        function appendOne() {
            append(data1);
        }

        function appendTwo() {
            append(data2);
        }

        function append(uint[] storage d) {
            d.push(1);
        }
    }

La función ``append`` puede funcionar en ambos ``data1`` y ``data2`` y sus
modificaciones serán guardadas permenentemente. Si quitas la palabra clave
``storage``, por defecto se usa ``memory`` para argumentos de función. Esto tiene
como efecto que en el punto donde ``append(data1)`` o ``append(data2)`` son llamadas,
una copia independiente de la variable de estado es creada en memoria y ``append``
opera en esta copia (que no soporta ``.push`` - pero eso es otro tema). Las
modificaciones a esta copia independiente no se llevan a ``data1`` o ``data2``.

Un error típico es declarar una variable local y presumir que será creada en
memoria, aunque será creada en storage::

    /// ESTE CONTRATO CONTIENE UN ERROR
    contract C {
        uint someVariable;
        uint[] data;

        function f() {
            uint[] x;
            x.push(2);
            data = x;
        }
    }

El tipo de la variable local ``x`` es ``unint[] storage``, pero ya que
storage no es asignada dinámicamente, tiene que ser asignada desde
una variable de estado antes de que pueda ser usada. Entonces nada de
espacio en storage será asignado para ``x``, pero en cambio funciona
sólo como un alias para variables pre existentes en storage.

Lo que pasará es que el compilador interpreta ``x`` como un pointer
storage y lo apuntará al slot de storage ``0`` por defecto.
Esto significa que ``unaVariable`` (que reside en slot ``0``
de storage) es modificada por ``x.push(2)``.

La manera correcta de hacerlo es la siguiente::

    contract C {
        uint someVariable;
        uint[] data;

        function f() {
            uint[] x = data;
            x.push(2);
        }
    }

¿Cuál es la diferencia entre ``bytes`` y ``byte[]``?
====================================================

``bytes`` es en general mas eficiente: Cuando se usa como argumentos a funciones (ej
en CALLDATA) o en memoria, cada elemento de un ``byte[]`` es acolchado a 32 bytes
que derrocha 31 bytes por elemento.

¿Es posible envíar un valor mientras se llama una función que está overloaded?
==============================================================================

Es una funcionalidad que falta conocida. https://www.pivotaltracker.com/story/show/92020468
como parte de https://www.pivotaltracker.com/n/projects/1189488

La mejor solución actualmente es introducir un caso de uso especial para gas y valor
y simplemente re-checkear si están presentes en el punto de resolución de overload.

*******************
Preguntas Avanzadas
*******************

¿Cómo se obtiene un número al azar en un contrato? (Emplementar un contrato de apuestas autónomo)
=================================================================================================

Obtener números al azar es a menudo una parte crucial de un proyecto crypto
y la mayoría de los errores vienen de malos generadores de números random.

Si no quieres que sea seguro, puedes contruir algo similar a la `moneda flipper <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/35_coin_flipper.sol>`_
si no, mejor usar otro contrato que entrega azar, como el `RANDAO <https://github.com/randao/randao>`_`

Obtener un valor devuelto de una función no constante desde otro contrato
=========================================================================

El punto clave es que el contrato que llama necesita saber sobre la función que intenta llamar.

Ver `ping.sol <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/45_ping.sol>`_
y `pong.sol <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/45_pong.sol>`_.


Hacer que un contrato haga algo apenas es minado por primera vez
================================================================

Usar el contructor. Cualquier cosa dentro de él será ejecutado cuando el contrato sea minado.

Ver `replicator.sol <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/50_replicator.sol>`_.

¿Cómo se crea un array de dos dimenciones?
==========================================

Ver `2D_array.sol <https://github.com/fivedogit/solidity-baby-steps/blob/master/contracts/55_2D_array.sol>`_.

Nótese que llenando un cuadrado de 10x10 de ``uint8`` + creación de contrato tomó mas de ``800,000``
gas en el momento de escribir este contrato. 17x17 tomó ``2,000,000`` gas. Con un límite a
3.14 millones, bueno, hay poco espacio para esto.

Nótese que simplemente "creando" un array es gratis, los costos son en rellenarlo.

Nota2: Optimizando acceso storage puede bajar los costes de gas muchísimo, porque
32 valores ``uint8`` pueden ser guardados en un slot simple. El problema es que estas
optimizaciones actualmente no funcionan en bucles y también tienen un problema con revisión
de límites.
Aunque, puedes obtener mejores resultados en el futuro.


What does ``p.recipient.call.value(p.amount)(p.data)`` do?
==========================================================

Every external function call in Solidity can be modified in two ways:

1. You can add Ether together with the call
2. You can limit the amount of gas available to the call

This is done by "calling a function on the function":

``f.gas(2).value(20)()`` calls the modified function ``f`` and thereby sending 20
Wei and limiting the gas to 2 (so this function call will most likely go out of
gas and return your 20 Wei).

In the above example, the low-level function ``call`` is used to invoke another
contract with ``p.data`` as payload and ``p.amount`` Wei is sent with that call.

What happens to a ``struct``'s mapping when copying over a ``struct``?
======================================================================

This is a very interesting question. Suppose that we have a contract field set up like such::

    struct user {
        mapping(string => address) usedContracts;
    }

    function somefunction {
       user user1;
       user1.usedContracts["Hello"] = "World";
       user user2 = user1;
    }

In this case, the mapping of the struct being copied over into the userList is ignored as there is no "list of mapped keys".
Therefore it is not possible to find out which values should be copied over.

How do I initialize a contract with only a specific amount of wei?
==================================================================

Currently the approach is a little ugly, but there is little that can be done to improve it.
In the case of a ``contract A`` calling a new instance of ``contract B``, parentheses have to be used around
``new B`` because ``B.value`` would refer to a member of ``B`` called ``value``.
You will need to make sure that you have both contracts aware of each other's presence and that ``contract B`` has a ``payable`` constructor.
In this example::

    contract B {
        function B() payable {}
    }


    contract A {
        address child;

        function test() {
            child = (new B).value(10)(); //construct a new B with 10 wei
        }
    }

Can a contract function accept a two-dimensional array?
=======================================================

This is not yet implemented for external calls and dynamic arrays -
you can only use one level of dynamic arrays.

What is the relationship between ``bytes32`` and ``string``? Why is it that ``bytes32 somevar = "stringliteral";`` works and what does the saved 32-byte hex value mean?
========================================================================================================================================================================

The type ``bytes32`` can hold 32 (raw) bytes. In the assignment ``bytes32 samevar = "stringliteral";``,
the string literal is interpreted in its raw byte form and if you inspect ``somevar`` and
see a 32-byte hex value, this is just ``"stringliteral"`` in hex.

The type ``bytes`` is similar, only that it can change its length.

Finally, ``string`` is basically identical to ``bytes`` only that it is assumed
to hold the UTF-8 encoding of a real string. Since ``string`` stores the
data in UTF-8 encoding it is quite expensive to compute the number of
characters in the string (the encoding of some characters takes more
than a single byte). Because of that, ``string s; s.length`` is not yet
supported and not even index access ``s[2]``. But if you want to access
the low-level byte encoding of the string, you can use
``bytes(s).length`` and ``bytes(s)[2]`` which will result in the number
of bytes in the UTF-8 encoding of the string (not the number of
characters) and the second byte (not character) of the UTF-8 encoded
string, respectively.


Can a contract pass an array (static size) or string or ``bytes`` (dynamic size) to another contract?
=====================================================================================================

Sure. Take care that if you cross the memory / storage boundary,
independent copies will be created::

    contract C {
        uint[20] x;

        function f() {
            g(x);
            h(x);
        }

        function g(uint[20] y) {
            y[2] = 3;
        }

        function h(uint[20] storage y) {
            y[3] = 4;
        }
    }

The call to ``g(x)`` will not have an effect on ``x`` because it needs
to create an independent copy of the storage value in memory
(the default storage location is memory). On the other hand,
``h(x)`` successfully modifies ``x`` because only a reference
and not a copy is passed.

Sometimes, when I try to change the length of an array with ex: ``arrayname.length = 7;`` I get a compiler error ``Value must be an lvalue``. Why?
==================================================================================================================================================

You can resize a dynamic array in storage (i.e. an array declared at the
contract level) with ``arrayname.length = <some new length>;``. If you get the
"lvalue" error, you are probably doing one of two things wrong.

1. You might be trying to resize an array in "memory", or

2. You might be trying to resize a non-dynamic array.

::

    int8[] memory memArr;        // Case 1
    memArr.length++;             // illegal
    int8[5] storageArr;          // Case 2
    somearray.length++;          // legal
    int8[5] storage storageArr2; // Explicit case 2
    somearray2.length++;         // legal

**Important note:** In Solidity, array dimensions are declared backwards from the way you
might be used to declaring them in C or Java, but they are access as in
C or Java.

For example, ``int8[][5] somearray;`` are 5 dynamic ``int8`` arrays.

The reason for this is that ``T[5]`` is always an array of 5 ``T``'s,
no matter whether ``T`` itself is an array or not (this is not the
case in C or Java).

Is it possible to return an array of strings (``string[]``) from a Solidity function?
=====================================================================================

Not yet, as this requires two levels of dynamic arrays (``string`` is a dynamic array itself).

If you issue a call for an array, it is possible to retrieve the whole array? Or must you write a helper function for that?
===========================================================================================================================

The automatic getter function for a public state variable of array type only returns
individual elements. If you want to return the complete array, you have to
manually write a function to do that.


What could have happened if an account has storage value(s) but no code?  Example: http://test.ether.camp/account/5f740b3a43fbb99724ce93a879805f4dc89178b5
==========================================================================================================================================================

The last thing a constructor does is returning the code of the contract.
The gas costs for this depend on the length of the code and it might be
that the supplied gas is not enough. This situation is the only one
where an "out of gas" exception does not revert changes to the state,
i.e. in this case the initialisation of the state variables.

https://github.com/ethereum/wiki/wiki/Subtleties

After a successful CREATE operation's sub-execution, if the operation returns x, 5 * len(x) gas is subtracted from the remaining gas before the contract is created. If the remaining gas is less than 5 * len(x), then no gas is subtracted, the code of the created contract becomes the empty string, but this is not treated as an exceptional condition - no reverts happen.


What does the following strange check do in the Custom Token contract?
======================================================================

::

    require((balanceOf[_to] + _value) >= balanceOf[_to]);

Integers in Solidity (and most other machine-related programming languages) are restricted to a certain range.
For ``uint256``, this is ``0`` up to ``2**256 - 1``. If the result of some operation on those numbers
does not fit inside this range, it is truncated. These truncations can have
`serious consequences <https://en.bitcoin.it/wiki/Value_overflow_incident>`_, so code like the one
above is necessary to avoid certain attacks.


More Questions?
===============

If you have more questions or your question is not answered here, please talk to us on
`gitter <https://gitter.im/ethereum/solidity>`_ or file an `issue <https://github.com/ethereum/solidity/issues>`_.
