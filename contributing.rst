##########
Contribuir
##########

¡La ayuda siempre es bienvenida!

Para comenzar, puedes intentar :ref:`building-from-source` para familiarizarte
con los componentes de Solidity y el proceso de build. También, puede ser útil
de especializarse en escribir smart contrats en Solidity.

En particular, necesitamos ayuda en las siguientes áreas:

* Mejorando la documentación
* Respondiendo a las preguntas de otros usarios en `StackExchange
  <https://ethereum.stackexchange.com>`_ y el `Gitter Solidity
  <https://gitter.im/ethereum/solidity>`_
* Corrigiendo y respondiendo a los `issues del GitHub Solidity
  <https://github.com/ethereum/solidity/issues>`_, especialmente esos
  taggeados como `up-for-grabs <https://github.com/ethereum/solidity/issues?q=is%3Aopen+is%3Aissue+label%3Aup-for-grabs>`_
  que están destinados a contribuidores externos como temas introductorios.

Cómo reportar un Issue
======================

Para reportar un issue, porfavor usar el
`Issue tracker de GitHub <https://github.com/ethereum/solidity/issues>`_. Cuando
reportes un issue, por favor menciona los siguientes detalles:

* Qué versión de SOlidity estás usando
* Cuál es el código fuente (si es aplicable)
* En cuál plataforma estás usando
* Cómo reproducir el resultado
* Cuál fue el resultado del issue
* Cuál fue el resultado esperado

Reducir el código fuente que causó el issue a lo mínimo es siempre muy
útil y a veces incluso aclara un malentendido.

Flujo de trabajo para Pull Requests
===================================

A fin de contribuir, haz un fork de la rama ``develop`` y haz tus cambios ahí.
Tus mensajes commit deben detallar *porqué* hiciste el cambio además de *lo que*
hiciste (al menos que sea un pequeño cambio).

Si necesitas hacer un pull de ``develop`` después de haber hecho tu fork (por ejemplo,
para resolver potenciales conflictos de merge), evita utilizar ``git merge``
en lugar de ``git rebase`` para tu rama.

Adicionalmente, si estás escribiendo una nueva funcionalidad, porfavor asegúrate de
hacer tests unitarios Boost y ponerlos en ``test/``.

Sin embargo, si estás haciendo cambios más grandes, consulta con el canal Gitter primero.

Finalmente, siempre asegúrate de respetar los `estándares de coding
<https://raw.githubusercontent.com/ethereum/cpp-ethereum/develop/CodingStandards.txt>`_
de este proyecto. También, aunque hacemos testing CI, hace tests de tu código y asegúrate que
puedas hacer un build localmente antes de enviar un pull request.

Gracias por tu ayuda!

Corriendo los tests de compilador
=================================

Solidity incluye diferentes tipos de tests. Están incluídos en la aplicación llamada
``soltest``. Algunos de ellos requieren el cliente ``cpp-ethereum`` en modo test.

Para correr ``cpp-ethereum`` en modo test: ``eth --test -d /tmp/testeth``.

Para lanzar los tests: ``soltest -- --ipcpath /tmp/testeth/geth.ipc``.

Para correr un subconjunto de los tests, se pueden usar filtros:
``soltest -t TestSuite/TestName -- --ipcpath /tmp/testeth/geth.ipc``, donde ``TestName`` puede ser un comodín ``*``.

Alternativamente, hay un script de testing en ``scripts/test.sh`` que ejecuta todos los tests.
