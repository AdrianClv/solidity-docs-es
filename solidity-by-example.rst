##########################
Solidity mediante ejemplos
##########################

.. index:: voting, ballot

.. _voting:

********
Votación
********

El siguiente contrato es bastante complejo, pero muestra
muchas de las características de Solidity. Es un contrato
de votación. Uno de los principales problemas de la votación
electrónica es la asignación de los derechos de voto a las
personas correctas para evitar la manipulación. No vamos a resolver
todos los problemas, pero por lo menos vamos a ver cómo se puede
hacer un sistema de votación delegado que permita contar votos
de forma **automática y completamente transparente** al mismo tiempo.

La idea es crear un contrato por cada votación,
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

        // Una matriz dinámica de estructuras de datos de tipo `Proposal`.
        Proposal[] public proposals;

        // Crea una nueva votación para elegir uno de los `proposalNames`.
        function Ballot(bytes32[] proposalNames) {
            chairperson = msg.sender;
            voters[chairperson].weight = 1;

            // Para cada nombre propuesto
            // crea un nuevo objeto de tipo Proposal y lo añade
            // al final del array.
            for (uint i = 0; i < proposalNames.length; i++) {
                // `Proposal({...})` crea una nuevo objeto de tipo Proposal
                // de forma temporal y se añade al final de `proposals`
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

            // Propaga la delegación en tanto que `to` también delegue.
            // Por norma general, los bucles son muy peligrosos
            // porque si tienen muchas iteracciones puede darse el caso
            // de que empleen más gas del disponible en un bloque.
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
                // Si la persona en la que se ha delegado el voto
                // todavía no ha votado, se añade al peso de su voto.
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

            // Si `proposal` está fuera del rango de la matriz,
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
de voto a todos los participantes. ¿Se te ocurre una forma mejor?

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
            // de pujas ha finalizado.
            require(now <= (auctionStart + biddingTime));

            // Si la puja no es la más alta,
            // envía el dinero de vuelta.
            require(msg.value > highestBid);

            if (highestBidder != 0) {
                // Devolver el dinero usando
                // highestBidder.send(highestBid) es un riesgo
                // de seguridad, porque la llamada puede ser evitada
                // por el usuario elevando la pila de llamadas a 1023.
                // Siempre es más seguro dejar que los receptores
                // saquen su propio dinero.
                pendingReturns[highestBidder] += highestBid;
            }
            highestBidder = msg.sender;
            highestBid = msg.value;
            HighestBidIncreased(msg.sender, msg.value);
        }

        /// Retira una puja que fue superada.
        function withdraw() returns (bool) {
            var amount = pendingReturns[msg.sender];
            if (amount > 0) {
                // Es importante poner esto a cero porque el receptor
                // puede llamar de nuevo a esta función como parte
                // de la recepción antes de que `send` devuelva su valor.
                pendingReturns[msg.sender] = 0;

                if (!msg.sender.send(amount)) {
                    // Aquí no es necesario lanzar una excepción.
                    // Basta con reiniciar la cantidad que se debe devolver.
                    pendingReturns[msg.sender] = amount;
                    return false;
                }
            }
            return true;
        }

        /// Finaliza la subasta y envía la puja más alta al beneficiario.
        function auctionEnd() {
            // Es una buena práctica estructurar las funciones que interactúan
            // con otros contratos (i.e. llaman a funciones o envían ether)
            // en tres fases:
            // 1. comprobación de las condiciones
            // 2. ejecución de las acciones (pudiendo cambiar las condiciones)
            // 3. interacción con otros contratos
            // Si estas fases se entremezclasen, otros contratos podrían
            // volver a llamar a este contrato y modificar el estado
            // o hacer que algunas partes (pago de ether) se ejecute multiples veces.
            // Si se llama a funciones internas que interactúan con otros contratos,
            // deben considerarse como interacciones con contratos externos.

            // 1. Condiciones
            require(now >= (auctionStart + biddingTime)); // la subasta aún no ha acabado
            require(!ended); // esta función ya ha sido llamada

            // 2. Ejecución
            ended = true;
            AuctionEnded(highestBidder, highestBid);

            // 3. Interacción
            beneficiary.transfer(highestBid);
        }
    }

Subasta a ciegas
================

A continuación, se va a extender la subasta abierta anterior
a una subasta a ciegas. La ventaja de una subasta a ciegas
es que conforme se acaba el plazo de pujas, no aumenta la presión.
Crear una subasta a ciegas en una plataforma de computación
transparente puede parecer contradictorio, pero la criptografía
lo hace posible.

Durante el **periodo de puja**, un pujador no envía su puja
como tal, sino una versión hasheada de la misma.
Puesto que en la actualidad se considera que es prácticamente
imposible encontrar dos valores (suficientemente largos)
cuyos hashes son iguales, el pujador realiza la puja de esa forma.
Tras el periodo de puja, los pujadores tienen que revelar
sus pujas. Para ello, envían los valores descrifrados y el
contrato comprueba que el valor del hash se corresponde con
el proporcionado durante el periodo de puja.

Otra complicación es cómo hacer la subasta
**vinculante y ciega** al mismo tiempo: la única manera de
evitar que el pujador no envíe el dinero tras ganar
la subasta, es haciendo que lo envíe junto con la puja.
Puesto que las transferencias de valor no pueden ser
ocultadas en Ethereum, cualquiera podrá ver la cantidad.

El siguiente contrato soluciona este problema al
aceptar cualquier valor que sea al menos tan alto como
lo pujado. Puesto que esto sólo se podrá comprobar durante
la fase de revelación, algunas pujas podrán ser **inválidas**.
Esto es así a propósito (incluso sirve para prevenir errores
en caso de envíar pujas con valores muy altos). Los pujadores
pueden confundir a su competencia realizando multiples pujas
inválidas con valores altos o bajos.

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

        // Retiradas permitidas de pujas previas
        mapping(address => uint) pendingReturns;

        event AuctionEnded(address winner, uint highestBid);

        /// Los modificadores son una forma cómoda de validar los
        /// inputs de las funciones. Abajo se puede ver cómo
        /// `onlyBefore` se aplica a `bid`.
        /// El nuevo cuerpo de la función pasa a ser el del modificador,
        /// sustituyendo `_` por el anterior cuerpo de la función.
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

        /// Efectúa la puja de manera oculta con `_blindedBid`=
        /// keccak256(value, fake, secret).
        /// El ether enviado sólo se recuperará si la puja se revela de
        /// forma correcta durante la fase de revelacin. La puja es
        /// válida si el ether junto al que se envía es al menos "value"
        /// y "fake" no es cierto. Poner "fake" como verdadero y no enviar
        /// la cantidad exacta, son formas de ocultar la verdadera puja
        /// y aún así realizar el depósito necesario. La misma dirección
        /// puede realizar múltiples pujas.
        function bid(bytes32 _blindedBid)
            payable
            onlyBefore(biddingEnd)
        {
            bids[msg.sender].push(Bid({
                blindedBid: _blindedBid,
                deposit: msg.value
            }));
        }

        /// Revela tus pujas ocultas. Recuperarás los fondos de todas
        /// las pujas inválidas ocultadas de forma correcta y de
        /// todas las pujas salvo en aquellos casos en que sea la más alta.
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
                    // La puja no ha sido correctamente revelada.
                    // No se recuperan los fondos depositados.
                    continue;
                }
                refund += bid.deposit;
                if (!fake && bid.deposit >= value) {
                    if (placeBid(msg.sender, value))
                        refund -= value;
                }
                // Hace que el emisor no pueda reclamar dos veces
                // el mismo depósito.
                bid.blindedBid = 0;
            }
            msg.sender.transfer(refund);
        }

        // Esta función es "internal", lo que significa que sólo
        // se podrá llamar desde el propio contrato (o contratos
        // que deriven de él).
        function placeBid(address bidder, uint value) internal
                returns (bool success)
        {
            if (value <= highestBid) {
                return false;
            }
            if (highestBidder != 0) {
                // Devolverle el dinero de la puja
                // al anterior pujador con la puja más alta.
                pendingReturns[highestBidder] += highestBid;
            }
            highestBid = value;
            highestBidder = bidder;
            return true;
        }

        /// Retira una puja que ha sido superada.
        function withdraw() returns (bool) {
            var amount = pendingReturns[msg.sender];
            if (amount > 0) {
                // Es importante poner esto a cero porque el receptor
                // puede llamar a esta función de nuevo como parte
                // de la recepción antes de que `send` devuelva su valor.
                // (ver la observacin de arriba sobre condiciones -> efectos
                // -> interacción).
                pendingReturns[msg.sender] = 0;

                if (!msg.sender.send(amount)){
                    // Aquí no es necesario lanzar una excepción.
                    // Basta con reiniciar la cantidad que se debe devolver.
                    pendingReturns[msg.sender] = amount;
                    return false;
                }
            }
            return true;
        }

        /// Finaliza la subasta y envía la puja más alta
        /// al beneficiario.
        function auctionEnd()
            onlyAfter(revealEnd)
        {
            require(!ended);
            AuctionEnded(highestBidder, highestBid);
            ended = true;
            // Enviamos todo el dinero que tenemos, porque
            // parte de las devoluciones pueden haber fallado.
            beneficiary.transfer(this.balance);
        }
    }

.. index:: purchase, remote purchase, escrow

*************************
Compra a distancia segura
*************************

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

        /// Aborta la compra y reclama el ether.
        /// Sólo puede ser llamado por el vendedor
        /// antes de que el contrato se cierre.
        function abort()
            onlySeller
            inState(State.Created)
        {
            Aborted();
            state = State.Inactive;
            seller.transfer(this.balance);
        }

        /// Confirma la compra por parte del comprador.
        /// La transacción debe incluir la cantidad de ether
        /// multiplicada por 2. El ether quedará bloqueado
        /// hasta que se llame a confirmReceived.
        function confirmPurchase()
            inState(State.Created)
            condition(msg.value == (2 * value))
            payable
        {
            PurchaseConfirmed();
            buyer = msg.sender;
            state = State.Locked;
        }

        /// Confirma que tú (el comprador) has recibido el
        /// artículo. Esto desbloqueará el ether.
        function confirmReceived()
            onlyBuyer
            inState(State.Locked)
        {
            ItemReceived();
            // Es importante que primero se cambie el estado
            // para evitar que los contratos a los que se llama
            // abajo mediante `send` puedan volver a ejecutar esto.
            state = State.Inactive;

            // NOTA: Esto permite bloquear los fondos tanto al comprador
            // como al vendedor - debe usarse el patrón withdraw.

            buyer.transfer(value);
            seller.transfer(this.balance);
        }
    }

*******************
Canal de micropagos
*******************

Por escribir.
