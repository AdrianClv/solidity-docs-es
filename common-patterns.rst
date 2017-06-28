################
Patrones Comunes
################

.. index:: withdrawal

.. _withdrawal_pattern:

**********************
Retiro desde Contratos
**********************

El método recomendado de envío de fondos después de un efecto
es usando el patrón de retiro (withdrawal pattern). Aunque el método
más intuitivo de envío de Ether, del resultado de un efecto, es
un llamado directo de ``send``, esto no es recomendado ya que
introduce un potencial riesgo de seguridad. Puedes leer más
de esto en la página :ref:`security_considerations`.

Este es un ejemplo del patrón de retiro in práctica en un
contrato donde el objetivo es enviar la mayor cantidad del Ether
al contrato a fin de convertirse en el más "adinerado", inspirado por
`King of the Ether <https://www.kingoftheether.com/>`_.

En el siguiente contrato, si dejas de ser el más adinerado,
recibes los fondos de la persona quien te destronó.

::

    pragma solidity ^0.4.11;

    contract WithdrawalContract {
        address public richest;
        uint public mostSent;

        mapping (address => uint) pendingWithdrawals;

        function WithdrawalContract() payable {
            richest = msg.sender;
            mostSent = msg.value;
        }

        function becomeRichest() payable returns (bool) {
            if (msg.value > mostSent) {
                pendingWithdrawals[richest] += msg.value;
                richest = msg.sender;
                mostSent = msg.value;
                return true;
            } else {
                return false;
            }
        }

        function withdraw() {
            uint amount = pendingWithdrawals[msg.sender];
            // Remember to zero the pending refund before
            // sending to prevent re-entrancy attacks
            pendingWithdrawals[msg.sender] = 0;
            msg.sender.transfer(amount);
        }
    }

Esto en lugar de el patrón más intuitivo de envío:

::

    pragma solidity ^0.4.11;

    contract SendContract {
        address public richest;
        uint public mostSent;

        function SendContract() payable {
            richest = msg.sender;
            mostSent = msg.value;
        }

        function becomeRichest() payable returns (bool) {
            if (msg.value > mostSent) {
                // Esta línea puede causar problemas (explicado abajo).
                richest.transfer(msg.value);
                richest = msg.sender;
                mostSent = msg.value;
                return true;
            } else {
                return false;
            }
        }
    }

Nótese que, en este ejemplo, un atacante puede bloquear
el contrato en un estado inútil haciendo que ``richest``
sea la dirección de un contrato que tiene como función una
respaldo (ej. usando ``revert()`` o solo consumiendo más de
2300 estipendio de gas). De esa forma, cuando ``transfer``
es llamado para enviar fondos al contrato "envenenado", fallará
y también fallará la función ``becomeRichest``, bloqueando el
contrario para siempre.

Por el contrario, si usas el patrón "withdrawl" del primer ejemplo,
el atacante sólo puede causar que su propio withdrawl falle y no
el resto del contrato.

.. index:: access;restricting

********************
Restringiendo Acceso
********************

Restringiendo acceso es un patrón común para contratos.
Nótese que nunca se puede restringir un humano o ordenador
de leer el contenido de una transacción o del estado de un
contrato. Lo puedes hacer un poco más difícil de leer usando
criptografía, pero si tu contrato debe leer los datos, todos
podrán leerlo también.

Puedes restringir acceso de lectura al estado de tu contrato
por **otros contratos**. Esto es, en realidad, por defecto
al menos que declares tus variables ``public``.

Además, puedes restringir quién puede hacer modificaciones
al estado de tu contrato o quien puede llamar las funciones
y de eso se trata esta sección.

.. index:: function;modifier

El uso de **modificadores de funciones** (function modifiers)
hace estas restricciones altamente lisibles.

::

    pragma solidity ^0.4.11;

    contract AccessRestriction {
        // Estas serán asignadas en la fase de
        // construcción, donde `msg.sender` es
        // el account que crea este contrato.
        address public owner = msg.sender;
        uint public creationTime = now;

        // Modificadores pueden usarse para
        // cambiar el cuerpo de una función.
        // Si el modificador es usado, agregará
        // un chequeo que sólo pasa si la
        // función es llamada desde una cierta
        // dirección.
        modifier onlyBy(address _account)
        {
            require(msg.sender == _account);
            // No olvides el "_;"!
            // Esto será reemplazado por el cuerpo
            // de la función cuando el modificador
            // será activado.
            _;
        }

        /// Hacer `_newOwner` el nuevo owner de
        /// este contrato.
        function changeOwner(address _newOwner)
            onlyBy(owner)
        {
            owner = _newOwner;
        }

        modifier onlyAfter(uint _time) {
            require(now >= _time);
            _;
        }

        /// Borrar información de ownership.
        /// Sólo puede llamarse 6 semanas
        /// después que el contrato hay sido
        /// creado.
        function disown()
            onlyBy(owner)
            onlyAfter(creationTime + 6 weeks)
        {
            delete owner;
        }

        // Este modificador requiere un cierto pago
        // de fee que sea asociado con una llamada
        // de función.
        // Si el llamador envió demasiado, será
        // reembolsado, pero sólo después del cuerpo
        // de la función.
        // Esto era peligroso antes de la versión
        // 0.4.0 de solidity, donde era posible
        // de saltar la parte después de `_;`.
        modifier costs(uint _amount) {
            require(msg.value >= _amount);
            _;
            if (msg.value > _amount)
                msg.sender.send(msg.value - _amount);
        }

        function forceOwnerChange(address _newOwner)
            costs(200 ether)
        {
            owner = _newOwner;
            // sólo una condición de ejemplo
            if (uint(owner) & 0 == 1)
                // Esto no se hacía antes de Solidity
                // 0.4.0
                return;
            // reembolsar los fees excesivos
        }
    }

Una manera más especializada de acceder a funciones
que pueden ser restringidas será visto en el próximo
ejemplo.

.. index:: state machine

*****************
Máquina de Estado
*****************

Los contratos a menudo actúan como una máquina de estado,
que significa que tienen ciertas **etapas** en donde se
comportan de manera diferente o en donde distintas funciones
pueden ser llamadas. Una llamada de función a menudo
termina una etapa y pasa el contrato a la siguiente
etapa (especialmente si el contrato modela **interaction**).
También es común que algunas etapas será automáticamente
alcanzadas a cierto punto en el **tiempo**.

Como un ejemplo de esto es el contrato ciego de contrato
que comienza en la etapa "aceptando ofertas ciegas", luego
pasa a "revelando ofertas" que es finalizado por
"determinar resultado de subasta".

.. index:: function;modifier

Modificadores de funciones pueden ser usado en esta
situación para modelar los estados y cuidar
el uso incorrecto del contrato.

Ejemplo
=======

En el siguiente ejemplo,
el modificador ``atStage`` asegura que la función
pueda sólo ser llamada desde una cierta etapa.

Transiciones automáticas temporizadas son manejadas
por el modificador ``timeTransitions``, quien
debe usarse para toas las funciones.

.. nota::
    **EL Ordén del Modificador Importa**.
    Si atStage es combinado
    con timesTransitions, asegúrate que puedas
    mencionarlo después de éste, para que la nueva
    etapa sea tomada en cuenta.

Finalmente, el modificador ``transitionNext`` puede
ser usado automáticamente para ir a la próxima etapa
cuando la función termina.

.. nota::
    **El Modificador Puede Ser Omitido**.
    Esto sólo se aplica a Solidity antes de la versión
    0.4.0:
    Ya que los modificadores son aplicados simplemente
    remplazando código y no usando llamados de funciones,
    el código puede ser omitido si la función en sí usa
    return. Si es lo que quieres hacer, asegúrate
    de llamar nextStage manualmente desde esas funciones.
    Comenzando con la versión 0.4.0 ,modificar código
    correrá incluso si la función explícitamente
    retorna.

::

    pragma solidity ^0.4.11;

    contract StateMachine {
        enum Stages {
            AcceptingBlindedBids,
            RevealBids,
            AnotherStage,
            AreWeDoneYet,
            Finished
        }

        // Ésta es la etapa actual.
        Stages public stage = Stages.AcceptingBlindedBids;

        uint public creationTime = now;

        modifier atStage(Stages _stage) {
            require(stage == _stage);
            _;
        }

        function nextStage() internal {
            stage = Stages(uint(stage) + 1);
        }

        // Hacer transiciones temporizadas. Asegúrate de
        // mencionar este modificador primero, si no, la
        // seguridad no tomará en cuenta la nueva etapa.
        modifier timedTransitions() {
            if (stage == Stages.AcceptingBlindedBids &&
                        now >= creationTime + 10 days)
                nextStage();
            if (stage == Stages.RevealBids &&
                    now >= creationTime + 12 days)
                nextStage();
            // Las otras etapas transición por transición
            _;
        }

        // ¡El orden de los modificadores importa aquí!
        function bid()
            payable
            timedTransitions
            atStage(Stages.AcceptingBlindedBids)
        {
            // No implementaremos esto aquí
        }

        function reveal()
            timedTransitions
            atStage(Stages.RevealBids)
        {
        }

        // Este modificador pasa a la próxima etapa
        // una vez terminada la función.
        modifier transitionNext()
        {
            _;
            nextStage();
        }

        function g()
            timedTransitions
            atStage(Stages.AnotherStage)
            transitionNext
        {
        }

        function h()
            timedTransitions
            atStage(Stages.AreWeDoneYet)
            transitionNext
        {
        }

        function i()
            timedTransitions
            atStage(Stages.Finished)
        {
        }
    }
