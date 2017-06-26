.. index:: Bugs

.. _known_bugs:

#########################
Lista de errore conocidos
#########################

Abajo encontrarás una lista en formato JSON de algunos de los errores de seguridad
en el compilador Solidity. El archivo en sí está hosteado en el repositorio github
`<https://github.com/ethereum/solidity/blob/develop/docs/bugs.json>`_.
La lista comienza con la versión 0.3.0, y sólo los errores conocidos antes de esa versíon no
están listados.

Hay otro archivo llamado `bugs_by_version.json
<https://github.com/ethereum/solidity/blob/develop/docs/bugs_by_version.json>`_,
que puede usarse para revisar que errores afectan una versión específica del compilador.

Herramientas de verificación de código fuente y otras herramientas para contratos
deben consultar esta lista con los suientes criterios:

 - Es relativamente sospechoso que un contrato se haya compilado con una version
   nightly y no con una release. Esta lista no mantiene un registro de la versiones
   nightly.
 - También es sospechoso cuando un contrato fue compilado con una version que no fue
   la máß reciente en el momento que el contrato due creado. Para contratos
   creados de otros contratos, tienes que seguir la cadena de creación de
   de vuelta a una transacción y revisar la fecha de esa transacción.
 - Es muy sospechoso que un contrato haya sido compilado con un compilador que contiene
   un erro conocido y que el contrato fue creado a un momento donde una versión del
   compilador con un fix ya tiene release.

El archivo JSON es una lista de objetos, una para cada error,
con la siguiete información:

nombre (name)
    Nombre único asignado al error
resumen (summary)
    Pequeño resumen del error
descripción (description)
    Descripción detallada del error
enlace (link)
    URL de un sitio web con más información, opcional
introducido (introduced)
    La primera versión del compilador que contiene ese error, opcional
corregido (fixed)
    La primera versión del compilador que ya no contiene el error
publicado (publish)
    La fecha cuando el error se hizo públicamente conocido, opcional
gravedad (severity)
    Gravedad del error: baja, media, alta. Considera
    la descubribilidad en tests de contratos, probabilidad de ocurrir
    y potencial daño hecho por él.
condiciones (conditions)
    Condiciones que tienen que cumplirse para iniciar el error. Actualmente
    eso es un objeto que puede contener un valor booleano ``optimizer``. que
    significa que el optimizador tiene que ser activado para repetir el error.
    Si ninguna condición es alcanzada, presumir que el error está presente.

.. literalinclude:: bugs.json
   :language: js
