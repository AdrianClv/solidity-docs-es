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
            // do something with _a and _b.
        }
    }

Parámetros de salida
--------------------

Los parámetros de salida se pueden declarar con la misma sintaxis después de la palabra reservada ``returns``. Por ejemplo, supongamos que deseamos devolver dos resultados:
La suma y el producto de dos valores dados, entonces escribiríamos un código como este::

    contract Simple {
        function arithmetics(uint _a, uint _b) returns (uint o_sum, uint o_product) {
            o_sum = _a + _b;
            o_product = _a * _b;
        }
    }

Los nombres de los parámetros de salida se pueden omitir.
Los valores de saluda se pueden especificar también usando sentencias ``return``.
Las sentencias ``return`` también son capaces de devolver múltiples valores, ver :ref:`multi-return`.
Los parámetros de retorno se inicializan a cero; si no se especifica esxplícitamente su valor, permanecen con dicho valor cero.

Los parámetros de entrada y salida se pueden usar como expresiones en el cuerpo de la función. En este caso, también pueden ir en el lado izquierdo de una asignación.

.. index:: if, else, while, do/while, for, break, continue, return, switch, goto

Estructuras de control
======================

La mayoría de las estructuras de controlos disponibles en JavaScript, también lo están en Solidity exceptuando ``switch`` y ``goto``. Esto significa que tenemos: ``if``, ``else``, ``while``, ``do``, ``for``, ``break``, ``continue``, ``return``, ``? :``, con la semántica habitual conocida de C o JavaScript.

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

Las funciones del conrtrato actual pueden ser llamadas directamente("internamente"), y también recursivamente, como se puede ver en este ejemplo sin sentido::

    contract C {
        function g(uint a) returns (uint ret) { return f(); }
        function f() returns (uint ret) { return g(7) + f(); }
    }

Estas llamadas de función son traducidas en simples saltos dentro de la máquina virtual de Ethereum (EVM). Esto tiene como consecuencia que la memoria actual no se limpia, p.ej.: pasando referencias de memoria a las funciones internamente llamadas es muy eficiente. Sólo las funciones del mismo contrato pueden ser llamadas internamente.

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

The modifier ``payable`` has to be used for ``info``, because otherwise, the `.value()`
option would not be available.

Note that the expression ``InfoFeed(addr)`` performs an explicit type conversion stating
that "we know that the type of the contract at the given address is ``InfoFeed``" and
this does not execute a constructor. Explicit type conversions have to be
handled with extreme caution. Never call a function on a contract where you
are not sure about its type.

We could also have used ``function setFeed(InfoFeed _feed) { feed = _feed; }`` directly.
Be careful about the fact that ``feed.info.value(10).gas(800)``
only (locally) sets the value and amount of gas sent with the function call and only the
parentheses at the end perform the actual call.

Function calls cause exceptions if the called contract does not exist (in the
sense that the account does not contain code) or if the called contract itself
throws an exception or goes out of gas.

.. warning::
    Any interaction with another contract imposes a potential danger, especially
    if the source code of the contract is not known in advance. The current
    contract hands over control to the called contract and that may potentially
    do just about anything. Even if the called contract inherits from a known parent contract,
    the inheriting contract is only required to have a correct interface. The
    implementation of the contract, however, can be completely arbitrary and thus,
    pose a danger. In addition, be prepared in case it calls into other contracts of
    your system or even back into the calling contract before the first
    call returns. This means
    that the called contract can change state variables of the calling contract
    via its functions. Write your functions in a way that, for example, calls to
    external functions happen after any changes to state variables in your contract
    so your contract is not vulnerable to a reentrancy exploit.

Named Calls and Anonymous Function Parameters
---------------------------------------------

Function call arguments can also be given by name, in any order,
if they are enclosed in ``{ }`` as can be seen in the following
example. The argument list has to coincide by name with the list of
parameters from the function declaration, but can be in arbitrary order.

::

    pragma solidity ^0.4.0;

    contract C {
        function f(uint key, uint value) { ... }

        function g() {
            // named arguments
            f({value: 2, key: 3});
        }
    }

Omitted Function Parameter Names
--------------------------------

The names of unused parameters (especially return parameters) can be omitted.
Those names will still be present on the stack, but they are inaccessible.

::

    pragma solidity ^0.4.0;

    contract C {
        // omitted name for parameter
        function func(uint k, uint) returns(uint) {
            return k;
        }
    }
    

.. index:: ! new, contracts;creating

.. _creating-contracts:

Creating Contracts via ``new``
==============================

A contract can create a new contract using the ``new`` keyword. The full
code of the contract being created has to be known in advance, so recursive
creation-dependencies are not possible.

::

    pragma solidity ^0.4.0;

    contract D {
        uint x;
        function D(uint a) payable {
            x = a;
        }
    }


    contract C {
        D d = new D(4); // will be executed as part of C's constructor

        function createD(uint arg) {
            D newD = new D(arg);
        }

        function createAndEndowD(uint arg, uint amount) {
            // Send ether along with the creation
            D newD = (new D).value(amount)(arg);
        }
    }

As seen in the example, it is possible to forward Ether to the creation using the ``.value()`` option,
but it is not possible to limit the amount of gas. If the creation fails
(due to out-of-stack, not enough balance or other problems), an exception
is thrown.

Order of Evaluation of Expressions
==================================

The evaluation order of expressions is not specified (more formally, the order
in which the children of one node in the expression tree are evaluated is not
specified, but they are of course evaluated before the node itself). It is only
guaranteed that statements are executed in order and short-circuiting for
boolean expressions is done. See :ref:`order` for more information.

.. index:: ! assignment

Assignment
==========

.. index:: ! assignment;destructuring

Destructuring Assignments and Returning Multiple Values
-------------------------------------------------------

Solidity internally allows tuple types, i.e. a list of objects of potentially different types whose size is a constant at compile-time. Those tuples can be used to return multiple values at the same time and also assign them to multiple variables (or LValues in general) at the same time::

    contract C {
        uint[] data;

        function f() returns (uint, bool, uint) {
            return (7, true, 2);
        }

        function g() {
            // Declares and assigns the variables. Specifying the type explicitly is not possible.
            var (x, b, y) = f();
            // Assigns to a pre-existing variable.
            (x, y) = (2, 7);
            // Common trick to swap values -- does not work for non-value storage types.
            (x, y) = (y, x);
            // Components can be left out (also for variable declarations).
            // If the tuple ends in an empty component,
            // the rest of the values are discarded.
            (data.length,) = f(); // Sets the length to 7
            // The same can be done on the left side.
            (,data[3]) = f(); // Sets data[3] to 2
            // Components can only be left out at the left-hand-side of assignments, with
            // one exception:
            (x,) = (1,);
            // (1,) is the only way to specify a 1-component tuple, because (1) is
            // equivalent to 1.
        }
    }

Complications for Arrays and Structs
------------------------------------

The semantics of assignment are a bit more complicated for non-value types like arrays and structs.
Assigning *to* a state variable always creates an independent copy. On the other hand, assigning to a local variable creates an independent copy only for elementary types, i.e. static types that fit into 32 bytes. If structs or arrays (including ``bytes`` and ``string``) are assigned from a state variable to a local variable, the local variable holds a reference to the original state variable. A second assignment to the local variable does not modify the state but only changes the reference. Assignments to members (or elements) of the local variable *do* change the state.

.. index:: ! scoping, declarations, default value

.. _default-value:

Scoping and Declarations
========================

A variable which is declared will have an initial default value whose byte-representation is all zeros.
The "default values" of variables are the typical "zero-state" of whatever the type is. For example, the default value for a ``bool``
is ``false``. The default value for the ``uint`` or ``int`` types is ``0``. For statically-sized arrays and ``bytes1`` to ``bytes32``, each individual
element will be initialized to the default value corresponding to its type. Finally, for dynamically-sized arrays, ``bytes``
and ``string``, the default value is an empty array or string.

A variable declared anywhere within a function will be in scope for the *entire function*, regardless of where it is declared.
This happens because Solidity inherits its scoping rules from JavaScript.
This is in contrast to many languages where variables are only scoped where they are declared until the end of the semantic block.
As a result, the following code is illegal and cause the compiler to throw an error, ``Identifier already declared``::

    pragma solidity ^0.4.0;

    contract ScopingErrors {
        function scoping() {
            uint i = 0;

            while (i++ < 1) {
                uint same1 = 0;
            }

            while (i++ < 2) {
                uint same1 = 0;// Illegal, second declaration of same1
            }
        }

        function minimalScoping() {
            {
                uint same2 = 0;
            }

            {
                uint same2 = 0;// Illegal, second declaration of same2
            }
        }

        function forLoopScoping() {
            for (uint same3 = 0; same3 < 1; same3++) {
            }

            for (uint same3 = 0; same3 < 1; same3++) {// Illegal, second declaration of same3
            }
        }
    }

In addition to this, if a variable is declared, it will be initialized at the beginning of the function to its default value.
As a result, the following code is legal, despite being poorly written::

    function foo() returns (uint) {
        // baz is implicitly initialized as 0
        uint bar = 5;
        if (true) {
            bar += baz;
        } else {
            uint baz = 10;// never executes
        }
        return bar;// returns 5
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
