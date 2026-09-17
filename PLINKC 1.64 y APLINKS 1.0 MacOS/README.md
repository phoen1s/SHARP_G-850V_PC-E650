# PLINKC 1.64 y APLINKS 1.0 para macOS

Dos programas, uno a cada lado del cable serie.

| | dónde corre | qué es | versión |
|---|---|---|---|
| **PLINKC** | SHARP PC-E500 / E650 | driver de dispositivo: da la unidad `L:` | **1.64** — update by PHOENIX, 2026 |
| **APLINKS** | Mac | servidor: sirve una carpeta como esa unidad | **1.0 para macOS** — (c) 2026 PHOENIX |

Los dos son trabajo derivado. El protocolo, el disco virtual y el driver
original no son míos: ver [Créditos](#créditos).


---

# PLINKC ver 1.64

## Versiones

| versión | autor | año | qué aporta |
|---|---|---|---|
| PLINK.SYS 1.04 | N.Kon | 1990-94 | el driver original. Tres buffers fijos de 129 B, sin lista enlazada |
| PLINKC 1.60-1.62 | D. Mizobata | 1996-99 | caché de 8 sectores y modo 512 KB. Duplica la velocidad |
| **PLINKC 1.63** | PHOENIX | 2026 | `ORG` fijo en `&AD000`: sale del slot `S1:` |
| **PLINKC 1.64** | PHOENIX | 2026 | **+ volcado automático del buffer de escritura** |

Probado en una PC-E650 el 31 de agosto de 2026.

## Cambio 1 — el driver ya no vive en `S1:`

**El problema.** Con PLINKC instalado, `INIT "E:10K"` reseteaba la máquina y se
perdía todo. Al revés —crear `E:` primero e instalar después— funcionaba.

**La causa.** No era un fallo: era el diseño. El PLINKC original se carga en
`&BF000`, pide un bloque de memoria en el slot `S1:` y **se reubica dentro**
usando su tabla de relocalización. Pero `E:` y `F:` son bloques `RAMFILE` del
*mismo* slot: crear el disco RAM mueve el driver de sitio, y quedan colgando
sus punteros absolutos y la cadena de dispositivos del IOCS en `&BFCA2`.

**La solución.** `ORG &AD000`, dentro del área de código máquina reservada, que
no es de nadie más. Al no moverse nunca, sobra toda la maquinaria de
reubicación:

| | fichero | cuerpo | carga |
|---|---|---|---|
| PLINKC 1.62 original | 1554 B | 1538 B | `&BF000-&BF601`, y de ahí a `S1:` |
| PLINKC 1.64 | 1348 B | 1332 B | `&AD000-&AD533`, y ahí se queda |

259 bytes menos de código y tabla que ya no hacen falta.

`FILES "S1:"` ya no muestra ningún `PLINK.SYS`: esta versión no es un bloque de
memoria, es código máquina residente.

## Cambio 2 — el buffer se vuelca solo

**El problema.** Al guardar, BASIC pide un puntero al sector (orden `$17`) y
escribe **dentro** del buffer del driver. PLINKC no se entera de cuándo ha
terminado, y solo manda ese sector al servidor cuando necesita el hueco para
otro. Resultado: lo último que grabas se queda en la pocket hasta que haces
cualquier otra cosa con `L:`. Había que teclear `INIT "L:5"` detrás de cada
`SAVE`.

**El cambio.** 53 bytes. Un gancho al principio de `main` que expulsa el buffer
en cuanto llega **cualquier orden que no sea `$17`**, más la rutina `volcar`.

```
        cmp  a,$17
        jrz  nv_ya          ; es la de pedir puntero: no tocar
        pushu ba / i / x / y
        mv   ba,[sect_number]
        inc  ba
        jrz  nv_fin         ; -1 = aún no hay sector cargado
        bsr  volcar
nv_fin: popu y / x / i / ba
        mv   (flag),0
nv_ya:
```

| | cuerpo | fin |
|---|---|---|
| sin volcado (`PLINKCF.BIN`) | 1279 B | `&AD4FE` |
| con volcado (`PLINKC64.BIN`) | 1332 B | `&AD533` |


## Mapa de memoria

Reserva de 75 KB con `SET.BAS` (de E.KAKO) apuntando a `&AD000`:

| desde | hasta | qué |
|---|---|---|
| `&AD000` | `&AD533` | PLINKC 1.64 (lo que hay en el `.BIN`) |
| `&AD534` | `&ADA6F` | sus buffers y la caché: 1340 B que no van en el fichero |
| `&ADA70` | `&ADFFF` | libre |
| `&AE000` | | variables de los juegos |
| `&B0000` | | objetos |
| | `&BFC00` | tope del área reservada |

Ojo con la segunda fila: el `.BIN` acaba en `&AD533`, pero el driver declara con
`suborg` 1340 bytes más de área de trabajo —`sect_1`, `sect_2`, 7 sectores de
caché de FAT y 1 de datos— que no viajan en el fichero y sí están ocupados en
cuanto corre. El footprint real es `&AD000-&ADA6F`.

## Instalar

```
LOADM "L:PLINKC64.BIN"
CALL &AD000
```

Debe decir `Installed.` Si ya estaba, dice `Already exists.` y no hace nada.

## Comprobar

```
PRINT HEX$ (PEEK &BFCA2+PEEK &BFCA3*256+PEEK &BFCA4*65536)
```

Tiene que salir **AD13D**, la cabecera del driver dentro del binario. Su
estructura, verificada byte a byte sobre `PLINKC64.BIN`:

| dirección | contenido |
|---|---|
| `&AD13D` | puntero al siguiente driver de la cadena (3 B) |
| `&AD140` | número de dispositivo, asignado al instalar desde el 10 |
| `&AD141` | atributo `$83` |
| `&AD142` | entrada del cuerpo: `&AD172` |
| `&AD145` | nombre: `'L:',0` |

## Desinstalar

No hay `CALL` de desinstalar. Se desengancha de la cadena del IOCS a mano:

```
P=PEEK &BFCA2+PEEK &BFCA3*256+PEEK &BFCA4*65536
POKE &BFCA2,PEEK P,PEEK (P+1),PEEK (P+2)
```

Los tres bytes de `&AD13D` son el driver que había antes de instalarse; al
copiarlos a `&BFCA2` la cadena se cierra saltándoselo. `FILES "L:"` debe dar
error.

Sólo vale si PLINKC es el primero de la cadena, que es lo que comprueba el
`PRINT HEX$` de arriba.

El `LOADM` no hay que deshacerlo: son bytes muertos en el área reservada. Ahora
bien, siguen ahí, así que **`CALL &AD000` lo vuelve a instalar**. Para dejarlo
muerto de verdad:

```
POKE &AD000,&07
```

`07` es `RETF`: el `CALL` vuelve en el acto sin hacer nada. Y para recuperar
además los 12 KB, `SET.BAS` con `&B0000` — `MACWRK` es el suelo, no el techo,
así que los objetos de `&B0000` para arriba no se tocan.

Un RESET también lo quita, pero se lleva la reserva por delante.



---

# APLINKS para macOS 1.0

## Versiones

| versión | autor | año | sistema |
|---|---|---|---|
| APLINKS 1.03 | N.Kon | 1992-93 | MS-DOS (fuente en C) |
| APLINKS for Mac 1.03 | — | — | Mac clásico |
| aplinksw32 1.02e | — | — | Windows. Desde la 1.01 pone fecha y hora |
| **APLINKS para macOS 1.0** | PHOENIX | 2026 | macOS nativo |

Habla el protocolo de la 1.03 sin cambiarlo, así que sustituye a cualquiera de
los tres.

## Qué se ha mejorado sobre el original

1. **Sincroniza según escribes.** El original solo volcaba al hacer
   `INIT "L:D"`: si algo fallaba antes, se perdía todo. Aquí se vuelca en
   cuanto la línea lleva un cuarto de segundo parada — no sector a sector, para
   no escribir ficheros a medias con la FAT aún inconsistente.
2. **Modo 512 KB.** PLINKC lo soporta desde la 1.60; el servidor de 1993 no.
3. **Listado en vivo.** El panel `UNIDAD L:` se redibuja leyendo el disco
   virtual, o sea lo que ve la pocket de verdad, no la carpeta del Mac.
4. **Sin la línea engañosa de 0 bytes.** Al crear un fichero la pocket escribe
   la entrada de directorio con tamaño 0 y sólo al final pone el real. Antes eso
   se volcaba como un fichero vacío.
5. **Fecha y hora en los ficheros.** Como el aplinksw32 desde su 1.01.
6. **Detección automática del puerto.**
7. **Se oculta el nombre de usuario en las rutas.**
8. **Entradas de directorio repetidas.** Tras un `KILL` y un `SAVE` del mismo
   nombre pueden convivir la entrada caduca y la nueva. Se usa la más completa.
9. **Volcado automático,** con el driver PLINKC 1.64. Esto no es del servidor:
   son los 53 bytes de la pocket.

## Uso

1. `CARPETA` → Elegir... la que quieres ver como `L:`
2. `PUERTO` → tu adaptador USB-serie (`↻` para releer)
3. `VELOCIDAD` → la misma que en la pocket
4. `CONECTAR`

En la pocket, la cadena de `OPEN`:

```
9600,N,8,1,A,L,&H1A,N,N
```

Control de flujo: **ninguno**. Con CE-135T o CE-140T es lo correcto, y XON/XOFF
es imposible porque el protocolo manda datos binarios.

Para terminar, lo limpio es `INIT "L:D"` desde la pocket.

## Capacidad

| | 128 KB | 512 KB |
|---|---|---|
| capacidad | 125.440 B | 514.048 B |
| ficheros | 128 | 128 |

Los dos lados tienen que coincidir, pero de eso se encarga la pocket: al hacer
`INIT "L:5"` manda un aviso y el servidor cambia y recarga la carpeta él solo.
`INIT "L:1"` vuelve a 128. La ventana arranca en 512.

## Límites

- 128 ficheros como máximo, y sólo del primer nivel de la carpeta.
- Los nombres se fuerzan a 8.3 en mayúsculas. Si dos chocan, avisa al conectar.
- Cada fichero gasta sectores enteros de 128 bytes, y siempre uno de más.
- La carpeta se lee **al conectar**. Lo que añadas con el servidor en marcha no
  sale hasta que reconectes.
- Los ficheros viajan en binario, sin tocar nada: ni saltos de línea ni
  codificación. A propósito, como el APLINKS original.
- Lo que borres en `L:` con `KILL` se borra **también** en la carpeta del Mac.
  No hay papelera.

## Una fidelidad que parece un fallo

Al leer un sector de datos el servidor manda 129 bytes pero sólo rellena 128:
el último va con basura de la lectura anterior. Es un descuido del APLINKS de
1992, pero la caché de PLINKC usa ese byte como *lookahead* del sector
siguiente y está calibrada así. Está copiado a propósito: "arreglarlo"
descuadra la caché.

## La app

`APLINKS.app` es autónoma: 28 MB, arm64, firmada ad-hoc. 
 Se copia a cualquier Mac y funciona.


---

# Lo que no se pudo hacer

**Sectores de 256 o 512 bytes.** No por los contadores de 8 bits, que sí tienen
arreglo: `MV r,[r3+n]` tiene **un solo byte de desplazamiento**, y PLINKC
direcciona la caché con 13 desplazamientos derivados de `sectsiz`. Con 256 ya
se pasa (`cache_next` = 259). Mizobata metió los metadatos *dentro* de cada
entrada de caché, y eso sólo funciona mientras la entrada quepa en 8 bits.


**`MEM$="S1"` para ganar memoria.** Medido en la máquina: deja `S1:` en 64 KB, y
objeto más fuente son 78. La tarjeta *es* la página `&A0000`; el montaje depende
de `MEM$="B"`.

---

# Créditos

- **Protocolo y disco virtual:** APLINKS 1.03, (c) 1992,93 N.Kon.
- **Driver de la pocket:** PLINK.SYS 1.04, (c) 1990,93,94 N.Kon.
- **Caché y modo 512 KB:** PLINKC 1.62, (c) 1996,1997,1999 Daisuke Mizobata.
- **Reserva de memoria:** `SET.BAS` de E.KAKO.
- **PLINKC ver 1.64 y APLINKS para macOS 1.0:** (c) 2026 PHOENIX.

---

## Vídeo de referencia

https://www.youtube.com/watch?v=1Y448n0gjzc
