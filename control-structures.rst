#####################################
Expresiones y Estructuras de Control
#####################################

.. index:: ! parámetro, parámetro;entrada, parámetro;salida

Parámetros de entrada y de salida
=================================

Al igual que en Javascript, las funciones obtienen parámetros como entrada;
al contrario que en Javascript y C, estas también deberían devolver un número arbitrario de parámetros como salida.

Parámetros de entrada
---------------------

Los parámetros de entrada se declaran de la misma forma que las variables. Como una excepción, los parámetros no usados pueden omitir el nombre de la variable. Por ejemplo, si quisiéramos que nuestro contrato acepte un tipo de llamadas externas con dos enteros, el código quedaría similar a este::

    contract Simple {
        function taker(uint _a, uint _b) {
            // has algo con _a y _b.
        }
    }

Parámetros de salida
--------------------

Los parámetros de salida se pueden declarar con la misma sintaxis después de la palabra reservada ``returns``. Por ejemplo, supongamos que deseamos devolver dos resultados: la suma y el producto de dos valores dados. Entonces, escribiríamos un código como este::

    contract Simple {
        function arithmetics(uint _a, uint _b) returns (uint o_sum, uint o_product) {
            o_sum = _a + _b;
            o_product = _a * _b;
        }
    }

Los nombres de los parámetros de salida se pueden omitir.
Los valores de saluda se pueden especificar también usando sentencias ``return``.
Las sentencias ``return`` también son capaces de devolver múltiples valores, ver :ref:`multi-return`.
Los parámetros de retorno se inicializan a cero; si no se especifica esplícitamente su valor, permanecen con dicho valor cero.

Los parámetros de entrada y salida se pueden usar como expresiones en el cuerpo de la función. En este caso, también pueden ir en el lado izquierdo de una asignación.

.. index:: if, else, while, do/while, for, break, continue, return, switch, goto

Estructuras de control
======================

La mayoría de las estructuras de control disponibles en JavaScript, también lo están en Solidity exceptuando ``switch`` y ``goto``. Esto significa que tenemos: ``if``, ``else``, ``while``, ``do``, ``for``, ``break``, ``continue``, ``return``, ``? :``, con la semántica habitual conocida de C o JavaScript.

Los paréntesis no se pueden omitir para condicionales, pero sí las llaves alrededor de los cuerpos de las sentencias sencillas.

Hay que tener en cuenta que no hay conversión de tipos desde non-boolean a boolean como hay en C y JavaScript, por loo que ``if (1) { ... }`` *no* es válido en Solidity.

.. _multi-return:

Devolver múltiples valores
--------------------------

Cuando una función tiene múltiples valores de salida, ``return (v0, v1, ...,
vn)`` puede devolver múltiples valores. El número de componentes debe ser el mismo que el de parámetros de salida.

.. index:: ! función;llamada, función;interna, función;externa

.. _function-calls:

Llamadas de función
===================

Llamadas de función internas
----------------------------

Las funciones del conrtrato actual pueden ser llamadas directamente("internamente") y, también recursivamente como se puede ver en este ejemplo sin sentido funcional::

    contract C {
        function g(uint a) returns (uint ret) { return f(); }
        function f() returns (uint ret) { return g(7) + f(); }
    }

Estas llamadas de función son traducidas en simples saltos dentro de la máquina virtual de Ethereum (EVM). Esto tiene como consecuencia que la memoria actual no se limpia, así que pasar referencias de memoria a las funciones llamadas internamente es muy eficiente. Sólo las funciones del mismo contrato pueden ser llamadas internamente.

Llamadas de función externas
----------------------------

Las expresiones ``this.g(8);`` and ``c.g(2);`` (donde ``c`` es la instancia de un contrato) son también llamadas de función válidas, pero en esta ocasión, la función se llamará "externamente", mediante un message call y no directamente por saltos.
Por favor, es importante tener en cuenta que las llamadas de función en ``this`` no pueden ser usadas en el constructor, ya que el contrato en cuestión no se ha creado todavía.

Las funciones de otros contratos se tienen que llamar de forma externa. Para una llamada externa,
todos los argumentos de la función tienen que ser copiados en memoria.

Cuando se llama a funciones de otros contratos, la cantidad de Wei enviada con la llamada y el gas pueden especificarse con opciones especiales ``.value()`` y ``.gas()``, respectivamente::

    contract InfoFeed {
        function info() payable returns (uint ret) { return 42; }
    }


    contract Consumer {
        InfoFeed feed;
        function setFeed(address addr) { feed = InfoFeed(addr); }
        function callFeed() { feed.info.value(10).gas(800)(); }
    }

El modificador ``payable`` se tiene que usar para ``info``, porque de otra manera la opción `.value()`
no estaría disponible.

Destacar que la expresión ``InfoFeed(addr)`` realiza una conversión de tipo explícita afirmando que "sabemos que el tipo de contrato en la dirección dada es ``InfoFeed``" y este no ejecuta un constructor. Las conversiones de tipo explícitas tienen que ser gestionadas con extrema precaución. Nunca se debe llamar a una función en un contrato donde no se tiene seguridad de cuál es su tipo.

También se podría usar ``function setFeed(InfoFeed _feed) { feed = _feed; }`` directamente.
Hay que tener cuidado con el hecho de que ``feed.info.value(10).gas(800)``
sólo (localmente) establece el valor y la cantidfad de gas enviado con la llamada de función y, sólo el paréntesis al final realiza la llamada actual.

Las llamadas de función provocan excepciones si el contrato invocado no existe (en el sentido de que la cuenta no contiene código) o si el contrato invocado por sí mismo dispara una excepción o se queda sin gas.

.. warning::
    Cualquier interacción con otro contrato supone un daño potencial, especialmente si el código fuente del contrato no se conoce de antemano. El contrato actual pasa el control al contrato invocado y eso potencialmente podría suponer que haga cualquier cosa. Incluso si el contrato invocado hereda de un contrato padre conocido, el contrato del que hereda sólo requiere tener una interfaz correcta. La implementación del contrato, sin embargo, puede ser totalmente aleatoria y, por ello, crear un perjuicio. Además, hay que estar preparado en caso de que llame dentro de otros contratos del sistema o, incluso, volver al contrato que lo llama antes de que la primera llamada retorne. Esto significa que el contrato invocado puede cambiar variables de estado del contrato que le llama via sus funciones. Escribir tus funciones de esa manera, por ejemplo, llamadas a funciones externas ocurridas después de cualquier cambio en variables de estado en tu contrato, hace que este contrato no sea vulnerable a un código malicioso reejecutable.
    

Named Calls y parámetros de funciones anónimas
----------------------------------------------

Los argumentos de una llamada a una función pueden venir dados por el nombre, en cualquier orden, si están entre ``{ }`` como se puede ver en el siguiente ejemplo. La lista de argumentos tiene que coincidir por el nombre con la lista de parámetros de la declaración de la función, pero pueden estar en orden aleatorio.

::

    pragma solidity ^0.4.0;

    contract C {
        function f(uint key, uint value) { ... }

        function g() {
            // named arguments
            f({value: 2, key: 3});
        }
    }

Nombres de parámetros de función omitidos
-----------------------------------------

Los nombres de parámetros no usados (especialmente los de retorno) se pueden omitir.
Esos nombres estarán presentes en la pila, pero serán inaccesibles.

::

    pragma solidity ^0.4.0;

    contract C {
        //Se omite el nombre para el parámetro
        function func(uint k, uint) returns(uint) {
            return k;
        }
    }
    

.. index:: ! nuevo, contratos;creación

.. _creating-contracts:

Creando contratos via ``new``
=============================

Un contrato puede crear un nuevo contrato usando la palabra reservada ``new``. El código completo del contrato que se está creando tiene que ser conocido de antemano, por lo que no son posibles las dependencias de creación recursivas.

::

    pragma solidity ^0.4.0;

    contract D {
        uint x;
        function D(uint a) payable {
            x = a;
        }
    }


    contract C {
        D d = new D(4); // Se ejecutará como parte del constructor de C

        function createD(uint arg) {
            D newD = new D(arg);
        }

        function createAndEndowD(uint arg, uint amount) {
            // Envía Ether junto con la creación
            D newD = (new D).value(amount)(arg);
        }
    }

Como se ve en el ejemplo, es posible traspasar Ether a la creación usando la opción ``.value()``,
pero no es posible limitar la cantidad de gas. Si la creación falla
(debido al desbordamiento de la pila, falta de balance o cualquier otro problema), se dispara una excepción.

Orden de la evaluación de expresiones
=====================================

El orden de evaluación de expresiones no se especifica (más formalmente, el orden en el que los hijos de un nodo en el árbol de la expresión son evaluados no es especificado. Eso sí, son evaluados antes que el propio nodo). Sólo se garantiza que las sentencias son ejecutadas en orden y que se hace un cortocircuito para las expresiones booleanas. Ver :ref:`order` para más información.

.. index:: ! asignación

Asignación
==========

.. index:: ! asignación;desestructurar

Asignaciones para desestructurar y retornar múltiples valores
-------------------------------------------------------------

Solidity internamente permite tipos tupla, p.ej.: una lista de objetos de , potencialmente, diferentes tipos cuyo tamaño es constante en tiempo de compilación. Esas tuplas pueden ser usadas para retornar múltiples valores al mismo tiempo y, también, asignarlos a múltiples variables (o lista de valores en general) al mismo tiempo::

    contract C {
        uint[] data;

        function f() returns (uint, bool, uint) {
            return (7, true, 2);
        }

        function g() {
            //Declara y asigna variables. No es posible especificar el tipo de forma esplícita.
            var (x, b, y) = f();
            //Asigna a una variable pre-existente.
            (x, y) = (2, 7);
            // Truco común para intercambiar valores -- no funcoina con tipos de almacenamiento sin valor.
            (x, y) = (y, x);
            //Los componentes se pueden dejar fuera (también en declaraciones de variables).
            //Si la tupla acaba en un componente vacío,
            //el resto de los valores se descartan.
            (data.length,) = f(); // Establece la longitud a 7
            // Lo mismo se puede hacer en el lado izquierdo.
            (,data[3]) = f(); // Sets data[3] to 2
            //Los componentes sólo se pueden dejar en el lado izquierdo de las asignaciones, con
            // una excepción:
            (x,) = (1,);
            // (1,) es la única forma de especificar una tupla de un componente, porque (1) 
            // equivale a 1.
        }
    }

Complicaciones en Arrays y Structs
----------------------------------

La sintaxis de asignación es algo más complicada por tipos sin valor como arrays y structs.
Las asignaciones *a* variables de estado siempre crean una copia independiente. Por otro lado, asignar una variable local crea sólo una copia independiente para tipos elementales, como tipos estáticos que casan en 32 bytes. Si los structs o arrays (incluyendo ``bytes`` y ``string``) son asignados desde una variable de estado a una local, la variable local se queda una referencia a la variable de estado original. Una segunda asignación a la variable local no modifica el estado, sólo cambia la referencia. Las asignaciones a miembros (o elementos) de la variable local *hacen* cambiar el estado.

.. index:: ! scoping, declaraciones, valor por defecto

.. _default-value:

Scoping and declaraciones
=========================

Una variable que es declarada tendrá un valor inicial por defecto cuyo valor, representado en bytes, será todo ceros.
Los valores por defecto de variables son los típicos "estado-cero" cualquiera que sea el tipo. Por ejemplo, el valor por defecto para un ``bool`` es ``false``. El valor por defecto para un ``uint`` o ``int`` es ``0``. Para arrays de tamaño estático y ``bytes1`` hasta ``bytes32``, cada elemento individual será inicializado a un valor por defecto según sea su tipo. Finalmente, para arrays de tamaño dinámico, ``bytes``y ``string``, el valor por defecto es un array o string vacio.

Una variable declarada en cualquier punto de una función, estará dentro del alcance de *toda la función*, independientemente de donde se haya declarado. Esto ocurre porque Solidity hereda sus reglas de scoping de JavaScript.
Esto difiere de muchos lenguajes donde las variables sólo están en el alcance de donde se declaran hasta que acaba el bloque semántico.
Como consecuencia de esto, el código siguiente es ilegal y hace que el compilador devuelva un error porque el identificador se ha declarado previamente, ``Identifier already declared``::

    pragma solidity ^0.4.0;

    contract ScopingErrors {
        function scoping() {
            uint i = 0;

            while (i++ < 1) {
                uint same1 = 0;
            }

            while (i++ < 2) {
                uint same1 = 0;// Ilegal, seguna declaración para same1
            }
        }

        function minimalScoping() {
            {
                uint same2 = 0;
            }

            {
                uint same2 = 0;// Ilegal, seguna declaración para same2
            }
        }

        function forLoopScoping() {
            for (uint same3 = 0; same3 < 1; same3++) {
            }

            for (uint same3 = 0; same3 < 1; same3++) {// Ilegal, seguna declaración para same3
            }
        }
    }

Como añadido a esto, si la variable se declara, se inicializará al principio de la función con su valor por defecto.
Esto significa que el siguiente código es legal, aunque se haya escrito de manera un tanto pobre::

    function foo() returns (uint) {
        // baz se inicializa implícitamente a 0
        uint bar = 5;
        if (true) {
            bar += baz;
        } else {
            uint baz = 10;// Nunca se ejecuta
        }
        return bar;// devuelve 5
    }

.. index:: ! exception, ! throw

Exceptions
==========

There are some cases where exceptions are thrown automatically (see below). You can use the ``throw`` instruction to throw an exception manually. The effect of an exception is that the currently executing call is stopped and reverted (i.e. all changes to the state and balances are undone) and the exception is also "bubbled up" through Solidity function calls (exceptions are ``send`` and the low-level functions ``call``, ``delegatecall`` and ``callcode``, those return ``false`` in case of an exception).

Catching exceptions is not yet possible.

In the following example, we show how ``throw`` can be used to easily revert an Ether transfer and also how to check the return value of ``send``::

    pragma solidity ^0.4.0;

    contract Sharer {
        function sendHalf(address addr) payable returns (uint balance) {
            if (!addr.send(msg.value / 2))
                throw; // also reverts the transfer to Sharer
            return this.balance;
        }
    }

Currently, Solidity automatically generates a runtime exception in the following situations:

#. If you access an array at a too large or negative index (i.e. ``x[i]`` where ``i >= x.length`` or ``i < 0``).
#. If you access a fixed-length ``bytesN`` at a too large or negative index.
#. If you call a function via a message call but it does not finish properly (i.e. it runs out of gas, has no matching function, or throws an exception itself), except when a low level operation ``call``, ``send``, ``delegatecall`` or ``callcode`` is used.  The low level operations never throw exceptions but indicate failures by returning ``false``.
#. If you create a contract using the ``new`` keyword but the contract creation does not finish properly (see above for the definition of "not finish properly").
#. If you divide or modulo by zero (e.g. ``5 / 0`` or ``23 % 0``).
#. If you shift by a negative amount.
#. If you convert a value too big or negative into an enum type.
#. If you perform an external function call targeting a contract that contains no code.
#. If your contract receives Ether via a public function without ``payable`` modifier (including the constructor and the fallback function).
#. If your contract receives Ether via a public getter function.
#. If you call a zero-initialized variable of internal function type.
#. If a ``.transfer()`` fails.
#. If you call ``assert`` with an argument that evaluates to false.

While a user-provided exception is generated in the following situations:

#. Calling ``throw``.
#. Calling ``require`` with an argument that evaluates to ``false``.

Internally, Solidity performs a revert operation (instruction ``0xfd``) when a user-provided exception is thrown or the condition of
a ``require`` call is not met. In contrast, it performs an invalid operation
(instruction ``0xfe``) if a runtime exception is encountered or the condition of an ``assert`` call is not met. In both cases, this causes
the EVM to revert all changes made to the state. The reason for this is that there is no safe way to continue execution, because an expected effect
did not occur. Because we want to retain the atomicity of transactions, the safest thing to do is to revert all changes and make the whole transaction
(or at least call) without effect.

If contracts are written so that ``assert`` is only used to test internal conditions and ``require``
is used in case of malformed input, a formal analysis tool that verifies that the invalid
opcode can never be reached can be used to check for the absence of errors assuming valid inputs.
