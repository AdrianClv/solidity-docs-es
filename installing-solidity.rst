.. index:: ! installing

.. _installing-solidity:

###################
Instalando Solidity
###################

Control de Versiones
====================

Las versiones de Solidity siguen un `versionado semántico <https://semver.org>`_, y además
de los releases, también hay **nightly developement builds**. El funcionamiento de los nightly builds
no está garantizado y puede que contengan cambios no documentados o que no funcionen.
Recomendamos usar la última release. Los siguientes instaladores de paquetes usarán la release
más actual.

Remix
=====

Si sólo quieres probar Solidity para pequeños contratos, puedes usar
`Remix <https://remix.ethereum.org/>`_
que no necesita instalación. Si quieres usarlo sin conexión a internet,
puedes ir a https://github.com/ethereum/browser-solidity/tree/gh-pages
y bajar el .ZIP como se explica en esa página.

npm / Node.js
=============

Esta es probablemente la manera más portable y conveniente de instalar Solidity localmente.

Se proporciona una librería Javascript independiente de plataforma mediante la compilación del código C++
a Javascript usando Emscripten. Puede ser usado en proyectos directamente (como Remix).
Visita el repositorio `solc-js <https://github.com/ethereum/solc-js>`_ para ver las instrucciones.

También contiene una herramienta de línea de comandos llamada `solcjs` que puede ser instalada vía npm:

.. code:: bash

    npm install -g solc

.. note::

    Las opciones de línea de comandos de `solcjs` no son compatibles con `solc`, y las herramientas
    (tales como `geth`) que esperen el comportamiento de `solc` no funcionarán con `solcjs`.

Docker
======

Proveemos builds de docker actualizadas para el compilador. El repositorio
``stable`` contiene las versiones publicadas mientras que el ``nightly``
contiene cambios potencialmente inestables de la rama de desarrollo.

.. code:: bash

    docker run ethereum/solc:stable solc --version

Actualmente, la imagen de docker contiene el compilador ejecutable,
así que tendrás que enlazar las carpetas de código y de output.

Paquetes Binarios
=================

Los paquetes binarios de Solidity están disponibles en
`solidity/releases <https://github.com/ethereum/solidity/releases>`_.

También tenemos PPAs para Ubuntu. Para la versión estable más reciente.

.. code:: bash

    sudo add-apt-repository ppa:ethereum/ethereum
    sudo apt-get update
    sudo apt-get install solc

Si quieres la versión en desarrollo más reciente:

.. code:: bash

    sudo add-apt-repository ppa:ethereum/ethereum
    sudo add-apt-repository ppa:ethereum/ethereum-dev
    sudo apt-get update
    sudo apt-get install solc

Arch Linux también tiene paquetes, pero limitados a la versión de desarrollo más reciente:

.. code:: bash

    pacman -S solidity-git

Homebrew aún no tiene paquetes preconstruidos,
siguiendo una migración de Jenkins a TavisCI, pero Homebrew
todavía debería funcionar para construir desde el código.
Pronto se agregarán los paquetes preconstruidos.

.. code:: bash

    brew update
    brew upgrade
    brew tap ethereum/ethereum
    brew install solidity
    brew linkapps solidity

Si necesitas una versión específica de Solidity, puedes instalar
una fórmula Homebrew desde Github.

Ver
`solidity.rb commits on Github <https://github.com/ethereum/homebrew-ethereum/commits/master/solidity.rb>`_.

Sigue los enlaces de historia hasta que veas un enlace a un fichero de un
commit específico de ``solidity.rb``.

instalar con ``brew``:

.. code:: bash

    brew unlink solidity
    # Install 0.4.8
    brew install https://raw.githubusercontent.com/ethereum/homebrew-ethereum/77cce03da9f289e5a3ffe579840d3c5dc0a62717/solidity.rb

Gentoo también provee un paquete Solidity que puede instalarse con ``emerge``:

.. code:: bash

    demerge ev-lang/solidity

.. _building-from-source:

Construir desde el código
=========================

Clonar el Repositorio
---------------------

Para clonar el código fuente, ejecuta el siguiente comando:

.. code:: bash

    git clone --recursive https://github.com/ethereum/solidity.git
    cd solidity

Si quieres ayudar a desarrollar Solidity,
debes hacer un fork de Solidity y agregar tu fork personal como un remoto secundario:

.. code:: bash

    cd solidity
    git remote add personal git@github.com:[username]/solidity.git

Solidity tiene submódulos de git. Asegúrate que están cargados correctamente:

.. code:: bash

   git submodule update --init --recursive

Prerrequisitos - macOS
----------------------

Para macOS, asegúrate de tener la versión más reciente de
`Xcode instalada <https://developer.apple.com/xcode/download/>`_.
Esto contiene el `compilador Clang C++ <https://en.wikipedia.org/wiki/Clang>`_, las
herramientas que se necesitan para construir aplicaciones C++ en OS X.
Si estás instalando Xcode por primera vez, necesitarás aceptar las condiciones de uso
antes de poder hacer builds de línea de comandos:

.. code:: bash

    sudo xcodebuild -license accept

Nuestras builds OS X requieren instalar el gestor de paquetes
`Homebrew <http://brew.sh>`_ para instalar dependencias externas.
Aquí puedes ver cómo `desinstalar Homebrew
<https://github.com/Homebrew/homebrew/blob/master/share/doc/homebrew/FAQ.md#how-do-i-uninstall-homebrew>`_,
si alguna vez quieres empezar de nuevo.

Prerrequisitos - Windows
------------------------

Necesitarás instalar las siguientes dependencias para los builds de Solidity en Windows:

+------------------------------+--------------------------------------------------------+
| Software                     | Notas                                                  |
+==============================+========================================================+
| `Git para Windows`_          | Herramienta de línea de comandos para repositorios git.|
+------------------------------+--------------------------------------------------------+
| `CMake`_                     | Generador de build multi plataforma.                   |
+------------------------------+--------------------------------------------------------+
| `Visual Studio 2015`_        | Compilador C++ y entorno de desarrollo.                |
+------------------------------+--------------------------------------------------------+

.. _Git para Windows: https://git-scm.com/download/win
.. _CMake: https://cmake.org/download/
.. _Visual Studio 2015: https://www.visualstudio.com/products/vs-2015-product-editions


Dependencias Externas
---------------------

Ahora tenemos un script fácil de usar que instala todas las dependencias externas
en macOS, Windows y varias distros Linux. Esto solía ser un proceso manual de varias
etapas, pero ahora es una sola línea:

.. code:: bash

    ./scripts/install_deps.sh

o, en Windows:

.. code:: bat

    scripts\install_deps.bat

Build en línea de comandos
--------------------------

Construir Solidity es bastante similar en Linux, macOS y otros sistemas Unix:

.. code:: bash

    mkdir build
    cd build
    cmake .. && make

o aún más fácil:

.. code:: bash

    #nota: esto instalará los binarios de solc y soltest en usr/local/bin
    ./scripts/build.sh

Incluso para Windows:

.. code:: bash

    mkdir build
    cd build
    cmake -G "Visual Studio 14 2015 Win64" ..

Estas últimas instrucciones deberían resultar en la creación de
**solidity.sln** en ese directorio de build. Hacer doble click en ese
archivo debería abrir Visual Studio. Sugerimos construir
la configuración **RelWithDebugInfo**, pero todas funcionan.

O si no, puedes construir para Windows en la línea de comandos de la siguiente manera:

.. code:: bash

    cmake --build . --config RelWithDebInfo

La cadena de versión en detalle
===============================

La cadena de versión de Solidity está compuesta por 4 partes:

- el número de la versión
- etiqueta pre-release, en general en formato ``develop.YYYY.MM.DD`` o ``nightly.YYYY.MM.DD``
- commit en formato ``commit.GITHASH``
- la plataforma tiene número arbitrario de ítems, contiene detalles de la plataforma y el compilador

Si hay modificaciones locales, el commit tendrá el sufijo ``.mod``.

Éstas partes se combinan como es requerido por Semver, donde la etiqueta pre-release de Solidity equivale al pre-release de Semver y el commit Solidity y plataforma combinadas hacen el metadata del build de Semver.

Un ejemplo de release: ``0.4.8+commit.60cc1668.Emscripten.clang``.

Un ejemplo pre-release: ``0.4.9-nightly.2017.1.17+commit.6ecb4aa3.Emscripten.clang``

Información importante sobre versiones
======================================

Tras hacer un release, se incrementa el nivel de versión patch, porque asumimos que sólo
siguen cambios de nivel de patch. Cuando se integran los cambios, la versión se aumenta
de acuerdo a la versión Semver y la urgencia de los cambios. Finalmente, un release siempre se hace con la versión de la build nightly actual, pero sin el especificador ``prerelease``.

Ejemplo:

0. se hace el release 0.4.0
1. el nightly build tiene versión 0.4.1 a partir de ahora
2. se introducen cambios compatibles - no hay cambio en versión
3. se introduce cambios no compatibles - la versión se aumenta a 0.5.0
4. se hace el release 0.5.0

Este comportamiento funciona bien con la :ref:`versión pragma <version_pragma>`.
