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
            // con otros contratos (p.ej.: llaman a funciones o envían ether)
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

    
    pragma solidity >0.4.23 <0.7.0;

    contract BlindAuction {
        struct Bid {
            bytes32 blindedBid;
            uint deposit;
    }

    address payable public beneficiary;
    uint public biddingEnd;
    uint public revealEnd;
    bool public ended;

    mapping(address => Bid[]) public bids;

    address public highestBidder;
    uint public highestBid;

    // Retiradas permitidas de pujas previas
    mapping(address => uint) pendingReturns;

    event AuctionEnded(address winner, uint highestBid);

    // Los modificadores son una forma cómoda de validar los
    // inputs de las funciones. Abajo se puede ver cómo
    // `onlyBefore` se aplica a `bid`.
    // El nuevo cuerpo de la función pasa a ser el del modificador,
    // sustituyendo `_` por el anterior cuerpo de la función.
    modifier onlyBefore(uint _time) {
        require(now < _time); 
        _;
    }
    modifier onlyAfter(uint _time) {
        require(now > _time);
        _;
    }

    constructor(
        uint _biddingTime,
        uint _revealTime,
        address payable _beneficiary
    ) public {
        beneficiary = _beneficiary;
        biddingEnd = now + _biddingTime;
        revealEnd = biddingEnd + _revealTime;
    }

    // Efectúa la puja de manera oculta con `_blindedBid`=
    // keccak256(abi.encodePacked(value, fake, secret)).
    // El ether enviado sólo se reintegrará si la puja se revela de
    // forma correcta durante la fase de revelación. La puja es
    // válida si el ether envíado junto a la misma puja es al menos "value"
    // y "fake" como no verdadero (!=). Poner "fake" como verdadero y no enviar
    // la cantidad exacta, son formas de ocultar la verdadera puja
    // y aún así realizar el depósito necesario. La misma dirección
    // puede realizar múltiples pujas.
    function bid(bytes32 _blindedBid)
        public
        payable
        onlyBefore(biddingEnd)
    {
        bids[msg.sender].push(Bid({
            blindedBid: _blindedBid,
            deposit: msg.value
        }));
    }

    // Revela tus pujas ocultas. Recuperarás los fondos de todas
    // las pujas inválidas ocultadas de forma correcta y de
    // todas las pujas salvo en aquellos casos en que la puja sea la más alta.
    function reveal(
        uint[] memory _values,
        bool[] memory _fake,
        bytes32[] memory _secret
    )
        public
        onlyAfter(biddingEnd)
        onlyBefore(revealEnd)
    {
        uint length = bids[msg.sender].length;
        require(_values.length == length);
        require(_fake.length == length);
        require(_secret.length == length);

        uint refund;
        for (uint i = 0; i < length; i++) {
            Bid storage bidToCheck = bids[msg.sender][i];
            (uint value, bool fake, bytes32 secret) =
                    (_values[i], _fake[i], _secret[i]);
            if (bidToCheck.blindedBid != keccak256(abi.encodePacked(value, fake, secret))) {
                // La puja no ha sido correctamente revelada.
                // No se recuperan los fondos depositados.
                continue;
            }
            refund += bidToCheck.deposit;
            if (!fake && bidToCheck.deposit >= value) {
                if (placeBid(msg.sender, value))
                    refund -= value;
            }
            // Hace que el emisor no pueda reclamar dos veces
            // el mismo depósito.
            bidToCheck.blindedBid = bytes32(0);
        }
        require(msg.sender).transfer(refund);
    }

     /// Retira una puja que ha sido superada.
    function withdraw() public {
        uint amount = pendingReturns[msg.sender];
        if (amount > 0) {
            // Es importante poner esto a cero porque el receptor
            // puede llamar a esta función de nuevo como parte
            // de la recepción antes de que `send` devuelva su valor.
            // (ver la observacin de arriba sobre condiciones -> efectos
            // -> interacción).
            pendingReturns[msg.sender] = 0;

            msg.sender.transfer(amount);
        }
    }

    // Finaliza la subasta y envía la puja más alta
    // al beneficiario.
    function auctionEnd()
        public
        onlyAfter(revealEnd)
    {
        require(!ended);
        emit AuctionEnded(highestBidder, highestBid);
        ended = true;
        beneficiary.transfer(highestBid);
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
        if (highestBidder != address(0)) {
              // Devolverle el dinero de la puja al anterior pujador con la puja más alta.
            pendingReturns[highestBidder] += highestBid;
        }
        highestBid = value;
        highestBidder = bidder;
        return true;
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

En esta sección aprenderemos a desarrollar una implementación de ejemplo de un canal de pagos. Se usarán firmas criptográficas para realizar repetidas transferencias de Ether entre las mismas partes seguras, instantáneamente y sin comisiones de transacción. Para el ejemplo, necesitaremos entender cómo firmar y verificar las firmas, y configurar el canal.

Crear y verificar firmas
========================

Imagina que Alice quiere mandar una cantidad de Ether a Bob, p.ej.: Alice es el emisor y Bob es el receptor.

Alice solo necesita enviar un mensaje firmado cripográficamente off-chain (p.ej.: via email) a Bob y sería algo similar a llenar un cheque.

Alice y Bob usan firmas para autorizar transacciones, las cuales son posibles en los contratos inteligentes de la red Ethereum. Alice puede crear un contrato inteligente sencillo que le permita transferir Ether, pero en vez de llamar una función ella misma para iniciar un pago, ella puede dejar que Bob lo realice y a la vez pagar la comisión de transacción.

El contrato funciona de la siguiente forma:
    - Alice despliega el ``receiverPays``, agregando el Ether necesario para cubrir el pago que ella realizó.
    - Alice autoriza un pago al firmar un mensaje con su llave privada.
    - Alice envía el mensaje firmado criptográficamente a Bob. El mensaje no necesita mantenerse en secreto (lo explicaremos luego), y el mecanismo para enviarlo no es necesario.
    - Bob solicita su pago al presentar el mensaje firmado para el contrato inteligente, al verificar la autenticidad del mensaje, se liberan los fondos.
    

Creando la Firma
================

Alice no necesita interactuar con la red de Ethereum para firmar la transacción, el proceso es completamente offline. En este tutorial, firmaremos mensajes en el navegador usando `web3.js <https://github.com/ethereum/web3.js>`_ y `Metamask <https://metamask.io/>`_, usando el método descrito en `EIP-762 <https://github.com/ethereum/EIPs/pull/712>`_, ya que proporciona una serie de otros beneficios de seguridad.

::

    // El Hashing de primero hace las cosas más sencillas
    var hash = web3.utils.sha3("message to sign");
    web3.eth.personal.sign(hash, web3.eth.defaultAccount, function () {
        console.log("Signed");
    });
    
    ``web3.the.personal.sign`` antepone el tamaño del mensaje a los datos firmados. Desde que realizamos el hash primero, el mensaje siempre tendrá un tamaño de 32 Bytes. y por lo tanto este prefijo de longitud es siempre el mismo.
    

Qué Firmar
==========

Para un contrato que cumple con los pagos, el mensaje firmado debe de incuir:
    - La dirección del destinatario.
    - El monto a transferir.
    - La proteccción contra un `Ataque de Replay <https://es.wikipedia.org/wiki/Ataque_de_REPLAY>`_.

Un Ataque de Replay es cuando un mensaje firmado es reusado para reclamar autorización para una segunda acción. Para evitar una repetición de ataques usamos la misma técnica como en las mismas transacciones de Ethereum, un llamado nonce, el cual es un número de transacciones enviados por un cuenta. El contrato inteligente verifica si el nonce es usando varias veces.

Otro tipo de Ataque de Replay puede suceder cuando el propietario despliega un contrato inteligente de ``ReceiverPays``, realizando algún otro pago y luego destruye el contrato. Luego, él decide desplegar el contrato inteligente ``RecipientPays`` otra vez, pero el nuevo contrato no sabe cual nonce se utilizó en el despliege anterior, así el atacante puede utilizar el viejo mensaje de nuevo.

Alice puede protegerse contra estos ataques al incuir la dirección del contrato en el mensaje, y solamente los mensajes que estén en la dirección del contrato por sí mismo serán aceptados. Puedes encontrar un ejemplo en las primeras dos líneas de la función ``claimPayment()`` del contrato de ejemplo al final de esta sección

Empaquetar Argumentos
=====================

Ahora que hemos identificado qué información incluir en el mensaje firmado, estamos preparados para poner el mensaje junto, hashearlo y firmarlo. Por simplicidad, concatenamos los datos. La librería `ethereumjs-abi <https://github.com/ethereumjs/ethereumjs-abi>`_ proporciona una función llamada ``soliditySHA3`` que imita el comportamiento de la función ``keccak256`` que se utiliza en los argumentos codificados usando ``abi.encodePacked``. Aquí es una función de JavaScript que crear una firma adecuada para el ejemplo del ``ReceiverPays``:


::

    // recipient es la dirección que debería recibir el pago.
    // amount, en unidades wei, especifica cuanto ether debe ser enviado.
    // nonce puede ser un único número para así evitar Ataques de Replay.
    // contractAddress se usa para eveitar Ataque de Replay en contratos cruzados
    function signPayment(recipient, amount, nonce, contractAddress, callback) {
        var hash = "0x" + abi.soliditySHA3(
            ["address", "uint256", "uint256", "address"],
            [recipient, amount, nonce, contractAddress]
        ).toString("hex");

        web3.eth.personal.sign(hash, web3.eth.defaultAccount, callback);
    }
    

Recuperar el firmante del mensaje en Solidity
=============================================

En términos generales, Las firmas `ECDSA <https://es.wikipedia.org/wiki/ECDSA>`_ consisten en dos parámetros, ``r`` y ``s``. Las firmas en Ethereum incluyen un tercer parámetro llamado ``v``, que puede ser usado para verificar cuál cuenta con llave privada fue usada para firmar el mensaje, y el emisor de la transacción. Solidity proporciona una función incorporada llamada `ecrecover <https://docs.soliditylang.org/en/v0.8.6/units-and-global-variables.html#mathematical-and-cryptographic-functions>`_ que acepta un mensaje junto con los parámetros ``r``, ``s`` y ``v``, luego devuelve la dirección que fue usada para firmar el mensaje.

Extrayendo los Parámetros de Firma
==================================

Las firmas producidas por web3.js son la concatenación de ``r``, ``s``, y ``v``, así que el primer paso sería separar estos parámetros. Puedes hacer esto del lado del cliente, pero haciendo dentro del contrato inteligente, significa que solo necesitas enviar un parámetro de firma en vez de tres. Dividir un array de bytes en sus partes constituyentes es un desastre, así que usamos un ensamblador en línea (`inline assembly <https://docs.soliditylang.org/en/v0.8.6/assembly.html>`_) para realizar el trabajo en la fución ``SplitSignature`` (la tercera función en el contrato entero está al final de esta sección).

Calcular el Mensaje de Hash
===========================

El contrato necesita saber con exactitud cuáles son los parámetros firmados, y así debe recrear el mensaje desde los parámetros y usarlos para verificar la firma. Las funciones ``prefixed`` y ``recoverSigner`` lo ejecuta dentro de la función ``claimPayment``.

El Contrato Completo
====================

::

// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0 <0.9.0;
contract ReceiverPays {
    address owner = msg.sender;

    mapping(uint256 => bool) usedNonces;

    constructor() payable {}

    function claimPayment(uint256 amount, uint256 nonce, bytes memory signature) public {
        require(!usedNonces[nonce]);
        usedNonces[nonce] = true;

        // Esto recrea el mensaje que fue firmado por el cliente
        bytes32 message = prefixed(keccak256(abi.encodePacked(msg.sender, amount, nonce, this)));

        require(recoverSigner(message, signature) == owner);

        payable(msg.sender).transfer(amount);
    }

    // Destruye el contrato y reclama los fondos que sobran.
    function shutdown() public {
        require(msg.sender == owner);
        selfdestruct(payable(msg.sender));
    }

    // Métodos de firma.
    function splitSignature(bytes memory sig)
        internal
        pure
        returns (uint8 v, bytes32 r, bytes32 s)
    {
        require(sig.length == 65);

        assembly {
            // los primeros 32 bytes, luego el tamaño del prefijo.
            r := mload(add(sig, 32))
            // siguientes 32 bytes.
            s := mload(add(sig, 64))
            // último byte (primer byte de los siguientes 32 bytes).
            v := byte(0, mload(add(sig, 96)))
        }

        return (v, r, s);
    }

    function recoverSigner(bytes32 message, bytes memory sig)
        internal
        pure
        returns (address)
    {
        (uint8 v, bytes32 r, bytes32 s) = splitSignature(sig);

        return ecrecover(message, v, r, s);
    }

    // crea un hash prefijado para imitar el comportamiento de eth_sign.
    function prefixed(bytes32 hash) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hash));
    }
}


Escribiendo un Canal de Pago Simple
===================================
    
Alice ahora crea una implementación simple y completa de un canal de pagos; los canales usan firmas criptográficas para hacer que las transferencias repetidas en Ether sean seguras, instantáneas y sin comisión de transferencia.

¿Qué es un Canal de Pagos?
==========================

Los canales de pagos permite a los participantes hacer transferencias repetidas en Ether sin usar directamente las transacciones. Esto quiere decir que tu puedes evitar los atrasos y comisiones generadas por las transacciones. Exploraremos un canal de pago simple y unidireccional entre dos partes (Alice y Bob). Esto implica tres pasos:

    - Alice paga con Ether a través de un contrato inteligente. Esto "Abre" el canal de pago.
    - Alice firma los mensajes que especifican cuanto de Ether se le debe al destinario. Este paso se repite en cada pago.
    - Bob "cierra" el canal de pago, retirando la parte que le corresponde en Ether y enviando el resto de vuelta al emisor (sender).

Nota

Solamente los pasos 1 y 3 requieren las transacciones de Ethereum, El paso 2 significa que el sender transmite un mensaje firmado criptográficamente al destinatario a través de métodos off-chain (p.ej.: email). Esto quiere decir que solamente dos transacciones se requieren para soportar cualquier cantidad de transferencias.

Bob tiene la garantía de que recibirá sus pagos debido a que el contrato inteligente tiene custodia sobre el Ether y respeta el mensaje firmado valido. El contrato también impone un tiempo de espera, así que Alice tiene la garantía de recuperar sus fondos aunque el destinatario se niega a cerrar el canal. Depende de los participantes cuánto tiempo se mantiene abierto el canal de pago.

Para una transacción corta, como el pago de un servicio de Internet por consumo por minuto, el canal de pago debe mantenerse abierto por un tiempo determinado. Por otra parete, para un pago recurrente, como el pago de un empleado por hora de trabajo, el canal de pago debe mantenerse abierto por algunos meses del año.

Abriendo un Canal de Pago
=========================

Para abrir un canal de pago, Alice implementa el contrato inteligente, agregando el Ether que se depositará en custodia y especificará el destinatario y una duración máxima del canal. Esta es la función ``SimplePaymentChannel`` en el contrato al final de esta sección.

Realizando Pagos
================

Alice realiza los pagos al envíar mensajes firmados a Bob. Este paso se realiza totalmente fuera de la red de Ethereum. Los mensajes son firmados criptográficamente por el sender y luego son transmitidos directamente al destinatario.

Cada mensaje incluye la siguiente información:
    - La dirección del contrato inteligente usada para prevenir Ataques de Replay en contratos cruzados.
Making Payments
    - El monto total en Ethere que enviado al destinatario.
    
Un canal de pago se cierra una sola vez, al final de una serie de transferencias. debido a esto, solamente se canjea uno de los mensajes enviados. Esta es la razón Por la que cada mensaje especifica una cantidad total acumulada de Ether que se debe, en vez del monto de cada micropago individual. El destinatario, por naturaleza, elegirá canjear el mensaje más reciente porque es el que tiene el monto total más elevado. El nonce por mensaje no se necesita más, porque el contrato inteligente solo respeta un único mensaje. La dirección del contrato inteligente se usa aún para evitar que un mensaje destinado a un canal de pago se use para un canal diferente.

Aquí está el código modificado de JavaScript para firmar criptográficamente el mensaje de la sección anterior:

::

function constructPaymentMessage(contractAddress, amount) {
    return abi.soliditySHA3(
        ["address", "uint256"],
        [contractAddress, amount]
    );
}

function signMessage(message, callback) {
    web3.eth.personal.sign(
        "0x" + message.toString("hex"),
        web3.eth.defaultAccount,
        callback
    );
}

// contractAddress se usa para evitar ataques de Replay en contratos cruzados.
// amount, en unidades wei, especifica cuanto de Ethere debe de envíar.

function signPayment(contractAddress, amount, callback) {
    var message = constructPaymentMessage(contractAddress, amount);
    signMessage(message, callback);
}

Cerrando el Canal de Pago
=========================

Cuando Bob esté listo para recibir sus fondos, es momento de cerrar el canal de pago, llamando una función ``close`` en el contrato inteligente. Al cerrar este canal, se realiza el pago al destinatario el Ether que se le debe y el contrato se destruye, mandando el sobrante del Ether de vuelta a Alice. Para cerrar el canal, Bob necesita entregar un mensaje firmado a Alice.

Los contratos inteligentes deben verificar que el mensaje contiene una firma validad del emisor. El proceso para realizar esta verificación es el mismo como el proceso que el destinatario usa. Las funciones de Solidity ``isValidSignature`` y ``recoverSigner`` trabajan como sus contrapartes en JavaScript de la sección anterior, con la última sección tomada del contrato llamada ``ReceiverPays``.

Solo el destinatario del canal de pago puede llamar la función ``close``, quien naturalmente pasa el mensaje de pago más reciente porque ese mensaje lleva el compromiso de pago total más alto. Si al emisor, se le permite llamar a esta función, podría enviar una cantidad menor y engañar al destinatario con la deuda que le corresponde.

La función verifica que el mensaje firmado coincida con los parámetros dados. Si todo sale bien, se le envía la parte que le corresponde de Ether al destinatario, y el sender recibe el resto a través de ``selfdestruct``. Puedes ver la función ``close`` en el contrato completo.


Caducidad del Canal
===================

Bob puede cerrar el canal de pago en cualquier momento, pero si no lo hace, Alice necesita una manera de recuperar sus fondos bloqueados. Se estableció un tiempo de expiración al momento de implementar el contrato. Una vez que pasa ese tiempo, Alice puede llamar la función ``claimTimeout`` para recuperar sus fondos. Puedes ver esta función ``claimTimeout`` en el contrato completo.

Luego de llamar esta función, Bob no puede recibir ningún Ether, por lo tanto, es importante que Bob cierre el canal antes de que el tiempo de expiración llegué.


El Contrato Completo

::

// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0 <0.9.0;
contract SimplePaymentChannel {
    address payable public sender;      // La cuenta de quien envía el pago.
    address payable public recipient;   // La cuenta de quién recibe el pago.
    uint256 public expiration;  // Tiempo de expiración del contrato en caso de que el destinatario nunca lo cierre.

    constructor (address payable _recipient, uint256 duration)
        payable
    {
        sender = payable(msg.sender);
        recipient = _recipient;
        expiration = block.timestamp + duration;
    }

    // El destinatario puede cerrar el canal en cualquier momento al presentar un
    // monto firmado al emisor (sender). Al destinatario se le enviará esa cantidad,
    // y el resto se le regresará al sender
    function close(uint256 amount, bytes memory signature) public {
        require(msg.sender == recipient);
        require(isValidSignature(amount, signature));

        recipient.transfer(amount);
        selfdestruct(sender);
    }

    // El sender puede extender el tiempo de expiración en cualquier momento
    function extend(uint256 newExpiration) public {
        require(msg.sender == sender);
        require(newExpiration > expiration);

        expiration = newExpiration;
    }

    // Si el tiempo de expiración es alcanzado sin que el destinatario cierre el canal,
    // entonces todo el Ether es regresado al emisor (sender).
    function claimTimeout() public {
        require(block.timestamp >= expiration);
        selfdestruct(sender);
    }

    function isValidSignature(uint256 amount, bytes memory signature)
        internal
        view
        returns (bool)
    {
        bytes32 message = prefixed(keccak256(abi.encodePacked(this, amount)));

        // verifica que la firma sea del emisor
        return recoverSigner(message, signature) == sender;
    }

    // Todas las funciones siguientes son tomadas del capítulo
    // 'Crear y verificar firmas' chapter.

    function splitSignature(bytes memory sig)
        internal
        pure
        returns (uint8 v, bytes32 r, bytes32 s)
    {
        require(sig.length == 65);

        assembly {
            // first 32 bytes, after the length prefix
            r := mload(add(sig, 32))
            // second 32 bytes
            s := mload(add(sig, 64))
            // final byte (first byte of the next 32 bytes)
            v := byte(0, mload(add(sig, 96)))
        }

        return (v, r, s);
    }

    function recoverSigner(bytes32 message, bytes memory sig)
        internal
        pure
        returns (address)
    {
        (uint8 v, bytes32 r, bytes32 s) = splitSignature(sig);

        return ecrecover(message, v, r, s);
    }

    // Construye un hash prefijado para imitar el comportamiento de eth_sign.
    function prefixed(bytes32 hash) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hash));
    }
}


Nota

La función ``splitSignature`` no usa todos los pasos de verificación. Una implementación real sería usar una librería de testing más rigurosa, como la versión de `openZeppelin <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/ECDSA.sol>`_ en este código.

Verificando Pagos
=================

A diferencia de la sección anterior, los mensajes en un canal de pago no son canjeados al instante. El destinatario realiza un seguimiento del último mensaje y lo canjea cuando es momento de cerrar el canal de pago. Esto quiere decir que es importante que el destinatario realice su propia verificación en cada mensaje. De otra forma, no habrá garantía que el destinatario pueda cobrar.

El destinatario puede verificar cada mensaje realizando lo siguiente:
    - Verificar que la dirección del contrato en el mensaje coincida con el canal de pago
    - Verificando que el nuevo monto total sea el monto esperado.
    - Verificar que el nuevo total no exceda el monto del Ethere depositado.
    - Verificar que la firma sea validad y venga del canal del pago del emisor (sender).
    
Usaremos la librería `ethereumjs-util <https://github.com/ethereumjs/ethereumjs-util>`_ para escribir este proceso de verificación. El paso final se puede realizar de varias formas, pero nosotros usaremos JavaScript. El siguiente código toma prestado la función ``constructPaymentMessage`` desde el código escrito en JavaScript anteriormente:

::

// Esto imita el comportamiento prefijado del método JSON-RPC eth_sign.
function prefixed(hash) {
    return ethereumjs.ABI.soliditySHA3(
        ["string", "bytes32"],
        ["\x19Ethereum Signed Message:\n32", hash]
    );
}

function recoverSigner(message, signature) {
    var split = ethereumjs.Util.fromRpcSig(signature);
    var publicKey = ethereumjs.Util.ecrecover(message, split.v, split.r, split.s);
    var signer = ethereumjs.Util.pubToAddress(publicKey).toString("hex");
    return signer;
}

function isValidSignature(contractAddress, amount, signature, expectedSigner) {
    var message = prefixed(constructPaymentMessage(contractAddress, amount));
    var signer = recoverSigner(message, signature);
    return signer.toLowerCase() ==
        ethereumjs.Util.stripHexPrefix(expectedSigner).toLowerCase();
}


Contratos Modulares
===================

Un enfoque por módulos para crear tus contratos ayuda a reducir la complejidad y mejorar la legibilidad, lo que ayudará a identificar posibles bugs y vulnerabilidades durante el desarrollo o revisión del código. Si especificas y controlas el comportamiento de cada módulo por separado, las interacciones que tienes que considerar serían solamente aquellas entre las especificaciones y no en cada parte dinámica del contrato. En el siguiente ejemplo, el contrato usa el método ``move`` de la `librería <https://docs.soliditylang.org/en/v0.8.6/contracts.html#libraries>`_ ``balances`` para verificar los balances envíados entre las direcciones que esperas que coincidad. De esta forma, la librería ``Blances`` proporciona un componente aislado que realiza un seguimiento apropiado de los balances entre cada cuenta. Es fácil verificar que la librería ``Balances`` nunca produce saldos negativo o excesivos al total y la suma de todos los balances o saldos es invariante durante la vigencia del contrato.

::

// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.5.0 <0.9.0;

library Balances {
    function move(mapping(address => uint256) storage balances, address from, address to, uint amount) internal {
        require(balances[from] >= amount);
        require(balances[to] + amount >= balances[to]);
        balances[from] -= amount;
        balances[to] += amount;
    }
}

contract Token {
    mapping(address => uint256) balances;
    using Balances for *;
    mapping(address => mapping (address => uint256)) allowed;

    event Transfer(address from, address to, uint amount);
    event Approval(address owner, address spender, uint amount);

    function transfer(address to, uint amount) public returns (bool success) {
        balances.move(msg.sender, to, amount);
        emit Transfer(msg.sender, to, amount);
        return true;

    }

    function transferFrom(address from, address to, uint amount) public returns (bool success) {
        require(allowed[from][msg.sender] >= amount);
        allowed[from][msg.sender] -= amount;
        balances.move(from, to, amount);
        emit Transfer(from, to, amount);
        return true;
    }

    function approve(address spender, uint tokens) public returns (bool success) {
        require(allowed[msg.sender][spender] == 0, "");
        allowed[msg.sender][spender] = tokens;
        emit Approval(msg.sender, spender, tokens);
        return true;
    }

    function balanceOf(address tokenOwner) public view returns (uint balance) {
        return balances[tokenOwner];
    }
}

