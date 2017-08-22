.. index:: style, coding style

##############
Guía de estilo
##############

************
Introducción
************

Esta guía pretende proporcionar convenciones de codificación para escribir código con Solidity.
Esta guía debe ser entendida como un documento en evolución que cambiará con el tiempo, mientras nuevas convenciones útiles se encuentran y antiguas convenciones se vuelven obsoletas.

Muchos proyectos implementarán sus propias guías de estilo. En el caso de conflictos, las guías de estilo específicas del proyecto tendran prioridad.

La estructura y muchas de las recomendaciones de esta guía de estilo fueron tomadas de Python: `pep8 style guide <https://www.python.org/dev/peps/pep-0008/>`_.

El objetivo de esta guía *no* es ser la forma correcta o la mejor manera de escribir código con Solidity. El objetivo de esta guía es *consistencia*. Una cita de python `pep8 <https://www.python.org/dev/peps/pep-0008/#a-foolish-consistency-is-the-hobgoblin-of-little-minds>`_ capta bien este concepto.

    Una guía de estilo es sobre consistencia. La consistencia con esta guía de estilo es importante. La consistencia dentro de un proyecto es más importante. La consistencia dentro de un módulo o función es lo más importante.
    Pero sobre todo: saber cuándo ser inconsistente - a veces la guía de estilo simplemente no se aplica. En caso de duda, use su mejor juicio. Mire otros ejemplos y decida qué parece mejor. ¡Y no dude en preguntar!


*****************
Diseño del código
*****************


Sangría
=======

Utilice 4 espacios por nivel de sangría.

Tabulador o espacios
====================

Los espacios son el método de indentación preferido.

Se deben evitar la mezcla del tabulador y los espacios.

Líneas en blanco
================

Envuelva las declaraciones de nivel superior en el código de Solidity con dos líneas en blanco.

Sí::

    contract A {
        ...
    }


    contract B {
        ...
    }


    contract C {
        ...
    }

No::

    contract A {
        ...
    }
    contract B {
        ...
    }

    contract C {
        ...
    }

Dentro de un contrato rodeé las declaraciones de una función con una sola línea en blanco.

Las líneas en blanco se pueden omitir entre grupos de una frase relacionada (tales como las funciones stub en un contrato abstracto)

Sí::

    contract A {
        function spam();
        function ham();
    }


    contract B is A {
        function spam() {
            ...
        }

        function ham() {
            ...
        }
    }

No::

    contract A {
        function spam() {
            ...
        }
        function ham() {
            ...
        }
    }

Codificación de archivos de origen
==================================

Se prefiere la codificación del texto en UTF-8 or ASCII.

Importación
===========

Las declaraciones de importación siempre deben colocarse en la parte superior del archivo.

Sí::

    import "owned";


    contract A {
        ...
    }


    contract B is owned {
        ...
    }

No::

    contract A {
        ...
    }


    import "owned";


    contract B is owned {
        ...
    }

Orden de funciones
==================

La ordenación ayuda a que los lectores puedan identificar las funciones que pueden invocar y encontrar las definiciones de constructor y de retorno más fácilmente.

Las funciones deben agruparse de acuerdo con su visibilidad y ser ordenadas de acuerdo a:

- constructor
- fallback function (Si existe)
- external
- public
- internal
- private

Dentro de un grupo, coloque las funciones `` constant`` de último.

Sí::

    contract A {
        function A() {
            ...
        }
        
        function() {
            ...
        }
        
        // External functions
        // ...
        
        // External functions that are constant
        // ...
        
        // Public functions
        // ...
        
        // Internal functions
        // ...
        
        // Private functions
        // ...
    }

No::

    contract A {
        
        // External functions
        // ...

        // Private functions
        // ...

        // Public functions
        // ...

        function A() {
            ...
        }
        
        function() {
            ...
        }

        // Internal functions
        // ...       
    }

Espacios en blanco en expresiones
=================================

Evite los espacios en blanco irrazonables en las siguientes situaciones:

Inmediatamente entre paréntesis, llaves o corchetes, con la excepción de declaraciones de una función en una sola línea.

Sí::

    spam(ham[1], Coin({name: "ham"}));

No::

    spam( ham[ 1 ], Coin( { name: "ham" } ) );

Excepción::

    function singleLine() { spam(); }

Inmediatamente antes de una coma, punto y coma:

Sí::

    function spam(uint i, Coin coin);

No::

    function spam(uint i , Coin coin) ;

Más de un espacio alrededor de una asignación u otro operador para alinearlo con
  otro:

Sí::

    x = 1;
    y = 2;
    long_variable = 3;

No::

    x             = 1;
    y             = 2;
    long_variable = 3;

No incluya un espacio en blanco en la función de segunda opción:

Sí::

    function() {
        ...
    }

No::
   
    function () {
        ...
    }

Estructuras de control
======================

Las llaves que denotan el cuerpo de un contrato, biblioteca, funciones y estructuras
deberán:

* Abrir en la misma línea que la declaración
* Cerrar en la misma línea en el mismo nivel de sangría que el
  declaración.
* La llaves de apertura deben ser procedidas por un solo espacio.

Sí::

    contract Coin {
        struct Bank {
            address owner;
            uint balance;
        }
    }

No::

    contract Coin
    {
        struct Bank {
            address owner;
            uint balance;
        }
    }

Las mismas recomendaciones se aplican a las estructuras de control ``if``, ``else``, ``while``,
y ``for``.

Además, debería existir un único espacio entre las estructuras de control ``if``, ``while``, y ``for`` Y el bloque entre paréntesis que representa el condicional, así como un único espacio entre el bloque del paréntesis condicional y la llave de apertura.

Sí::

    if (...) {
        ...
    }

    for (...) {
        ...
    }

No::

    if (...)
    {
        ...
    }

    while(...){
    }

    for (...) {
        ...;}

For control structures whose body contains a single statement, omitting the
braces is ok *if* the statement is contained on a single line.

Yes::

    if (x < 10)
        x += 1;

No::

    if (x < 10)
        someArray.push(Coin({
            name: 'spam',
            value: 42
        }));

For ``if`` blocks which have an ``else`` or ``else if`` clause, the ``else`` should be
placed on the same line as the ``if``'s closing brace. This is an exception compared
to the rules of other block-like structures.

Yes::

    if (x < 3) {
        x += 1;
    } else if (x > 7) {
        x -= 1;
    } else {
        x = 5;
    }


    if (x < 3)
        x += 1;
    else
        x -= 1;

No::

    if (x < 3) {
        x += 1;
    }
    else {
        x -= 1;
    }

Function Declaration
====================

For short function declarations, it is recommended for the opening brace of the
function body to be kept on the same line as the function declaration.

The closing brace should be at the same indentation level as the function
declaration.

The opening brace should be preceeded by a single space.

Yes::

    function increment(uint x) returns (uint) {
        return x + 1;
    }

    function increment(uint x) public onlyowner returns (uint) {
        return x + 1;
    }

No::

    function increment(uint x) returns (uint)
    {
        return x + 1;
    }

    function increment(uint x) returns (uint){
        return x + 1;
    }

    function increment(uint x) returns (uint) {
        return x + 1;
        }

    function increment(uint x) returns (uint) {
        return x + 1;}

The visibility modifiers for a function should come before any custom
modifiers.

Yes::

    function kill() public onlyowner {
        selfdestruct(owner);
    }

No::

    function kill() onlyowner public {
        selfdestruct(owner);
    }

For long function declarations, it is recommended to drop each argument onto
it's own line at the same indentation level as the function body.  The closing
parenthesis and opening bracket should be placed on their own line as well at
the same indentation level as the function declaration.

Yes::

    function thisFunctionHasLotsOfArguments(
        address a,
        address b,
        address c,
        address d,
        address e,
        address f
    ) {
        doSomething();
    }

No::

    function thisFunctionHasLotsOfArguments(address a, address b, address c,
        address d, address e, address f) {
        doSomething();
    }

    function thisFunctionHasLotsOfArguments(address a,
                                            address b,
                                            address c,
                                            address d,
                                            address e,
                                            address f) {
        doSomething();
    }

    function thisFunctionHasLotsOfArguments(
        address a,
        address b,
        address c,
        address d,
        address e,
        address f) {
        doSomething();
    }

If a long function declaration has modifiers, then each modifier should be
dropped to it's own line.

Yes::

    function thisFunctionNameIsReallyLong(address x, address y, address z)
        public
        onlyowner
        priced
        returns (address)
    {
        doSomething();
    }

    function thisFunctionNameIsReallyLong(
        address x,
        address y,
        address z,
    )
        public
        onlyowner
        priced
        returns (address)
    {
        doSomething();
    }

No::

    function thisFunctionNameIsReallyLong(address x, address y, address z)
                                          public
                                          onlyowner
                                          priced
                                          returns (address) {
        doSomething();
    }

    function thisFunctionNameIsReallyLong(address x, address y, address z)
        public onlyowner priced returns (address)
    {
        doSomething();
    }

    function thisFunctionNameIsReallyLong(address x, address y, address z)
        public
        onlyowner
        priced
        returns (address) {
        doSomething();
    }

For constructor functions on inherited contracts whose bases require arguments,
it is recommended to drop the base constructors onto new lines in the same
manner as modifiers if the function declaration is long or hard to read.

Yes::

    contract A is B, C, D {
        function A(uint param1, uint param2, uint param3, uint param4, uint param5)
            B(param1)
            C(param2, param3)
            D(param4)
        {
            // do something with param5
        }
    }

No::

    contract A is B, C, D {
        function A(uint param1, uint param2, uint param3, uint param4, uint param5)
        B(param1)
        C(param2, param3)
        D(param4)
        {
            // do something with param5
        }
    }

    contract A is B, C, D {
        function A(uint param1, uint param2, uint param3, uint param4, uint param5)
            B(param1)
            C(param2, param3)
            D(param4) {
            // do something with param5
        }
    }

When declaring short functions with a single statement, it is permissible to do it on a single line.

Permissible::

    function shortFunction() { doSomething(); }

These guidelines for function declarations are intended to improve readability.
Authors should use their best judgement as this guide does not try to cover all
possible permutations for function declarations.

Mappings
========

TODO

Variable Declarations
=====================

Declarations of array variables should not have a space between the type and
the brackets.

Yes::

    uint[] x;

No::

    uint [] x;


Other Recommendations
=====================

* Strings should be quoted with double-quotes instead of single-quotes.

Yes::

    str = "foo";
    str = "Hamlet says, 'To be or not to be...'";

No::

    str = 'bar';
    str = '"Be yourself; everyone else is already taken." -Oscar Wilde';

* Surround operators with a single space on either side.

Yes::

    x = 3;
    x = 100 / 10;
    x += 3 + 4;
    x |= y && z;

No::

    x=3;
    x = 100/10;
    x += 3+4;
    x |= y&&z;

* Operators with a higher priority than others can exclude surrounding
  whitespace in order to denote precedence.  This is meant to allow for
  improved readability for complex statement. You should always use the same
  amount of whitespace on either side of an operator:

Yes::

    x = 2**3 + 5;
    x = 2*y + 3*z;
    x = (a+b) * (a-b);

No::

    x = 2** 3 + 5;
    x = y+z;
    x +=1;


******************
Naming Conventions
******************

Naming conventions are powerful when adopted and used broadly.  The use of
different conventions can convey significant *meta* information that would
otherwise not be immediately available.

The naming recommendations given here are intended to improve the readability,
and thus they are not rules, but rather guidelines to try and help convey the
most information through the names of things.

Lastly, consistency within a codebase should always supercede any conventions
outlined in this document.


Naming Styles
=============

To avoid confusion, the following names will be used to refer to different
naming styles.

* ``b`` (single lowercase letter)
* ``B`` (single uppercase letter)
* ``lowercase``
* ``lower_case_with_underscores``
* ``UPPERCASE``
* ``UPPER_CASE_WITH_UNDERSCORES``
* ``CapitalizedWords`` (or CapWords)
* ``mixedCase`` (differs from CapitalizedWords by initial lowercase character!)
* ``Capitalized_Words_With_Underscores``

.. note:: When using abbreviations in CapWords, capitalize all the letters of the abbreviation. Thus HTTPServerError is better than HttpServerError.


Names to Avoid
==============

* ``l`` - Lowercase letter el
* ``O`` - Uppercase letter oh
* ``I`` - Uppercase letter eye

Never use any of these for single letter variable names.  They are often
indistinguishable from the numerals one and zero.


Contract and Library Names
==========================

Contracts and libraries should be named using the CapWords style.


Events
======

Events should be named using the CapWords style.


Function Names
==============

Functions should use mixedCase.


Function Arguments
==================

When writing library functions that operate on a custom struct, the struct
should be the first argument and should always be named ``self``.


Local and State Variables
=========================

Use mixedCase.


Constants
=========

Constants should be named with all capital letters with underscores separating
words.  (for example:``MAX_BLOCKS``)


Modifiers
=========

Use mixedCase.


Avoiding Collisions
===================

* ``single_trailing_underscore_``

This convention is suggested when the desired name collides with that of a
built-in or otherwise reserved name.


General Recommendations
=======================

TODO
