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
al contrato a fin de convertirse en el mas "adinerado", inspirado por
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

Esto en lugar de el patrón mas intuitivo de envío:

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

Nótese que, en este ejemplo, un atacante puede atrapar
en un estado inútil

Notice that, in this example, an attacker could trap the
contract into an unusable state by causing ``richest`` to be
the address of a contract that has a fallback function
which fails (e.g. by using ``revert()`` or by just
conssuming more than the 2300 gas stipend). That way,
whenever ``transfer`` is called to deliver funds to the
"poisoned" contract, it will fail and thus also ``becomeRichest``
will fail, with the contract being stuck forever.

In contrast, if you use the "withdraw" pattern from the first example,
the attacker can only cause his or her own withdraw to fail and not the
rest of the contract's workings.

.. index:: access;restricting

******************
Restricting Access
******************

Restricting access is a common pattern for contracts.
Note that you can never restrict any human or computer
from reading the content of your transactions or
your contract's state. You can make it a bit harder
by using encryption, but if your contract is supposed
to read the data, so will everyone else.

You can restrict read access to your contract's state
by **other contracts**. That is actually the default
unless you declare make your state variables ``public``.

Furthermore, you can restrict who can make modifications
to your contract's state or call your contract's
functions and this is what this page is about.

.. index:: function;modifier

The use of **function modifiers** makes these
restrictions highly readable.

::

    pragma solidity ^0.4.11;

    contract AccessRestriction {
        // These will be assigned at the construction
        // phase, where `msg.sender` is the account
        // creating this contract.
        address public owner = msg.sender;
        uint public creationTime = now;

        // Modifiers can be used to change
        // the body of a function.
        // If this modifier is used, it will
        // prepend a check that only passes
        // if the function is called from
        // a certain address.
        modifier onlyBy(address _account)
        {
            require(msg.sender == _account);
            // Do not forget the "_;"! It will
            // be replaced by the actual function
            // body when the modifier is used.
            _;
        }

        /// Make `_newOwner` the new owner of this
        /// contract.
        function changeOwner(address _newOwner)
            onlyBy(owner)
        {
            owner = _newOwner;
        }

        modifier onlyAfter(uint _time) {
            require(now >= _time);
            _;
        }

        /// Erase ownership information.
        /// May only be called 6 weeks after
        /// the contract has been created.
        function disown()
            onlyBy(owner)
            onlyAfter(creationTime + 6 weeks)
        {
            delete owner;
        }

        // This modifier requires a certain
        // fee being associated with a function call.
        // If the caller sent too much, he or she is
        // refunded, but only after the function body.
        // This was dangerous before Solidity version 0.4.0,
        // where it was possible to skip the part after `_;`.
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
            // just some example condition
            if (uint(owner) & 0 == 1)
                // This did not refund for Solidity
                // before version 0.4.0.
                return;
            // refund overpaid fees
        }
    }

A more specialised way in which access to function
calls can be restricted will be discussed
in the next example.

.. index:: state machine

*************
State Machine
*************

Contracts often act as a state machine, which means
that they have certain **stages** in which they behave
differently or in which different functions can
be called. A function call often ends a stage
and transitions the contract into the next stage
(especially if the contract models **interaction**).
It is also common that some stages are automatically
reached at a certain point in **time**.

An example for this is a blind auction contract which
starts in the stage "accepting blinded bids", then
transitions to "revealing bids" which is ended by
"determine auction outcome".

.. index:: function;modifier

Function modifiers can be used in this situation
to model the states and guard against
incorrect usage of the contract.

Example
=======

In the following example,
the modifier ``atStage`` ensures that the function can
only be called at a certain stage.

Automatic timed transitions
are handled by the modifier ``timeTransitions``, which
should be used for all functions.

.. note::
    **Modifier Order Matters**.
    If atStage is combined
    with timedTransitions, make sure that you mention
    it after the latter, so that the new stage is
    taken into account.

Finally, the modifier ``transitionNext`` can be used
to automatically go to the next stage when the
function finishes.

.. note::
    **Modifier May be Skipped**.
    This only applies to Solidity before version 0.4.0:
    Since modifiers are applied by simply replacing
    code and not by using a function call,
    the code in the transitionNext modifier
    can be skipped if the function itself uses
    return. If you want to do that, make sure
    to call nextStage manually from those functions.
    Starting with version 0.4.0, modifier code
    will run even if the function explicitly returns.

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

        // This is the current stage.
        Stages public stage = Stages.AcceptingBlindedBids;

        uint public creationTime = now;

        modifier atStage(Stages _stage) {
            require(stage == _stage);
            _;
        }

        function nextStage() internal {
            stage = Stages(uint(stage) + 1);
        }

        // Perform timed transitions. Be sure to mention
        // this modifier first, otherwise the guards
        // will not take the new stage into account.
        modifier timedTransitions() {
            if (stage == Stages.AcceptingBlindedBids &&
                        now >= creationTime + 10 days)
                nextStage();
            if (stage == Stages.RevealBids &&
                    now >= creationTime + 12 days)
                nextStage();
            // The other stages transition by transaction
            _;
        }

        // Order of the modifiers matters here!
        function bid()
            payable
            timedTransitions
            atStage(Stages.AcceptingBlindedBids)
        {
            // We will not implement that here
        }

        function reveal()
            timedTransitions
            atStage(Stages.RevealBids)
        {
        }

        // This modifier goes to the next stage
        // after the function is done.
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
