#########################################
Introducción a los Contratos Inteligentes
#########################################

.. _contrato-inteligente-simple:

******************************
Un contrato inteligente simple
******************************

Vamos a comenzar con el ejemplo más básico. No pasa nada si no entiendes nada ahora, entraremos más en detalle posteriormente.

Almacenamiento
==============

::

    pragma solidity ^0.4.0;

    contract SimpleStorage {
        uint storedData;

        function set(uint x) {
            storedData = x;
        }

        function get() constant returns (uint) {
            return storedData;
        }
    }

La primera línea simplemente dice que el código fuente se ha escrito en la versión 0.4.0 de Solidity o en otra superior totalmente compatible (cualquiera anterior a la 0.5.0). Esto es para garantizar que el contrato no se va a comportar de una forma diferente con una versión más nueva del compilador. La palabra reservada ``pragma`` es llamada de esa manera porque, en general, los "pragmas" son instrucciones para el compilador que indican como este debe operar con el código fuente (p.ej.: `pragma once <https://en.wikipedia.org/wiki/Pragma_once>`_).  .

Un contrato para Solidity es una colección de código (sus *funciones*) y datos (su *estado*) que residen en una dirección específica en la blockchain de Ethereum. La línea ``uint storedData;`` declara una variable de estado llamada ``storedData`` del tipo ``uint`` (unsigned integer de 256 bits). Esta se puede entender como una parte única en una base de datos que puede ser consultada o modificada llamando a funciones del código que gestiona dicha base de datos. En el caso de Ethereum, este es siempre el contrato propietario. Y en este caso, las funciones ``set`` y ``get`` se pueden usar para modificar o consultar el valor de la variable.

Para acceder a una variable de estado, no es necesario el uso del prefijo ``this.`` como es habitual en otros lenguajes.

Este contrato no hace mucho todavía (debido a la infraestructura construída por Ethereum), simplemente permite a cualquiera almacenar un número accesible para todos sin un (factible) modo de prevenir la posibilidad de publicar este número. Por supuesto, cualquiera podría simplemente hacer una llamada ``set`` de nuevo con un valor diferente y sobreescribir el número inicial, pero este número siempre permanecería almacenado en la historía de la blockchain. Más adelante, veremos como imponer restricciones de acceso de tal manera que sólo tú puedas cambiar el número.
.. index:: ! submoneda

Ejemplo de Submoneda
====================

El siguiente contrato va a implementar la forma más sencilla de una criptomoneda. Se pueden generar monedas de la nada, pero sólo la persona que creó el contrato estará habilitada para hacerlo (es trivial implementar un esquema diferente de emisión). Es más, cualquiera puede enviar monedas a otros sin necesidad de registrarse con usuario y clave - sólo hace falta un par de claves Ethereum.


::

    pragma solidity ^0.4.0;

    contract Coin {
        // The keyword "public" makes those variables
        // readable from outside.
        address public minter;
        mapping (address => uint) public balances;

        // Events allow light clients to react on
        // changes efficiently.
        event Sent(address from, address to, uint amount);

        // This is the constructor whose code is
        // run only when the contract is created.
        function Coin() {
            minter = msg.sender;
        }

        function mint(address receiver, uint amount) {
            if (msg.sender != minter) return;
            balances[receiver] += amount;
        }

        function send(address receiver, uint amount) {
            if (balances[msg.sender] < amount) return;
            balances[msg.sender] -= amount;
            balances[receiver] += amount;
            Sent(msg.sender, receiver, amount);
        }
    }

Este contrato introduce algunos conceptos nuevos que vamos a detallar uno a uno.

La línea ``address public minter;`` declara una variable de estado de tipo address (dirección) que es públicamente accesible. El tipo ``address`` es un valor de 160-bit que no permite operaciones aritméticas. Es apropiado para almacenar direcciones de contratos o pares de claves pertenecientes a personas externas. La palabra reservada ``public`` genera automáticamente una función que permite el acceso al valor actual de la variable de estado. Sin esta palabra reservada, otros contratos no tienen manera de acceder a la variable.
Está función sería algo como esto::

    function minter() returns (address) { return minter; }

Por supuesto, añadir una función exactamente como esa no funcionará porque deberíamos tener una función y una variable de estado con el mismo nombre, pero afortunadamente, has cogido la idea - el compilador se lo imagina por tí.

.. index:: mapping

La siguiente línea, ``mapping (address => uint) public balances;`` también crea una variable de estado pública, pero se trata de un tipo de datos más complejo. El tipo mapea direcciones a enteros sin signo.
Los mapeos (Mappings) pueden ser vistos como tablas hash `hash tables <https://en.wikipedia.org/wiki/Hash_table>`_ que son virtualmente inicializadas de tal forma que cada clave candidata existe y es mapeada a un valor cuya representación en bytes es todo ceros. 
Esta anología no va mucho más allá, ya que no es posible obtener una lista de todas las claves de un mapeo, ni tampoco una lista de todos los valores. Por eso hay que tener en cuenta (o mejor, conservar una lista o usar un tipo de datos más avanzado) lo que se añade al mapping o usarlo en un contexto donde no es necesario, como este caso. La función getter creada mediante la palabra reservada ``public`` es un poco más compleja en este caso. De forma aproximada, es algo parecido a lo siguiente::

    function balances(address _account) returns (uint) {
        return balances[_account];
    }

Como se puede ver, se puede usar esta función para, de forma sencilla, consultar el balance de una única cuenta.

.. index:: event

La línea ``event Sent(address from, address to, uint amount);`` declara un evento que es disparado en la última línea de la ejecución 
``send``. Las interfaces de ususario (como las de servidor, por supuesto) pueden escuchar esos eventos que están siendo disparados en la blockchain sin mucho coste. Tan pronto son disparados, el listener tambiñen recibirá los argumentos ``from``, ``to`` y ``amount``, que hacen más fácil trazar las transacciones. Con el fin de escuchar este evento, se podría usar ::

    Coin.Sent().watch({}, '', function(error, result) {
        if (!error) {
            console.log("Coin transfer: " + result.args.amount +
                " coins were sent from " + result.args.from +
                " to " + result.args.to + ".");
            console.log("Balances now:\n" +
                "Sender: " + Coin.balances.call(result.args.from) +
                "Receiver: " + Coin.balances.call(result.args.to));
        }
    })

Es interesante como la función generada automáticamente ``balances`` es llamada desde la interfaz de usuario.

.. index:: coin

La función especial ``Coin`` es el constructor que es ejecutado durante la creación de un contrato y no puede ser llamada con posterioridad. Almacena permanentemente la dirección de la persona que crea el contrato: ``msg`` (junto con ``tx`` y ``block``) es una variable global mágica que contiene propiedades que permiten el acceso a la blockchain. ``msg.sender`` es siempre la dirección donde la función es siempre la dirección donde la llamada a la función actual (externa) es originada.

Finalmente, las funciones que actualmente concluirán con el contrato pueden ser llamadas por usuarios y contratos como son ``mint`` y ``send``. Si ``mint`` es llamado por cualquiera excepto la cuenta que creó el contrato, nada pasará. Por otro lado, ``send`` puede ser usado por todos (los que ya tienen algunas de estas monedas) para enviar monedas a cualquier otro. Hay que tener en cuenta que si se usa este contrato para enviar monedas a una dirección, no se verá reflejado cuando se busque la dirección en un explorador de la blockchain por el hecho de enviar monedas y que los balances sólo sean guardados en el almacenamiento particular de este contrato de moneda. Con el uso de eventos es relativamente sencillo crear un "explorador de la blockchain" que monitorice las transacciones y los balances de la nueva moneda.

.. _blockchain-basics:

*************************
Fundamentos de Blockchain
*************************

Las blockchains son un concepto no muy difícil de entender para desarrolladores. La razón es que la mayoría de las complicaciones (minería, `hashing <https://en.wikipedia.org/wiki/Cryptographic_hash_function>`_, `elliptic-curve cryptography <https://en.wikipedia.org/wiki/Elliptic_curve_cryptography>`_, `peer-to-peer networks <https://en.wikipedia.org/wiki/Peer-to-peer>`_, etc.) están justo ahí para proveer un conjunto de funcionalidades y espectativas. Una vez que aceptas estas funcionalidades tal cual vienen dadas, no tienes que preocuparte por la tecnología que lleva inmersa - o, ¿tienes que saber realmente cómo Amazon AWS funciona internamente para poder usarlo?.

.. index:: transacción

Transacciones
=============

Una blockchain es una base de datos transaccional globalmente compartida. Esto quiere decir que todos pueden leer las entradas en la base de datos simplemente participando en la red. Si quieres cambiar algo en la base de datos, tienes que crear una transacción a tal efecto que tiene qie ser aceptada por todos los demás.
La palabra transacción implica que el cambio que quieres hacer (asumiendo que quieres cambiar dos valores al mismo tiempo) no se hace o no se aplica completamente. Es más, mientras tu transacción es aplicada en la base de datos, ninguna otra transacción puede modificarla. 

Como un ejemplo, imagine una tabla que lista los balances de todas las cuentas en una divisa electrónica. Si una transferencia de una cuenta a otra es solicitada, la naturaleza transaccional de la base de datos garantiza que la cantidad que es sustraída de una cuenta, es añadida en la otra. Si por la razón que sea, añadir la cantidad a la cuenta de destino no es posible, la cuenta origen tampoco se modifica. 

Yendo más allá, una transacción es siempre firmada criptográficamente por el remitente (creador). Esto la hace más robusta para garantizar el acceso a modificaciones específicas de la base de datos. En el ejemplo de divisas electrónicas, un simple chequeo asegura que sólo la persona que posee las claves de la cuenta puede transferir dinero desde ella.

.. index:: ! bloque

Bloques
=======

Un obstáculo mayor que sobrepasar es el que, en términos de Bitcoin, es llamado un ataque de "doble gasto": ¿qué ocurre si dos transacciones existentes en la red quieren borrar una cuenta?, ¿un conflicto?.

La respuesta abstracta a esto es que no tienes de qué preocuparte. Un orden para estas transacciones será seleccitonado por tí, las transacciones se aglutinarán en lo que es llamado "bloque" y entonces serán ejecutadas y distribuídas entre todos los nodos participantes. Si dos transacciones se contradicen, la que concluye en segundo lugar será rechazada y no formará parte del bloque.

Estos bloques forman una secuencia lineal en el tiempo y del que viene la palabra cadema de bloques o "blockchain". Los bloques son añadidos a la cadena en intervalos regulares - para Ethereum esto viene a significar cada 17 segundos.

Como parte del "mecanismo de selcción de orden" (que se conoce como minería), tiene que pasar que los bloques sean revertidos de cuando en cuando, pero sólo en el extremo o "tip" de la cadena. Cuanto más bloques se añaden encima, menos problable es. Entonces, sucedería que tus transacciones sean revertidas e incluso borradas de la blockchain, y cuanto más esperes, menos problable será.


.. _maquina-virtual-ethereum:

.. index:: !evm, ! ethereum virtual machine

************************
Máquina Virtual Ethereum
************************

Introducción
============

La máquina virtual de Ethereum (EVM por sus siglas en inglés) es un entorno de ejecución de contratos inteligentes en Ethereum.Va más allá de una configuración tipo sandbox ya que se encuentra totalmente aislada, lo que significa que el código que se ejecuta en la EVM no tiene acceso a la red, ni al sistema de ficheros, ni a cualquier otro proceso. Incluso los contratos inteligentes tienen acceso limitado a otros contratos inteligentes.

.. index:: ! cuentas, direcciones, almacenamiento, balance

Cuentas
=======

Hay dos tipos de cuentas en Ethereum que comparten el mismo espacio de dirección: **Cuentas externas** que están controladas por un par de claves pública-privadas (p-ej.: humanos) y **Cuentas contrato** que están controladas por el código almacenado conjuntamente con la cuenta.

La dirección de una cuenta externa viene determinada por la clave pública mientras que la dirección de la cuenta contrato se define en el momento que se crea dicho contrato (se deriva de la dirección del creador y del número de transacciones enviadas desde esa dirección, el llamado "nonce").

Independientemente de que la cuenta almacene código, los dos tipos son tratados de forma equitativa por la EVM.

Cada cuenta tiene un almacenamiento persistente clave-valor que mapea palabras de 256-bit a palabras de 256-bit llamado **almacenamiento**.

Más allá, cada cuenta tiene un **balance** en Ether (in "Wei" para ser exactos) que puede ser modificado enviando transacciones que incluyen Ether.

.. index:: ! transaction

Transactions
============

A transaction is a message that is sent from one account to another
account (which might be the same or the special zero-account, see below).
It can include binary data (its payload) and Ether.

If the target account contains code, that code is executed and
the payload is provided as input data.

If the target account is the zero-account (the account with the
address ``0``), the transaction creates a **new contract**.
As already mentioned, the address of that contract is not
the zero address but an address derived from the sender and
its number of transactions sent (the "nonce"). The payload
of such a contract creation transaction is taken to be
EVM bytecode and executed. The output of this execution is
permanently stored as the code of the contract.
This means that in order to create a contract, you do not
send the actual code of the contract, but in fact code that
returns that code.

.. index:: ! gas, ! gas price

Gas
===

Upon creation, each transaction is charged with a certain amount of **gas**,
whose purpose is to limit the amount of work that is needed to execute
the transaction and to pay for this execution. While the EVM executes the
transaction, the gas is gradually depleted according to specific rules.

The **gas price** is a value set by the creator of the transaction, who
has to pay ``gas_price * gas`` up front from the sending account.
If some gas is left after the execution, it is refunded in the same way.

If the gas is used up at any point (i.e. it is negative),
an out-of-gas exception is triggered, which reverts all modifications
made to the state in the current call frame.

.. index:: ! storage, ! memory, ! stack

Storage, Memory and the Stack
=============================

Each account has a persistent memory area which is called **storage**.
Storage is a key-value store that maps 256-bit words to 256-bit words.
It is not possible to enumerate storage from within a contract
and it is comparatively costly to read and even more so, to modify
storage. A contract can neither read nor write to any storage apart
from its own.

The second memory area is called **memory**, of which a contract obtains
a freshly cleared instance for each message call. Memory is linear and can be
addressed at byte level, but reads are limited to a width of 256 bits, while writes
can be either 8 bits or 256 bits wide. Memory is expanded by a word (256-bit), when
accessing (either reading or writing) a previously untouched memory word (ie. any offset
within a word). At the time of expansion, the cost in gas must be paid. Memory is more
costly the larger it grows (it scales quadratically).

The EVM is not a register machine but a stack machine, so all
computations are performed on an area called the **stack**. It has a maximum size of
1024 elements and contains words of 256 bits. Access to the stack is
limited to the top end in the following way:
It is possible to copy one of
the topmost 16 elements to the top of the stack or swap the
topmost element with one of the 16 elements below it.
All other operations take the topmost two (or one, or more, depending on
the operation) elements from the stack and push the result onto the stack.
Of course it is possible to move stack elements to storage or memory,
but it is not possible to just access arbitrary elements deeper in the stack
without first removing the top of the stack.

.. index:: ! instruction

Instruction Set
===============

The instruction set of the EVM is kept minimal in order to avoid
incorrect implementations which could cause consensus problems.
All instructions operate on the basic data type, 256-bit words.
The usual arithmetic, bit, logical and comparison operations are present.
Conditional and unconditional jumps are possible. Furthermore,
contracts can access relevant properties of the current block
like its number and timestamp.

.. index:: ! message call, function;call

Message Calls
=============

Contracts can call other contracts or send Ether to non-contract
accounts by the means of message calls. Message calls are similar
to transactions, in that they have a source, a target, data payload,
Ether, gas and return data. In fact, every transaction consists of
a top-level message call which in turn can create further message calls.

A contract can decide how much of its remaining **gas** should be sent
with the inner message call and how much it wants to retain.
If an out-of-gas exception happens in the inner call (or any
other exception), this will be signalled by an error value put onto the stack.
In this case, only the gas sent together with the call is used up.
In Solidity, the calling contract causes a manual exception by default in
such situations, so that exceptions "bubble up" the call stack.

As already said, the called contract (which can be the same as the caller)
will receive a freshly cleared instance of memory and has access to the
call payload - which will be provided in a separate area called the **calldata**.
After it has finished execution, it can return data which will be stored at
a location in the caller's memory preallocated by the caller.

Calls are **limited** to a depth of 1024, which means that for more complex
operations, loops should be preferred over recursive calls.

.. index:: delegatecall, callcode, library

Delegatecall / Callcode and Libraries
=====================================

There exists a special variant of a message call, named **delegatecall**
which is identical to a message call apart from the fact that
the code at the target address is executed in the context of the calling
contract and ``msg.sender`` and ``msg.value`` do not change their values.

This means that a contract can dynamically load code from a different
address at runtime. Storage, current address and balance still
refer to the calling contract, only the code is taken from the called address.

This makes it possible to implement the "library" feature in Solidity:
Reusable library code that can be applied to a contract's storage, e.g. in
order to  implement a complex data structure.

.. index:: log

Logs
====

It is possible to store data in a specially indexed data structure
that maps all the way up to the block level. This feature called **logs**
is used by Solidity in order to implement **events**.
Contracts cannot access log data after it has been created, but they
can be efficiently accessed from outside the blockchain.
Since some part of the log data is stored in `bloom filters <https://en.wikipedia.org/wiki/Bloom_filter>`_, it is
possible to search for this data in an efficient and cryptographically
secure way, so network peers that do not download the whole blockchain
("light clients") can still find these logs.

.. index:: contract creation

Create
======

Contracts can even create other contracts using a special opcode (i.e.
they do not simply call the zero address). The only difference between
these **create calls** and normal message calls is that the payload data is
executed and the result stored as code and the caller / creator
receives the address of the new contract on the stack.

.. index:: selfdestruct

Self-destruct
=============

The only possibility that code is removed from the blockchain is
when a contract at that address performs the ``selfdestruct`` operation.
The remaining Ether stored at that address is sent to a designated
target and then the storage and code is removed from the state.

.. warning:: Even if a contract's code does not contain a call to ``selfdestruct``,
  it can still perform that operation using ``delegatecall`` or ``callcode``.

.. note:: The pruning of old contracts may or may not be implemented by Ethereum
  clients. Additionally, archive nodes could choose to keep the contract storage
  and code indefinitely.

.. note:: Currently **external accounts** cannot be removed from the state.
