# Placa emuladora CE-104F para Sharp PC-E650 y compatibles


Se ha desarrollado un módulo de hardware que emula la unidad de disquete Sharp CE-140F, utilizando una tarjeta SD y un microcontrolador STM32.

En los años 80 y principios de los 90, uno de los mayores inconvenientes de los Sharp Pocket PC era la falta de una forma cómoda de guardar y cargar programas. Ingresar listados en BASIC a mano era una tarea frustrante, y aunque las cintas de casete eran el medio más común y económico, el uso de disquetes representaba una mejora importante.

La unidad Sharp CE-140F era una extensión rara y costosa compatible con varios modelos de Sharp (como los PC-1403, PC-E500, PC-E650, entre otros).

La solución propuesta emula esta unidad física, permitiendo a los Sharp Pocket PC gestionar archivos de manera completa a través de una tarjeta SD, haciendo posible el intercambio de datos con una PC moderna.





La url del proyecto es:
https://github.com/ffxx68/Sharp_ce140f_emul

Se necesita conectar el STM32 a la placa de Pokoyama Danna para su funcionamiento:
https://booth.pm/ja/items/4941857

El driver original tiene una pequeña limitación con los programas de más de 20K de tamaño, los cuales no pueden ser grabados desde el SD al PC-E650. He cambiado el código y recompilado para que admita programas de, por lo menos, 40K, modificando las opciones del archivo commands.h.

// communication data depth (max file size during LOAD)
if defined TARGET_NUCLEO_L432KC
define OUT_BUF_SIZE 40000
define IN_BUF_SIZE 2000

