.. index:: ! installing

.. _installing-solidity:

####################
Instalando Solidity
####################

Control de Versión
==================

Las versiones de Solidity siguen `versioning semantica <https://semver.org>`_ y además
de los releases, también hay **nightly developement builds**. Los nightly builds no están
garantizados a funcionar y puede que contengan cambios no documentados o que no funcionen.
Recomendamos usar la última release. Los instaladores de paquetes siguientes usarán la release
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

Esto es probablemente la manera más portable y conveniente de instalar Solidity localmente.

Una librería Javascript independiente de plataforma está proporcionada compilando la fuente C++
a Javascript usando Emscripten. Puede ser usado en proyectos directamente (como Remix).
Refiérete al repositorio `solc-js <https://github.com/ethereum/solc-js>`_ para ver las instrucciones.

También contiene una herramienta de línea de comando llamada `solcjs` que puede ser instala vía npm:

.. code:: bash

    npm install -g solc

.. nota::

    Las opciones de línea de comando de `solcjs` no son compatibles con `solc` y herramientas
    (tales como `geth`) esperando que el comportamiento de `solc` no funcione con `solcjs`.

Docker
======

Proveemos builds docker al día para el compilador. El repositorio
``stable`` contiene las versiones released mientras que el ``nightly``
contiene cambios potencialmente inestables de la rama de desarrollo.

.. code:: bash

    docker run ethereum/solc:stable solc --version

Actualmente, la imagen docker contiene el compilador ejecutable,
así que tendrás que enlazar las carpetas de fuente y de output.

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

Homebrew aún no tiene paquetes preconstruidos (pre-built bottles),
siguiendo una migración de Jenkins a TavisCI, pero Homebrew
debería aún funcionar para construir desde la fuente (build-from-source).
Se agregarán los paquetes preconstruidos pronto.

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

Seguir los enlaces de historia hasta que veas un archivo crudo de un
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

Construir desde la fuente
=========================

Clonar el Repositorio
---------------------

Para clonar el código fuente, ejecuta el comando siguiente:

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
---------------------

Para macOS, asegúrate que tiene la versión más reciente de
`Xcode installed <https://developer.apple.com/xcode/download/>`_.
Esto contiene el `compilador Clang C++ <https://en.wikipedia.org/wiki/Clang>`_, las
herramientas que se necesitan para construir aplicaciones C++ en OS X.
Si estás instalando Xcode por primera vez, necesitarás aceptar las condiciones de uso
antes de poder hacer builds de línea de comando:

.. code:: bash

    sudo xcodebuild -license accept

Nuestras builds OS X requieren instalar el gestor de paquetes
`Homebrew <http://brew.sh>`_ para instalar dependencias externas.
Aquí puedes ver cómo `desinstalar Homebrew
<https://github.com/Homebrew/homebrew/blob/master/share/doc/homebrew/FAQ.md#how-do-i-uninstall-homebrew>`_,
si alguna vez quieres empezar de nuevo.

Prerrequisitos - Windows
-----------------------

Necesitarás instalar las siguientes dependencias para los builds de Solidity en Windows:

+------------------------------+-------------------------------------------------------+
| Software                     | Notas                                                 |
+==============================+=======================================================+
| `Git para Windows`_          | Herramienta de línea de comando para repositorios git.|
+------------------------------+-------------------------------------------------------+
| `CMake`_                     | Generador de build multi plataforma.                  |
+------------------------------+-------------------------------------------------------+
| `Visual Studio 2015`_        | compilador C++ y entorno desarrollo.                  |
+------------------------------+-------------------------------------------------------+

.. _Git para Windows: https://git-scm.com/download/win
.. _CMake: https://cmake.org/download/
.. _Visual Studio 2015: https://www.visualstudio.com/products/vs-2015-product-editions


Dependencias Externas
---------------------

Ahora tenemos un script simple de uso que instala todos las dependencias externas
en macOS, Windows y varias distros Linux. Esto solía ser un proceso manual de varias
etapas, pero ahora es una sólo línea:

.. code:: bash

    ./scripts/install_deps.sh

o, en Windows:

.. code:: bat

    scripts\install_deps.bat

Build en Línea de comandos
--------------------------

Construir Solidity es bastante similar en Linux, macOS y otros sistemas Unix:

.. code:: bash

    mkdir build
    cd build
    cmake .. && make

o aún más fácil:

.. code:: bash

    #nota: esto instalará binarios solc y soltest en usr/local/bin
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

O si no, puedes construir para Windows en la línea de comandos, así:

.. code:: bash

    cmake --build . --config RelWithDebInfo

La cadena de versión en detalle
===============================

La cadena de versión de Solidity está compuesta por 4 partes:

- el número de la versión
- tag pre-release, en general en formato ``develop.YYYY.MM.DD`` o ``nightly.YYYY.MM.DD``
- commit en formato ``commit.GITHASH``
- plataforma tiene número arbitrario de ítems, contiene detalles de la plataforma y compilador

Si es que hay modificaciones locales, el commit será postfixed con ``.mod``.

Éstas partes son combinadas son requeridas por Semver, donde la tag pre-release de Solidity equivale al pre-release
de Semver y el commit Solidity y plataforma combinadas hacen el metadata del build de Semver.

Un ejemplo de release: ``0.4.8+commit.60cc1668.Emscripten.clang``.

Un ejemplo pre-release: ``0.4.9-nightly.2017.1.17+commit.6ecb4aa3.Emscripten.clang``

Información importante sobre versiones
======================================

Luego de hacer un release, la versión de nível de patch es levantada, porque asumimos que sólo
siguen cambios de nivel de patch. Cuando los cambios son integrados, la versión será aumentada
de acuerdo a la versión Semver y la urgencia de los cambios. Finalmente, un release siempre está
hecho con la versión de la build nightly actual, pero sin el especificador ``prerelease``.

Ejemplo:

0. se hace el release 0.4.0
1. el nightly build tiene versión 0.4.1 desde ahora
2. cambios sin ruptura se introducen - no hay cambio en versión
3. cambios con ruptura de introducen - versión se aumenta a 0.5.0
4. se hace el release 0.5.0

Este comportamiento funciona bien con el :ref:`version pragma <version_pragma>`.
