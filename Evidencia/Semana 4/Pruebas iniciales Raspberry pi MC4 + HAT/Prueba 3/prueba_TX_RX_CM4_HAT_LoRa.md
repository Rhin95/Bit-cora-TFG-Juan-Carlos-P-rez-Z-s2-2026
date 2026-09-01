---
created: 2026-08-28T22:33
updated: 2026-08-29T12:53
---
# Prueba 3: TX y RX del CM4 + HAT LoRa contra LilyGo T3 S3

**Objetivo:** Validar que el conjunto **CM4 + Waveshare SX1303 HAT** puede **recibir** paquetes LoRa transmitidos desde una T3 S3, y **transmitir** paquetes LoRa que sean recibidos por otra T3 S3. Ambos sentidos de comunicación en 915 MHz, banda US915 (Costa Rica).

**Prerrequisitos:**
- Prueba Intermedia (validación del HAT) completada con éxito.
- Dos LilyGo T3 S3 programadas:
  - **T3 S3 #1 (TX):** transmite paquetes LoRa periódicos.
  - **T3 S3 #2 (RX):** escucha paquetes LoRa y los muestra por Serial.
- sx1302_hal ya compilado y probado (`~/sx1302_hal/`).
- Antenas de 915 MHz conectadas a los 3 dispositivos.
- Sync word en las T3 S3 configurado a **`0x12` (privado)** — ver nota 1 al final.

**Parámetros de la prueba (todos deben coincidir en TX y RX):**

| Parámetro        | Valor                        | Fuente en T3 S3                                    |
| ---------------- | ---------------------------- | -------------------------------------------------- |
| Frecuencia       | 915.0 MHz                    | `platformio.ini:114` (`-DCONFIG_RADIO_FREQ=915.0`) |
| Bandwidth        | 125 kHz                      | default rama `USING_SX1262`                        |
| Spreading Factor | 12 (variable con `TEST_SF`)  | `Transmit_Interrupt.ino:153`                       |
| Coding Rate      | **4/6** (`setCodingRate(6)`) | fijo ambos                                         |
| Preámbulo        | **16 símbolos**              | fijo ambos                                         |
| Sync Word        | **`0x34` (publico)**         |                                                    |
| CRC              | habilitado                   | `setCRC(true)`                                     |
| Header           | explícito                    | default RadioLib                                   |
| IQ               | normal (no invertido)        | default                                            |
| Potencia TX      | 22 dBm (T3 S3 TX)            | solo relevante en el nodo TX                       |

⚠️ **Cualquier discrepancia en un parámetro = 0 paquetes recibidos.** LoRa no negocia parámetros.

---

## Parte A: Preparación del entorno

### Paso A.1 — Ubicación física de los dispositivos

Para las primeras pruebas, colocá los dispositivos en la misma mesa, **separados al menos 30-50 cm** entre sí. Estar demasiado cerca puede saturar el receptor y dar errores de CRC.

Distribución recomendada:
- CM4 + HAT en un extremo
- T3 S3 TX a 30-50 cm de la CM4
- T3 S3 RX a 30-50 cm de la T3 S3 TX (puede estar del otro lado de la CM4)

### Paso A.2 — Conectar y abrir monitor serial de cada T3 S3

Para poder ver qué está transmitiendo/recibiendo cada T3 S3, conectalas por USB a tu PC y abrí un monitor serial (Arduino IDE, PlatformIO o `putty`/`minicom`) para cada una:

- **Baud rate:** el que hayas configurado en el firmware (típicamente 115200)
- Una ventana serial por cada T3 S3, para poder ver los logs simultáneamente

### Paso A.3 — Conectarse a la CM4 por SSH

Desde tu PC:

```bash
ssh jcperezz@cm4-jk-gateway.local
```

---

## Parte B: Configurar el sx1302_hal en modo privado (sync word 0x12)

Por defecto, el sx1302_hal está compilado para modo **público** (sync word `0x34`, típico de LoRaWAN público). Como estamos usando `0x12` (privado), hay que verificar/ajustar la configuración de los programas de prueba.

### Paso B.1 — Ver las opciones disponibles del test RX

```bash
cd ~/sx1302_hal/libloragw
./test_loragw_hal_rx -h
```

Buscá en la salida si aparece una opción tipo `-k` o `--public`:

- Si aparece `-k 0` → correlo con `-k 0` para modo privado (0x12).
- Si aparece `-k 1` → es el default público (0x34).
- Si NO aparece esa opción → hay que modificar el código fuente del test (ver Paso B.3).

### Paso B.2 — Ver las opciones del test TX

```bash
./test_loragw_hal_tx -h
```

Igual que arriba: buscá `-k` para modo público/privado.

### Paso B.3 — Si NO existe el flag, modificar el fuente

Si los tests no exponen ese parámetro por CLI, hay que cambiar el default en el código fuente. Editá:

```bash
nano ~/sx1302_hal/libloragw/tst/test_loragw_hal_rx.c
```

Buscá una línea que contenga `boardconf.lorawan_public` o similar (usa `Ctrl+W` en nano):

```c
boardconf.lorawan_public = true;
```

Cambiala a:

```c
boardconf.lorawan_public = false;
```

Guardá (`Ctrl+O`, Enter) y salí (`Ctrl+X`).

Hacé lo mismo con `test_loragw_hal_tx.c` en el mismo directorio.

Después recompilá:

```bash
cd ~/sx1302_hal
make clean all
make all
```

**Nota:** cuando `lorawan_public = false`, el HAL usa sync word `0x12` (privado). Cuando es `true`, usa `0x34` (público).

---

## Parte C: Prueba de RECEPCIÓN (T3 S3 → CM4)

En esta prueba, la **T3 S3 #1 transmite** paquetes LoRa y la **CM4 + HAT los recibe**.

### Paso C.1 — Iniciar el receptor en la CM4

```bash
cd ~/sx1302_hal/libloragw
sudo ./test_loragw_hal_rx -a 915.0 -b 915.0 -r 1250
```

(Si tenía flag `-k`, agregalo: `sudo ./test_loragw_hal_rx -a 915.0 -b 915.0 -r 1250 -k 0`.)

**Salida esperada al arrancar:**

```
===== sx1302 HAL RX test =====
INFO: rxpkt buffer size is set to 16
INFO: Select channel mode 0
CoreCell reset through GPIO23...
Opening SPI communication interface
Note: chip version is 0x12 (v1.2)
INFO: found temperature sensor on port 0x39
Waiting for packets...
```

**⚠️ Importante:** El SX1303 escucha **múltiples canales y todos los SF (SF7 a SF12) simultáneamente**. No hay que configurarle "SF12" explícitamente — cualquier paquete válido en el rango de canales configurados será recibido, sin importar qué SF use, siempre y cuando use el mismo sync word que la CM4.

**Distribución de canales:** con `-a 915.0 -b 915.0`, los 8 multi-SF channels caen aproximadamente entre 914.8 y 915.2 MHz. Tu T3 S3 TX en 915.0 MHz cae **exactamente en el canal central**, así que la recepción es óptima.

### Paso C.2 — Activar la T3 S3 TX

Si aún no está encendida, alimentala. Debería empezar a transmitir según su firmware. En su monitor serial deberías ver algo tipo:

```
[TX] Sending packet #1
[TX] TX done. Waiting...
```

### Paso C.3 — Verificar la recepción en la CM4

En la terminal SSH de la CM4, cada vez que la T3 S3 transmita, deberías ver un bloque de información como:

```
---
Nb valid packets received: 1 CRC OK, 0 CRC ERROR, 0 NO CRC
----- Received packet -----
  count_us: 1234567890
  chan   : 5
  status : CRC_OK
  size   : 12
  modulation : LORA
  bandwidth  : 125000 Hz
  datarate   : SF12
  coderate   : 4/6
  freq_hz    : 915000000
  RSSI       : -45.5 dBm
  SNR        : 9.5 dB
  payload    : 48 65 6C 6C 6F ...
```

**Qué mirar:**
- **`status: CRC_OK`** → paquete recibido sin errores ✅
- **`datarate: SF12`** y **`bandwidth: 125000 Hz`** y **`coderate: 4/6`** → coinciden con la T3 S3
- **`freq_hz: 915000000`** → 915 MHz
- **`RSSI`** → intensidad de señal en dBm. Cerca (30 cm), esperá entre -30 y -60 dBm.
- **`SNR`** → signal-to-noise ratio en dB. Con SF12 se puede recibir hasta con SNR de -20 dB.
- **`payload`** → los bytes reales que envió el TX (en hex).

### Paso C.4 — Interpretar contadores

Cada pocos segundos vas a ver una línea de resumen:

```
Nb valid packets received: 15 CRC OK, 0 CRC ERROR, 0 NO CRC
```

- **CRC OK**: paquetes recibidos y validados
- **CRC ERROR**: paquetes recibidos pero corruptos (típicamente por interferencia o dispositivos muy cerca)
- **NO CRC**: paquetes sin CRC (no debería aparecer, tu T3 S3 tiene `setCRC(true)`)

### Paso C.5 — Detener el receptor

Presioná `Ctrl+C`. Vas a ver un resumen final con estadísticas totales.

**Resultado esperado de la prueba C:** al menos varios paquetes con `CRC_OK`, con `datarate: SF12`, `coderate: 4/6`, `bandwidth: 125000 Hz` y `freq_hz: 915000000`.

---

## Parte D: Prueba de TRANSMISIÓN (CM4 → T3 S3)

En esta prueba, la **CM4 + HAT transmite** paquetes LoRa y la **T3 S3 #2 los recibe**.

### Paso D.1 — Preparar la T3 S3 RX

Encendé la T3 S3 #2 (RX) y abrí su monitor serial. Debería estar esperando paquetes:

```
[RX] LoRa Receiver ready. Listening at 915.0 MHz, SF12, BW125, CR4/6, SW=0x12...
```

### Paso D.2 — Transmitir desde la CM4

En la sesión SSH de la CM4:

```bash
cd ~/sx1302_hal/libloragw
sudo ./test_loragw_hal_tx -r 1250 -f 915.0 -s 12 -b 125 -c 2 -p 14 -n 10 -z 16
```

**Parámetros:**
- `-r 1250`: tipo de radio SX1250
- `-f 915.0`: frecuencia de transmisión en MHz
- `-s 12`: Spreading Factor 12
- `-b 125`: Bandwidth 125 kHz
- **`-c 2`: Coding Rate 4/6** (valor `1` = 4/5, **`2` = 4/6**, `3` = 4/7, `4` = 4/8)
- `-p 14`: potencia de transmisión en dBm (14 dBm ≈ 25 mW)
- `-n 10`: cantidad de paquetes a transmitir
- `-z 16`: tamaño del payload en bytes

Verificá también con `./test_loragw_hal_tx -h` si tiene una opción para preámbulo (típicamente `-o`). Si sí, agregala: `-o 16`. Si no, el default de sx1302_hal es 8 símbolos, y tu T3 S3 RX está configurado a 16 → **posible incompatibilidad**. Ver Paso D.4.

**Salida esperada:**

```
===== sx1302 HAL TX test =====
CoreCell reset through GPIO23...
Opening SPI communication interface
Note: chip version is 0x12 (v1.2)
INFO: found temperature sensor on port 0x39
INFO: Starting concentrator...
INFO: concentrator started, packet can be sent
Sending LoRa packet on 915.0 MHz (BW125kHz, SF12, CR4/6, 16 bytes payload) at 14 dBm
Sending packet 1/10 ... done
...
Sending packet 10/10 ... done
```

Cada paquete SF12 con 16 bytes y preámbulo 16 tarda aproximadamente **1.8 segundos** en el aire, así que la transmisión completa dura unos 20 segundos.

### Paso D.3 — Verificar recepción en la T3 S3 RX

En el monitor serial de la T3 S3 #2, deberías ver cada paquete:

```
[RX] Packet received!
  RSSI: -52 dBm
  SNR: 8.5 dB
  Length: 16 bytes
  Data: 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
```

### Paso D.4 — Si NO recibe paquetes, revisar preámbulo

Si la CM4 transmite (los logs muestran "done") pero la T3 S3 no ve nada, el problema más probable es el **preámbulo**:

- La T3 S3 está configurada a **16 símbolos** de preámbulo.
- El sx1302_hal por defecto usa **8 símbolos**.
- El receptor busca el preámbulo del tamaño esperado, y si no coincide, descarta el paquete.

**Solución A** — Configurar el preámbulo en el TX de la CM4:
Si `test_loragw_hal_tx -h` mostró una opción `-o`, usala:
```bash
sudo ./test_loragw_hal_tx -r 1250 -f 915.0 -s 12 -b 125 -c 2 -p 14 -n 10 -z 16 -o 16
```

**Solución B** — Cambiar el preámbulo en la T3 S3 RX a 8:
En el firmware de la T3 S3 RX:
```cpp
radio.setPreambleLength(8);
```
Y recompilar/reflashear.

---

## Parte E: Prueba simultánea RX + TX (opcional)

⚠️ **Nota:** `test_loragw_hal_rx` y `test_loragw_hal_tx` **no pueden correr al mismo tiempo** porque ambos toman control exclusivo del concentrador. Para RX/TX simultáneo (full-duplex) hay que usar `lora_pkt_fwd` (el packet forwarder), que sí maneja ambos sentidos con planificación.

Para esta guía, corré una prueba a la vez: primero RX (Parte C), después TX (Parte D).

---

## Resumen — Estado de comunicación LoRa

Al terminar esta prueba deberías tener validado:

| Sentido | Emisor | Receptor | Test |
|---|---|---|---|
| **RX** | T3 S3 #1 (TX) | CM4 + HAT | `test_loragw_hal_rx` recibe paquetes con `CRC_OK`, `SF12`, `CR 4/6` |
| **TX** | CM4 + HAT | T3 S3 #2 (RX) | `test_loragw_hal_tx` envía 10 paquetes, T3 S3 los ve todos |

**Con esto queda validada la comunicación LoRa bidireccional entre CM4 y dispositivos T3 S3** en modo LoRa privado con los parámetros exactos que usa tu proyecto.

---

## ⚠️ Notas importantes y precauciones

### 1. Sync word 0xAB no funciona con el SX1303 → cambiar a 0x12

El firmware original de las T3 S3 usa **`setSyncWord(0xAB)`**, un valor arbitrario que RadioLib permite en el chip SX1262. **El SX1303 (concentrador) NO acepta sync words arbitrarios** — solo soporta los dos oficiales de LoRaWAN:
- **`0x34`** — modo público (redes tipo TTN)
- **`0x12`** — modo privado

Para esta prueba, cambiar en el firmware de **ambas** T3 S3:

```cpp
radio.setSyncWord(0x12);   // en lugar de 0xAB
```

Y recompilar/reflashear.

**Por qué:** el demodulador del SX1303 filtra los paquetes por sync word en hardware antes de entregarlos al firmware. Un paquete con sync word `0xAB` es descartado sin siquiera aparecer en las estadísticas. No hay forma de "ver" esos paquetes con el HAL estándar.

**Alternativa (no recomendada):** si querés mantener `0xAB` en las T3 S3, tendrías que dejar de usar el concentrador SX1303 y en su lugar usar el chip **SX1261 auxiliar** que el HAT también incluye (originalmente pensado para LBT/Spectral Scan). Este sí acepta sync words arbitrarios, pero perdés todas las ventajas del concentrador multi-canal/multi-SF y hay que reescribir la aplicación desde cero.

### 2. Los parámetros DEBEN coincidir exactamente

LoRa no negocia parámetros. Si TX y RX difieren en **cualquiera** de estos, no hay comunicación:
- Frecuencia
- Bandwidth
- Spreading Factor
- Coding Rate
- Sync Word
- Preamble length
- IQ inversion (invert/normal)
- CRC habilitado/deshabilitado
- Header explícito/implícito

Con el default del sx1302_hal y tu T3 S3 hay dos diferencias a controlar: **sync word** (0x34 default vs 0x12 privado) y **preámbulo** (8 default vs 16 en tu T3 S3). Los pasos B y D.4 cubren cómo alinearlos.

### 3. Modo privado (0x12) vs LoRaWAN público

Usar `lorawan_public = false` (sync word `0x12`) es totalmente válido para aplicaciones **no-LoRaWAN** privadas, como es tu caso (baliza custom). No implica que sea LoRaWAN privado, solo que el sync word usado no es el público de TTN.

Tu aplicación puede ser un protocolo completamente propio sobre LoRa "puro", sin usar LoRaWAN, y esto es perfectamente compatible con el SX1303 usando sync word `0x12`.

### 4. La banda 915 MHz en Costa Rica

- Costa Rica está en Región 2 ITU y usa **US915**.
- Potencia máxima permitida sin licencia: **30 dBm (1 W) EIRP** en la mayoría de canales.
- La T3 S3 a 22 dBm y la CM4 a 14 dBm están dentro del límite.
- **Nunca uses 868 MHz (EU868)** — no autorizada en Costa Rica y tu HAT no está sintonizado para esa banda.

### 5. SF12 es lento pero de mayor alcance

Con SF12 + BW125 + CR 4/6 + preámbulo 16 símbolos + payload 16 bytes:
- **Airtime por paquete:** ~1.8 segundos
- **Data rate efectivo:** ~180 bps
- **Sensibilidad SX1303:** ~-137 dBm
- **Alcance típico:** varios km en línea de vista, cientos de metros con obstrucción

Para depuración rápida podés bajar a SF7 temporalmente (airtime < 100 ms), pero para la baliza marítima SF12 tiene sentido para maximizar alcance sobre el mar.

### 6. Coding Rate 4/6 vs 4/5

CR 4/6 (setCodingRate(6) en RadioLib) es más robusto que 4/5 pero más lento — agrega 20% más de bytes de FEC. Elección razonable para un enlace crítico como una baliza de emergencia.

En `test_loragw_hal_tx` se mapea así:
- `-c 1` → 4/5
- `-c 2` → **4/6** ← el tuyo
- `-c 3` → 4/7
- `-c 4` → 4/8

### 7. RSSI y SNR: qué esperar

| Condición | RSSI típico | SNR típico |
|---|---|---|
| Dispositivos a 30 cm | -30 a -50 dBm | +8 a +10 dB |
| Misma habitación (5 m) | -60 a -80 dBm | +5 a +10 dB |
| Muros/pisos (20-50 m) | -90 a -110 dBm | -5 a +5 dB |
| Límite de detección SF12 | -137 dBm | -20 dB |

Si el RSSI es muy alto (> -30 dBm) y ves CRC errors → estás **saturando el receptor**. Separá los dispositivos o bajá la potencia de TX.

### 8. RX y TX no simultáneos con test_*

`test_loragw_hal_rx` y `test_loragw_hal_tx` son programas de prueba con acceso exclusivo al concentrador. Para full-duplex necesitás `lora_pkt_fwd`.

### 9. Si no recibe nada, orden de diagnóstico

1. **TX transmite** → mirá el monitor serial de la T3 S3 TX (mensajes de "TX done").
2. **RX corre sin errores** → `test_loragw_hal_rx` muestra "Waiting for packets..." sin errores previos.
3. **Antenas conectadas** en los 3 dispositivos.
4. **Frecuencia exacta** → 915.0 MHz en ambos extremos.
5. **SF, BW, CR** → SF12, BW125, CR 4/6 en ambos extremos.
6. **Sync word** → `0x12` en ambos extremos (T3 S3 con `setSyncWord(0x12)`, sx1302_hal con `lorawan_public = false`).
7. **Preámbulo** → 16 en ambos extremos (o 8 en ambos), no mezclar.
8. **CRC** → habilitado en ambos.
9. **IQ** → normal en ambos (no invertido).
10. **Distancia** → al menos 30 cm entre dispositivos.


# Resultados de as pruebas 

## Prueba TX T3 S3 -> GATEGAY RX 



T3 S3

![](imagenes/Pasted%20image%2020260828235611.png)

GATEWAY
![](imagenes/Pasted%20image%2020260828235705.png)