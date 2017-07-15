##########################
Solidity mediante ejemplos
##########################

.. index:: voting, ballot

.. _voting:

********
Votación
********

El siguiente contrato es bastante complejo, pero muestra
muchas de las caractersticas de Solidity. Es un contrato
de votación. Uno de los principales problemas de la votación
electrónica es la asignación de los derechos de voto a las
personas correctas para evitar la manipulación. No vamos a resolver
todos los problemas, pero por lo menos vamos a ver cómo se puede
hacer un sistema de votación delegado que permita contar votos
de forma **automática y completamente transparente** al mismo tiempo.

La idea es crear un contrato por votación,
junto con un breve nombre para cada opción.
El creador del contrato, que ejercerá de presidente,
será el encargado de dar derecho a voto a cada
dirección una a una.

Las personas que controlen estas direcciones podrán
votar o delegar su voto a una persona de confianza.

Cuando finalice el periodo de votación, ``winningProposal()``
devolverá la propuesta más votada.

::

    pragma solidity ^0.4.11;

    /// @title Votación con voto delegado
    contract Ballot {
        // Declara un nuevo tipo de dato complejo, que será
        // usado para almacenar variables.
        // Representará a un único votante.
        struct Voter {
            uint weight; // el peso del voto se acumula mediante la delegación de votos
            bool voted;  // true si esa persona ya ha votado
            address delegate; // persona a la que se delega el voto
            uint vote;   // índice de la propuesta votada
        }

        // Representa una única propuesta.
        struct Proposal {
            bytes32 name;   // nombre corto (hasta 32 bytes)
            uint voteCount; // número de votos acumulados
        }

        address public chairperson;

        // Declara una variable de estado que
        // almacena una estructura de datos `Voter` para cada posible dirección.
        mapping(address => Voter) public voters;

        // Una array dinámico de estructuras de datos de tipo `Proposal`.
        Proposal[] public proposals;

        /// Crea una nueva votación para elegir uno de los `proposalNames`.
        function Ballot(bytes32[] proposalNames) {
            chairperson = msg.sender;
            voters[chairperson].weight = 1;

            // Para cada nombre propuesto
            // crea un nuevo objeto de tipo Proposal y lo añade
            // al final del array.
            for (uint i = 0; i < proposalNames.length; i++) {
                // `Proposal({...})` crea una nuevo objeto de tipo Proposal
                // de forma temporal y añade al final de `proposals`
                // mediante `proposals.push(...)`.
                proposals.push(Proposal({
                    name: proposalNames[i],
                    voteCount: 0
                }));
            }
        }

        // Da a `voter` el derecho a votar en esta votación.
        // Sólo puede ser ejecutado por `chairperson`.
        function giveRightToVote(address voter) {
            // Si el argumento de `require` da como resultado `false`,
            // finaliza la ejecución y revierte todos los cambios
            // producidos en el estado y los balances de Ether.
            // A veces es buena idea usar esto por si las funciones
            // están siendo ejecutadas de forma incorrecta. Pero ten en cuenta
            // que de esta forma se consumirá todo el gas enviado
            // (está previsto que esto cambie en el futuro).
            require((msg.sender == chairperson) && !voters[voter].voted);
            voters[voter].weight = 1;
        }

        /// Delega tu voto a `to`.
        function delegate(address to) {
            // Asignación por referencia
            Voter sender = voters[msg.sender];
            require(!sender.voted);

            // No se permite la delegación a uno mismo.
            require(to != msg.sender);

            // Propaga la delegación en tanto que
            // `to` también delegue.
            // Por norma general, los bucles son muy peligrosos
            // porque si tienen muchas iteracciones puede darse el caso
            // de que empleen más gas del disponible en un boque.
            // En este caso, eso implica que la delegación no será ejecutada,
            // pero en otros casos puede suponer que un
            // contrato se quede completamente bloqueado.
            while (voters[to].delegate != address(0)) {
                to = voters[to].delegate;

                // Encontramos un bucle en la delegación. No está permitido.
                require(to != msg.sender);
            }

            // Dado que `sender` es una referencia, esto
            // modifica `voters[msg.sender].voted`
            sender.voted = true;
            sender.delegate = to;
            Voter delegate = voters[to];
            if (delegate.voted) {
                // Si la persona en la que se ha delegado el voto ya ha votado,
                // se añade directamente al número de votos.
                proposals[delegate.vote].voteCount += sender.weight;
            } else {
                // Si la persona en la que se ha delegado el voto todavía no ha votado,
                // se añade al peso de su voto.
                delegate.weight += sender.weight;
            }
        }

        /// Da tu voto (incluyendo los votos que te han delegado)
        /// a la propuesta `proposals[proposal].name`.
        function vote(uint proposal) {
            Voter sender = voters[msg.sender];
            require(!sender.voted);
            sender.voted = true;
            sender.vote = proposal;

            // Si `proposal` está fuera del rango del array,
            // esto lanzará automáticamente una excepción y
            // se revocarán todos los cambios
            proposals[proposal].voteCount += sender.weight;
        }

        /// @dev Calcula la propuesta ganadora teniendo en cuenta
        // todos los votos realizados.
        function winningProposal() constant
                returns (uint winningProposal)
        {
            uint winningVoteCount = 0;
            for (uint p = 0; p < proposals.length; p++) {
                if (proposals[p].voteCount > winningVoteCount) {
                    winningVoteCount = proposals[p].voteCount;
                    winningProposal = p;
                }
            }
        }

        // Llama a la función winningProposal() para obtener
        // el índice de la propuesta ganadora y así luego
        // devolver el nombre.
        function winnerName() constant
                returns (bytes32 winnerName)
        {
            winnerName = proposals[winningProposal()].name;
        }
    }

Posibles mejoras
================

Actualmente, hacen falta muchas transacciones para dar derecho
a voto a todos los participantes. ¿Se te ocurre una forma mejor?

.. index:: auction;blind, auction;open, blind auction, open auction

****************
Subasta a ciegas
****************

En esta sección vamos a ver lo fácil que es crear
un contrato para hacer una subasta a ciegas en Ethereum.
Comenzaremos con una subasta en donde todo el mundo pueda
ver las pujas que se van haciendo. Posteriormente, ampliaremos
este contrato para convertirlo en una subasta a ciegas
en donde no sea posible ver las pujas reales hasta que finalice el
periodo de pujas.

.. _simple_auction:

Subasta abierta sencilla
========================

La idea general del siguiente contrato de subasta es que
cualquiera puede enviar sus pujas durante el periodo de pujas.
Como parte de la puja se envía el dinero / ether para ligar
a los pujadores con sus pujas. Si la puja más alta es superada,
la anterior puja más alta recupera su dinero.
Tras el periodo de pujas, el contrato tiene que ser llamado
manualmente por parte de los beneficiarios para recuperar
su dinero - los contratos no pueden activarse por sí mismos.

::

    pragma solidity ^0.4.11;

    contract SimpleAuction {
        // Parámetros de la subasta. Los tiempos son
        // o timestamps estilo unix (segundos desde 1970-01-01)
        // o periodos de tiempo en segundos.
        address public beneficiary;
        uint public auctionStart;
        uint public biddingTime;

        // Estado actual de la subasta.
        address public highestBidder;
        uint public highestBid;

        // Retiradas de dinero permitidas de las anteriores pujas
        mapping(address => uint) pendingReturns;

        // Fijado como true al final, no permite ningún cambio.
        bool ended;

        // Eventos que serán emitidos al realizar algún cambio
        event HighestBidIncreased(address bidder, uint amount);
        event AuctionEnded(address winner, uint amount);

        // Lo siguiente es lo que se conoce como un comentario natspec,
        // se identifican por las tres barras inclinadas.
        // Se mostrarán cuando se pregunte al usuario
        // si quiere confirmar la transacción.

        /// Crea una subasta sencilla con un periodo de pujas
        /// de `_biddingTime` segundos. El beneficiario de
        /// las pujas es la dirección `_beneficiary`.
        function SimpleAuction(
            uint _biddingTime,
            address _beneficiary
        ) {
            beneficiary = _beneficiary;
            auctionStart = now;
            biddingTime = _biddingTime;
        }

        /// Puja en la subasta con el valor enviado
        /// en la misma transacción.
        /// El valor pujado sólo será devuelto
        /// si la puja no es ganadora.
        function bid() payable {
            // No hacen falta argumentos, toda
            // la información necesaria es parte de
            // la transacción. La palabra payable
            // es necesaria para que la función pueda recibir Ether.

            // Revierte la llamada si el periodo
            // se pujas ha finalizado.
            require(now <= (auctionStart + biddingTime));

            // Si la puja no es la más alta,
            // envía el dinero de vuelta.
            require(msg.value > highestBid);

            if (highestBidder != 0) {
                // Sending back the money by simply using
                // highestBidder.send(highestBid) is a security risk
                // because it can be prevented by the caller by e.g.
                // raising the call stack to 1023. It is always safer
                // to let the recipients withdraw their money themselves.
                pendingReturns[highestBidder] += highestBid;
            }
            highestBidder = msg.sender;
            highestBid = msg.value;
            HighestBidIncreased(msg.sender, msg.value);
        }

        /// Withdraw a bid that was overbid.
        function withdraw() returns (bool) {
            var amount = pendingReturns[msg.sender];
            if (amount > 0) {
                // It is important to set this to zero because the recipient
                // can call this function again as part of the receiving call
                // before `send` returns.
                pendingReturns[msg.sender] = 0;

                if (!msg.sender.send(amount)) {
                    // No need to call throw here, just reset the amount owing
                    pendingReturns[msg.sender] = amount;
                    return false;
                }
            }
            return true;
        }

        /// End the auction and send the highest bid
        /// to the beneficiary.
        function auctionEnd() {
            // It is a good guideline to structure functions that interact
            // with other contracts (i.e. they call functions or send Ether)
            // into three phases:
            // 1. checking conditions
            // 2. performing actions (potentially changing conditions)
            // 3. interacting with other contracts
            // If these phases are mixed up, the other contract could call
            // back into the current contract and modify the state or cause
            // effects (ether payout) to be performed multiple times.
            // If functions called internally include interaction with external
            // contracts, they also have to be considered interaction with
            // external contracts.

            // 1. Conditions
            require(now >= (auctionStart + biddingTime)); // auction did not yet end
            require(!ended); // this function has already been called

            // 2. Effects
            ended = true;
            AuctionEnded(highestBidder, highestBid);

            // 3. Interaction
            beneficiary.transfer(highestBid);
        }
    }

Blind Auction
=============

The previous open auction is extended to a blind auction
in the following. The advantage of a blind auction is
that there is no time pressure towards the end of
the bidding period. Creating a blind auction on a
transparent computing platform might sound like a
contradiction, but cryptography comes to the rescue.

During the **bidding period**, a bidder does not
actually send her bid, but only a hashed version of it.
Since it is currently considered practically impossible
to find two (sufficiently long) values whose hash
values are equal, the bidder commits to the bid by that.
After the end of the bidding period, the bidders have
to reveal their bids: They send their values
unencrypted and the contract checks that the hash value
is the same as the one provided during the bidding period.

Another challenge is how to make the auction
**binding and blind** at the same time: The only way to
prevent the bidder from just not sending the money
after he won the auction is to make her send it
together with the bid. Since value transfers cannot
be blinded in Ethereum, anyone can see the value.

The following contract solves this problem by
accepting any value that is at least as large as
the bid. Since this can of course only be checked during
the reveal phase, some bids might be **invalid**, and
this is on purpose (it even provides an explicit
flag to place invalid bids with high value transfers):
Bidders can confuse competition by placing several
high or low invalid bids.


::

    pragma solidity ^0.4.11;

    contract BlindAuction {
        struct Bid {
            bytes32 blindedBid;
            uint deposit;
        }

        address public beneficiary;
        uint public auctionStart;
        uint public biddingEnd;
        uint public revealEnd;
        bool public ended;

        mapping(address => Bid[]) public bids;

        address public highestBidder;
        uint public highestBid;

        // Allowed withdrawals of previous bids
        mapping(address => uint) pendingReturns;

        event AuctionEnded(address winner, uint highestBid);

        /// Modifiers are a convenient way to validate inputs to
        /// functions. `onlyBefore` is applied to `bid` below:
        /// The new function body is the modifier's body where
        /// `_` is replaced by the old function body.
        modifier onlyBefore(uint _time) { require(now < _time); _; }
        modifier onlyAfter(uint _time) { require(now > _time); _; }

        function BlindAuction(
            uint _biddingTime,
            uint _revealTime,
            address _beneficiary
        ) {
            beneficiary = _beneficiary;
            auctionStart = now;
            biddingEnd = now + _biddingTime;
            revealEnd = biddingEnd + _revealTime;
        }

        /// Place a blinded bid with `_blindedBid` = keccak256(value,
        /// fake, secret).
        /// The sent ether is only refunded if the bid is correctly
        /// revealed in the revealing phase. The bid is valid if the
        /// ether sent together with the bid is at least "value" and
        /// "fake" is not true. Setting "fake" to true and sending
        /// not the exact amount are ways to hide the real bid but
        /// still make the required deposit. The same address can
        /// place multiple bids.
        function bid(bytes32 _blindedBid)
            payable
            onlyBefore(biddingEnd)
        {
            bids[msg.sender].push(Bid({
                blindedBid: _blindedBid,
                deposit: msg.value
            }));
        }

        /// Reveal your blinded bids. You will get a refund for all
        /// correctly blinded invalid bids and for all bids except for
        /// the totally highest.
        function reveal(
            uint[] _values,
            bool[] _fake,
            bytes32[] _secret
        )
            onlyAfter(biddingEnd)
            onlyBefore(revealEnd)
        {
            uint length = bids[msg.sender].length;
            require(_values.length == length);
            require(_fake.length == length);
            require(_secret.length == length);

            uint refund;
            for (uint i = 0; i < length; i++) {
                var bid = bids[msg.sender][i];
                var (value, fake, secret) =
                        (_values[i], _fake[i], _secret[i]);
                if (bid.blindedBid != keccak256(value, fake, secret)) {
                    // Bid was not actually revealed.
                    // Do not refund deposit.
                    continue;
                }
                refund += bid.deposit;
                if (!fake && bid.deposit >= value) {
                    if (placeBid(msg.sender, value))
                        refund -= value;
                }
                // Make it impossible for the sender to re-claim
                // the same deposit.
                bid.blindedBid = 0;
            }
            msg.sender.transfer(refund);
        }

        // This is an "internal" function which means that it
        // can only be called from the contract itself (or from
        // derived contracts).
        function placeBid(address bidder, uint value) internal
                returns (bool success)
        {
            if (value <= highestBid) {
                return false;
            }
            if (highestBidder != 0) {
                // Refund the previously highest bidder.
                pendingReturns[highestBidder] += highestBid;
            }
            highestBid = value;
            highestBidder = bidder;
            return true;
        }

        /// Withdraw a bid that was overbid.
        function withdraw() returns (bool) {
            var amount = pendingReturns[msg.sender];
            if (amount > 0) {
                // It is important to set this to zero because the recipient
                // can call this function again as part of the receiving call
                // before `send` returns (see the remark above about
                // conditions -> effects -> interaction).
                pendingReturns[msg.sender] = 0;

                if (!msg.sender.send(amount)){
                    // No need to call throw here, just reset the amount owing
                    pendingReturns[msg.sender] = amount;
                    return false;
                }
            }
            return true;
        }

        /// End the auction and send the highest bid
        /// to the beneficiary.
        function auctionEnd()
            onlyAfter(revealEnd)
        {
            require(!ended);
            AuctionEnded(highestBidder, highestBid);
            ended = true;
            // We send all the money we have, because some
            // of the refunds might have failed.
            beneficiary.transfer(this.balance);
        }
    }

.. index:: purchase, remote purchase, escrow

********************
Safe Remote Purchase
********************

::

    pragma solidity ^0.4.11;

    contract Purchase {
        uint public value;
        address public seller;
        address public buyer;
        enum State { Created, Locked, Inactive }
        State public state;

        function Purchase() payable {
            seller = msg.sender;
            value = msg.value / 2;
            require((2 * value) == msg.value);
        }

        modifier condition(bool _condition) {
            require(_condition);
            _;
        }

        modifier onlyBuyer() {
            require(msg.sender == buyer);
            _;
        }

        modifier onlySeller() {
            require(msg.sender == seller);
            _;
        }

        modifier inState(State _state) {
            require(state == _state);
            _;
        }

        event Aborted();
        event PurchaseConfirmed();
        event ItemReceived();

        /// Abort the purchase and reclaim the ether.
        /// Can only be called by the seller before
        /// the contract is locked.
        function abort()
            onlySeller
            inState(State.Created)
        {
            Aborted();
            state = State.Inactive;
            seller.transfer(this.balance);
        }

        /// Confirm the purchase as buyer.
        /// Transaction has to include `2 * value` ether.
        /// The ether will be locked until confirmReceived
        /// is called.
        function confirmPurchase()
            inState(State.Created)
            condition(msg.value == (2 * value))
            payable
        {
            PurchaseConfirmed();
            buyer = msg.sender;
            state = State.Locked;
        }

        /// Confirm that you (the buyer) received the item.
        /// This will release the locked ether.
        function confirmReceived()
            onlyBuyer
            inState(State.Locked)
        {
            ItemReceived();
            // It is important to change the state first because
            // otherwise, the contracts called using `send` below
            // can call in again here.
            state = State.Inactive;

            // NOTE: This actually allows both the buyer and the seller to
            // block the refund - the withdraw pattern should be used.

            buyer.transfer(value);
            seller.transfer(this.balance);
        }
    }

*******************
Canal de micropagos
*******************

Por escribir.
