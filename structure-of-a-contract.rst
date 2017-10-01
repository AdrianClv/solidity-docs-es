.. index:: contract, state variable, function, event, struct, enum, function;modifier

.. _contract_structure:

*************************
Estructura de un contrato
*************************

Los Contratos en Solidity son similares a las clases de los lenguajes orientados a objetos.
Cualquier contrato puede contener declaraciones del tipo :ref:`variables de estado <structure-state-variables>`, :ref:`funciones <structure-functions>`, :ref:`modificadores de función <structure-function-modifiers>`, :ref:`eventos <structure-events>`, :ref:`structs <structure-structs-types>` y :ref:`enums <structure-enum-types>`.
Además, los contratos pueden heredar de otros contratos.

.. _structure-state-variables:

Variables de estado
===================

Las variables de estado son valores que están permanentemente almacenados en una parte del contrato conocida como storage del contrato.

::

  pragma solidity ^0.4.0;

  contract SimpleStorage {
      uint storedData; // Variable de estado
      // ...
  }

Véase la sección :ref:`tipos <types>` para conocer los diferentes tipos válidos de variables de estado y :ref:`visibilidad-y-getters <visibility-and-getters>` para conocer las distintas posibilidades de visibilidad que pueden tener las variables de estado.

.. _structure-functions:

Funciones
=========

Las funciones son las unidades ejecutables del código dentro de un contrato.

::

  pragma solidity ^0.4.0;

  contract SimpleAuction {
      function bid() payable { // Función
          // ...
      }
  }

Las :ref:`llamadas a una función <function-calls>` pueden ocurrir dentro o fuera de la misma. Una función puede tener varios niveles de :ref:`visibilidad <visibility-and-getters>` con respecto a otros contratos.

.. _structure-function-modifiers:

Modificadores de función
========================

Los modificadores de función se usan para enmendar de un modo declarativo la semántica de las funciones (véase :ref:`modificadores <modifiers>` en la sección sobre contratos).

::

  pragma solidity ^0.4.11;

  contract Purchase {
      address public seller;

      modifier onlySeller() { // Modificador
          require(msg.sender == seller);
          _;
      }

      function abort() onlySeller { // Uso de modificador
          // ...
      }
  }

.. _structure-events:

Eventos
=======

Los eventos son interfaces de conveniencia con los servicios de registro de la EVM (Máquina Virtual de Ethereum).

::

  pragma solidity ^0.4.0;

  contract SimpleAuction {
      event HighestBidIncreased(address bidder, uint amount); // Evento

      function bid() payable {
          // ...
          HighestBidIncreased(msg.sender, msg.value); // Lanzamiento del evento
      }
  }

Véase :ref:`eventos <events>` en la sección sobre contratos para tener más información sobre cómo se declaran los eventos y cómo se pueden usar dentro de una dapp.

.. _structure-structs-types:

Tipos de structs
================

Las estructuras de datos (Structs) son tipos definidos por el propio usuario y pueden agrupar mútiples variables (véase :ref:`structs <structs>` en la sección sobre tipos).

::

  pragma solidity ^0.4.0;

  contract Ballot {
      struct Voter { // Structs
          uint weight;
          bool voted;
          address delegate;
          uint vote;
      }
  }

.. _structure-enum-types:

Tipos de enum
=============

Los enumerados (Enums) se usan para crear tipos con un conjunto de valores finitos y están definidos por el propio usuario (véase :ref:`enums <enums>` en la sección sobre tipos).

::

  pragma solidity ^0.4.0;

  contract Purchase {
      enum State { Created, Locked, Inactive } // Enum
  }
