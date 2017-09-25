******************
Uso del Compilador
******************

.. index:: ! commandline compiler, compiler;commandline, ! solc, ! linker

.. _commandline-compiler:

Utilizar el Compilador de Línea de Comandos
******************************

Uno de los objetivos de compilación del repositorio de Solidity es ``solc``, el compilador de línea de comandos de solidity.

Utilizando ``solc --help`` proporciona una explicación de todas las opciones. El compilador puede producir varias salidas, desde binarios simples y ensamblaje sobre un árbol de sintaxis abstracto (árbol de análisis) hasta estimaciones de uso de gas.
Si sólo deseas compilar un único archivo, lo ejecutas como ``solc --bin sourceFile.sol`` y se imprimirá el binario. Antes de implementar tu contrato, activa el optimizador mientras compilas usando ``solc --optimize --bin sourceFile.sol``. Si deseas obtener algunas de las variantes de salida más avanzadas de ``solc``, probablemente sea mejor decirle que salga todo por ficheros separados usando``solc -o outputDirectory --bin --ast --asm sourceFile.sol``.

El compilador de línea de comandos leerá automáticamente los archivos importados del sistema de archivos, aunque es posible proporcionar un redireccionamiento de ruta utilizando ``prefix=path`` de la siguiente manera:

::

    solc github.com/ethereum/dapp-bin/=/usr/local/lib/dapp-bin/ =/usr/local/lib/fallback file.sol

Esencialmente esto instruye al compilador a buscar cualquier cosa que empiece con
``github.com/ethereum/dapp-bin/`` bajo ``/usr/local/lib/dapp-bin`` y si no localiza el fichero ahí, mirará en ``/usr/local/lib/fallback`` (el prefijo vacío siempre coincide). ``solc`` no leerá ficheros del sistema de ficheros que se encuentren fuera de los objetivos de reasignación y fuera de los directorios donde se especifica explícitamente la fuente de ficheros donde residen, con lo cual cosas como 

solo funcionan si le añades ``import "/etc/passwd";``, así que sólo funcionan si se añades ``=/`` como una reasignacion.

Si hay coincidencias múltiples debido a reasignaciones, se selecciona el prefijo común más largo.

Por razones de seguridad, el compilador tiene restricciones a qué directorios puede acceder. Las rutas de acceso (y sus subdirectorios) de los archivos de origen especificados en la línea de comandos y las rutas definidas por las reasignaciones se permiten para las instrucciones de importación, pero todo lo demás se rechaza. Se pueden permitir rutas adicionales (y sus subdirectorios) a través del cambio ``--allow-paths /sample/path,/another/sample/path``.

Si tus contratos usan :ref:`libraries <libraries>`, notaras que el bytecode contiene subcadenas del formulario ``__LibraryName______``. Puedes utilizar ``solc`` como un enlazador, lo que significa que insertará las direcciones de la biblioteca en esos puntos:

Agregue ``--libraries "Math:0x12345678901234567890 Heap:0xabcdef0123456"`` a su comando para proporcionar una dirección para cada biblioteca o almacenar la cadena en un archivo (una biblioteca por línea) y ejecutar `` solc`` usando `` --libraries fileName``.

Si ``solc`` se llama con la opción ``--link``, todos los archivos de entrada se interpretan como binarios desvinculados (codificados en hexadecimal) en el ``__LibraryName____``-format dado anteriormente y están enlazados in situ (si la entrada se lee desde stdin, se escribe en stdout). Todas las opciones excepto ``--libraries`` son ignorados (incluyendo ``-o``) en este caso.

Si ``solc`` se llama con la opción ``--standard-json``, esperará una entrada JSON (como se explica a continuación) en la entrada estándar, y devolverá una salida JSON a la salida estándar.

.. _compiler-api:

Compilador de entrada y salida JSON Descripción
******************************************

Estos formatos JSON son utilizados por la API del compilador y están disponibles a través de ``solc``. Estos están sujetos a cambios, algunos campos son opcionales (como se ha señalado), pero está dirigido a hacer sólo cambios compatibles hacia atrás.

La API del compilador espera una entrada con formato JSON y genera el resultado de la compilación en una salida con formato JSON.

Por supuesto, los comentarios no se permiten y se utilizan aquí sólo con fines explicativos.

Descripción de entrada
-----------------

.. code-block:: none

    {
        // Requerido: Lenguaje del código fuente, tal como "Solidity", "serpent", "lll", "assembly", etc.
        language: "Solidity",
        // Requerido
        sources:
        {
        // Las teclas aquí son los nombres "globales" de los ficheros fuente,
        // las importaciones pueden utilizar otros ficheros mediante remappings (vér más abajo).
        "myFile.sol":
        {
          // Opcional: keccak256 hash del fichero fuente
          // Se utiliza para verificar el contenido recuperado si se importa a través de URLs.
          "keccak256": "0x123...",
          // Requerido (a menos que se use "contenido", ver abajo): URL (s) al fichero fuente.
          // URL(s) deben ser importadas en este orden y el resultado debe ser verificado contra el fichero
          // keccak256 hash (si está disponible). Si el hash no coincide con ninguno de los
          // URL(s) resultado en el éxito, un error debe ser elevado.
          "urls":
          [
            "bzzr://56ab...",
            "ipfs://Qma...",
            "file:///tmp/path/to/file.sol"
          ]
        },
        "mortal":
        {
          // Opcional: keccak256 hash del fichero fuente
          "keccak256": "0x234...",
          // Requerido (a menos que se use "urls"): contenido literal del fichero fuente
          "content": "contract mortal is owned { function kill() { if (msg.sender == owner) selfdestruct(owner); } }"
        }
        },
        // Opcional
        settings:
        {
        // Opcional: Lista ordenada de remappings
        remappings: [ ":g/dir" ],
        // Opcional: Ajustes de optimización (activación de valores predeterminados a false)
        optimizador: {
          enabled: true,
          runs: 500
        },
        // Configuración de metadatos (opcional)
        metadata: {
          // Usar sólo contenido literal y no URLs (falso por defecto)
          useLiteralContent: true
        },
        // Direcciones de las bibliotecas. Si no todas las bibliotecas se dan aquí, puede resultar con objetos no vinculados cuyos datos de salida son diferentes.
        libraries: {
          // La clave superior es el nombre del fichero fuente donde se utiliza la biblioteca.
          // Si se utiliza remappings, este fichero fuente debe coincidir con la ruta global después de que se hayan aplicado los remappings.
          // Si esta clave es una cadena vacía, se refiere a un nivel global.

          "myFile.sol": {
            "MyLib": "0x123123..."
          }
        }
        // Para seleccionar las salidas deseadas se puede utilizar lo siguiente.
        // Si este campo se omite, el compilador se carga y comprueba el tipo, pero no genera ninguna salida aparte de errores.
        // La clave de primer nivel es el nombre del fichero y la segunda es el nombre del contrato, donde el nombre vacío del contrato se refiere al fichero mismo,
        // mientras que la estrella se refiere a todos los contratos.
        //
        // Las clases de mensajes disponibles son las siguientes:
        //   abi - ABI
        //   ast - AST de todos los ficheros fuente
        //   legacyAST - legado AST de todos los ficheros fuente
        //   devdoc - Documentación para desarrolladores (natspec)
        //   userdoc - Documentación de usuario (natspec)
        //   metadata - Metadatos
        //   ir - Nuevo formato de ensamblaje antes del desazucarado
        //   evm.assembly - Nuevo formato de ensamblaje después del desazucarado
        //   evm.legacyAssembly - Formato de ensamblaje antiguo en JSON
        //   evm.bytecode.object - Objeto bytecode
        //   evm.bytecode.opcodes - Lista de Opcodes
        //   evm.bytecode.sourceMap - Asignación de fuentes (útil para depuración)
        //   evm.bytecode.linkReferences - Referencias de enlace (si es objeto no enlazado)
        //   evm.deployedBytecode* - Desplegado bytecode (tiene las mismas opciones que evm.bytecode)
        //   evm.methodIdentifiers - La lista de funciones de hashes 
        //   evm.gasEstimates - Funcion de estimación de gas
        //   ewasm.wast - eWASM S-formato de expresiones (no compatible con atm)
        //   ewasm.wasm - eWASM formato binario (no compatible con atm)
        //
        // Ten en cuenta que el uso de `evm`, `evm.bytecode`, `ewasm`, etc. seleccionara cada
        // parte objetiva de esa salida.
        //
        outputSelection: {
          // Habilita los metadatos y las salidas de bytecode de cada contrato.
          "*": {
            "*": [ "metadata", "evm.bytecode" ]
          },
          // Habilitar la salida abi y opcodes de MyContract definida en el fichero def.
          "def": {
            "MyContract": [ "abi", "evm.opcodes" ]
          },
          // Habilita la salida del mapa de fuentes de cada contrato individual.
          "*": {
            "*": [ "evm.sourceMap" ]
          },
          // Habilita la salida AST heredada de cada archivo.
          "*": {
            "": [ "legacyAST" ]
          }
        }
      }
    }

Output Description
------------------

.. code-block:: none

    {
        // Opcional: no está presente si no se han encontrado errores/avisos
        errors: [
        {
          // Opcional: Ubicación dentro del fichero fuente.
          sourceLocation: {
            file: "sourceFile.sol",
            start: 0,
            end: 100
          ],
          // Obligatorio: Tipo de error, como "TypeError", "InternalCompilerError", "Exception", etc
          type: "TypeError",
          // Obligatorio: Componente donde se originó el error, como "general", "ewasm", etc.
          component: "general",
          // Obligatorio ("error" o "warning")
          severity: "error",
          // Obligatorio
          message: "Invalid keyword"
          // Opcional: el mensaje formateado con la ubicación de origen
          formattedMessage: "sourceFile.sol:100: Invalid keyword"
        }
        ],
        // Contiene las salidas a nivel de fichero. Puede ser limitado/filtrado por los ajustes de outputSelection.
        sources: {
        "sourceFile.sol": {
          // Identificador (utilizado en los mapas fuente)
          id: 1,
          // El objeto AST
          ast: {},
          // El objeto legado AST 
          legacyAST: {}
        }
        },
        // Contiene las salidas contract-level. Puede ser limitado/filtrado por los ajustes de outputSelection.
        contracts: {
        "sourceFile.sol": {
          // Si el idioma utilizado no tiene nombres de contrato, este campo debe ser igual a una cadena vacía.
          "ContractName": {
            // El Contrado de Ethereum ABI. Si está vacío, se representa como una matriz vacía.
            // Ver https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI
            abi: [],
            // Ver la documentación de salida de metadatos (cadena JSON seriada)
            metadata: "{...}",
            // Documentación de usuario (natspec)
            userdoc: {},
            // Documentación para desarrolladores (natspec)
            devdoc: {},
            // Representación intermedia (cadena)
            ir: "",
            // EVM-salidas relacionadas 
            evm: {
              // Montaje (cadena)
              assembly: "",
              // Antiguo estilo ensamblaje (objeto)
              legacyAssembly: {},
              // Bytecode y detalles relacionados.
              bytecode: {
                // El bytecode como una cadena hexadecimal.
                object: "00fe",
                // Lista de Opcodes (cadena)
                opcodes: "",
                // El mapeo de fuentes como una cadena. Ve la definición del mapeo de fuentes.
                sourceMap: "",
                // Si se da, este es un objeto no ligado.
                linkReferences: {
                  "libraryFile.sol": {
                    // Traslados de bytes en el bytecode. El enlace sustituye a los 20 bytes que se encuentran allí.
                    "Library1": [
                      { start: 0, length: 20 },
                      { start: 200, length: 20 }
                    ]
                  }
                }
              },
              // La misma disposición que la anterior.
              deployedBytecode: { },
              // La lista de hashes de función
              methodIdentifiers: {
                "delegate(address)": "5c19a95c"
              },
              // Funcion de estimados de gas
              gasEstimates: {
                creation: {
                  codeDepositCost: "420000",
                  executionCost: "infinite",
                  totalCost: "infinite"
                },
                external: {
                  "delegate(address)": "25000"
                },
                internal: {
                  "heavyLifting()": "infinite"
                }
              }
            },
            // eWASM resultados relacionados
            ewasm: {
              // S-formato de expressiones
              wast: "",
              // Formato Binario (cadena hexagonal)
              wasm: ""
            }
          }
        }
      }
    }
