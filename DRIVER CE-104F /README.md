# DRIVER CE-140F emulator

Firmware para el emulador de disquetera Sharp CE-140F, partiendo del port a
STM32CubeIDE de ffxx68 (rama `CubeIDE`) y llevándolo a funcionar.

Probado en: **PC-E650** con la placa **v1.5 de PoyokomaDanna** (Nucleo L432KC).

**Versión actual: v4.4**

La base v1.1 es el port de ffxx68 tal cual estaba.

```javascript
CE140F emulator v4.4 init
```

## Qué funciona

| Comando | Estado |
| --- | --- |
| `LOAD` | ✅ |
| `SAVE` (binario) | ✅ |
| `SAVE ,A` (ASCII) | ✅ **no funcionaba ni en el firmware original** |
| `FILES` y navegación (↑ ↓ SHIFT+↓) | ✅ |
| Comodines: `FILES"X:C*.BAS"` | ✅ **funcionalidad nueva** |
| `KILL` | ✅ |
| `OPEN` / `CLOSE` / `PRINT#` / `INPUT#` | ✅ |
| `DSKF` | ✅ (`DSKF 1` da los MB libres reales; `DSKF 2` devuelve 65535 fijo) |
| `CHAIN` / `MERGE` | ✅ |
| `EOF` | ✅ **funcionalidad nueva** |
| `NAME` | ✅ **funcionalidad nueva** |
| `SET` (protección de ficheros) | ✅ **funcionalidad nueva** |
| `COPY` | ✅ **funcionalidad nueva** |
| `LFILES` | ✅ abriendo antes el puerto serie (ver abajo) |
| `LOC` y `LOF` | ✅ **funcionalidad nueva** |
| `INIT` | ❌ sin implementar, a propósito |

Nota sobre los comodines: el Sharp expande el patrón antes de enviarlo. Funciona
`C*.BAS`, pero **no** `*.BIN` ni `C*` — esos los rechaza el Sharp, que ni llega a
enviar el comando. El equivalente sí funciona: `????????.BIN`, `C???????.???`.

## Historial de versiones

### v1.2 — Driver de tarjeta SD
`user_diskio.c` era el esqueleto vacío de CubeMX. Añadido `sd_spi.c`: driver
MMC/SDC en SPI1 (PA5/PA6/PA7, CS en PB5), el mismo cableado que la versión Mbed.

### v1.3 — Un solo montaje del sistema de ficheros
Cada comando llamaba a `f_mount()`, que invalidaba los ficheros abiertos, así
que `LOAD`, `SAVE` y `OPEN` no podían funcionar.

### v1.4 — Un solo `main()`
`main.c` y `main.cpp` definían ambos `main` y el de C no llamaba a `app_main()`.
Eliminado.

### v1.5 — Memoria
`.bss` ocupaba 48976 de los 49152 bytes de SRAM1: quedaban 176 bytes de pila y
se desbordaba. Los buffers grandes pasaron a la SRAM2 (~8,6 KB de pila).

### v1.6 — Flanco de BUSY perdido (arregla el `SAVE`)
El Sharp levanta BUSY con el nibble puesto y el flanco se perdía al desmontar el
envío. Se comprueba ahora también el nivel.

### v1.7 — El UART, un solo escritor
`bitReady()` escribía por el puerto serie **dentro de la interrupción** y
`HAL_UART_Transmit` no es reentrante. Toda la salida pasa ahora por el buffer,
que vacía el bucle principal.

### v1.8 — Ficheros de macOS ocultos
Los `._*` y `.DS_Store` aparecían en el listado y descuadraban la numeración.
`FILES` y `FILES_LIST` comparten ahora el mismo filtro.

### v1.9 — Temporización del ACK
Pulso de ACK ensanchado (`BIT_DELAY_1` 1000 → 2000 µs, `BIT_DELAY_2` → 1500),
medido en placa: de 4 fallos de 56 secuencias (7,1%) a 0 de 90.
**No tocar sin medir**: 2600/1500 rompe la navegación del listado.

### v2.0 — `OPEN` con número de fichero 1 (arregla el `SAVE ,A`)
El `SAVE ,A` abre su destino con el número 1, que `inDataBuf[16] - 2` convertía
en 255 en un `uint8_t`; la validación lo rechazaba. Ahora los números 0 y 1 van
al manejador del `SAVE`.

### v2.1 — `CLOSE` con dueño explícito
`CLOSE` no cerraba el fichero del `SAVE ,A` (dejaba 0 bytes), pero cerrarlo
siempre rompía el `SAVE` binario. Ahora `CLOSE` solo cierra lo que abrió un
`OPEN`.

### v2.2 — El byte de EOF sobrante en `LOAD`
Al final se mandaba el `EOF` de `fgetc()` (−1) como dato: un `0xFF` en el cable
que el Sharp leía como línea sin número (*undefined line*). El Mbed hace lo
mismo.

### v2.3 — Comodines en `FILES`
El manual los documenta y el firmware los ignoraba. El Sharp manda el patrón ya
expandido (un `?` por posición), así que basta comparar en formato 8.3. El
espacio es literal, no comodín.

### v2.4 — Handshake del código de dispositivo por eventos
**El cambio más importante.** `bitReady()` subía el ACK a ciegas tras un tiempo
fijo; el manual dice que el ACK va en el **flanco de bajada**, que estaba sin
enganchar. Arregló de golpe `FILES` a secas, `FILES"X:"` y el salto de página.

### v2.5 — Lectura por mayoría y recuperación tras checksum
Cada nibble se lee tres veces y se toma el valor mayoritario bit a bit. Además,
un error de checksum dejaba la secuencia a medias con el Sharp esperando hasta
BREAK; ahora se cierra limpiamente.

### v2.6 — `DSKF` monta la tarjeta
`DSKF 1` devolvía 65535 si era el primer comando tras el reinicio:
`f_getfree()` sin volumen montado. También informa de la geometría por el
puerto serie.

### v2.7 — Buffer de LOAD a 40000 (regresión del issue #6)
El port había dejado `OUT_BUF_SIZE` en 38000; el valor correcto es **40000**
([issue #6](https://github.com/ffxx68/Sharp_ce140f_emul/issues/6)). Con el
valor bajo un fichero grande se trunca en silencio y la máquina se queda
colgada hasta BREAK.

### v2.8 — `LOAD` de ficheros grandes sin límite de tamaño
Subir el buffer no basta con 48 KB de RAM. Ahora el fichero no se acumula
entero: al llenarse el buffer se envía lo que hay y se sigue leyendo. El límite
de tamaño desaparece. Verificado: `PLATAV.BIN` (54.040 bytes) en ~94 s
(~2,44 ms por byte).

### v2.9 — Un `1A` dentro del fichero es fin de fichero
Los ficheros de texto terminan en `1A` (Ctrl-Z) y el `LOAD` ASCII lo mandaba
como carácter más, rompiendo una línea del listado. Ahora corta la lectura y
envía el marcador de fin. Solo afecta al ASCII; en binario (`0x0F`) un `1A` es
un dato más.

### v3.0 a v3.2 — Un fichero que no existe ya no cuelga la máquina
`LOAD "X:NOEXISTE.BAS"` dejaba el Sharp esperando para siempre (igual que el
firmware Mbed). Arreglado también el `0x17`, que respondía un `0xFF` pelado.
En `ProcessCommand()`: si un manejador termina sin poner un byte, se manda
`0xFF` de todas formas — error, nunca máquina bloqueada.

### v3.3 — El byte de error: `0x10`, no `0xFF`
Un `KILL` fallido era indistinguible de uno correcto: contestábamos `0xFF` y el
Sharp lo da por bueno. Medido en la máquina: `0x10` = **I/O ERROR**, `0x80` =
`Error` sin número, `0xFF` = éxito. Constante `ERR_REPLY` = `0x10` en `KILL`,
`SAVE`, `OPEN`, `CLOSE`, `PRINT#`, `INPUT#` y la red de seguridad. No se tocan
la trama de 6 bytes del `LOAD` ni el `0xFF` de fin de listado de `FILES`.

### v3.4 — Montar la tarjeta en todos los comandos
`OPEN ... FOR OUTPUT` daba I/O ERROR como primer comando tras encender, porque
`process_OPEN` no monta el volumen y solo lo hacían `FILES`, `LOAD`, `SAVE` y
`DSKF`. El montaje pasó a `ProcessCommand()`.

### v3.5 — `EOF` implementado
El código `0x1A` caía en «comando desconocido» (`0x00` = no es el final) y un
bucle de lectura normal no terminaba nunca. El byte de respuesta **es el
valor**: basta contestar `0x01` al llegar al final. El fichero viene en
`inDataBuf[1]`.

### v3.6 — `NAME` implementado, y cómo se averigua un comando
`NAME "X:VIEJO" AS "NUEVO"` no tenía manejador. Los comandos no reconocidos
**vuelcan su trama en el log** — método que sirvió para sacar `COPY`, `SET`,
`LOC` y `LOF`.

### v3.7 y v3.8 — `COPY` y `SET`
Dos nombres 8.3 por el cable, igual que `NAME`: origen en el byte 3, destino en
el 17 (`COPY` copia por bloques de 512 bytes). En `SET` el atributo **no viaja
como letra**: `"P"` = `00 01`, `" "` = `FF FE`. La protección la aplica FatFs
con solo lectura: `KILL` y `OPEN ... FOR OUTPUT` de un fichero protegido dan
I/O ERROR.

### v3.9 — `DIRLIST.TXT`, el listado como fichero
Extra de este firmware. `LFILES` imprime por el bus de 11 patillas, así que una
impresora en `COM:` queda fuera, y BASIC no puede meter nombres en variables.
Abrir `X:DIRLIST.TXT` para lectura escribe el listado en la tarjeta (nombre
8.3, tamaño, `P` si protegido) y devuelve un fichero normal. El nombre se
reserva y no aparece en `FILES`.

Después se supo que **`LFILES` sí imprime por el puerto serie** si se abre antes
(`OPEN "COM:"` + `LFILES "X:"`, verificado con impresora Sanei SM1-21). La
redirección ocurre dentro del Sharp, no en el bus. `DIRLIST.TXT` sigue útil
para meter nombres en variables.

### v4.0 a v4.2 — `LOC` y `LOF`
Los dos últimos que quedaban (`LOC` estaba comentado en el firmware original;
el código de `LOF`, el `0x1B`, salió del volcado de tramas). Llegan igual:
comando, número de fichero y checksum. **No son iguales**: `LOC` responde con
valor de **2 bytes** y `LOF` con **3** — al `LOC` se le mandaron 3 y el log
dijo `SO Err2 pos: 4`. `LOC` cuenta registros de 256 (redondeo hacia arriba),
`LOF` devuelve bytes.

Con esto solo queda `INIT` sin implementar, a propósito: formatear no aporta
nada en una tarjeta SD y es el único comando capaz de borrarla entera.

### v4.3 y v4.4 — La `P` de los ficheros protegidos: no se puede
La `P` del listado **no es posible**, medido: la respuesta son 17 bytes fijos
(con uno más la máquina para: `SO Err2 pos: 17`); puesta dentro, ni `FILES` ni
`LFILES` la dibujan; y el Sharp no pide el atributo por otro lado.

La v4.3 la dio por resuelta por un malentendido — la `P` que se veía venía de
`DIRLIST.TXT`, que la escribe este firmware, no del `LFILES` — y la v4.4 lo
revierte. La protección en sí funciona: `KILL` y `OPEN ... FOR OUTPUT` la
respetan. Para ver qué está protegido, el camino es `DIRLIST.TXT`.

## Notas de uso

- **Nombres 8.3 y en mayúsculas.** Uno de 9 letras da ERROR 8 al listar o cargar.
- **Enciende antes el emulador y después el Sharp.** Reduce los ERROR 8 de arranque.
- **No mezcles formatos sobre el mismo nombre**: borra antes de regrabar un
fichero ASCII como binario, o usa nombres distintos.
- Si aparecen errores intermitentes tras un rato largo de uso, mira las pilas del
Sharp: el nivel de las señales queda justo en el umbral y es sensible a eso.

---

## Vídeo de referencia

https://www.youtube.com/watch?v=bGOUfeed6xE