.. index:: ! contract

#########
Contratos
#########

Los contratos en Solidity son similares a las clases en los lenguajes orientados a objeto. Los contratos contienen datos persistentes almacenados en variables de estados y funciones que pueden modificar estas variables. Llamar a una función de un contrato diferente (instancia) realizará una llamada a una función de la EVM (Máquina Virtual de Ethereum) para que cambie el contexto de manera que las variables de estado no estén accesibles.

.. index:: ! contract;creation

***************
Crear contratos
***************

Los contratos pueden crearse "desde fuera" o desde contratos en Solidity. Cuando se crea un contrato, su constructor (una función con el mismo nombre que el contrato) se ejecuta una sola vez.

El constructor es opcional. Se admite un solo constructor, lo que significa que la sobrecarga no está soportada.

Desde ``web3.js``, es decir la API de JavaScript, esto se hace de la siguiente manera::

    // Es necesario especificar alguna fuente, incluido el nombre del contrato para los parametros de abajo
    var source = "contract CONTRACT_NAME { function CONTRACT_NAME(uint a, uint b) {} }";

    // El array json del ABI generado por el compilador
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

.. index:: constructor;arguments

Internamente, los argumentos del constructor son transmitidos después del propio código del contrato, pero no se tiene que preocupar de eso si utiliza ``web3.js``.

Si un contrato quiere crear otros contratos, el creador tiene que conocer el código fuente (y el binario) del contrato a crear. Eso significa que la creación de dependencias cíclicas es imposible.

::

    pragma solidity ^0.4.0;

    contract OwnedToken {
        // TokenCreator es un contrato que está definido más abajo. 
        // No hay problema en referenciarlo, siempre y cuando no esté 
        // siendo utilizado para crear un contrato nuevo.
        TokenCreator creator;
        address owner;
        bytes32 name;

        // Esto es el constructor que registra el creador y el nombre 
        // que se le ha asignado
        function OwnedToken(bytes32 _name) {
            // Se accede a las variables de estado por su nombre
            // y no, por ejemplo, por this.owner. Eso también se aplica 
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
            // se pueden convertir a direcciones de forma implícita.
            if (msg.sender == address(creator))
                name = newName;
        }

        function transfer(address newOwner) {
            // Solo el creador puede transferir el token.
            if (msg.sender != owner) return;
            // También vamos a querer preguntar al creador 
            // si la transferencia ha salido bien. Note que esto
            // tiene como efecto llamar a una función del contrato 
            // que está definida más abajo. Si la llamada no funciona
            // (p.ej si no queda gas), la ejecución se para aquí inmediatamente.
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
            // es el tipo más cercano disponible en el ABI.
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
            // Verifica una condición arbitraria
            address tokenAddress = msg.sender;
            return (keccak256(newOwner) & 0xff) == (bytes20(tokenAddress) & 0xff);
        }
    }

.. index:: ! visibility, external, public, private, internal

.. _visibility-and-getters:

*********************
Visibilidad y getters
*********************

Ya que Solidity sólo conoce dos tipos de llamadas a una función (las internas que no generan una llamada a la EVM (también llamadas "message calls") y las externas que sí generan una llamada a la EVM), hay cuatro tipos de visibilidad para las funciones y las variables de estado.

Una función puede especificarse como ``external``, ``public``, ``internal`` o ``private``. Por defecto una función es ``public``. Para las variables de estado, el tipo ``external`` no es posible y el tipo por defecto es ``internal``.

``external``: Las funciones externas son parte de la interfaz del contrato, lo que significa que pueden llamarse desde otros contratos y vía transacciones. Una función externa ``f`` no puede llamarse internamente (por ejemplo ``f()`` no funciona, pero ``this.f()`` funciona). Las funciones externas son a veces más eficientes cuando reciben grandes arrays de datos.
    
``public``: Las funciones públicas son parte de la interfaz del contrato y pueden llamarse internamente o vía mensajes. Para las variables de estado públicas, se genera una función getter automática (ver más abajo).

``internal``: Estas funciones y variables de estado sólo pueden llamarse internamente (es decir, desde dentro del contrato actual o desde contratos de derivan del mismo), sin poder usarse ``this``.

``private``: Las funciones y variables de estado privadas sólo están visibles para el contrato en el que se han definido y no para contratos de derivan del mismo.

.. note:: Todo lo que está definido dentro de un contrato es visible para todos los observadores externos. Definir algo como ``private`` sólo impide que otros contratos puedan acceder y modificar la información, pero esta información siempre será visible para todo el mundo, incluso fuera de la blockchain.

El especificador de visibilidad se pone después del tipo para las variables de estado y entre la lista de parámetros y la lista de parámetros de retorno para las funciones.

::

    pragma solidity ^0.4.0;

    contract C {
        function f(uint a) private returns (uint b) { return a + 1; }
        function setData(uint a) internal { data = a; }
        uint public data;
    }

En el siguiente ejemplo, ``D``, puede llamar a ``c.getData()`` para recuperar el valor de ``data`` en el almacén de estados, pero no puede llamar a ``f``. El contrato ``E`` deriva de ``C`` y, por lo tanto, puede llamar a ``compute``.

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

.. index:: ! getter;function, ! function;getter

Funciones getter
================

El compilador crea automáticamente funciones getter para todas las variables de estado **públicas**. En el contrato que se muestra abajo, el compilador va a generar una función llamada ``data`` que no lee ningún argumento y devuelve un ``uint``, el valor de la variable de estado ``data``. La inicialización de las variables de estado se puede hacer en el momento de la declaración. 

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

Las funciones getter tienen visibilidad externa. Si se accede al símbolo internamente (es decir sin ``this.``), entonces se evalúa como una variable de estado. Si se accede al símbolo externamente, (es decir con ``this.``), entonces se evalúa como una función.

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

.. index:: ! function;modifier

.. _modifiers:

**************************
Modificadores de funciones
**************************

Se pueden usar los modificadores para cambiar el comportamiento de las funciones de una manera ágil. Por ejemplo, los modificadores son capaces de comprobar automáticamente una condición antes de ejecutar una función. Los modificadores son propiedades heredables de los contratos y pueden ser sobrescritos por contratos derivados.

::

    pragma solidity ^0.4.11;

    contract owned {
        function owned() { owner = msg.sender; }
        address owner;
        
        // Este contrato sólo define un modificador pero no lo usa, se va a utilizar en un contrato derivado.
        // El cuerpo de la función se inserta donde aparece el símbolo especial "_;" en la definición del modificador.
        // Esto significa que si el propietario llama a esta función, la función se ejecuta, pero en otros casos devolverá una excepción.
        modifier onlyOwner {
            require(msg.sender == owner);
            _;
        }
    }


    contract mortal is owned {
        // Este contrato hereda del modificador "onlyOwner" desde "owned" y lo aplica a la función "close", lo que tiene como efecto que las llamadas a "close" solamente tienen efecto si las hace el propietario registrado.
        function close() onlyOwner {
            selfdestruct(owner);
        }
    }


    contract priced {
        // Los modificadores pueden recibir argumentos:
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

        // Aquí es importante facilitar también la palabra clave "payable", de lo contrario la función rechazaría automáticamente todos los ethers que le mandemos. 
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
        /// La declaración `return 7` asigna 7 al valor devuelto, pero aún así ejecuta la declaración `locked = false` en el modificador.
        function f() noReentrancy returns (uint) {
            require(msg.sender.call());
            return 7;
        }
    }

Se pueden aplicar varios modificadores a una misma función especificándolos en una lista separada por espacios en blanco. Serán evaluados en el orden presentado en la lista.

.. warning::
	En una versión anterior de Solidity, declaraciones del tipo ``return`` dentro de funciones que contienen modificadores se comportaban de otra manera. 

	Lo que se devuelve explícitamente de un modificador o del cuerpo de una función solo sale del modificador actual o del cuerpo de la función actual. Las variables que se devuelven están asignadas y el control de flujo continúa después del "_" en el modificador que precede.

	Se aceptan expresiones arbitrarias para los argumentos del modificador y en ese contexto, todos los símbolos visibles desde la función son visibles en el modificador. Símbolos introducidos en el modificador no son visibles en la función (ya que pueden cambiar por sobreescritura).

.. index:: ! constant

******************************
Variables de estado constantes
******************************

Las variables de estado pueden declarase como ``constant``. En este caso, se tienen que asignar desde una expresión que es una constante en tiempo de compilación. Las expresiones que acceden al almacenamiento, datos sobre la blockchain (p.ej ``now``, ``this.balance`` o ``block.number``), datos sobre la ejecución (``msg.gas``) o que hacen llamadas a contratos externos, están prohibidas. Las expresiones que puedan tener efectos colaterales en el reparto de memoria están permitidas, pero las que puedan tener efectos colaterales en otros objetos de memoria no lo están. Las funciones por defecto ``keccak256``, ``sha256``, ``ripemd160``, ``ecrecover``, ``addmod`` y ``mulmod`` están permitidas (aunque hacen llamadas a contratos externos).

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


.. _constant-functions:

********************
Funciones constantes
********************

En el caso en que una función se declare como constante, promete no modificar el estado.

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
  El compilador todavía no impone que un método constante no modifique el estado.

.. index:: ! fallback function, function;fallback

.. _fallback-function:

****************
Función fallback
****************

Un contrato puede tener exactamente una sola función sin nombre. Esta función no puede tener argumentos ni puede devolver nada. Se ejecuta si, al llamar al contrato, ninguna de las otras funciones del contrato se corresponde al identificador de función proporcionado (o si no se hubiera proporcionado ningún dato).

Además, esta función se ejecutará siempre y cuando el contrato sólo reciba Ether (sin datos). En este caso en general hay muy poco gas disponible para una llamada a una función (para ser preciso, 2300 gas), por eso es importante hacer las funciones fallback lo más baratas posible.

En particular, las siguientes operaciones consumirán más gas de lo que se da como estipendio para una función fallback.

- Escribir en storage
- Crear un contrato
- Llamar a una función externa que consume una cantidad de gas significativa
- Mandar Ether

Asegúrese por favor de testear su función fallback meticulosamente antes de desplegar el contrato para asegurarse de que su coste de ejecución es menor de 2300 gas.

.. warning::
    Los contratos que reciben Ether directamente (sin una llamada a una función, p.ej usando ``send`` o ``transfer``) pero que no tienen definida una función fallback, van a lanzar una excepción, devolviendo el Ether (nótese que esto era diferente antes de la versión v0.4.0 de Solidity). Por lo tanto, si desea que su contrato reciba Ether, tiene que implementar una función fallback.

::

    pragma solidity ^0.4.0;

    contract Test {
        // Se llama a esta función para todos los mensajes enviados a este contrato (no hay otra función). Enviar Ether a este contrato lanza una excepción, porque la función fallback no tiene el modificador "payable".
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

.. index:: ! event

.. _events:

*******
Eventos
*******

Los eventos permiten el uso conveniente de la capacidad de registro del EVM, que a su vez puede "llamar" a callbacks de JavaScript en la interfaz de usuario de una dapp que escucha a esos eventos.

Los eventos son miembros heredables de los contratos. Cuando se les llama, hacen que los argumentos se guarden en el registro de transacciones - una estructura de datos especial en la blockchain. Estos registros están asociados con la dirección del contrato y serán incorporados en la blockchain y allí permanecerán siempre que un bloque esté accesible (eso es: para siempre con Frontier y con Homestead, pero puede cambiar con Serenity). Los datos de registros y de eventos no están disponibles desde dentro de los contratos (ni siquiera desde el contrato que los ha creado).

Se pueden hacer pruebas SPV para los registros, de manera que si una entidad externa proporciona un contrato con dicha prueba, se puede comprobar que el registro realmente existe en la blockchain. Dicho esto, tenga en cuenta que las cabeceras de bloque deben proporcionarse porque el contrato sólo lee los últimos 256 hashes de bloque. 

Hasta tres parámetros pueden recibir el atributo ``indexed``, lo que hará que se busque por los respectivos parámetros. En la interfaz de usuario, es posible filtrar por los valores específicos de argumentos indexados.

Si se utilizan arrays como argumentos indexados (incluyendo ``string`` y ``bytes``), en su lugar se guarda su hash Keccak-256 como asunto.

El hash de la firma de un evento es uno de los asuntos, excepto si declaras el evento con el especificador ``anonymous``. Esto significa que no es posible filtrar por eventos anónimos específicos por su nombre.

Todos los argumentos no indexados se guardarán en la parte de datos del registro.

.. note::
    No se guardan los argumentos indexados propiamente dichos. Uno sólo puede buscar por los valores, pero es imposible recuperar los valores en sí.

::

    pragma solidity ^0.4.0;

    contract ClientReceipt {
        event Deposit(
            address indexed _from,
            bytes32 indexed _id,
            uint _value
        );

        function deposit(bytes32 _id) payable {
            // Cualquier llamada a esta función (por muy anidada que sea) puede ser detectada desde la API de JavaScript con un filtro para que se llame a `Deposit`.
            Deposit(msg.sender, _id, msg.value);
        }
    }

Su uso en la API de JavaScript sería como sigue:

::

    var abi = /* abi generado por el compilador */;
    var ClientReceipt = web3.eth.contract(abi);
    var clientReceipt = ClientReceipt.at(0x123 /* dirección */);

    var event = clientReceipt.Deposit();

    // mirar si hay cambios
    event.watch(function(error, result){
        // el resultado contendrá varias informaciones incluyendo los argumentos proporcionados en el momento de la llamada a Deposit.
        if (!error)
            console.log(result);
    });

    // O ejecutar una funcin callback para empezar a escuchar de inmediato
    var event = clientReceipt.Deposit(function(error, result) {
        if (!error)
            console.log(result);
    });

.. index:: ! log

Interfaz a registros de bajo nivel
==================================

También es posible acceder al mecanismo de logging a través de la interfaz de bajo nivel mediante las funciones ``log0``, ``log1``, ``log2``, ``log3`` y ``log4``. ``logi`` toma ``i + 1`` parámetros del tipo ``bytes32``, donde el primer argumento se utiliza para la parte de datos del log y los otros como asuntos. La llamada al evento de arriba puede realizarse de una manera similar a esta:

::

    log3(
        msg.value,
        0x50cb9fe53daa9737b786ab3646f04d0150dc50ef4e75f59509d83667ad5adb20,
        msg.sender,
        _id
    );

donde el numero hexadecimal largo es igual a ``keccak256("Deposit(address,hash256,uint256)")``, la firma del evento.

Recursos adicionales para entender los eventos
==============================================

- `Documentación de Javascript <https://github.com/ethereum/wiki/wiki/JavaScript-API#contract-events>`_
- `Ejemplo de uso de los eventos <https://github.com/debris/smart-exchange/blob/master/lib/contracts/SmartExchange.sol>`_
- `Cómo acceder a eventos con js <https://github.com/debris/smart-exchange/blob/master/lib/exchange_transactions.js>`_

.. index:: ! inheritance, ! base class, ! contract;base, ! deriving

********
Herencia
********

Solidity soporta multiples herencias copiando el código, incluyendo el polimorfismo. 

Todas las llamadas a funciones son virtuales, lo que significa que es la función más derivada la que se llama, excepto cuando el nombre del contrato se menciona explícitamente.

Cuando un contrato hereda de múltiples contratos, un solo contrato está creado en la blockchain, y el código de todos los contratos base está copiado dentro del contrato creado.

El sistema general de herencia es muy similar al de
`Python <https://docs.python.org/3/tutorial/classes.html#inheritance>`_,
especialmente en lo que se refiere a herencias multiples.

En el siguiente ejemplo se dan más detalles.

::

    pragma solidity ^0.4.0;

    contract owned {
        function owned() { owner = msg.sender; }
        address owner;
    }


    // Usar "is" para derivar de otro contrato. Los contratos derivados
    // pueden acceder a todos los miembros no privados, incluidas las
    // funciones internas y variables de estado. A éstas sin embargo
    // no se puede acceder externamente mediante `this`.
    contract mortal is owned {
        function kill() {
            if (msg.sender == owner) selfdestruct(owner);
        }
    }


    // Estos contratos abstractos sólo se proporcionan para que el compilador
    // sepa de la interfaz. Nótese que la función no tiene cuerpo. Si un contrato
    // no implementa todas las funciones, sólo puede usarse como interfaz.
    contract Config {
        function lookup(uint id) returns (address adr);
    }


    contract NameReg {
        function register(bytes32 name);
        function unregister();
     }


    // Las herencias multiples son posibles. Nótese que "owned" también es una clase base
    // de "mortal", aun así hay una sóla instancia de "owned" (igual que para las herencias virtuales en C++).
    contract named is owned, mortal {
        function named(bytes32 name) {
            Config config = Config(0xd5f9d8d94886e70b06e474c3fb14fd43e2f23970);
            NameReg(config.lookup(1)).register(name);
        }

        // Las funciones pueden ser sobreescritas por otras funciones con el mismo nombre
	// y el mismo numero/tipo de entradas. Si la función que sobreescribe tiene distintos
	// tipos de parámetros de salida, esto provocará un error. 
        // Tanto las llamadas a funciones locales como las que están basadas en mensajes
	// tienen en cuenta estas sobreescrituras.
        function kill() {
            if (msg.sender == owner) {
                Config config = Config(0xd5f9d8d94886e70b06e474c3fb14fd43e2f23970);
                NameReg(config.lookup(1)).unregister();
                // Sigue siendo posible llamar a una función específica que ha sido sobreescrita.
                mortal.kill();
            }
        }
    }


    // Si un constructor acepta un argumento, es necesario proporcionarlo en la cabecera
    // (o de forma similar a como se hace con los modificadores, en el constructor
    // del contrato derivado (ver más abajo)).
    contract PriceFeed is owned, mortal, named("GoldFeed") {
       function updateInfo(uint newInfo) {
          if (msg.sender == owner) info = newInfo;
       }

       function get() constant returns(uint r) { return info; }

       uint info;
    }

Nótese que arriba llamamos a ``mortal.kill()`` para "reenviar" la orden de destrucción. Hacerlo de esta forma es problemático, como se puede ver en el siguiente ejemplo.

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

Una llamada a ``Final.kill()`` llamará a ``Base2.kill`` al ser la última sobreescritura, pero esta función obviará ``Base1.kill``, básicamente porque ni siquiera sabe de la existencia de ``Base1``. La forma de solucionar esto es usando ``super``.

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

Es similar a la búsqueda de métodos virtual

Si ``Base1`` llama a una función de ``super``, no simplemente llama a esta función en uno de sus contratos base. En su lugar, llama a esta función en el siguiente contrato base en el último grafo de herencias, por lo tanto llama a ``Base2.kill()`` (nótese que la secuencia final de herencia es -- empezando por el contrato más derivado: Final, Base1, Base2, mortal, owned). La función a la que se llama cuando se usa super no se sabe en el contexto de la clase donde se usa, aunque su tipo es conocido. Es similar a la búsqueda de métodos virtuales. 

.. index:: ! base;constructor

Argumentos para constructores base
==================================

Se requiere que los contratos derivados proporcionen todos los argumentos necesarios para los constructores base. Esto se puede hacer de dos maneras.

::

    pragma solidity ^0.4.0;

    contract Base {
        uint x;
        function Base(uint _x) { x = _x; }
    }


    contract Derived is Base(7) {
        function Derived(uint _y) Base(_y * _y) {
        }
    }

Una es directamente en la lista de herencias (``is Base(7)``). La otra es en la misma línea en que un modificador se invoca como parte de la cabecera de un constructor derivado (``Base(_y * _y)``). La primera manera es más conveniente si el argumento del constructor es una constante y define el comportamiento del contrato o por lo menos lo describe. La segunda manera se tiene que usar si los argumentos del constructor de la base dependen de los argumentos del contrato derivado. Si, como en este ejemplo sencillo, ambos sitios están utilizados, el argumento estilo modificador tiene la prioridad.

.. index:: ! inheritance;multiple, ! linearization, ! C3 linearization

Herencia múltiple y linearización
=================================

Los lenguajes que permiten herencias múltiples tienen que lidiar con varios problemas. Uno es el `Problema del diamante <https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem>`_.
Solidity le sigue la pista a Python y utiliza la "`Linearización C3 <https://en.wikipedia.org/wiki/C3_linearization>`_" para forzar un orden específico en el DAG de las clases base. Esto hace que se consigua la propiedad deseada de monotonicidad pero impide algunos grafos de herencia. El orden en el que las clases base se van dando con la instrucción ``is`` es especialmente importante. En el siguiente código, Solidity dará el error "Linearization of inheritance graph impossible".

::

    pragma solidity ^0.4.0;

    contract X {}
    contract A is X {}
    contract C is A, X {}

El motivo de que se produzca este error es que ``C`` solicita a ``X`` que sobreescriba ``A`` (especificando ``A``, ``X`` en este orden), pero el propio ``A`` solicita sobreescribir ``X``, lo que presenta una contradicción que no puede resolverse.

Una regla simple para recordar es especificar las clases base en el orden desde "la más base" hasta "la más derivada".

Heredar distintos tipos de miembros con el mismo nombre
=======================================================

Cuando la herencia termina en un contrato con una función y un modificador con el mismo nombre, se considera esta herencia un error.
Este error también se produciría en el caso en que un evento y un modificador tuvieran el mismo nombre, así como con una función y un evento con el mismo nombre. 
Como excepción, una variable de estado getter puede sobre escribir una función pública. 

.. index:: ! contract;abstract, ! abstract contract

********************
Contratos abstractos
********************

Las funciones de un contrato pueden carecer de una implementación como pasa en el siguiente ejemplo (nótese que la cabecera de declaración de la función se termina con un ``;``).

::

    pragma solidity ^0.4.0;

    contract Feline {
        function utterance() returns (bytes32);
    }

Estos contratos no pueden compilarse (aunque contengan funciones implementadas junto con funciones no implementadas), pero pueden usarse como contratos base.

::

    pragma solidity ^0.4.0;

    contract Cat is Feline {
        function utterance() returns (bytes32) { return "miaow"; }
    }

Si un contrato hereda de un contrato abstracto y éste no implementa todas las funciones no implementadas con sobrescritura, él mismo será un contrato abstracto.

.. index:: ! contract;interface, ! interface contract

**********
Interfaces
**********

Las interfaces son similares a los contratos abstractos, pero no pueden tener ninguna función implementada. Y hay más restricciones:

#. No pueden heredar otros contratos o interfaces.
#. No pueden definir contructores.
#. No pueden definir variables.
#. No pueden definir structs.
#. No pueden definir enums.

Es posible que en el futuro, algunas de estas restricciones se levanten.

Las interfaces son limitadas a lo que básicamente el contrato ABI puede representar, y la conversion entre el ABI y la interfaz debería hacerse sin perdida de información.

Las interfaces se indican por su propia palabra clave:

::

    interface Token {
        function transfer(address recipient, uint amount);
    }

Los contratos pueden heredar interfaces como lo heredarían otros contratos.

.. index:: ! library, callcode, delegatecall

.. _libraries:

*********
Librerías
*********

Las librerías son similares a los contratos, pero su propósito es que se desplieguen una sola vez a una dirección especifica y su código se pueda reutilizar utilizando la característica ``DELEGATECALL`` (``CALLCODE`` hasta Homestead) de la EVM. Lo que significa que si se llama a las funciones de una librería, su código es ejecutado en el contexto del contrato que llama, es decir, ``this`` apunta al contrato que llama y en especial, se puede acceder al almacenamiento del contrato que llama. Como una librería es un trozo de código fuente aislado, una librería sólo puede acceder a las variables de estado de un contrato emisor si estas variables están específicamente proporcionadas (de lo contrario, no tendría la posibilidad de nombrarlas).

Las librería pueden considerarse como contratos base implícitos del contrato que las usa.
Las librerías no son explícitamente visibles en la jerarquía de herencia, pero las llamadas a las funciones de una librería se parecen completamente a las llamadas a funciones de contratos base explícitos (``L.f()`` si ``L`` es el nombre de la librería). Además, las funciones ``internal`` de las librerías son visibles en todos los contratos, como si la librería fuera un contrato base. Por supuesto las llamadas a funciones internas utilizan las normas de llamadas internas, lo que significa que todos los tipos internos pueden ser enviados y que los tipos de memoria serán enviados mediante referencia y no copiados.
Para realizar esta operación en la EVM, se incluirá en el contrato emisor el código de las funciones internas de la librería y todas las funciones llamadas desde dentro usando el comando habitual ``JUMP`` en lugar del ``DELEGATECALL``.

.. index:: using for, set

El siguiente ejemplo ilustra cómo usar las librerías (pero asegúrese de leer :ref:`using for <using-for>` para tener un ejemplo más avanzado de cómo implementar un set):

::

    pragma solidity ^0.4.11;

    library Set {
      // Definimos un nuevo tipo de datos para un struct que se va a utilizar para
      // conservar sus datos en el contrato que efectúa la llamanda.
      struct Data { mapping(uint => bool) flags; }

      // Nótese que el primer parametro es del tipo "referencia de almacenamiento",
      // por lo tanto solamente su dirección de almacenamiento y no su contenido se
      // envía como parte de la llamada. Esto es una característica especial de las
      // funciones de librerías. Es idiomático llamar al primer parámetro 'self', si
      // la función puede verse como un método de este objeto.
      function insert(Data storage self, uint value)
          returns (bool)
      {
          if (self.flags[value])
              return false; // ya está
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
            // Las funciones de librerías pueden llamarse sin una instancia
	    // específica de la librería, ya que la "instancia" es el contrato actual.
            require(Set.insert(knownValues, value));
        }
        // En este contrato, si se quiere, también se puede acceder directamente a knownValues.flags.
    }

No es por supuesto obligatorio usar las librerías de esta manera -  también pueden usarse sin definir tipos de datos struct. Las funciones también funcionan sin parámetros de referencia de almacenamiento, y pueden tener multiples parámetros de referencia de almacenamiento y en cualquier posición.

Las llamadas a ``Set.contains``, ``Set.insert`` y ``Set.remove`` son compiladas como llamadas (``DELEGATECALL``) en un contrato/librería externa. Si se usan librerías, hay que asegurarse de que de verdad se realiza una llamada a una función externa.
``msg.sender``, ``msg.value`` y ``this`` conservarán sus valores en esta llamada, aunque hasta Homestead, por el uso de `CALLCODE`, ``msg.sender`` y ``msg.value`` cambiaban.

En el siguiente ejemplo se muestra cómo usar tipos de memoria y funciones internas en las librerías para implementar tipos a medida sin la necesidad de usar llamadas a funciones externas:

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
                // ¡Qué mal! Tenemos que añadir un "limb"
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

Puesto que el compilador no puede saber en qué dirección será desplegada la librería, estas direcciones deben ser insertadas en el bytecode final por un *linker* (véase :ref:`commandline-compiler` para saber cómo usar el compilador de lineas de comando para establecer vínculos). Si las direcciones no están facilitadas como argumentos al compilador, el código hex compilado contendrá marcadores de posición de la forma ``__Set______`` (donde ``Set`` es el nombre de la librería). La dirección puede ser facilitada manualmente remplazando cada uno de estos 40 símbolos por la codificación hexadecimal de la dirección del contrato de la librería.

Las restricciones para las librerías con respecto a las restricciones para los contratos son las siguientes:

- No hay variables de estado
- No puede heredar ni ser heredadas
- No pueden recibir Ether

(Puede que estas restricciones se levanten en un futuro.)

.. index:: ! using for, library

.. _using-for:

*********
Using For
*********

La directiva ``using A for B;`` se puede usar para adjuntar funciones de librería (desde la librería ``A``) a cualquier tipo (``B``). Estas funciones recibirán el objeto con el que se les llama como su primer parámetro (igual que con el parámetro ``self`` en Python).

El efecto que tiene esta directiva ``using A for *;`` es que las funciones de la librería ``A`` se adjunten a cualquier tipo.

En ambas situaciones, todas las funciones se adjuntan, incluso las funciones donde el tipo del primer parámetro no coincide con el tipo del objeto. El tipo se comprueba en el punto en que se llama a la función y se resuelven problemas de sobrecarga de la función.

Con el alcance actual, que por ahora está limitado a un contrato, la directiva ``using A for B;`` está activa. Más adelante tendrá un alcance global, lo que hará que cuando se incluya un módulo, sus tipos de datos, incluyendo las funciones de librería, estarán disponibles sin tener que añadir más código.

Volvamos a escribir el ejemplo de Set del apartado :ref:`librerías <libraries>` de la siguiente manera:

::

    pragma solidity ^0.4.11;

    // Es el mismo código que antes pero sin los comentarios
    library Set {
      struct Data { mapping(uint => bool) flags; }

      function insert(Data storage self, uint value)
          returns (bool)
      {
          if (self.flags[value])
            return false; // está
          self.flags[value] = true;
          return true;
      }

      function remove(Data storage self, uint value)
          returns (bool)
      {
          if (!self.flags[value])
              return false; // no está
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
        using Set for Set.Data; // este es el cambio importante
        Set.Data knownValues;

        function register(uint value) {
            // Aquí, cada una de las variables con el tipo Set.Data tiene una función miembro correspondiente.
            // La siguiente llamada es idéntica a Set.insert(knownValues, value)
            require(knownValues.insert(value));
        }
    }

También es posible extender los tipos elementales de la siguiente manera:

::

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
            // Esto es lo que realiza la llamada a la función de la librería
            uint index = data.indexOf(_old);
            if (index == uint(-1))
                data.push(_new);
            else
                data[index] = _new;
        }
    }

Nótese que cualquier llamada a una librería es en realidad una llamada a una función de la EVM. Esto significa que si se envían tipos de memoria o de valor, se va a realizar una copia, incluso de la variable ``self``. La única situación en la que no se va a realizar una copia es cuando se utilizan variables que hacen referencia al almacenamiento.
