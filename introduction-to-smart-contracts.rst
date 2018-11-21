#########################################
Introducción a los Contratos Inteligentes
#########################################

.. _simple-smart-contract:

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

El siguiente contrato va a implementar la forma más sencilla de una criptomoneda. Se pueden generar monedas de la nada, pero sólo la persona que creó el contrato estará habilitada para hacerlo (es trivial implementar un esquema diferente de emisión). Es más, cualquiera puede enviar monedas a otros sin necesidad de registrarse con usuario y contraseña - sólo hace falta un par de claves de Ethereum.


::

    pragma solidity ^0.4.0;

    contract Coin {
        // La palabra clave "public" hace que dichas variables
        // puedan ser leídas desde fuera.
        address public minter;
        mapping (address => uint) public balances;
        
        // Los eventos permiten a los clientes ligeros reaccionar
        // de forma eficiente a los cambios.
        event Sent(address from, address to, uint amount);

        // Este es el constructor cuyo código
        // sólo se ejecutará cuando se cree el contrato.
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

La línea ``address public minter;`` declara una variable de estado de tipo address (dirección) que es públicamente accesible. El tipo ``address`` es un valor de 160 bits que no permite operaciones aritméticas. Es apropiado para almacenar direcciones de contratos o pares de claves pertenecientes a personas externas. La palabra reservada ``public`` genera automáticamente una función que permite el acceso al valor actual de la variable de estado. Sin esta palabra reservada, otros contratos no tienen manera de acceder a la variable.
Esta función sería algo como esto::

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

La línea ``event Sent(address from, address to, uint amount);`` declara un evento que es disparado en la última línea de la ejecución de 
``send``. Las interfaces de ususario (como las de servidor, por supuesto) pueden escuchar esos eventos que están siendo disparados en la blockchain sin mucho coste. Tan pronto son disparados, el listener también recibirá los argumentos ``from``, ``to`` y ``amount``, que hacen más fácil trazar las transacciones. Con el fin de escuchar este evento, se podría usar ::

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

La función especial ``Coin`` es el constructor que se ejecuta durante la creación de un contrato y no puede ser llamada con posterioridad. Almacena permanentemente la dirección de la persona que crea el contrato: ``msg`` (junto con ``tx`` y ``block``) es una variable global mágica que contiene propiedades que permiten el acceso a la blockchain. ``msg.sender`` es siempre la dirección desde donde se origina la llamada a la función actual (externa).

Finalmente, las funciones que realmente habrá en el contrato y que podrán ser llamadas por usuarios y contratos como son ``mint`` y ``send``. Si se llama a ``mint`` desde una cuenta distinta a la del creador del contrato, no ocurrirá nada. Por otro lado, ``send`` puede ser usado por todos (los que ya tienen algunas de estas monedas) para enviar monedas a cualquier otro. Hay que tener en cuenta que si se usa este contrato para enviar monedas a una dirección, no se verá reflejado cuando se busque la dirección en un explorador de la blockchain por el hecho de enviar monedas, y que los balances sólo serán guardados en el almacenamiento de este contrato de moneda. Con el uso de eventos es relativamente sencillo crear un "explorador de la blockchain" que monitorice las transacciones y los balances de la nueva moneda.

.. _blockchain-basics:

*************************
Fundamentos de Blockchain
*************************

Las blockchains son un concepto no muy difícil de entender para desarrolladores. La razón es que la mayoría de las complicaciones (minería, `hashes <https://en.wikipedia.org/wiki/Cryptographic_hash_function>`_, `criptografa de curva elíptica <https://en.wikipedia.org/wiki/Elliptic_curve_cryptography>`_, `redes P2P <https://en.wikipedia.org/wiki/Peer-to-peer>`_, etc.) están justo ahí para proveer un conjunto de funcionalidades y espectativas. Una vez que aceptas estas funcionalidades tal cual vienen dadas, no tienes que preocuparte por la tecnología que lleva inmersa - o, ¿tienes que saber realmente cómo funciona internamente Amazon AWS para poder usarlo?.

.. index:: transaction

Transacciones
=============

Una blockchain es una base de datos transaccional globalmente compartida. Esto quiere decir que todos pueden leer las entradas en la base de datos simplemente participando en la red. Si quieres cambiar algo en la base de datos, tienes que crear una transacción a tal efecto que tiene que ser aceptada por todos los demás.
La palabra transacción implica que el cambio que quieres hacer (asumiendo que quieres cambiar dos valores al mismo tiempo) o se aplica por completo, o no se realiza. Es más, mientras tu transacción es aplicada en la base de datos, ninguna otra transacción puede modificarla. 

Como ejemplo, imagine una tabla que lista los balances de todas las cuentas en una divisa electrónica. Si se solicita una transferencia de una cuenta a otra, la naturaleza transaccional de la base de datos garantiza que la cantidad que es sustraída de una cuenta, es añadida en la otra. Si por la razón que sea, no es posible añadir la cantidad a la cuenta de destino, la cuenta origen tampoco se modifica. 

Yendo más allá, una transacción es siempre firmada criptográficamente por el remitente (creador). Esto la hace más robusta para garantizar el acceso a modificaciones específicas de la base de datos. En el ejemplo de divisas electrónicas, un simple chequeo asegura que sólo la persona que posee las claves de la cuenta puede transferir dinero desde ella.

.. index:: ! block

Bloques
=======

Un obstáculo mayor que sobrepasar es el que, en términos de Bitcoin, se llama ataque de "doble gasto": ¿qué ocurre si dos transacciones existentes en la red quieren borrar una cuenta?, ¿un conflicto?.

La respuesta abstracta a esto es que no tienes de qué preocuparte. El orden de las transacciones se seleccionará por ti, las transacciones se aglutinarán en lo que es llamado "bloque" y entonces serán ejecutadas y distribuídas entre todos los nodos participantes. Si dos transacciones se contradicen, la que concluye en segundo lugar será rechazada y no formará parte del bloque.

Estos bloques forman una secuencia lineal en el tiempo de la que viene la palabra cadena de bloques o "blockchain". Los bloques son añadidos a la cadena en intervalos regulares - para Ethereum esto viene a significar cada 17 segundos.

Como parte del "mecanismo de selección de orden" (que se conoce como minería), tiene que pasar que los bloques sean revertidos de cuando en cuando, pero sólo en el extremo o "tip" de la cadena. Cuantos más bloques se añaden encima, menos probable es. En ese caso, lo que ocurre es que tus transacciones son revertidas e incluso borradas de la blockchain, pero cuanto más esperes, menos probable será.


.. _the-ethereum-virtual-machine:

.. index:: !evm, ! ethereum virtual machine

***************************
Máquina Virtual de Ethereum
***************************

Introducción
============

La máquina virtual de Ethereum (EVM por sus siglas en inglés) es un entorno de ejecución de contratos inteligentes en Ethereum. Va más allá de una configuración tipo sandbox ya que se encuentra totalmente aislada, lo que significa que el código que se ejecuta en la EVM no tiene acceso a la red, ni al sistema de ficheros, ni a ningún otro proceso. Incluso los contratos inteligentes tienen acceso limitado a otros contratos inteligentes.

.. index:: ! account, address, storage, balance

Cuentas
=======

Hay dos tipos de cuentas en Ethereum que comparten el mismo espacio de dirección: **Cuentas externas** que están controladas por un par de claves pública-privada (p-ej.: humanos) y **Cuentas contrato** que están controladas por el código almacenado conjuntamente con la cuenta.

La dirección de una cuenta externa viene determinada por la clave pública mientras que la dirección de la cuenta contrato se define en el momento en que se crea dicho contrato (se deriva de la dirección del creador y del número de transacciones enviadas desde esa dirección, el llamado "nonce").

Independientemente de que la cuenta almacene código, los dos tipos se tratan de forma equitativa por la EVM.

Cada cuenta tiene un almacenamiento persistente clave-valor que mapea palabras de 256 bits a palabras de 256 bits llamado **almacenamiento**.

Además, cada cuenta tiene un **balance** en Ether (en "Wei" para ser exactos) que puede ser modificado enviando transacciones que incluyen Ether.

.. index:: ! transaction

Transacciones
=============

Una transacción es un mensaje que se envía de una cuenta a otra (que debería ser la misma o la especial cuenta-cero, ver más adelante). Puede incluir datos binarios (payload) y Ether.

Si la cuenta destino contiene código, este es ejecutado y el payload se provee como dato de entrada.

Si la cuenta destino es la cuenta-cero (la cuenta con dirección ``0``), la transacción crea un **nuevo contrato**. Como se ha mencionado, la dirección del contrato no es la dirección cero, sino de una dirección derivada del emisor y su número de transacciones enviadas (el "nonce"). Los datos binarios de la transacción que crea el contrato son obtenidos como bytecode por la EVM y ejecutados. La salida de esta ejecución es permanentemente almacenada como el código del contrato. Esto significa que para crear un contrato, no se envía el código actual del contrato, realmente se envía código que nos devuelve ese código final.

.. index:: ! gas, ! gas price

Gas
===

En cuanto se crean, cada transacción se carga con una determinada cantidad de **gas**,
cuyo propósito es limitar la cantidad de trabajo que se necesita para ejecutar la transacción y pagar por esta ejecución. Mientras la EVM ejecuta la transacción, el gas se gasta gradualmente según unas reglas específicas.

El **precio del gas** (gas price) es un valor establecido por el creador de la transacción, quien tiene que pagar el ``gas_price * gas`` desde la cuenta de envío. Si queda algo de gas después de la ejecución, se le reembolsa.

Si se ha gastado todo el gas en un punto (p.ej.: es negativo),
se lanza una excepción de out-of-gas, que revierte todas las modificaciones hechas al estado en el contexto de la ejecución actual.

.. index:: ! storage, ! memory, ! stack

Almacenamiento, Memoria y la Pila
=================================

Cada cuenta tiene un área de memoria persistente que se llama **almacenamiento**.
El almacenamiento es un almacén clave-valor que mapea palabras de 256 bits con palabras de 256 bits.
No es posible enumerar el almacenamiento interno desde un contrato y es comparativamente costoso leer y, más todavía, modificar el almacenamiento. Un contrato no pueder leer ni escribir en otro almacenamiento que no sea el suyo.

La segunda área de memoria se conoce como **memoria**, de la que un contrato obtiene de forma ágil una instancia clara de cada message call (llamada de mensaje). La memoria es lineal y puede ser tratada a nivel de byte, pero las lecturas están limitadas a un ancho de 256 bits, mientras que las escrituras puden ser tanto de 8 bits como de 256 bits de ancho. La memoria se expande por palabras (256 bits), cuando se accede (tanto para leer o escribir) a una palabra de memoria sin modificar previamente (p.ej.: cualquier offset de una palabra). En el momento de expansión, se debe pagar el coste en gas. La memoria es más costosa cuanto más crece (escala cuadráticamente).

La EVM no es una máquina de registro, es una máquina de pila por lo que todas las operaciones se hacen en un área llamada la **pila**. Tiene un espacio máximo de 1024 elementos y contiene palabras de 256 bits. El acceso a la pila está limitado a su cima de la siguiente manera:
Es posible copiar uno de los 16 elementos superiores a la cima de la pila o intercambiar el elemento superior justo después de uno de los 16 elementos superiores.
El resto de operaciones cogen los dos elementos más superiores (o uno, o más, dependiendo de la operación) de la pila y ponen el resultado en ella. Por supuesto, es posible mover elementos de la pila al almacenamiento o a la memoria, pero no es posible acceder simplemente a elementos arbitrarios más profundos dentro de la pila sin, primeramente, borrar los que ya están encima.

.. index:: ! instruction

Conjunto de instrucciones
=========================

El conjunto de instrucciones de la EVM se mantiene mínimo con el objetivo de evitar implementaciones incorrectas que podrían causar problemas de consenso. Todas las instrucciones operan con el tipo de datos básico, palabras de 256 bits.
Las operaciones de aritmética habitual, bit, lógica y de comparación están presentes.
Se permiten tanto los saltos condicionales como los no condicionales. Es más, los contratos pueden acceder a propiedades relevantes del bloque actual como su número y timestamp.

.. index:: ! message call, function;call

Message Calls
=============

Los contratos pueden llamar a otros contratos o enviar Ether a cuentas que no sean de contratos usando message calls. Los Message calls son similares a las transacciones, en el sentido de que tienen un origen, un destino, datos, Ether, gas y datos de retorno. De hecho, cada transacción consiste en un message call de alto nivel que de forma consecutiva puede crear message calls posteriores.

Un contrato puede decidir cuánto de su **gas** restante podría ser enviado con el message call interno y cuánto quiere retener. Si ocurre una excepción de out-of-gas durante la llamada interna (o cualquier otra excepción), se mostrará como un valor de error introducido dentro de la pila. En este caso, sólo se gasta el gas enviado junto con la llamada.
En Solidity, el contrato que hace la llamada causa una excepción manual por defecto en estas situaciones, por lo que esas excepciones ascienden en la pila de llamada. 

Como se ha mencionado, el contrato llamado (que podría ser el mismo que el que hace la llamada) recibirá una instancia de memoria vacía y tendrá acceso a los datos de la llamada - que serán provistos en un área separada que se llama **calldata**.
Después de finalizar su ejecución, puede devolver datos que serán almacenados en una localización en la memoria del que hace la llamada que éste ha reservado previamente.

Las llamadas están **limitadas** a la profundidad de 1024, lo que quiere decir que para operaciones más complejas, se debería preferir bucles sobre llamadas recursivas.

.. index:: delegatecall, callcode, library

Delegatecall / Callcode y librerías
===================================

Existe una variante especial de message call llamada **delegatecall**
que es idéntica a un message call con la excepción de que el código en la dirección destino se ejecuta en el contexto del que hace la llamada y ``msg.sender`` y ``msg.value`` no cambian sus valores.

Esto significa que un contrato puede cargar código dinámicamente desde una dirección diferente en tiempo de ejecución. El almacenamiento, la dirección actual y el balance siguen referenciando al contrato que realiza la llamada, sólo se coge el código desde la dirección llamada.

Esto hace posible implementar la funcionalidad de "librería" en Solidity:
Código de librería reusable que se puede aplicar a un almacenamiento de contrato, por ejemplo, con el fin de implementar una estructura de datos compleja.

.. index:: log

Logs
====

Es posible almacenar datos en una estructura de datos indexada que mapea todo el recorrido hasta el nivel de bloque. Esta funcionalidad llamada **logs** se usa en Solidity para implementar **eventos**.
Los contratos no pueden acceder a los datos del log después de crearse, pero pueden ser accedidos desde fuera de la blockchain de forma eficiente. Como parte de los datos del log se guardan en  `bloom filters <https://en.wikipedia.org/wiki/Bloom_filter>`_, es posible buscar estos datos eficientemente y criptográficamente de manera segura, por lo que los otros miembros de la red que no se han descargado la blockchain entera ("light clients") todavía pueden buscarlos.

.. index:: contract creation

Creación
========

Los contratos pueden incluso crear otros contratos usando un opcode especial (p.ej.: ellos no llaman simplemente a la dirección cero). La única diferencia entre estos **create calls** y los message calls normales es que los datos son ejecutados y el resultado almacenado como código y el llamador / creador recibe la dirección del nuevo contrato en la pila.

.. index:: selfdestruct

Auto-destrucción
================

La única posibilidad de borrar el código de la blockchain es cuando un contrato en esa dirección realiza una operación de ``selfdestruct``. Los Ether restantes almacenados en esa dirección son enviados al destinatario designado y, entonces, se borran el almacenamiento y el código del estado.

.. warning:: Aunque un contrato no contenga una llamada a ``selfdestruct``,
  todavía podría hacer esa operación mediante ``delegatecall`` o ``callcode``.

.. note:: La eliminación de contratos antiguos puede, o no, ser implementada en clientes de Ethereum. Adicionalmente, los nodos de archivo podrían elegir mantener el almacenamiento del contrato y el código de forma indefinida.

.. note:: Actualmente las **cuentas externas** no se pueden borrar del estado.
