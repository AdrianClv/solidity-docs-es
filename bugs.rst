.. index:: Bugs

.. _known_bugs:

##########################
Lista de errores conocidos
##########################

Abajo encontrarás una lista en formato JSON de algunos de los errores de seguridad
en el compilador Solidity. El archivo en sí está alojado en el `repositorio de Github
<https://github.com/ethereum/solidity/blob/develop/docs/bugs.json>`_.
La lista comienza con la versión 0.3.0, y sólo los errores conocidos antes de esa versíon no
están listados.

Hay otro archivo llamado `bugs_by_version.json
<https://github.com/ethereum/solidity/blob/develop/docs/bugs_by_version.json>`_,
que puede usarse para revisar qué errores afectan a una versión específica del compilador.

Herramientas de verificación de código fuente y otras herramientas para contratos
deben consultar esta lista con los siguientes criterios:

 - Es relativamente sospechoso que un contrato se haya compilado con una versión
   nightly y no con una release. Esta lista no mantiene un registro de las versiones
   nightly.
 - También es sospechoso cuando un contrato fue compilado con una versión que no era
   la más reciente en el momento que el contrato fue creado. Para contratos
   creados de otros contratos, tienes que seguir la cadena de creación de
   vuelta a una transacción y revisar la fecha de esa transacción.
 - Es muy sospechoso que un contrato haya sido compilado con un compilador que contiene
   un error conocido, y que se cree cuando una versión del compilador con una corrección
   ya haya sido publicada.

El archivo JSON de errores conocidos es una lista de objetos, uno por cada error,
con la siguiente información:

nombre (name)
    Nombre único asignado al error
resumen (summary)
    Pequeña descripción del error
descripción (description)
    Descripción detallada del error
enlace (link)
    URL de un sitio web con más información, opcional
introducido (introduced)
    La primera versión del compilador que contiene ese error, opcional
corregido (fixed)
    La primera versión del compilador que ya no contiene el error
publicado (publish)
    La fecha en la cual el error se hizo públicamente conocido, opcional
gravedad (severity)
    Gravedad del error: baja, media, alta. Considera la facilidad
    de ser detectado en tests de contratos, probabilidad de ocurrir
    y potencial daño.
condiciones (conditions)
    Condiciones que tienen que cumplirse para iniciar el error. Actualmente
    es un objeto que puede contener un valor booleano ``optimizer``, que
    significa que el optimizador tiene que ser activado para reproducir el error.
    Si no se da ninguna condición, se da por hecho que el error está presente.

.. literalinclude:: bugs.json
   :language: js
