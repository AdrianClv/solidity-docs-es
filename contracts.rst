.. index:: ! contratos

#########
Contratos
#########

Los contratos en Solidity son similares a las clases en los lenguajes orientados a objeto. Los contratos contienen datos persistentes almacenados en variables de estados y funciones que pueden modificar estas variables. Llamar a una función de un contrato diferente (instancia) realizará una llamada a una función del EVM (Máquina Virtual de Ethereum) para que cambie el contexto de manera que las variables de estado no esten accesibles.

.. index:: ! contrato;creacion

***************
Crear Contratos
***************

Contratos pueden crearse "desde fuera" o desde contratos en Solidity. Cunando se crea un contrato, su constructor (una función con el mismo nombre que el contrato) se ejecuta una sola vez.

El contructor es opcional. Se admite un solo contsructor, lo que significa que sobrecargar no está soportado.

Desde ``web3.js``, es decir la API de JavaScript, esto se hace de la siguiente manera::

    // Es necesario especificar alguna fuente, incluido el nombre del contrato para los parametros de abajo
    var source = "contract CONTRACT_NAME { function CONTRACT_NAME(uint a, uint b) {} }";

    // La matriz abi en json generada por el compilador
    var abiArray = [
        {
            "inputs":[
                {"name":"x","type":"uint256"},
                {"name":"y","type":"uint256"}
            ],
            "type":"constructor"
        },
        {
            "constant":true,
            "inputs":[],
            "name":"x",
            "outputs":[{"name":"","type":"bytes32"}],
            "type":"function"
        }
    ];

    var MyContract_ = web3.eth.contract(source);
    MyContract = web3.eth.contract(MyContract_.CONTRACT_NAME.info.abiDefinition);
    
    // Desplegar el nuevo contrato
    var contractInstance = MyContract.new(
        10,
        11,
        {from: myAccount, gas: 1000000}
    );

.. index:: constructor;argumentos

Internamente, los argumentos del constructor son transmitidos después del propio código del contrato, pero no se tiene que preocupar de eso si utiliza ``web3.js``.

Si un contrato quiere crear otros contrato, el creador del código fuente (y el binario) del contrato nuevamente creado tiene que estar informado. Eso significa que la creación de dependencias cíclicas es imposible.

::

    pragma solidity ^0.4.0;

    contract OwnedToken {
        // TokenCreator es un contrato que està definido más abajo. 
        // No hay problema en referenciarlo, siempre y cuando no está 
        // siendo utilizado para crear un contrato nuevo.
        TokenCreator creator;
        address owner;
        bytes32 name;

        // Esto es el constructor que registra el creador y el nombre 
        // que se le ha asignado
        function OwnedToken(bytes32 _name) {
            // Se accede a las variables de estado por su nombre
            // y no por ejemplo por this.owner. Eso también se aplica 
            // a las funciones y, especialmente en los constructores, 
            // solo está permitido llamarlas de esa manera ("internal"), 
            // porque el propio contrato no existe todavía.
            owner = msg.sender;
            // Hacemos una conversión explícita de tipo, desde `address`
            // a `TokenCreator` y asumimos que el tipo del contrato que hace
            // la llamada es TokenCreator, ya que realmente no hay
            // formas de corroborar eso.
            creator = TokenCreator(msg.sender);
            name = _name;
        }

        function changeName(bytes32 newName) {
            // Solo el creador puede modificar el nombre --
            // la comparación es posible ya que los contratos 
            // se pueden implicitamente convertir a direcciones.
            if (msg.sender == address(creator))
                name = newName;
        }

        function transfer(address newOwner) {
            // Solo el creador actual puede transferir el token.
            if (msg.sender != owner) return;
            // También vamos a querer preguntar al creador 
            // si la transferencia ha salido bien. Note que esto
            // tiene como efecto llamar a una función del contrato 
            // que está definido más abajo. Si la llamada no funciona
            // (p.ej si no queda gas), la ejecución para aquí inmediatamente.
            if (creator.isTokenTransferOK(owner, newOwner))
                owner = newOwner;
        }
    }

    contract TokenCreator {
        function createToken(bytes32 name)
           returns (OwnedToken tokenAddress)
        {
            // Crea un contrato para crear un nuevo Token.
            // Del lado de JavaScript, el tipo que se nos devuelve
            // simplemente es la dirección ("address"), ya que ese
            // es el tipo más cerca disponible en el ABI.
            return new OwnedToken(name);
        }

        function changeName(OwnedToken tokenAddress, bytes32 name) {
            // De nuevo, el tipo externo de "tokenAddress" 
            // simplemente es "address".
            tokenAddress.changeName(name);
        }

        function isTokenTransferOK(
            address currentOwner,
            address newOwner
        ) returns (bool ok) {
            // Verifica un condición arbitraria
            address tokenAddress = msg.sender;
            return (keccak256(newOwner) & 0xff) == (bytes20(tokenAddress) & 0xff);
        }
    }

.. index:: ! visibilidad, externa, pública, privada, interna

.. _visibilidad-y-getters:

*********************
Visibilidad y Getters
*********************

Ya que Solidity sólo conoce dos tipos de llamadas a una función (las internas que no generan una llamada al EVM (también llamadas "llamadas mensaje") y las externas que si generan una llamada al EVM), hay cuatro tipos de visibilidades para las funciones y las variables de estado.

Una función puede especificarse como ``externa``, ``pública``, ``interna`` o ``privada``. Por defecto una función es ``pública``. Para las variables de estado, el tipo ``externa`` no es posible y el tipo por defecto es ``interna``.

``externa``: Funciones externas son parte de la interfaz del contrato, lo que significa que pueden llamarse desde otros contratos y vía transacciones. Una función externa ``f`` no puede llamarse internamente (por ejemplo ``f()`` no funciona, pero ``this.f()`` funciona). Las funciones externas son a veces más eficientes cuando reciben grandes matrices de datos.
    
``pública``: Funciones públicas son parte de la interfaz del contrato y pueden llarmarse internamente o vía mensajes. Para las variables de estado públicas, se genera una función getter automática (ver más abajo).

``interna``: Estas funciones y variables de estado sólo pueden llamarse internamente (es decir desde dentro del contrato actual o desde contratos de derivan del mismo), sin poder usarse ``this``.

``private``: Las funciones y variables de estado privadas sólo están visibles para el contrato en el que se han definido y no para contratos de derivan del mismo.

.. note:: Todo lo que está definido dentro de un contrato es visible para todos los observadores externos. Definir algo como ``privado`` sólo impide que otros contratos puedan acceder y modificar la información, pero esta información siempre será visible para todo el mundo, incluso fuera de la blockchain.

Es especificador de visibilidad se pone después del tipo para las variables de estado y entre la lista de parametros y la lista de parametros que devuelven información para las funciones.

::

    pragma solidity ^0.4.0;

    contract C {
        function f(uint a) private returns (uint b) { return a + 1; }
        function setData(uint a) internal { data = a; }
        uint public data;
    }

En el siguiente ejemplo, ``D``, puede llamar a ``c.getData()`` para recuperar el valor de ``data`` en el almacén de estado, pero no puede llamar a ``f``. El contrato ``E`` deriva de ``C`` y, por lo tanto, puede llamar a ``compute``.

::

    pragma solidity ^0.4.0;

    contract C {
        uint private data;

        function f(uint a) private returns(uint b) { return a + 1; }
        function setData(uint a) { data = a; }
        function getData() public returns(uint) { return data; }
        function compute(uint a, uint b) internal returns (uint) { return a+b; }
    }


    contract D {
        function readData() {
            C c = new C();
            uint local = c.f(7); // error: el ???miembro (member) "f" no es visible
            c.setData(3);
            local = c.getData();
            local = c.compute(3, 5); // error: el ???miembro (member) "compute" no es visible
        }
    }


    contract E is C {
        function g() {
            C c = new C();
            uint val = compute(3, 5);  // acceso a un miembro interno ???(from derivated to parent contract)
        }
    }

.. index:: ! getter;funcion, ! funcion;getter

Funciones Getter
================

El compilador crea automaticamente funciones getter para todas las variables de estado **publicas**. En el contrato que se muestra abajo, el compilador va a generar una función llamada ``data`` que no lee ningún argumento y devuelve un ``unint``, el valor de la variable de estado ``data``. La inicialización de las variables de estado se puede hacer en el momento de la declaración. 

::

    pragma solidity ^0.4.0;

    contract C {
        uint public data = 42;
    }


    contract Caller {
        C c = new C();
        function f() {
            uint local = c.data();
        }
    }

Las funciones getter tienen visibilidad externa. Si se accede al símbolo internamente (es decir sin ``this.``), entonces se evalua como un variables de estado. Si se accede al símbolo externamente, (es decir con ``this.``), entonces se evalua como una función.

::

    pragma solidity ^0.4.0;

    contract C {
        uint public data;
        function x() {
            data = 3; // acceso interno
            uint val = this.data(); // acceso externo
        }
    }

El siguiente ejemplo es un poco más complejo:

::

    pragma solidity ^0.4.0;

    contract Complex {
        struct Data {
            uint a;
            bytes3 b;
            mapping (uint => uint) map;
        }
        mapping (uint => mapping(bool => Data[])) public data;
    }

Nos va a generar una función de la siguiente forma:

::

    function data(uint arg1, bool arg2, uint arg3) returns (uint a, bytes3 b) {
        a = data[arg1][arg2][arg3].a;
        b = data[arg1][arg2][arg3].b;
    }

Notese que se ha omitido el mapeo en el struct porque no hay una buena manera de dar la clave para hacer el mapeo.

.. index:: ! funcion;modifier

.. _modifiers:

*******************
Funciones Modifiers
*******************

Se pueden usar los Modifiers para cambiar el comportamiento de las funciones de una manera ágil. Por ejemplo, los Modifiers son capaces de comprobar automaticamente una condición antes de ejecutar una función. Los Modifiers son propiedades heredables de los contratos y pueden ser sobreescritos por contratos derivados.

::

    pragma solidity ^0.4.11;

    contract owned {
        function owned() { owner = msg.sender; }
        address owner;
        
        // Este contrato solo define un Modifier pero lo usa – se va a utilizar en un contrato derivado.
        // El cuerpo de la función se incerta donde aparece el símbolo especial "_;" en la definición del Modifier.
        // Esto significa que si el propietario llama a esta función, la función se ejecuta, pero en otros casos devolverá un error (???exception).
        modifier onlyOwner {
            require(msg.sender == owner);
            _;
        }
    }


    contract mortal is owned {
        // Este contrato hereda del Modifier "onlyOwner" desde "owned" y lo aplica a la función "close", lo que tiene como efecto que las llamadas a "close" solamente tienen efecto si las hacen el propietario registrado.
        function close() onlyOwner {
            selfdestruct(owner);
        }
    }


    contract priced {
        // Los Modifiers pueden recibir argumentos:
        modifier costs(uint price) {
            if (msg.value >= price) {
                _;
            }
        }
    }


    contract Register is priced, owned {
        mapping (address => bool) registeredAddresses;
        uint price;

        function Register(uint initialPrice) { price = initialPrice; }

        // Aquí es importante facilitar también la palabra clave "payable", de lo contrario la función rechazaría automaticamente todos los Ether que le mandemos. 
        function register() payable costs(price) {
            registeredAddresses[msg.sender] = true;
        }

        function changePrice(uint _price) onlyOwner {
            price = _price;
        }
    }

    contract Mutex {
        bool locked;
        modifier noReentrancy() {
            require(!locked);
            locked = true;
            _;
            locked = false;
        }

        /// Esta función está protegida por un mutex, lo que significa que llamdas reentrantes desde dentro del msg.sender.call no pueden llamar a f de nuevo.
        /// La declaración `return 7` asigna 7 al valor devuelto, pero aún así ejecuta la declaración `locked = false` en el Modifier.
        function f() noReentrancy returns (uint) {
            require(msg.sender.call());
            return 7;
        }
    }

Múltiples Modifiers pueden ser aplicados a una misma función especificándolos en una lista separada por espacios en blanco. Serán evaluados en el orden presentado en la lista.

.. warning::
    En una versión anterior de Solidity, declaraciones del tipo ``return`` dentro de funciones que contienen Modifiers se comportaban de otra manera. 

Lo que se devuelve explicitamente de un Modifier o del cuerpo de una función solo sale del Modifier actual o del cuerpo de la función actual. Las variables que se devuelven están asignadas y el control de flujo continúa después del "_" en el Modifier que precede.

Se aceptan expresiones arbitrarias para los argumentos del Modifier y en ese contexto, todos los símbolos visibles desde la función son visibles en el Modifier. Símbolos introducidos en el Modifier no son visibles en la función (ya que pueden cambiar por sobreescritura).

.. index:: ! constante

******************************
Variables de Estado Constantes
******************************

Las variables de estado pueden declarase como ``constantes``. En este caso, se tienen que asignar desde una expresión que es una constante en momento de compilación. Las expresiones que acceden al almacenamiento, datos sobre la blockchain (p.ej ``now``, ``this.balance`` o ``block.number``), datos sobre la ejecución (``msg.gas``) o que hacen llamadas a contratos externos, están prohibidas. Las expresiones que puedan tener efectos colaterales en el reparto de memoria están permitidas, pero las que puedan tener efectos colaterales en otros objetos de memoria no lo son. Las funciones por defecto ``keccak256``, ``sha256``, ``ripemd160``, ``ecrecover``, ``addmod`` y ``mulmod`` están permitidas (aunque hacen llamadas a contratos externos).

Se permiten efectos colaterales en el repartidor de memoria porque debe ser posible construir objetos complejos como p.ej lookup-tables. Esta funcionalidad todavía no se puede usar tal cual. 

El compilador no guarda un espacio de almacenamiento para estas variables, y se remplaza cada ocurrencia por su respectiva expresión constante (que puede ser compilada como un valor simple por el optimizador).

En este momento, no todos los tipos para las constantes están implementados. Los únicos tipos implementados por ahora son los tipos de valor y las cadenas de texto (string).

::

    pragma solidity ^0.4.0;

    contract C {
        uint constant x = 32**22 + 8;
        string constant text = "abc";
        bytes32 constant myHash = keccak256("abc");
    }


.. _funciones-constantes:

********************
Funciones Constantes
********************

En el caso en que un función se declare como constante, promete no modificar el estado.

::

    pragma solidity ^0.4.0;

    contract C {
        function f(uint a, uint b) constant returns (uint) {
            return a * (b + 42);
        }
    }

.. note::
  Los metodos getter están marcados como constantes. 

.. warning::
  El compilador todavía no impone que un método constante no modifica el estado.

.. index:: ! fallback function, function;fallback

.. _fallback-function:

*****************
Fallback Function
*****************

A contract can have exactly one unnamed function. This function cannot have
arguments and cannot return anything.
It is executed on a call to the contract if none of the other
functions match the given function identifier (or if no data was supplied at
all).

Furthermore, this function is executed whenever the contract receives plain
Ether (without data).  In such a context, there is usually very little gas available to
the function call (to be precise, 2300 gas), so it is important to make fallback functions as cheap as
possible.

In particular, the following operations will consume more gas than the stipend provided to a fallback function:

- Writing to storage
- Creating a contract
- Calling an external function which consumes a large amount of gas
- Sending Ether

Please ensure you test your fallback function thoroughly to ensure the execution cost is less than 2300 gas before deploying a contract.

.. warning::
    Contracts that receive Ether directly (without a function call, i.e. using ``send`` or ``transfer``)
    but do not define a fallback function
    throw an exception, sending back the Ether (this was different
    before Solidity v0.4.0). So if you want your contract to receive Ether,
    you have to implement a fallback function.

::

    pragma solidity ^0.4.0;

    contract Test {
        // This function is called for all messages sent to
        // this contract (there is no other function).
        // Sending Ether to this contract will cause an exception,
        // because the fallback function does not have the "payable"
        // modifier.
        function() { x = 1; }
        uint x;
    }


    // This contract keeps all Ether sent to it with no way
    // to get it back.
    contract Sink {
        function() payable { }
    }


    contract Caller {
        function callTest(Test test) {
            test.call(0xabcdef01); // hash does not exist
            // results in test.x becoming == 1.

            // The following call will fail, reject the
            // Ether and return false:
            test.send(2 ether);
        }
    }

.. index:: ! event

.. _events:

******
Events
******

Events allow the convenient usage of the EVM logging facilities,
which in turn can be used to "call" JavaScript callbacks in the user interface
of a dapp, which listen for these events.

Events are
inheritable members of contracts. When they are called, they cause the
arguments to be stored in the transaction's log - a special data structure
in the blockchain. These logs are associated with the address of
the contract and will be incorporated into the blockchain
and stay there as long as a block is accessible (forever as of
Frontier and Homestead, but this might change with Serenity). Log and
event data is not accessible from within contracts (not even from
the contract that created them).

SPV proofs for logs are possible, so if an external entity supplies
a contract with such a proof, it can check that the log actually
exists inside the blockchain.  But be aware that block headers have to be supplied because
the contract can only see the last 256 block hashes.

Up to three parameters can
receive the attribute ``indexed`` which will cause the respective arguments
to be searched for: It is possible to filter for specific values of
indexed arguments in the user interface.

If arrays (including ``string`` and ``bytes``) are used as indexed arguments, the
Keccak-256 hash of it is stored as topic instead.

The hash of the signature of the event is one of the topics except if you
declared the event with ``anonymous`` specifier. This means that it is
not possible to filter for specific anonymous events by name.

All non-indexed arguments will be stored in the data part of the log.

.. note::
    Indexed arguments will not be stored themselves.  You can only
    search for the values, but it is impossible to retrieve the
    values themselves.

::

    pragma solidity ^0.4.0;

    contract ClientReceipt {
        event Deposit(
            address indexed _from,
            bytes32 indexed _id,
            uint _value
        );

        function deposit(bytes32 _id) payable {
            // Any call to this function (even deeply nested) can
            // be detected from the JavaScript API by filtering
            // for `Deposit` to be called.
            Deposit(msg.sender, _id, msg.value);
        }
    }

The use in the JavaScript API would be as follows:

::

    var abi = /* abi as generated by the compiler */;
    var ClientReceipt = web3.eth.contract(abi);
    var clientReceipt = ClientReceipt.at(0x123 /* address */);

    var event = clientReceipt.Deposit();

    // watch for changes
    event.watch(function(error, result){
        // result will contain various information
        // including the argumets given to the Deposit
        // call.
        if (!error)
            console.log(result);
    });

    // Or pass a callback to start watching immediately
    var event = clientReceipt.Deposit(function(error, result) {
        if (!error)
            console.log(result);
    });

.. index:: ! log

Low-Level Interface to Logs
===========================

It is also possible to access the low-level interface to the logging
mechanism via the functions ``log0``, ``log1``, ``log2``, ``log3`` and ``log4``.
``logi`` takes ``i + 1`` parameter of type ``bytes32``, where the first
argument will be used for the data part of the log and the others
as topics. The event call above can be performed in the same way as

::

    log3(
        msg.value,
        0x50cb9fe53daa9737b786ab3646f04d0150dc50ef4e75f59509d83667ad5adb20,
        msg.sender,
        _id
    );

where the long hexadecimal number is equal to
``keccak256("Deposit(address,hash256,uint256)")``, the signature of the event.

Additional Resources for Understanding Events
==============================================

- `Javascript documentation <https://github.com/ethereum/wiki/wiki/JavaScript-API#contract-events>`_
- `Example usage of events <https://github.com/debris/smart-exchange/blob/master/lib/contracts/SmartExchange.sol>`_
- `How to access them in js <https://github.com/debris/smart-exchange/blob/master/lib/exchange_transactions.js>`_

.. index:: ! inheritance, ! base class, ! contract;base, ! deriving

***********
Inheritance
***********

Solidity supports multiple inheritance by copying code including polymorphism.

All function calls are virtual, which means that the most derived function
is called, except when the contract name is explicitly given.

When a contract inherits from multiple contracts, only a single
contract is created on the blockchain, and the code from all the base contracts
is copied into the created contract.

The general inheritance system is very similar to
`Python's <https://docs.python.org/3/tutorial/classes.html#inheritance>`_,
especially concerning multiple inheritance.

Details are given in the following example.

::

    pragma solidity ^0.4.0;

    contract owned {
        function owned() { owner = msg.sender; }
        address owner;
    }


    // Use "is" to derive from another contract. Derived
    // contracts can access all non-private members including
    // internal functions and state variables. These cannot be
    // accessed externally via `this`, though.
    contract mortal is owned {
        function kill() {
            if (msg.sender == owner) selfdestruct(owner);
        }
    }


    // These abstract contracts are only provided to make the
    // interface known to the compiler. Note the function
    // without body. If a contract does not implement all
    // functions it can only be used as an interface.
    contract Config {
        function lookup(uint id) returns (address adr);
    }


    contract NameReg {
        function register(bytes32 name);
        function unregister();
     }


    // Multiple inheritance is possible. Note that "owned" is
    // also a base class of "mortal", yet there is only a single
    // instance of "owned" (as for virtual inheritance in C++).
    contract named is owned, mortal {
        function named(bytes32 name) {
            Config config = Config(0xd5f9d8d94886e70b06e474c3fb14fd43e2f23970);
            NameReg(config.lookup(1)).register(name);
        }

        // Functions can be overridden by another function with the same name and
        // the same number/types of inputs.  If the overriding function has different
        // types of output parameters, that causes an error.
        // Both local and message-based function calls take these overrides
        // into account.
        function kill() {
            if (msg.sender == owner) {
                Config config = Config(0xd5f9d8d94886e70b06e474c3fb14fd43e2f23970);
                NameReg(config.lookup(1)).unregister();
                // It is still possible to call a specific
                // overridden function.
                mortal.kill();
            }
        }
    }


    // If a constructor takes an argument, it needs to be
    // provided in the header (or modifier-invocation-style at
    // the constructor of the derived contract (see below)).
    contract PriceFeed is owned, mortal, named("GoldFeed") {
       function updateInfo(uint newInfo) {
          if (msg.sender == owner) info = newInfo;
       }

       function get() constant returns(uint r) { return info; }

       uint info;
    }

Note that above, we call ``mortal.kill()`` to "forward" the
destruction request. The way this is done is problematic, as
seen in the following example::

    pragma solidity ^0.4.0;

    contract mortal is owned {
        function kill() {
            if (msg.sender == owner) selfdestruct(owner);
        }
    }


    contract Base1 is mortal {
        function kill() { /* do cleanup 1 */ mortal.kill(); }
    }


    contract Base2 is mortal {
        function kill() { /* do cleanup 2 */ mortal.kill(); }
    }


    contract Final is Base1, Base2 {
    }

A call to ``Final.kill()`` will call ``Base2.kill`` as the most
derived override, but this function will bypass
``Base1.kill``, basically because it does not even know about
``Base1``.  The way around this is to use ``super``::

    pragma solidity ^0.4.0;

    contract mortal is owned {
        function kill() {
            if (msg.sender == owner) selfdestruct(owner);
        }
    }


    contract Base1 is mortal {
        function kill() { /* do cleanup 1 */ super.kill(); }
    }


    contract Base2 is mortal {
        function kill() { /* do cleanup 2 */ super.kill(); }
    }


    contract Final is Base2, Base1 {
    }

If ``Base1`` calls a function of ``super``, it does not simply
call this function on one of its base contracts.  Rather, it 
calls this function on the next base contract in the final
inheritance graph, so it will call ``Base2.kill()`` (note that
the final inheritance sequence is -- starting with the most
derived contract: Final, Base1, Base2, mortal, owned).
The actual function that is called when using super is
not known in the context of the class where it is used,
although its type is known. This is similar for ordinary
virtual method lookup.

.. index:: ! base;constructor

Arguments for Base Constructors
===============================

Derived contracts need to provide all arguments needed for
the base constructors. This can be done in two ways::

    pragma solidity ^0.4.0;

    contract Base {
        uint x;
        function Base(uint _x) { x = _x; }
    }


    contract Derived is Base(7) {
        function Derived(uint _y) Base(_y * _y) {
        }
    }

One way is directly in the inheritance list (``is Base(7)``).  The other is in
the way a modifier would be invoked as part of the header of
the derived constructor (``Base(_y * _y)``). The first way to
do it is more convenient if the constructor argument is a
constant and defines the behaviour of the contract or
describes it. The second way has to be used if the
constructor arguments of the base depend on those of the
derived contract. If, as in this silly example, both places
are used, the modifier-style argument takes precedence.

.. index:: ! inheritance;multiple, ! linearization, ! C3 linearization

Multiple Inheritance and Linearization
======================================

Languages that allow multiple inheritance have to deal with
several problems.  One is the `Diamond Problem <https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem>`_.
Solidity follows the path of Python and uses "`C3 Linearization <https://en.wikipedia.org/wiki/C3_linearization>`_"
to force a specific order in the DAG of base classes. This
results in the desirable property of monotonicity but
disallows some inheritance graphs. Especially, the order in
which the base classes are given in the ``is`` directive is
important. In the following code, Solidity will give the
error "Linearization of inheritance graph impossible".

::

    pragma solidity ^0.4.0;

    contract X {}
    contract A is X {}
    contract C is A, X {}

The reason for this is that ``C`` requests ``X`` to override ``A``
(by specifying ``A, X`` in this order), but ``A`` itself
requests to override ``X``, which is a contradiction that
cannot be resolved.

A simple rule to remember is to specify the base classes in
the order from "most base-like" to "most derived".

Inheriting Different Kinds of Members of the Same Name
======================================================

When the inheritance results in a contract with a function and a modifier of the same name, it is considered as an error.
This error is produced also by an event and a modifier of the same name, and a function and an event of the same name.
As an exception, a state variable getter can override a public function.

.. index:: ! contract;abstract, ! abstract contract

******************
Abstract Contracts
******************

Contract functions can lack an implementation as in the following example (note that the function declaration header is terminated by ``;``)::

    pragma solidity ^0.4.0;

    contract Feline {
        function utterance() returns (bytes32);
    }

Such contracts cannot be compiled (even if they contain implemented functions alongside non-implemented functions), but they can be used as base contracts::

    pragma solidity ^0.4.0;

    contract Cat is Feline {
        function utterance() returns (bytes32) { return "miaow"; }
    }

If a contract inherits from an abstract contract and does not implement all non-implemented functions by overriding, it will itself be abstract.

.. index:: ! contract;interface, ! interface contract

**********
Interfaces
**********

Interfaces are similar to abstract contracts, but they cannot have any functions implemented. There are further restrictions:

#. Cannot inherit other contracts or interfaces.
#. Cannot define constructor.
#. Cannot define variables.
#. Cannot define structs.
#. Cannot define enums.

Some of these restrictions might be lifted in the future.

Interfaces are basically limited to what the Contract ABI can represent, and the conversion between the ABI and
an Interface should be possible without any information loss.

Interfaces are denoted by their own keyword:

::

    interface Token {
        function transfer(address recipient, uint amount);
    }

Contracts can inherit interfaces as they would inherit other contracts.

.. index:: ! library, callcode, delegatecall

.. _libraries:

************
Libraries
************

Libraries are similar to contracts, but their purpose is that they are deployed
only once at a specific address and their code is reused using the ``DELEGATECALL``
(``CALLCODE`` until Homestead)
feature of the EVM. This means that if library functions are called, their code
is executed in the context of the calling contract, i.e. ``this`` points to the
calling contract, and especially the storage from the calling contract can be
accessed. As a library is an isolated piece of source code, it can only access
state variables of the calling contract if they are explicitly supplied (it
would have no way to name them, otherwise).

Libraries can be seen as implicit base contracts of the contracts that use them.
They will not be explicitly visible in the inheritance hierarchy, but calls
to library functions look just like calls to functions of explicit base
contracts (``L.f()`` if ``L`` is the name of the library). Furthermore,
``internal`` functions of libraries are visible in all contracts, just as
if the library were a base contract. Of course, calls to internal functions
use the internal calling convention, which means that all internal types
can be passed and memory types will be passed by reference and not copied.
To realize this in the EVM, code of internal library functions
and all functions called from therein will be pulled into the calling
contract, and a regular ``JUMP`` call will be used instead of a ``DELEGATECALL``.

.. index:: using for, set

The following example illustrates how to use libraries (but
be sure to check out :ref:`using for <using-for>` for a
more advanced example to implement a set).

::

    pragma solidity ^0.4.11;

    library Set {
      // We define a new struct datatype that will be used to
      // hold its data in the calling contract.
      struct Data { mapping(uint => bool) flags; }

      // Note that the first parameter is of type "storage
      // reference" and thus only its storage address and not
      // its contents is passed as part of the call.  This is a
      // special feature of library functions.  It is idiomatic
      // to call the first parameter 'self', if the function can
      // be seen as a method of that object.
      function insert(Data storage self, uint value)
          returns (bool)
      {
          if (self.flags[value])
              return false; // already there
          self.flags[value] = true;
          return true;
      }

      function remove(Data storage self, uint value)
          returns (bool)
      {
          if (!self.flags[value])
              return false; // not there
          self.flags[value] = false;
          return true;
      }

      function contains(Data storage self, uint value)
          returns (bool)
      {
          return self.flags[value];
      }
    }


    contract C {
        Set.Data knownValues;

        function register(uint value) {
            // The library functions can be called without a
            // specific instance of the library, since the
            // "instance" will be the current contract.
            require(Set.insert(knownValues, value));
        }
        // In this contract, we can also directly access knownValues.flags, if we want.
    }

Of course, you do not have to follow this way to use
libraries - they can also be used without defining struct
data types. Functions also work without any storage
reference parameters, and they can have multiple storage reference
parameters and in any position.

The calls to ``Set.contains``, ``Set.insert`` and ``Set.remove``
are all compiled as calls (``DELEGATECALL``) to an external
contract/library. If you use libraries, take care that an
actual external function call is performed.
``msg.sender``, ``msg.value`` and ``this`` will retain their values
in this call, though (prior to Homestead, because of the use of `CALLCODE`, ``msg.sender`` and
``msg.value`` changed, though).

The following example shows how to use memory types and
internal functions in libraries in order to implement
custom types without the overhead of external function calls:

::

    pragma solidity ^0.4.0;

    library BigInt {
        struct bigint {
            uint[] limbs;
        }

        function fromUint(uint x) internal returns (bigint r) {
            r.limbs = new uint[](1);
            r.limbs[0] = x;
        }

        function add(bigint _a, bigint _b) internal returns (bigint r) {
            r.limbs = new uint[](max(_a.limbs.length, _b.limbs.length));
            uint carry = 0;
            for (uint i = 0; i < r.limbs.length; ++i) {
                uint a = limb(_a, i);
                uint b = limb(_b, i);
                r.limbs[i] = a + b + carry;
                if (a + b < a || (a + b == uint(-1) && carry > 0))
                    carry = 1;
                else
                    carry = 0;
            }
            if (carry > 0) {
                // too bad, we have to add a limb
                uint[] memory newLimbs = new uint[](r.limbs.length + 1);
                for (i = 0; i < r.limbs.length; ++i)
                    newLimbs[i] = r.limbs[i];
                newLimbs[i] = carry;
                r.limbs = newLimbs;
            }
        }

        function limb(bigint _a, uint _limb) internal returns (uint) {
            return _limb < _a.limbs.length ? _a.limbs[_limb] : 0;
        }

        function max(uint a, uint b) private returns (uint) {
            return a > b ? a : b;
        }
    }


    contract C {
        using BigInt for BigInt.bigint;

        function f() {
            var x = BigInt.fromUint(7);
            var y = BigInt.fromUint(uint(-1));
            var z = x.add(y);
        }
    }

As the compiler cannot know where the library will be
deployed at, these addresses have to be filled into the
final bytecode by a linker
(see :ref:`commandline-compiler` for how to use the
commandline compiler for linking). If the addresses are not
given as arguments to the compiler, the compiled hex code
will contain placeholders of the form ``__Set______`` (where
``Set`` is the name of the library). The address can be filled
manually by replacing all those 40 symbols by the hex
encoding of the address of the library contract.

Restrictions for libraries in comparison to contracts:

- No state variables
- Cannot inherit nor be inherited
- Cannot receive Ether

(These might be lifted at a later point.)

.. index:: ! using for, library

.. _using-for:

*********
Using For
*********

The directive ``using A for B;`` can be used to attach library
functions (from the library ``A``) to any type (``B``).
These functions will receive the object they are called on
as their first parameter (like the ``self`` variable in
Python).

The effect of ``using A for *;`` is that the functions from
the library ``A`` are attached to any type.

In both situations, all functions, even those where the
type of the first parameter does not match the type of
the object, are attached. The type is checked at the
point the function is called and function overload
resolution is performed.

The ``using A for B;`` directive is active for the current
scope, which is limited to a contract for now but will
be lifted to the global scope later, so that by including
a module, its data types including library functions are
available without having to add further code.

Let us rewrite the set example from the
:ref:`libraries` in this way::

    pragma solidity ^0.4.11;

    // This is the same code as before, just without comments
    library Set {
      struct Data { mapping(uint => bool) flags; }

      function insert(Data storage self, uint value)
          returns (bool)
      {
          if (self.flags[value])
            return false; // already there
          self.flags[value] = true;
          return true;
      }

      function remove(Data storage self, uint value)
          returns (bool)
      {
          if (!self.flags[value])
              return false; // not there
          self.flags[value] = false;
          return true;
      }

      function contains(Data storage self, uint value)
          returns (bool)
      {
          return self.flags[value];
      }
    }


    contract C {
        using Set for Set.Data; // this is the crucial change
        Set.Data knownValues;

        function register(uint value) {
            // Here, all variables of type Set.Data have
            // corresponding member functions.
            // The following function call is identical to
            // Set.insert(knownValues, value)
            require(knownValues.insert(value));
        }
    }

It is also possible to extend elementary types in that way::

    pragma solidity ^0.4.0;

    library Search {
        function indexOf(uint[] storage self, uint value) returns (uint) {
            for (uint i = 0; i < self.length; i++)
                if (self[i] == value) return i;
            return uint(-1);
        }
    }


    contract C {
        using Search for uint[];
        uint[] data;

        function append(uint value) {
            data.push(value);
        }

        function replace(uint _old, uint _new) {
            // This performs the library function call
            uint index = data.indexOf(_old);
            if (index == uint(-1))
                data.push(_new);
            else
                data[index] = _new;
        }
    }

Note that all library calls are actual EVM function calls. This means that
if you pass memory or value types, a copy will be performed, even of the
``self`` variable. The only situation where no copy will be performed
is when storage reference variables are used.
