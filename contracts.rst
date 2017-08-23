.. index:: ! contratos

#########
Contratos
#########

Los contratos en Solidity son similares a las clases en los lenguajes orientados a objeto. Los contratos contienen datos persistentes almacenados en variables de estados y funciones que pueden modificar estas variables. Llamar a una función de un contrato diferente (instancia) realizará una llamada a una función del EVM (Máquina Virtual de Ethereum) para que cambie el contexto de manera que las variables de estado no estén accesibles.

.. index:: ! contrato;creacion

***************
Crear Contratos
***************

Contratos pueden crearse "desde fuera" o desde contratos en Solidity. Cuando se crea un contrato, su constructor (una función con el mismo nombre que el contrato) se ejecuta una sola vez.

El constructor es opcional. Se admite un solo constructor, lo que significa que sobrecargar no está soportado.

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
        // TokenCreator es un contrato que está definido más abajo. 
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
            // se pueden implícitamente convertir a direcciones.
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

Ya que Solidity sólo conoce dos tipos de llamadas a una función (las internas que no generan una llamada al EVM (también llamadas "llamadas mensaje") y las externas que si generan una llamada al EVM), hay cuatro tipos de visibilidad para las funciones y las variables de estado.

Una función puede especificarse como ``externa``, ``pública``, ``interna`` o ``privada``. Por defecto una función es ``pública``. Para las variables de estado, el tipo ``externa`` no es posible y el tipo por defecto es ``interna``.

``externa``: Funciones externas son parte de la interfaz del contrato, lo que significa que pueden llamarse desde otros contratos y vía transacciones. Una función externa ``f`` no puede llamarse internamente (por ejemplo ``f()`` no funciona, pero ``this.f()`` funciona). Las funciones externas son a veces más eficientes cuando reciben grandes matrices de datos.
    
``pública``: Funciones públicas son parte de la interfaz del contrato y pueden llamarse internamente o vía mensajes. Para las variables de estado públicas, se genera una función getter automática (ver más abajo).

``interna``: Estas funciones y variables de estado sólo pueden llamarse internamente (es decir desde dentro del contrato actual o desde contratos de derivan del mismo), sin poder usarse ``this``.

``private``: Las funciones y variables de estado privadas sólo están visibles para el contrato en el que se han definido y no para contratos de derivan del mismo.

.. note:: Todo lo que está definido dentro de un contrato es visible para todos los observadores externos. Definir algo como ``privado`` sólo impide que otros contratos puedan acceder y modificar la información, pero esta información siempre será visible para todo el mundo, incluso fuera de la blockchain.

Es especificador de visibilidad se pone después del tipo para las variables de estado y entre la lista de parámetros y la lista de parámetros que devuelven información para las funciones.

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
            uint local = c.f(7); // error: el miembro "f" no es visible
            c.setData(3);
            local = c.getData();
            local = c.compute(3, 5); // error: el miembro "compute" no es visible
        }
    }


    contract E is C {
        function g() {
            C c = new C();
            uint val = compute(3, 5);  // acceso a un miembro interno (desde un contrato derivado a su contrato padre)
        }
    }

.. index:: ! getter;funcion, ! funcion;getter

Funciones Getter
================

El compilador crea automáticamente funciones getter para todas las variables de estado **públicas**. En el contrato que se muestra abajo, el compilador va a generar una función llamada ``data`` que no lee ningún argumento y devuelve un ``unint``, el valor de la variable de estado ``data``. La inicialización de las variables de estado se puede hacer en el momento de la declaración. 

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

Las funciones getter tienen visibilidad externa. Si se accede al símbolo internamente (es decir sin ``this.``), entonces se evalúa como un variables de estado. Si se accede al símbolo externamente, (es decir con ``this.``), entonces se evalúa como una función.

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

Se pueden usar los Modifiers para cambiar el comportamiento de las funciones de una manera ágil. Por ejemplo, los Modifiers son capaces de comprobar automáticamente una condición antes de ejecutar una función. Los Modifiers son propiedades heredables de los contratos y pueden ser sobrescritos por contratos derivados.

::

    pragma solidity ^0.4.11;

    contract owned {
        function owned() { owner = msg.sender; }
        address owner;
        
        // Este contrato solo define un Modifier pero lo usa – se va a utilizar en un contrato derivado.
        // El cuerpo de la función se inserta donde aparece el símbolo especial "_;" en la definición del Modifier.
        // Esto significa que si el propietario llama a esta función, la función se ejecuta, pero en otros casos devolverá una excepción.
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

        // Aquí es importante facilitar también la palabra clave "payable", de lo contrario la función rechazaría automáticamente todos los Ether que le mandemos. 
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

        /// Esta función está protegida por un mutex, lo que significa que llamadas reentrantes desde dentro del msg.sender.call no pueden llamar a f de nuevo.
        /// La declaración `return 7` asigna 7 al valor devuelto, pero aún así ejecuta la declaración `locked = false` en el Modifier.
        function f() noReentrancy returns (uint) {
            require(msg.sender.call());
            return 7;
        }
    }

Múltiples Modifiers pueden ser aplicados a una misma función especificándolos en una lista separada por espacios en blanco. Serán evaluados en el orden presentado en la lista.

.. warning::
    En una versión anterior de Solidity, declaraciones del tipo ``return`` dentro de funciones que contienen Modifiers se comportaban de otra manera. 

Lo que se devuelve explícitamente de un Modifier o del cuerpo de una función solo sale del Modifier actual o del cuerpo de la función actual. Las variables que se devuelven están asignadas y el control de flujo continúa después del "_" en el Modifier que precede.

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
  Los métodos getter están marcados como constantes. 

.. warning::
  El compilador todavía no impone que un método constante no modifica el estado.

.. index:: ! funcion fallback, funcion;fallback

.. _fallback-function:

****************
Función Fallback
****************

Un contrato puede tener exactamente una sola función sin nombre. Esta función no puede tener argumentos ni puede devolver nada. Se ejecuta si, al llamar al contrato, ninguna de las otras funciones del contrato se corresponde al identificador de función proporcionado (o si no se hubiera proporcionado ningún dato).

Además, esta función se ejecutará siempre y cuando el contrato sólo recibe Ether (sin dato). En este caso en general hay muy poco gas disponible para una llamada a una función (para ser preciso, 2300 gas), por eso es importante hacer las funciones fallback las más baratas posible.

En particular, las siguientes operaciones consumirán más gas que  lo que se paga (???stipend) para una función fallback.
In particular, the following operations will consume more gas than the stipend provided to a fallback function:

- Escribir al ???(storage)
- Crear un contrato
- Llamar a una función externa que consume una cantidad de gas significativa
- Mandar Ether

Asegúrese por favor de testear su función fallback meticulosamente antes de desplegar el contrato para asegurarse de que su coste de ejecución es menor de 2300 gas.

.. warning::
Los contratos que reciben Ether directamente (sin una llamada a una función, p.ej usando ``send`` o ``transfer``) pero que no tienen definida una función fallback, van a devolver una excepción (???exception), devolviendo el Ether (nótese que esto era diferente antes de la versión v0.4.0 de Solidity). Por lo tanto, si desea que su contrato reciba Ether, tiene que implementar una función fallback.

::

    pragma solidity ^0.4.0;

    contract Test {
		    // Se llama a esta función para todos los mensajes enviados a este contrato (no hay otra función). Enviar Ether a este contrato va a devolver una excepción, porque la función fallback no tiene el modificador "payable".
        function() { x = 1; }
        uint x;
    }


    // Este contrato guarda todo el Ether que se le envía sin posibilidad de recuperarlo.
    contract Sink {
        function() payable { }
    }


    contract Caller {
        function callTest(Test test) {
            test.call(0xabcdef01); // el hash no existe
            // resulta en que test.x se vuelve == 1.

            // La siguiente llamada falla, devuelve el Ether y devuelve un error:
            test.send(2 ether);
        }
    }

.. index:: ! evento

.. _eventos:

*******
Eventos
*******

Los eventos permiten el uso conveniente de la capacidad de registro del EVM, que a su vez puede "llamar" a callbacks de JavaScript en la interfaz de usuario de una dapp que escucha a esos eventos.

Los eventos son miembros heredables de los contratos. Cuando se les llama, hacen que los argumentos se guarden en el registro de transacciones - una estructura de datos especial en la blockchain. Estos registros están asociados con la dirección del contrato y serán incorporados en la blockchain y allí permanecerán siempre que un bloque esté accesible (eso es: para siempre con Frontier y con Homestead, pero puede cambiar con Serenity). Los datos de registros y de eventos no están disponibles desde dentro de los contratos (ni siquiera desde el contrato que los ha creado).

Se pueden hacer pruebas SPV (???SPV proofs) para los registros, de manera que si una entidad externa proporciona un contrato con dicha prueba, se puede comprobar que el registro realmente existe en la blockchain. Dicho esto, tenga en cuenta que las cabeceras de bloque deben proporcionarse porque el contrato  sólo lee los últimos 256 hashes de bloque. 

Hasta tres parámetros pueden recibir el atributo ``indexed``, lo que hará que se busque por los respectivos parámetros. En la interfaz de usuario, es posible filtrar por los valores específicos de argumentos indexados.

Si se utilizan matrices como argumentos indexados (incluyendo ``string`` y ``bytes``), en cambio se guarda su hash Keccak-256 como un tópico (???topic).

El hash de la firma de un evento es uno de los tópicos, excepto si usted ha declarado el evento con el especificador ``anonymous``. Esto significa que no es posible filtrar por eventos anónimos específicos por su nombre.

Todos los argumentos no indexados se guardarán en la parte de datos del registro.

.. note::
		No se guardan los argumentos indexados propiamente dichos. Uno sólo puede buscar por los valores, pero es imposible recuperar los valores ellos mismos.

::

    pragma solidity ^0.4.0;

    contract ClientReceipt {
        event Deposit(
            address indexed _from,
            bytes32 indexed _id,
            uint _value
        );

        function deposit(bytes32 _id) payable {
            // Cualquier llamada a esta función (por muy anidado que sea) puede ser detectada desde la API de JavaScript con un filtro para que se llame a `Deposit`.
            Deposit(msg.sender, _id, msg.value);
        }
    }

Su uso en la API de JavaScript sería como sigue:

::

    var abi = /* abi tal que ha sido generado por el compilador */;
    var ClientReceipt = web3.eth.contract(abi);
    var clientReceipt = ClientReceipt.at(0x123 /* dirección */);

    var event = clientReceipt.Deposit();

    // mirar si hay cambios
    event.watch(function(error, result){
        // el resultado contendrá varias informaciones incluyendo los argumentos proporcionados en el momento de la llamada a Deposit.
        if (!error)
            console.log(result);
    });

    // O hacer una retro llamada (???callback) para empezar a mirar de inmediato
    var event = clientReceipt.Deposit(function(error, result) {
        if (!error)
            console.log(result);
    });

.. index:: ! registro

Interfaz a registros de bajo nivel
==================================

También es posible acceder al mecanismo de logging a través de la interfaz de bajo nivel mediante las funciones ``log0``, ``log1``, ``log2``, ``log3`` y ``log4``. ``logi`` toma ``i + 1`` parámetros del tipo ``bytes32``, donde el primer argumento se utiliza para la parte de datos del log y los otros como tópicos. La llamada al evento aquí arriba puede realizarse de una manera similar a esta:

::

    log3(
        msg.value,
        0x50cb9fe53daa9737b786ab3646f04d0150dc50ef4e75f59509d83667ad5adb20,
        msg.sender,
        _id
    );

donde el numero hexadecimal largo es igual a ``keccak256("Deposit(address,hash256,uint256)")``, la firma del evento.

Recursos Adicional para Entender los Eventos
============================================

- `Documentación de Javascript <https://github.com/ethereum/wiki/wiki/JavaScript-API#contract-events>`_
- `Ejemplo de uso de los eventos <https://github.com/debris/smart-exchange/blob/master/lib/contracts/SmartExchange.sol>`_
- `Como acceder a eventos con js <https://github.com/debris/smart-exchange/blob/master/lib/exchange_transactions.js>`_

.. index:: ! inheritance, ! base class, ! contract;base, ! deriving

********
Herencia
********

Solidity soporta multiples herencias copiando el código, incluyendo el polimorfismo. 

Todas las llamadas a funciones son virtuales, lo que significa que es la función la más derivada la que se llama, excepto cuando el nombre del contrato es explícitamente mencionado.

Cuando un contrato hereda de múltiples contratos, un solo contrato está creado en la blockchain, y el código de todos los contratos base está copiado dentro del contrato creado.

El sistema general de herencia es muy similar al de `Python <https://docs.python.org/3/tutorial/classes.html#inheritance>`_,
especialmente en lo que se refiere a herencias multiples.

En el siguiente ejemplo se dan más detalles.

::

    pragma solidity ^0.4.0;

    contract owned {
        function owned() { owner = msg.sender; }
        address owner;
    }


		// Usar "is" para derivar de otro contrato. Los contratos derivados pueden acceder a todos los miembros no privados, incluidas las funciones internas y variables de estado. A éstas sin embargo no se puede acceder externamente mediante `this`.
    contract mortal is owned {
        function kill() {
            if (msg.sender == owner) selfdestruct(owner);
        }
    }


		// Estos contratos abstractos solo se proporcionan para que el compilador sepa de la interfaz. Nótese que la función no tiene cuerpo. Si un contrato no implementa todas las funciones, solo puede usarse como interfaz.
    contract Config {
        function lookup(uint id) returns (address adr);
    }


    contract NameReg {
        function register(bytes32 name);
        function unregister();
     }


		// Las herencias multiples son posibles. Nótese que "owned" también es una clase base de "mortal", aun así hay una sóla instancia de "owned" (igual que para las herencias virtuales en C++).
    contract named is owned, mortal {
        function named(bytes32 name) {
            Config config = Config(0xd5f9d8d94886e70b06e474c3fb14fd43e2f23970);
            NameReg(config.lookup(1)).register(name);
        }

        // Las funciones pueden ser invalidadas por otras funciones con el mismo nombre y el mismo numero/tipo de entradas. Si la función que invalida tiene distintos tipos de parámetros de salida, esto provocará un error. 
        // Tanto las llamadas a funciones locales como a las que están basadas en mensaje toman en cuenta estas invalidaciones.
        function kill() {
            if (msg.sender == owner) {
                Config config = Config(0xd5f9d8d94886e70b06e474c3fb14fd43e2f23970);
                NameReg(config.lookup(1)).unregister();
                // Sigue siendo posible llamar a una función específica que ha sido invalidadas.
                mortal.kill();
            }
        }
    }


    // Si un constructor acepta un argumento, es necesario proporcionarlo en la cabecera (o ???modifier-invocation-style al constructor del contrato derivado (ver más abajo)).
    contract PriceFeed is owned, mortal, named("GoldFeed") {
       function updateInfo(uint newInfo) {
          if (msg.sender == owner) info = newInfo;
       }

       function get() constant returns(uint r) { return info; }

       uint info;
    }

Nótese que arriba, llamamos a ``mortal.kill()`` para "reenviar" la orden de destrucción. Hacerlo de esta forma es problemático, como se puede ver en el siguiente ejemplo.

::

    pragma solidity ^0.4.0;

    contract mortal is owned {
        function kill() {
            if (msg.sender == owner) selfdestruct(owner);
        }
    }


    contract Base1 is mortal {
        function kill() { /* hacer limpieza 1 */ mortal.kill(); }
    }


    contract Base2 is mortal {
        function kill() { /* hacer limpieza 2 */ mortal.kill(); }
    }


    contract Final is Base1, Base2 {
    }

Una llamada a ``Final.kill()`` llamará a ``Base2.kill`` como la invalidación la más derivada, pero esta función obviará ``Base1.kill``, básicamente porque no siquiera sabe de la existencia de ``Base1``. La forma de solucionar esto es usando ``super``.

::

    pragma solidity ^0.4.0;

    contract mortal is owned {
        function kill() {
            if (msg.sender == owner) selfdestruct(owner);
        }
    }


    contract Base1 is mortal {
        function kill() { /* hacer limpieza 1 */ super.kill(); }
    }


    contract Base2 is mortal {
        function kill() { /* hacer limpieza 2 */ super.kill(); }
    }


    contract Final is Base2, Base1 {
    }

Si ``Base1`` llama a una función de ``super``, no simplemente llama a esta función en uno de sus contratos base. En cambio, llama a esta función en el siguiente contrato base en el ultimo gráfico de herencias, por lo tanto llama a ``Base2.kill()`` (nótese que la secuencia final de herencia es -- empezando por el contrato el más derivado: Final, Base1, Base2, mortal, owned). La función real a la que se llama cuando se usa super no se sabe en el contexto de la clase donde se usa, aunque su tipo es conocido. Esto es similar para métodos habituales de búsqueda virtual. 

.. index:: ! base;constructor

Argumentos para Constructores Base
==================================

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
