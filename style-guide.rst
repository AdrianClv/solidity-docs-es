.. index:: style, coding style

##############
Guía de estilo
##############

************
Introducción
************

Esta guía pretende proporcionar convenciones de codificación para escribir código con Solidity.
Esta guía debe ser entendida como un documento en evolución que cambiará con el tiempo según aparecen nuevas convenciones útiles y antiguas convenciones se vuelven obsoletas.

Muchos proyectos implementarán sus propias guías de estilo. En el caso de conflictos, las guías de estilo específicas del proyecto tendrán prioridad.

La estructura y muchas de las recomendaciones de esta guía de estilo fueron tomadas de Python: `guía de estilo pep8 <https://www.python.org/dev/peps/pep-0008/>`_.

El objetivo de esta guía *no* es ser la forma correcta o la mejor manera de escribir código con Solidity. El objetivo de esta guía es la *consistencia*. Una cita de python `pep8 <https://www.python.org/dev/peps/pep-0008/#a-foolish-consistency-is-the-hobgoblin-of-little-minds>`_ capta bien este concepto.

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

Dentro de un contrato, rodee las declaraciones de una función con una sola línea en blanco.

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

Se prefiere la codificación del texto en UTF-8 o ASCII.

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
- función fallback (Si existe)
- external
- public
- internal
- private

Dentro de un grupo, coloque las funciones ``constant`` al final.

Sí::

    contract A {
        function A() {
            ...
        }

        function() {
            ...
        }

        // Funciones external
        // ...

        // Funciones external que son constantes
        // ...

        // Funciones public
        // ...

        // Funciones internal
        // ...

        // Funciones private
        // ...
    }

No::

    contract A {

        // Funciones external
        // ...

        // Funciones private
        // ...

        // Funciones public
        // ...

        function A() {
            ...
        }

        function() {
            ...
        }

        // Funciones internal
        // ...       
    }

Espacios en blanco en expresiones
=================================

Evite los espacios en blanco superfluos en las siguientes situaciones:

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

Más de un espacio alrededor de una asignación u otro operador para alinearlo con otro:

Sí::

    x = 1;
    y = 2;
    long_variable = 3;

No::

    x             = 1;
    y             = 2;
    long_variable = 3;

No incluya un espacio en blanco en la función fallback:

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

Las llaves que denotan el cuerpo de un contrato, biblioteca, funciones y estructuras deberán:

* Abrir en la misma línea que la declaración
* Cerrar en su propia línea en el mismo nivel de sangría que la declaración.
* La llave de apertura debe ser procedida por un solo espacio.

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

Las mismas recomendaciones se aplican a las estructuras de control ``if``, ``else``, ``while`` y ``for``.

Además, debería existir un único espacio entre las estructuras de control ``if``, ``while``, y ``for``, y el bloque entre paréntesis que representa el condicional, así como un único espacio entre el bloque del paréntesis condicional y la llave de apertura.

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

Para las estructuras de control cuyo cuerpo sólo contiene declaraciones únicas, se pueden omitir los corchetes *si* la declaración cabe en una sola línea.

Sí::

    if (x < 10)
        x += 1;

No::

    if (x < 10)
        someArray.push(Coin({
            name: 'spam',
            value: 42
        }));

Para los bloques ``if`` que contienen una condición ``else`` o ``else if``, el ``else`` debe estar en la misma línea que el corchete de cierre del ``if``. Esto es una excepción en comparación con las reglas de otras estructuras de tipo bloque.

Sí::

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

Declaración de funciones
========================

Para declaraciones de función cortas, se recomienda dejar el corchete de apertura del cuerpo de la función en la misma línea que la declaración de la función.

El corchete de cierre debe estar al mismo nivel de sangría que la declaración de la función.

El corchete de apertura debe estar precedido por un solo espacio.

Sí::

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

Se debe especificar la visibilidad de los modificadores para una función antes de cualquier modificador personalizado.

Sí::

    function kill() public onlyowner {
        selfdestruct(owner);
    }

No::

    function kill() onlyowner public {
        selfdestruct(owner);
    }

Para las declaraciones de función largas, se recomienda dejar a cada argumento su propia línea al mismo nivel de sangría que el cuerpo de la función. El paréntesis de cierre y el corchete de apertura deben de estar en su propia línea también y con el mismo nivel de sangría que la declaración de la función.

Sí::

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

Si una declaración de función larga tiene modificadores, cada uno de ellos debe de estar en su propia línea.

Sí::

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

Para las funciones de tipo constructor en contratos heredados que requieren argumentos, si la declaración de la función es larga o difícil de leer, se recomienda poner cada constructor base en su propia línea de la misma manera que con los modificadores.

Sí::

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

Cuando se declara funciones cortas con una sola declaración, está permitido hacerlo en una solo línea.

Permisible::

    function shortFunction() { doSomething(); }

Esta guía sobre la declaración de funciones está pensada para mejorar la legibilidad. Sin embargo, los autores deberían utilizar su mejor juicio, ya que esta guía tampoco intenta cubrir todas las posibles permutaciones para las declaraciones de función.

Mapeo
=====

Pendiente de hacer

Declaración de variables
========================

La declaración de variables tipo array no deben incluir un espacio entre el tipo y el corchete.

Sí::

    uint[] x;

No::

    uint [] x;


Otras recommendaciones
======================

* Los strings deben de citarse con comillas dobles en lugar de comillas simples.

Sí::

    str = "foo";
    str = "Hamlet says, 'To be or not to be...'";

No::

    str = 'bar';
    str = '"Be yourself; everyone else is already taken." -Oscar Wilde';

* Se envuelve los operadores con un solo espacio de cada lado.

Sí::

    x = 3;
    x = 100 / 10;
    x += 3 + 4;
    x |= y && z;

No::

    x=3;
    x = 100/10;
    x += 3+4;
    x |= y&&z;

* Para los operadores con una prioridad mayor que otros, se pueden omitir los espacios de cada lado del operador para marcar la precedencia. Esto se hace para mejorar la legibilidad de declaraciones complejas. Se debe usar siempre el mismo número de espacios a cada lado de un operador.

Sí::

    x = 2**3 + 5;
    x = 2*y + 3*z;
    x = (a+b) * (a-b);

No::

    x = 2** 3 + 5;
    x = y+z;
    x +=1;


************************
Convención sobre nombres
************************

Las convenciones sobre nombres son extremadamente útiles siempre y cuando se usen de forma ámplia. El uso de diferentes convenciones puede transmitir *meta* información significativa a la que de otro modo no tendríamos acceso de manera inmediata.

Las recomendaciones de nombres que se dan aquí están pensadas para mejorar la legibilidad, y por lo tanto no se deben considerar como reglas. Son más bien una guía para intentar transmitir la mayor información posible a través del nombre de las cosas.

Finalmente, la consistencia dentro de un bloque de código siempre debe prevalecer sobre cualquier convención destacada en este documento.


Estilos para poner nombres
==========================

Para evitar confusiones, se usarán los siguiente nombres para referirse a diferentes estilos para poner nombres.

* ``b`` (letra minúscula única)
* ``B`` (letra mayúscula única)
* ``minuscula``
* ``minuscula_con_guiones_bajos``
* ``MAYUSCULA``
* ``MAYUSCULA_CON_GUIONES_BAJOS``
* ``PalabrasConLaInicialEnMayuscula`` (también llamado CapWords)
* ``mezclaEntreMinusculaYMayuscula`` (¡distinto a PalabrasConLaInicialEnMayuscula por el uso de una minúscula en la letra inicial!)
* ``Palabras_Con_La_Inicial_En_Mayuscula_Y_Guiones_Bajos``

.. note:: Cuando se usan abreviaciones en CapWords, usar mayúsculas para todas las letras de la abreviación. Es decir que HTTPServerError es mejor que HttpServerError


Nombres a evitar
================

* ``l`` - Letra minúscula ele
* ``O`` - Letra mayúscula o
* ``I`` - Letra mayúscula i

No usar jamás ninguna de estas letras únicas para nombrar una variable. Estas letras generalmente no se diferencian de los dígitos uno y cero.


Contratos y librerías de nombres
================================

Contratos y librerías deben de nombrarse usando el estilo CapWord (PalabrasConLaInicialEnMayuscula).


Eventos
=======

Los eventos deben de nombrarse usando el estilo CapWord (PalabrasConLaInicialEnMayuscula).


Nombres para funciones
======================

Las funciones deben de nombrarse usando el estilo mezclaEntreMinusculaYMayuscula.


Argumentos de funciones
=======================

Cuando se escriben funciones de librerías que operan sobre un struct personalizado, el struct debe ser el primer argumento y debe nombrarse siempre ``self``.


Variables locales y de estado
=============================

Usar el estilo mezclaEntreMinusculaYMayuscula.


Constantes
==========

Las constantes deben de nombrarse con todas las letras mayúsculas y guiones bajos para separar las palabras (p.ej. ``MAX_BLOCKS``).


Modificadores
=============

Usar el estilo mezclaEntreMinusculaYMayuscula.


Evitar Colisiones
=================

* ``guion_bajo_unico_con_cola_``

Se recomienda usar esta convención cuando el nombre deseado colisiona con un nombre inherente al sistema o reservado.


Recommendaciones generales
==========================

Pendiente de hacer
