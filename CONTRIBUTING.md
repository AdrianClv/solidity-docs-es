## Contribuciones
Para editar un fichero, primero entra en la pestaña "Pull requests" para comprobar que no está siendo traducido por otra persona.

Para mejor organización, sólo debe traducirse un fichero por Pull request.

### Qué traducir
Este documento pretende ser una versión en español de la documentación original, de modo que lo que se escriba aquí debe ser un reflejo de lo que haya en la versión en inglés. Esto quiere decir que si crees que falta algo por documentar o puedes mejorar la documentación (que no traducción), el proceso que debes seguir es proponerlo primero en la versión en inglés y, una vez aceptado allí, reflejar aquí la versión en español.

Los documentos que se deben traducir son los **documentos .rst**. Dentro de estos, se debe traducir **el texto y los comentarios del código**. Los nombres de los contratos, variables y demás no deben ser traducidos.

Ahora mismo se encuentra en periodo de actualización, puedes ver los cambios a realizar en el siguiente gist: https://gist.github.com/chriseth/5e06fb08433a03d1e4c05224f4287264

### Estilo
Algunas cosas a tener en cuenta cuando se traduzca un documento:
* El código, nombres de funciones y variables no se traducen (los comentarios sí).
* Palabras que pese a estar en inglés sean de uso común en español no se traducen (ej: blockchain, timestamp, etc.)
* Los asteriscos (\*), signos de igual (=), almohadillas (#), guiones (-), etc.  que acompañen a títulos, deberán extender su longitud a la longitud del título. Ejemplo:
~~~
************
Título corto
************

Títular más largo
=================
~~~
* En inglés, existe la norma de empezar la primera letra de cada palabra "importante" de un título en mayúscula. En español esta norma no existe, así que mejor empezar con mayúscula sólo la primera letra (y nombres propios y demás).
* Las rutas de enlaces del siguiente estilo no se traducen:
~~~
-.. _structure-state-variables:
~~~
* En los enlaces a las rutas del apartado anterior, se traduce el nombre del enlace, no la ruta. Ejemplo:
~~~
:ref:`variables de estado <structure-state-variables>`
~~~

### Etiquetas
* work in progress: ya hay alguien trabajando en el fichero.
* needs review: el fichero ya ha sido traducido, necesita al menos una revisión de una persona distinta al traductor para incorporarlo al repositorio final.
* open: el fichero estaba siendo traducido por alguien, pero esa persona está inactiva. Puedes continuar con la traducción a partir de donde lo dejó el anterior.

### Pasos para editar
Elige el fichero que quieras traducir y haz click en el logo del lapicero de arriba a la derecha, en donde pone "Edit this file":
![](http://i.imgur.com/B6jRrsZ.png)

Se abrirá un editor de texto online. Ahí puedes hacer los cambios que consideres necesarios:
![](http://i.imgur.com/RMrIMWW.png)

En la pestaña "Preview changes" puedes ver cómo va quedando el resultado de los cambios realizados:
![](http://i.imgur.com/hUlAlcj.png)

Cuando quieras guardar el fichero (recomiendo hacerlo al poco de empezar para que otros usuarios puedan ver que estás trabajando en el fichero), ve al botón verde de abajo, donde pone "Propose file change":
![](http://i.imgur.com/F24VZal.png)

Esto te llevará a una nueva ventana donde puedes revisar los cambios realizados. Tras comprobar que todo es correcto, dale a "Create pull request" 2 veces (no hace falta cambiar el nombre del pull request):
![](http://i.imgur.com/CZrIxLK.png)

![](http://i.imgur.com/EmDXLd1.png)

Ahora, si vas a la pestaña "Pull requests" puedes ver tu trabajo en curso junto con los de otras personas. Para ver el estado de tu edición, haz click en tu Pull request, y luego en la pestaña "Files changed".
![](http://i.imgur.com/TOymUt4.png)

Si quieres continuar, desde "Files changes" haz click en editar el fichero como en el primer paso. Al acabar, haz click en el botón verde de abajo "Commit changes" como en la siguiente imagen:
![](http://i.imgur.com/mSu1fZK.png)
