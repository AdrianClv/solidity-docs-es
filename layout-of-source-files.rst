********************************************
Composición de un fichero fuente de Solidity
********************************************

Los ficheros fuente pueden contener un número arbitrario de definiciones de contrato, incluir directivas y directivas de pragma.

.. index:: ! pragma, version

.. _version_pragma:

Versión de Pragma
=================

Es altamente recomendable anotar los ficheros fuente con la versión de pragma para impedir que se compilen con una version posterior del compilador que podría introducir cambios incompatibles. Intentamos mantener este tipo de cambios al mínimo y que los cambios que introduzcamos y modifiquen la semántica requieran también un cambio en la ***sintáxis, pero esto no siempre es posible. Por ese motivo, siempre es buena idea repasar el registro de cambios por lo menos para las nuevas versiones que contienen cambios de ruptura. Estas nuevas versiones siempre tendrán un número de versión del tipo ``0.x.0`` o ``x.0.0``.

La versión de pragma se usa de la siguiente manera:

  pragma solidity ^0.4.0;

Un fichero como este no se compilará con un compilador con una versión anterior a ``0.4.0`` y tampoco funcionará con un compilador que tiene una versión posterior a ``0.5.0`` (se especifíca esta segunda condición usando el ``^``). La idea detrás de esto es que no va a haber un cambio de ruptura antes de la versión ``0.5.0``, así que podemos estar seguro de que nuestro código se compilará de la manera en que nosotros esperamos. No fijamos la versión exacta del compilador, de manera que podemos liberar nuevas versiones que corrigen bugs.

Se puede especificar reglas mucho más complejas para la versión del compilador, la expresión sigue a las utilizadas por `npm <https://docs.npmjs.com/misc/semver>`_.

.. index:: source file, ! import

.. _import:

Importar otros ficheros fuente
==============================

Sintáxis y semántica
--------------------

Solidity soporta la importación de declaraciones, que son muy similares a las que se hacen en JavaScript (a partir de ES6), aunque Solidity no conoce el concepto de "default export".

A nivel global, se puede usar la importación de declaraciones de la siguiente manera:

::

  import "filename";

Esta declaración importa todos los símbolos globales de "filename" (junto con los símbolos importados desde allí) en el alcance global actual (diferente que en ES6 pero compatible con versiones anteriores de Solidity).

::

  import * as symbolName from "filename";

...crea un nuevo símbolo global ``symbolName`` cuyos miembros son todos los símbolos globales de ``"filename"``.

::

  import {symbol1 as alias, symbol2} from "filename";

...crea un nuevo símbolo global ``alias`` y ``symbol2`` que referencia respectivamente ``symbol1`` y ``symbol2`` desde ``"filename"``.

Esta otra sintaxis no forma parte de ES6 pero es probablemente conveniente:

::

  import "filename" as symbolName;

lo que es equivalente a ``import * as symbolName from "filename";``.

Ruta
----

En lo que hemos visto más arriba, ``filename`` siempre se trata como una ruta con el ``/`` como separador de directorio, ``.`` como el directorio actual y ``..`` como el directorio ***padre. Cuando ``.`` o ``..`` es seguido por un carácter excepto ``/``, no se considera como el directorio actual o directorio padre. Se tratan todos los nombres de ruta como rutas absolutas, a no ser que empiecen por ``.`` o el directorio padre ``..``.

Para importar un fichero ``x`` desde el mismo directorio que el fichero actual, se usa `import "./x" as x;``. Si en lugar de esa expresión se usa ``import "x" as x;``, podría ser que se referencie un fichero distinto (***en un "include directory" global).

Cómo se resuelve la ruta depende del compilador (ver más abajo). En general, la jerarquía de directorios no necesita mapear estrictamente su sistema local de ficheros, también puede mapear recursos que se descubren con por ejemplo ipfs, http or git.

Uso en compiladores actuales
----------------------------

Cuando se invoca el compilador, no sólo se puede especificar la manera en que se descubre el primer elemento de la ruta, también es posible especificar un prefijo de ruta de remapeo, de tal manera que por ejemplo ``github.com/ethereum/dapp-bin/library`` se remapee por ``/usr/local/dapp-bin/library`` y el compilador lea el fichero desde allí. Si bien se pueden hacer múltiples remapeos, se intentará primero con el remapeo con la clave más larga. Esto permite "fallback-remapping" con, por ejemplo, ``""`` mapea a ``/usr/local/include/solidity``. Además, estos remapeos pueden depender del contexto, lo que permite configurar paquetes para importar por ejemplo diferentes versiones de una librería con el mismo nombre.

**solc**:

Para solc (el compilador de línea de comando), los remapeos se proporcionan como argumentos ``context:prefix=target``, donde tanto la parte ``context:`` como la parte ``=target`` son opcionales (en este caso target toma por defecto el valor de prefix). Todos los valores que remapean que son ficheros estándares son compilados (incluyendo sus dependencias). Este mecanismo es completamente compatible con versiones anteriores (siempre y cuando ningún nombre de fichero contenga = o :) y por lo tanto no es un cambio de ruptura. Todas las importaciones en los ficheros en el directorio ``context`` (o debajo de el) que importa un fichero que empieza con un ``prefix`` están redireccionados reemplazando ``prefix`` por ``target``.

Como ejemplo, si clonas ``github.com/ethereum/dapp-bin/`` en tu local a ``/usr/local/dapp-bin``, puedes usar lo siguiente en tu código fuente:

::

  import "github.com/ethereum/dapp-bin/library/iterable_mapping.sol" as it_mapping;

y luego se corre el compilador de esa manera:

.. code-block:: bash

  solc github.com/ethereum/dapp-bin/=/usr/local/dapp-bin/ source.sol

Como ejemplo un poco más complejo, imaginemos que nos basamos en algún módulo que utiliza un versión muy antigua de dapp-bin. Esta versión anterior de dapp-bin se comprueba en ``/usr/local/dapp-bin_old``, y luego se puede usar:

.. code-block:: bash

  solc module1:github.com/ethereum/dapp-bin/=/usr/local/dapp-bin/ \
       module2:github.com/ethereum/dapp-bin/=/usr/local/dapp-bin_old/ \
       source.sol

así, todas las importaciones en ``module2`` apuntan a la versión anterior pero las importaciones en ``module1`` consiguen la nueva versión.

Nótese que solc sólo permite incluir ficheros desde algunos directorios. Tienen que estar en el diretorio (o subdirectorio) de uno de los ficheros fuente explícitamente especificado o en el directorio (o subdirectorio) de un ***target de remapeo. Si se desea permitir inclusiones directas absolutas, sólo hace falta añadir el ``=/`` de remapeo.

Si hay múltiples remapeos que conducen a un fichero válido, se elige el remapeo con el prefijo común más largo.

**Remix**:

`Remix <https://remix.ethereum.org/>`_ proporciona un remapeo automático para github y también recupera automáticamente el fichero desde la red: se puede importar el mapeo iterable con por ejemplo ``import "github.com/ethereum/dapp-bin/library/iterable_mapping.sol" as it_mapping;``.

A futuro se podrían añadir otros proveedores de código fuente.


.. index:: ! comment, natspec

Comentarios
===========

Se aceptan los comentarios de línea simple (``//``) y los comentarios de múltiples líneas (``/*...*/``).

::

  // Esto es un comentario de simple línea.

  /*
  Esto es un comentario
  de múltiples líneas.
  */


Adicionalmente, existe otro tipo de comentario llamado natspec, pero la documentación todavía no está escrita. Estos comentarios se escriben con tres barras ***óblicas (``///``) o como un bloque de comentarios con doble asterísco (``/** ... */``) y se deben usar justo arriba de las declaraciones de función o instrucciones. Para documentar funciones, anotar condiciones para la verificación formal y para proporcionar un **texto de confirmation** al usuario cuando intentan invocar una función, se puede usar etiquetas del tipo `Doxygen <https://en.wikipedia.org/wiki/Doxygen>`_ dentro de estos comentarios.

En el siguiente ejemplo, documentamos el título del contrato, la explicación para los dos parametros de entrada y para los dos valores de retorno.

::

    pragma solidity ^0.4.0;

 /** @title Shape calculator.*/
 contract shapeCalculator{
     /**@dev Calculates a rectangle's surface and perimeter.
      * @param w Width of the rectangle.
      * @param h Height of the rectangle.
      * @return s The calculated surface.
      * @return p The calculated perimeter.
      */
     function rectangle(uint w, uint h) returns (uint s, uint p) {
         s = w * h;
         p = 2 * (w + h);
     }
 }
