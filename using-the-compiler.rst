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
        // Required: Source code language, such as "Solidity", "serpent", "lll", "assembly", etc.
        language: "Solidity",
        // Required
        sources:
        {
        // The keys here are the "global" names of the source files,
        // imports can use other files via remappings (see below).
        "myFile.sol":
        {
          // Optional: keccak256 hash of the source file
          // It is used to verify the retrieved content if imported via URLs.
          "keccak256": "0x123...",
          // Required (unless "content" is used, see below): URL(s) to the source file.
          // URL(s) should be imported in this order and the result checked against the
          // keccak256 hash (if available). If the hash doesn't match or none of the
          // URL(s) result in success, an error should be raised.
          "urls":
          [
            "bzzr://56ab...",
            "ipfs://Qma...",
            "file:///tmp/path/to/file.sol"
          ]
        },
        "mortal":
        {
          // Optional: keccak256 hash of the source file
          "keccak256": "0x234...",
          // Required (unless "urls" is used): literal contents of the source file
          "content": "contract mortal is owned { function kill() { if (msg.sender == owner) selfdestruct(owner); } }"
        }
        },
        // Optional
        settings:
        {
        // Optional: Sorted list of remappings
        remappings: [ ":g/dir" ],
        // Optional: Optimizer settings (enabled defaults to false)
        optimizer: {
          enabled: true,
          runs: 500
        },
        // Metadata settings (optional)
        metadata: {
          // Use only literal content and not URLs (false by default)
          useLiteralContent: true
        },
        // Addresses of the libraries. If not all libraries are given here, it can result in unlinked objects whose output data is different.
        libraries: {
          // The top level key is the the name of the source file where the library is used.
          // If remappings are used, this source file should match the global path after remappings were applied.
          // If this key is an empty string, that refers to a global level.
          "myFile.sol": {
            "MyLib": "0x123123..."
          }
        }
        // The following can be used to select desired outputs.
        // If this field is omitted, then the compiler loads and does type checking, but will not generate any outputs apart from errors.
        // The first level key is the file name and the second is the contract name, where empty contract name refers to the file itself,
        // while the star refers to all of the contracts.
        //
        // The available output types are as follows:
        //   abi - ABI
        //   ast - AST of all source files
        //   legacyAST - legacy AST of all source files
        //   devdoc - Developer documentation (natspec)
        //   userdoc - User documentation (natspec)
        //   metadata - Metadata
        //   ir - New assembly format before desugaring
        //   evm.assembly - New assembly format after desugaring
        //   evm.legacyAssembly - Old-style assembly format in JSON
        //   evm.bytecode.object - Bytecode object
        //   evm.bytecode.opcodes - Opcodes list
        //   evm.bytecode.sourceMap - Source mapping (useful for debugging)
        //   evm.bytecode.linkReferences - Link references (if unlinked object)
        //   evm.deployedBytecode* - Deployed bytecode (has the same options as evm.bytecode)
        //   evm.methodIdentifiers - The list of function hashes
        //   evm.gasEstimates - Function gas estimates
        //   ewasm.wast - eWASM S-expressions format (not supported atm)
        //   ewasm.wasm - eWASM binary format (not supported atm)
        //
        // Note that using a using `evm`, `evm.bytecode`, `ewasm`, etc. will select every
        // target part of that output.
        //
        outputSelection: {
          // Enable the metadata and bytecode outputs of every single contract.
          "*": {
            "*": [ "metadata", "evm.bytecode" ]
          },
          // Enable the abi and opcodes output of MyContract defined in file def.
          "def": {
            "MyContract": [ "abi", "evm.opcodes" ]
          },
          // Enable the source map output of every single contract.
          "*": {
            "*": [ "evm.sourceMap" ]
          },
          // Enable the legacy AST output of every single file.
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
      // Optional: not present if no errors/warnings were encountered
      errors: [
        {
          // Optional: Location within the source file.
          sourceLocation: {
            file: "sourceFile.sol",
            start: 0,
            end: 100
          ],
          // Mandatory: Error type, such as "TypeError", "InternalCompilerError", "Exception", etc
          type: "TypeError",
          // Mandatory: Component where the error originated, such as "general", "ewasm", etc.
          component: "general",
          // Mandatory ("error" or "warning")
          severity: "error",
          // Mandatory
          message: "Invalid keyword"
          // Optional: the message formatted with source location
          formattedMessage: "sourceFile.sol:100: Invalid keyword"
        }
      ],
      // This contains the file-level outputs. In can be limited/filtered by the outputSelection settings.
      sources: {
        "sourceFile.sol": {
          // Identifier (used in source maps)
          id: 1,
          // The AST object
          ast: {},
          // The legacy AST object
          legacyAST: {}
        }
      },
      // This contains the contract-level outputs. It can be limited/filtered by the outputSelection settings.
      contracts: {
        "sourceFile.sol": {
          // If the language used has no contract names, this field should equal to an empty string.
          "ContractName": {
            // The Ethereum Contract ABI. If empty, it is represented as an empty array.
            // See https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI
            abi: [],
            // See the Metadata Output documentation (serialised JSON string)
            metadata: "{...}",
            // User documentation (natspec)
            userdoc: {},
            // Developer documentation (natspec)
            devdoc: {},
            // Intermediate representation (string)
            ir: "",
            // EVM-related outputs
            evm: {
              // Assembly (string)
              assembly: "",
              // Old-style assembly (object)
              legacyAssembly: {},
              // Bytecode and related details.
              bytecode: {
                // The bytecode as a hex string.
                object: "00fe",
                // Opcodes list (string)
                opcodes: "",
                // The source mapping as a string. See the source mapping definition.
                sourceMap: "",
                // If given, this is an unlinked object.
                linkReferences: {
                  "libraryFile.sol": {
                    // Byte offsets into the bytecode. Linking replaces the 20 bytes located there.
                    "Library1": [
                      { start: 0, length: 20 },
                      { start: 200, length: 20 }
                    ]
                  }
                }
              },
              // The same layout as above.
              deployedBytecode: { },
              // The list of function hashes
              methodIdentifiers: {
                "delegate(address)": "5c19a95c"
              },
              // Function gas estimates
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
            // eWASM related outputs
            ewasm: {
              // S-expressions format
              wast: "",
              // Binary format (hex string)
              wasm: ""
            }
          }
        }
      }
    }
