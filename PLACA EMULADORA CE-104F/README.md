# Placa emuladora CE-104F - SHARP PC-E650


Se ha desarrollado un módulo de hardware que emula la unidad de disquete Sharp CE-140F, utilizando una tarjeta SD y un microcontrolador STM32.

En los años 80 y principios de los 90, uno de los mayores inconvenientes de los Sharp Pocket PC era la falta de una forma cómoda de guardar y cargar programas. Ingresar listados en BASIC a mano era una tarea frustrante, y aunque las cintas de casete eran el medio más común y económico, el uso de disquetes representaba una mejora importante.

La unidad Sharp CE-140F era una extensión rara y costosa compatible con varios modelos de Sharp (como los PC-1403, PC-E500, PC-E650, entre otros).

La solución propuesta emula esta unidad física, permitiendo a los Sharp Pocket PC gestionar archivos de manera completa a través de una tarjeta SD, haciendo posible el intercambio de datos con una PC moderna.


<p align="center">
<img src="https://github.com/user-attachments/assets/f8fdfb40-48c0-4324-acc9-169b6ee33f65" width="400">
</p>
<p align="center">


La url del proyecto es:
https://github.com/ffxx68/Sharp_ce140f_emul

Se necesita conectar el STM32 a la placa de Pokoyama Danna para su funcionamiento:
https://booth.pm/ja/items/4941857

El driver original tiene una pequeña limitación con los programas de más de 20K de tamaño, los cuales no pueden ser grabados desde el SD al PC-E650. He cambiado el código y recompilado para que admita programas de, por lo menos, 40K, modificando las opciones del archivo commands.h.

// communication data depth (max file size during LOAD)   
if defined TARGET_NUCLEO_L432KC.    
define OUT_BUF_SIZE 40000    
define IN_BUF_SIZE 2000

## UPDATE 2026

He creado un **nuevo driver** para la placa emuladora CE-140F, puedes 
descargarlo en la carpeta **DRIVER CE-104F** : arregla lo que
no funcionaba del original y además añade comandos que nunca tuvo.

Parte del port a STM32CubeIDE de ffxx68, que compilaba pero no llegaba a
funcionar, y lo lleva hasta el final.

**Lo que estaba roto y ya no:**

- 📂 `FILES` fallaba al bajar por el listado
- ⚡ `LOADM` no podía con ficheros grandes — ahora entran 54 KB sin límite

**Lo que no existía en ninguna versión anterior:**

- `SAVE ,A` — guardar en ASCII
- Comodines en `FILES` (`FILES "X:C*.BAS"`)
- `EOF`, `NAME`, `COPY`, `SET`, `LOC` y `LOF`

**Resultado:** funcionan **18 de los 19 comandos** del manual del CE-140F. El
único que falta es `INIT`, dejado fuera a propósito: en una tarjeta SD
formatear no aporta nada.

Probado en una PC-E650 con la placa v1.5 de PoyokomaDanna.