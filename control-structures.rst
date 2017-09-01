####################################
Expresiones y estructuras de control
####################################

.. index:: ! parameter, parameter;input, parameter;output

Parámetros de entrada y de salida
=================================

Al igual que en Javascript, las funciones obtienen parámetros como entrada. Por otro lado, al contrario que en Javascript y C, estas también deberían devolver un número aleatorio de parámetros como salida.

Parámetros de entrada
---------------------

Los parámetros de entrada se declaran de la misma forma que las variables. Como una excepción, los parámetros no usados pueden omitir el nombre de la variable. Por ejemplo, si quisiéramos que nuestro contrato acepte un tipo de llamadas externas con dos enteros, el código quedaría similar a este::

    contract Simple {
        function taker(uint _a, uint _b) {
            // hace algo con _a y _b.
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
Los valores de salida se pueden especificar también usando declaraciones ``return``.
Las declaraciones ``return`` también son capaces de devolver múltiples valores, ver :ref:`multi-return`.
Los parámetros de retorno se inicializan a cero; si no se especifica explícitamente su valor, permanecen con dicho valor cero.

Los parámetros de entrada y salida se pueden usar como expresiones en el cuerpo de la función. En este caso, también pueden ir en el lado izquierdo de una asignación.

.. index:: if, else, while, do/while, for, break, continue, return, switch, goto

Estructuras de control
======================

La mayoría de las estructuras de control disponibles en JavaScript, también lo están en Solidity exceptuando ``switch`` y ``goto``. Esto significa que tenemos: ``if``, ``else``, ``while``, ``do``, ``for``, ``break``, ``continue``, ``return``, ``? :``, con la semántica habitual conocida de C o JavaScript.

Los paréntesis no se pueden omitir para condicionales, pero sí las llaves alrededor de los cuerpos de las declaraciones sencillas.

Hay que tener en cuenta que no hay conversión de tipos desde non-boolean a boolean como hay en C y JavaScript, por lo que ``if (1) { ... }`` *no* es válido en Solidity.

.. _multi-return:

Devolver múltiples valores
--------------------------

Cuando una función tiene múltiples parámetros de salida, ``return (v0, v1, ...,
vn)`` puede devolver múltiples valores. El número de componentes debe ser el mismo que el de parámetros de salida.

.. index:: ! function;call, function;internal, function;external

.. _function-calls:

Llamadas a funciones
====================

Llamadas a funciones internas
-----------------------------

Las funciones del contrato actual pueden ser llamadas directamente ("internamente") y, también, recursivamente como se puede ver en este ejemplo sin sentido funcional::

    contract C {
        function g(uint a) returns (uint ret) { return f(); }
        function f() returns (uint ret) { return g(7) + f(); }
    }

Estas llamadas a funciones son traducidas en simples saltos dentro de la máquina virtual de Ethereum (EVM). Esto tiene como consecuencia que la memoria actual no se limpia, así que pasar referencias de memoria a las funciones llamadas internamente es muy eficiente. Sólo las funciones del mismo contrato pueden ser llamadas internamente.

Llamadas a funciones externas
-----------------------------

Las expresiones ``this.g(8);`` and ``c.g(2);`` (donde ``c`` es la instancia de un contrato) son también llamadas válidas, pero en esta ocasión, la función se llamará "externamente" mediante un message call y no directamente por saltos.
Por favor, es importante tener en cuenta que las llamadas a funciones en ``this`` no pueden ser usadas en el constructor, ya que el contrato en cuestión no se ha creado todavía.

Las funciones de otros contratos se tienen que llamar de forma externa. Para una llamada externa,
todos los argumentos de la función tienen que ser copiados en memoria.

Cuando se llama a funciones de otros contratos, la cantidad de Wei enviada con la llamada y el gas pueden especificarse con las opciones especiales ``.value()`` y ``.gas()``, respectivamente::

    contract InfoFeed {
        function info() payable returns (uint ret) { return 42; }
    }


    contract Consumer {
        InfoFeed feed;
        function setFeed(address addr) { feed = InfoFeed(addr); }
        function callFeed() { feed.info.value(10).gas(800)(); }
    }

El modificador ``payable`` se tiene que usar para ``info``, porque de otra manera la opción `.value()` no estaría disponible.

Destacar que la expresión ``InfoFeed(addr)`` realiza una conversión de tipo explícita afirmando que "sabemos que el tipo de contrato en la dirección dada es ``InfoFeed``", sin ejecutar un constructor. Las conversiones de tipo explícitas tienen que ser gestionadas con extrema precaución. Nunca se debe llamar a una función en un contrato donde no se sabe con seguridad cuál es su tipo.

También se podría usar ``function setFeed(InfoFeed _feed) { feed = _feed; }`` directamente.
Hay que tener cuidado con el hecho de que ``feed.info.value(10).gas(800)`` sólo (localmente) establece el valor y la cantidad de gas enviada con la llamada a la función. Sólo tras el último paréntesis se realiza realmente la llamada.

Las llamadas a funciones provocan excepciones si el contrato invocado no existe (en el sentido de que la cuenta no contiene código) o si el contrato invocado por sí mismo dispara una excepción o se queda sin gas.

.. warning::
    Cualquier interacción con otro contrato supone un daño potencial, especialmente si el código fuente del contrato no se conoce de antemano. El contrato actual pasa el control al contrato invocado y eso potencialmente podría suponer que haga cualquier cosa. Incluso si el contrato invocado hereda de un contrato padre conocido, el contrato del que hereda sólo requiere tener una interfaz correcta. La implementación del contrato, sin embargo, puede ser totalmente aleatoria y, por ello, crear un perjuicio. Además, hay que estar preparado en caso de que llame dentro de otros contratos del sistema o, incluso, volver al contrato que lo llama antes de que la primera llamada retorne. Esto significa que el contrato invocado puede cambiar variables de estado del contrato que le llama mediante sus funciones. Escribir tus funciones de manera que realicen, por ejemplo, llamadas a funciones externas ocurridas después de cualquier cambio en variables de estado en tu contrato, hace que este contrato no sea vulnerable a un ataque de reentrada.
    

Llamadas con nombre y parámetros de funciones anónimas
------------------------------------------------------

Los argumentos de una llamada a una función pueden venir dados por el nombre, en cualquier orden, si están entre ``{ }`` como se puede ver en el siguiente ejemplo. La lista de argumentos tiene que coincidir por el nombre con la lista de parámetros de la declaración de la función, pero pueden estar en orden aleatorio.

::

    pragma solidity ^0.4.0;

    contract C {
        function f(uint key, uint value) { ... }

        function g() {
            // argumentos con nombre
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
        // Se omite el nombre para el parámetro
        function func(uint k, uint) returns(uint) {
            return k;
        }
    }
    

.. index:: ! new, contracts;creation

.. _creating-contracts:

Creando contratos mediante ``new``
==================================

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

El orden de evaluación de expresiones no se especifica (más formalmente, el orden en el que los hijos de un nodo en el árbol de la expresión son evaluados no es especificado. Eso sí, son evaluados antes que el propio nodo). Sólo se garantiza que las sentencias se ejecutan en orden y que se hace un cortocircuito para las expresiones booleanas. Ver :ref:`order` para más información.

.. index:: ! assignment

Asignación
==========

.. index:: ! assignment;destructuring

Asignaciones para desestructurar y retornar múltiples valores
-------------------------------------------------------------

Solidity internamente permite tipos tupla, p.ej.: una lista de objetos de, potencialmente, diferentes tipos cuyo tamaño es constante en tiempo de compilación. Esas tuplas pueden ser usadas para retornar múltiples valores al mismo tiempo y, también, asignarlos a múltiples variables (o lista de valores en general) también al mismo tiempo::

    contract C {
        uint[] data;

        function f() returns (uint, bool, uint) {
            return (7, true, 2);
        }

        function g() {
            // Declara y asigna variables. No es posible especificar el tipo de forma explícita.
            var (x, b, y) = f();
            // Asigna a una variable pre-existente.
            (x, y) = (2, 7);
            // Truco común para intercambiar valores -- no funciona con tipos de almacenamiento sin valor.
            (x, y) = (y, x);
            // Los componentes se pueden dejar fuera (también en declaraciones de variables).
            // Si la tupla acaba en un componente vacío,
            // el resto de los valores se descartan.
            (data.length,) = f(); // Establece la longitud a 7
            // Lo mismo se puede hacer en el lado izquierdo.
            (,data[3]) = f(); // Sets data[3] to 2
            // Los componentes sólo se pueden dejar en el lado izquierdo de las asignaciones, con
            // una excepción:
            (x,) = (1,);
            // (1,) es la única forma de especificar una tupla de un componente, porque (1) 
            // equivale a 1.
        }
    }

Complicaciones en Arrays y Structs
----------------------------------

La sintaxis de asignación es algo más complicada para tipos sin valor como arrays y structs.
Las asignaciones *a* variables de estado siempre crean una copia independiente. Por otro lado, asignar una variable local crea una copia independiente sólo para tipos elementales, como tipos estáticos que encajan en 32 bytes. Si los structs o arrays (incluyendo ``bytes`` y ``string``) son asignados desde una variable de estado a una local, la variable local se queda una referencia a la variable de estado original. Una segunda asignación a la variable local no modifica el estado, sólo cambia la referencia. Las asignaciones a miembros (o elementos) de la variable local *hacen* cambiar el estado.

.. index:: ! scoping, declarations, default value

.. _default-value:

Scoping y declaraciones
=========================

Una variable cuando se declara tendrá un valor inicial por defecto que, representado en bytes, será todo ceros.
Los valores por defecto de las variables son los típicos "estado-cero" cualquiera que sea el tipo. Por ejemplo, el valor por defecto para un ``bool`` es ``false``. El valor por defecto para un ``uint`` o ``int`` es ``0``. Para arrays de tamaño estático y ``bytes1`` hasta ``bytes32``, cada elemento individual será inicializado a un valor por defecto según sea su tipo. Finalmente, para arrays de tamaño dinámico, ``bytes``y ``string``, el valor por defecto es un array o string vacío.

Una variable declarada en cualquier punto de una función estará dentro del alcance de *toda la función*, independientemente de donde se haya declarado. Esto ocurre porque Solidity hereda sus reglas de scoping de JavaScript.
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
                uint same1 = 0; // Ilegal, segunda declaración para same1
            }
        }

        function minimalScoping() {
            {
                uint same2 = 0;
            }

            {
                uint same2 = 0; // Ilegal, segunda declaración para same2
            }
        }

        function forLoopScoping() {
            for (uint same3 = 0; same3 < 1; same3++) {
            }

            for (uint same3 = 0; same3 < 1; same3++) { // Ilegal, segunda declaración para same3
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

Excepciones
===========

Hay algunos casos en los que las excepciones se lanzan automáticamente (ver más adelante). Se puede usar la instrucción ``throw`` para lanzarlas manualmente. La consecuencia de una excepción es que la llamada que se está ejecutando en ese momento se para y se revierte (todos los cambios en los estados y balances se deshacen) y la excepción también se genera mediante llamadas de función de Solidity (las excepciones ``send`` y las funciones de bajo nivel ``call``, ``delegatecall`` y ``callcode``, todas ellas devuelven ``false`` en caso de una excepción).

Todavía no es posible capturar excepciones.

En el siguiente ejemplo, se enseña como ``throw`` se puede usar para revertir fácilmente una transferencia de Ether y, además, se enseña como comprobar el valor de retorno de ``send``::

    pragma solidity ^0.4.0;

    contract Sharer {
        function sendHalf(address addr) payable returns (uint balance) {
            if (!addr.send(msg.value / 2))
                throw; // También revierte la transferencia de Sharer
            return this.balance;
        }
    }

Actualmente, Solidity genera automáticamente una excepción en tiempo de ejecución en las siguientes situaciones:

#. Si se accede a un array en un índice demasiado largo o negativo (ejemplo: ``x[i]`` donde ``i >= x.length`` o ``i < 0``).
#. Si se accede a un ``bytesN`` de longitud fija en un índice demasiado largo o negativo.
#. Si se llama a una función con un message call, pero no finaliza adecuadamente (ejemplo: se queda sin gas, no tiene una función de matching, o dispara una excepción por sí mismo), exceptuando el caso en el que se use una operación de bajo nivel ``call``, ``send``, ``delegatecall`` o ``callcode``. Las operaciones de bajo nivel nunca disparan excepciones, pero indican fallos devolviendo ``false``.
#. Si se crea un contrato usando la palabra reservada ``new``, pero la creación del contrato no finaliza correctamente (ver más arriba la definición de "no finalizar correctamente").
#. Si se divide o se hace módulo por cero (ejemplos: ``5 / 0`` o ``23 % 0``).
#. Si se hace un movimiento por una cantidad negativa.
#. Si se convierte un valor muy grande o negativo en un tipo enum.
#. Si se realiza una llamada de función externa apuntando a un contrato que no contiene código.
#. Si un contrato recibe Ether mediante una función sin el modificador ``payable`` (incluyendo el constructor y la función de fallback).
#. Si un contrato recibe Ether mediante una función getter pública.
#. Si se llama a una variable inicializada a cero de un tipo de función interna.
#. Si un ``.transfer()`` falla.
#. Si se invoca con ``assert`` junto con un argumento que evalúa a falso.

Un usuario genera una exepcin en las siguientes situaciones:

#. Llamando a ``throw``.
#. Llamando a ``require`` junto con un argumento que evalúa a ``false``.

Internamente, Solidity realiza una operación de revertir (revert, instrucción ``0xfd``) cuando una excepción provista por un usuario se lanza o la condición de la llamada ``require`` no se satisface. Por contra, realiza una operación inválida (instrucción ``0xfe``) si una excepción en tiempo de ejecución aparece o la condición de una llamada ``assert`` no se satisface. En ambos casos, esto ocasiona que la EVM revierta todos los cambios de estado acaecidos. El motivo de todo esto es que no existe un modo seguro de continuar con la ejecución debido a que no sucedió el efecto esperado. Como se quiere mantener la atomicidad de las transacciones, lo más seguro es revertir todos los cambios y hacer que la transacción no tenga ningún efecto en su totalidad o, como mínimo, en la llamada.

En el caso de que los contratos se escriban de tal manera que ``assert`` sólo sea usado para probar condiciones internas y ``require`` se use en caso de que haya una entrada malformada, una herramienta de análisis formal que verifique que el opcode inválido nunca pueda ser alcanzado, se podría usar para chequear la ausencia de errores asumiendo entradas válidas.
