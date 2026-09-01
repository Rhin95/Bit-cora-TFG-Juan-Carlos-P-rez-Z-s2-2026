---
created: 2026-08-19T23:35
updated: 2026-08-28T22:01
---
# Guía de Puesta en Marcha: Raspberry Pi CM4 + Waveshare SX1303 LoRaWAN Gateway HAT

## Placas utilizadas

| Componente | Modelo | Amazon ASIN |
|---|---|---|
| Placa base (carrier board) | Waveshare Mini Base Board (A) — CM4-IO-BASE-A | B092D8T71P |
| HAT de gateway LoRa | Waveshare SX1303 915M LoRaWAN Gateway HAT (con módulo GNSS L76K) | B0DKJJJLX5 |
| Módulo de cómputo | Raspberry Pi Compute Module 4 (con eMMC) | (el que ya tenés) |

---

## 1. Lista completa de lo que necesitás

### 1.1 Hardware

- **Raspberry Pi CM4** (con eMMC y/o WiFi, según tu variante).
- **Waveshare CM4-IO-BASE-A** (carrier board / placa base).
- **Waveshare SX1303 915M LoRaWAN Gateway HAT** (con módulo Mini-PCIe y L76K GNSS).
- **Cable USB-C a USB-A** — para conectar la placa base a tu PC y flashear el eMMC.
- **Fuente USB-C de 5V / 2.5A mínimo** — para alimentar la placa base en operación normal (puede ser un cargador de celular USB-C de buena calidad, 5V 3A recomendado).
- **Cable Ethernet RJ45** — para conectar la Pi a tu red local (recomendado para la primera configuración).
- **Antena LoRa 915 MHz** con conector SMA — viene incluida con el HAT normalmente. Verificá que venga una; si no, comprá una antena 915 MHz con conector SMA macho.
- **Antena GPS** — el HAT trae una antena GPS cerámica pasiva. Debe colocarse con la cara **sin etiqueta hacia arriba**, en un lugar con vista al cielo.
- **LilyGo T3 S3 (SX1262, 915 MHz)** — tu dispositivo de prueba LoRa. Ya lo tenés programado con los ejemplos de LilyGo para recepción.
- **Cable USB-C** para el T3 S3 (para alimentarlo y ver el monitor serial).

### 1.2 Herramientas de software (a instalar en tu PC)

| Software | Para qué sirve | Dónde descargarlo |
|---|---|---|
| **Raspberry Pi Imager** | Grabar Raspberry Pi OS en el eMMC del CM4 | [raspberrypi.com/software](https://www.raspberrypi.com/software/) |
| **rpiboot** (RPiBoot Setup) | Hacer que tu PC vea el eMMC del CM4 como un disco USB | Windows: [RPiBoot installer](https://github.com/raspberrypi/usbboot/raw/master/win32/rpiboot_setup.exe). Linux/Mac: compilar desde [github.com/raspberrypi/usbboot](https://github.com/raspberrypi/usbboot) |
| **Cliente SSH** | Conectarte a la Pi sin monitor | Windows: PuTTY o Windows Terminal (ssh viene integrado en Win10/11). Linux/Mac: terminal integrada |
| **Arduino IDE** (ya lo tenés) | Para los sketches del T3 S3 | [arduino.cc](https://www.arduino.cc/en/software) |
| **Monitor serial** | Ver la salida del T3 S3 | El propio Arduino IDE (Serial Monitor) o PuTTY |

### 1.3 Herramientas físicas

- Destornillador Phillips pequeño (si necesitás fijar el CM4 a la placa base con tornillos).
- Nada más. No se necesita soldadura.

---


## 3. Prueba 2 — LoRa: transmitir desde el HAT y recibir en el T3 S3

### Objetivo

Montar el HAT SX1303 sobre la CM4-IO-BASE-A, compilar las herramientas de Semtech (sx1302_hal), y hacer que el gateway transmita un paquete LoRa raw a 915 MHz que el LilyGo T3 S3 reciba usando los ejemplos de LilyGo.


### 3.1 Prueba de recepción raw LoRa (escuchar paquetes del T3 S3)

Ahora vamos a usar la herramienta `test_loragw_hal_rx` que viene dentro de `libloragw` para poner al concentrador en modo de escucha raw en 915 MHz.

**Paso 12.** Compilá los tests de libloragw (si no se compilaron ya):

```bash
cd ~/sx1302_hal/libloragw
make
```

**Paso 13.** Ejecutá el test de recepción:

```bash
cd ~/sx1302_hal/libloragw/
./test_loragw_hal_rx -a 915.0 -b 915.0 -r 1250
```

Parámetros:
- `-a 915.0` → Radio A escuchando en 915.0 MHz
- `-b 915.0` → Radio B escuchando en 915.0 MHz
- `-r 1250` → Tipo de radio: SX1250 (el que usa tu HAT con SX1303)

El programa va a quedarse esperando paquetes. Verás algo como:
```
INFO: concentrator started, packet can now be received
```

**Paso 14.** En tu **LilyGo T3 S3**, cargá el ejemplo de **transmisión** (sender/TX) de la librería de LilyGo. Asegurate de que:
- La frecuencia esté en **915.0 MHz** (debe coincidir con lo que pusiste en `-a`).
- El spreading factor sea **SF7** (el test de Semtech usa SF7 por defecto).
- El ancho de banda sea **125 kHz** (BW125, el estándar LoRa).

Si estás usando el ejemplo `SX1262_Transmit` de RadioLib (que es lo que usan los ejemplos de LilyGo), la configuración relevante es:

```cpp
// En el sketch del T3 S3 (transmisor)
radio.setFrequency(915.0);
radio.setSpreadingFactor(7);
radio.setBandwidth(125.0);
```

**Paso 15.** Enciende el T3 S3 con el sketch de TX cargado. En la terminal SSH donde corre `test_loragw_hal_rx`, deberías empezar a ver paquetes recibidos:

```
----- LoRa packet -----
  count_us: XXXXXX
  size:     XX
  chan:     X
  status:  0x01 (CRC OK)
  freq_hz: 915000000
  rssi:    -XX.X
  snr:     X.X
```

Si ves paquetes con `CRC OK`: **la comunicación LoRa funciona**.

Presioná **Ctrl+C** para detener el test.

> **Nota:** Este test usa recepción raw, no LoRaWAN completo. Es suficiente para validar que el hardware de radio funciona en ambos extremos. Para un gateway LoRaWAN completo con packet forwarder y conexión a The Things Network, eso es una configuración posterior.

### 3.6 Prueba alternativa: usar el Packet Forwarder para escuchar

Si el test raw no captura paquetes (por diferencias de configuración), podés usar el packet forwarder con la configuración US915:

```bash
cd ~/sx1302_hal/packet_forwarder/

# Copiar la configuración para US915 con radios SX1250
cp global_conf.json.sx1250.US915 global_conf.json

# Ejecutar (sin conectar a un servidor real, solo para ver paquetes)
./lora_pkt_fwd
```

El packet forwarder escuchará en los 8 canales US915 estándar y mostrará en consola cada paquete LoRa que reciba.

---

## 4. Prueba 3 — GNSS: leer posición del módulo L76K

### Objetivo

Leer datos de posición GPS/BeiDou desde el módulo L76K integrado en el HAT, usando Python por UART.

### 4.1 Habilitar UART

**Paso 1.** Desde SSH:

```bash
sudo raspi-config
```

Navegá a: **Interface Options** → **Serial Port**.

Te va a hacer dos preguntas:
1. "Would you like a login shell to be accessible over serial?" → **No**
2. "Would you like the serial port hardware to be enabled?" → **Yes**

Seleccioná **OK** → **Finish** → **Reiniciar**.

**Paso 2.** Verificá que el UART está activo:

```bash
ls /dev/ttyS0
```

Debe responder `/dev/ttyS0` sin error. (En algunos CM4, puede ser `/dev/ttyAMA0` — probá ambos si uno no funciona.)

### 4.2 Probar con minicom (lectura directa de NMEA)

**Paso 3.** Instalá minicom:

```bash
sudo apt install -y minicom
```

**Paso 4.** Abrí el puerto serial del L76K:

```bash
sudo minicom -D /dev/ttyS0 -b 9600
```

Parámetros:
- `-D /dev/ttyS0` → puerto serial (UART0 de la Pi, conectado al TX/RX del L76K a través del HAT).
- `-b 9600` → velocidad 9600 baudios (la velocidad por defecto del L76K).

**Paso 5.** Deberías ver sentencias NMEA desplazándose en la pantalla:

```
$GNRMC,123456.00,A,0958.12345,N,08412.67890,W,0.0,0.0,200826,,,A*XX
$GNGGA,123456.00,0958.12345,N,08412.67890,W,1,08,1.2,1100.0,M,-25.0,M,,*XX
$GNVTG,,T,,M,0.0,N,0.0,K,A*XX
```

- Si ves sentencias con datos de latitud/longitud (`0958.xxxxx,N` y `08412.xxxxx,W`): **el GPS tiene fix y funciona**.
- Si ves sentencias vacías (`$GNRMC,,V,,,,,,,,,,N*XX`): el GPS aún no tiene fix. Poné la antena GPS al aire libre o junto a una ventana y esperá 1-3 minutos para el primer fix.

Para salir de minicom: presioná **Ctrl+A**, luego **X**, luego **Enter**.

### 4.3 Leer GPS con Python

**Paso 6.** Instalá pyserial:

```bash
pip3 install pyserial --break-system-packages
```

**Paso 7.** Creá el script de prueba:

```bash
nano ~/test_gps.py
```

Pegá este contenido:

```python
#!/usr/bin/env python3
"""
Prueba de lectura del módulo GNSS L76K por UART.
Lee sentencias NMEA y extrae latitud/longitud de las sentencias $GNRMC.
"""
import serial
import sys

PUERTO = "/dev/ttyS0"  # Probar /dev/ttyAMA0 si este no funciona
BAUDRATE = 9600

def parsear_gnrmc(sentencia):
    """Parsea una sentencia $GNRMC y devuelve lat, lon si hay fix."""
    campos = sentencia.split(",")
    if len(campos) < 7:
        return None
    
    estado = campos[2]  # A = válido, V = no válido
    if estado != "A":
        return None
    
    # Latitud: DDMM.MMMMM
    lat_raw = campos[3]
    lat_dir = campos[4]
    # Longitud: DDDMM.MMMMM
    lon_raw = campos[5]
    lon_dir = campos[6]
    
    if not lat_raw or not lon_raw:
        return None
    
    # Convertir a grados decimales
    lat_deg = int(lat_raw[:2]) + float(lat_raw[2:]) / 60.0
    if lat_dir == "S":
        lat_deg = -lat_deg
    
    lon_deg = int(lon_raw[:3]) + float(lon_raw[3:]) / 60.0
    if lon_dir == "W":
        lon_deg = -lon_deg
    
    return lat_deg, lon_deg

def main():
    print(f"Abriendo {PUERTO} a {BAUDRATE} baud...")
    print("Esperando datos GNSS (puede tardar 1-3 min para primer fix)...")
    print("Presioná Ctrl+C para salir.\n")
    
    try:
        ser = serial.Serial(PUERTO, BAUDRATE, timeout=2)
    except serial.SerialException as e:
        print(f"ERROR: No se pudo abrir {PUERTO}: {e}")
        print("Probá con /dev/ttyAMA0 editando la variable PUERTO.")
        sys.exit(1)
    
    try:
        while True:
            linea = ser.readline().decode("utf-8", errors="ignore").strip()
            if not linea:
                continue
            
            # Mostrar todas las sentencias NMEA
            print(f"NMEA: {linea}")
            
            # Si es GNRMC, intentar parsear posición
            if linea.startswith("$GNRMC") or linea.startswith("$GPRMC"):
                pos = parsear_gnrmc(linea)
                if pos:
                    lat, lon = pos
                    print(f"\n>>> POSICIÓN: Lat={lat:.6f}°, Lon={lon:.6f}° <<<\n")
                else:
                    print("    (sin fix todavía...)")
    
    except KeyboardInterrupt:
        print("\nDetenido por el usuario.")
    finally:
        ser.close()

if __name__ == "__main__":
    main()
```

Guardá con **Ctrl+O**, **Enter**, **Ctrl+X**.

**Paso 8.** Ejecutá el script:

```bash
python3 ~/test_gps.py
```

Verás las sentencias NMEA y, cuando el GPS obtenga fix, la posición en grados decimales:

```
NMEA: $GNRMC,183025.00,A,0958.12345,N,08412.67890,W,0.0,0.0,200826,,,A*5E
>>> POSICIÓN: Lat=9.968724°, Lon=-84.211315° <<<
```

Presioná **Ctrl+C** para detener.

**¡Prueba 3 completada!** El módulo GNSS está leyendo posición.

---

## 5. Resumen de estado después de las 3 pruebas

| Prueba | Qué validaste | Estado esperado |
|---|---|---|
| 1. Hola Mundo | CM4 arranca, SSH funciona, Python funciona | ✅ |
| 2. LoRa | HAT SX1303 recibe paquetes del T3 S3 a 915 MHz | ✅ |
| 3. GNSS | L76K devuelve coordenadas GPS válidas por UART | ✅ |

## 6. Próximos pasos sugeridos

Después de validar las 3 pruebas:

1. **Configurar el packet forwarder** completo con `global_conf.json.sx1250.US915` y conectarlo a The Things Network (o ChirpStack local).
2. **Escribir la lógica en Python** que lea el GPS, maneje la cola de datos, y decida cuándo usar el fallback satelital.
3. **Configurar el T3 S3 como nodo LoRaWAN** (no solo raw LoRa) para integrarse con el gateway completo.
4. **Justificar la banda 915 MHz** en el anteproyecto citando el PNAF (Decreto 44010-MICITT) para la Región 2 UIT.

---

## 7. Referencia rápida de comandos

```bash
# Conectarse por SSH
ssh pi@cm4-gateway.local

# Reiniciar
sudo reboot

# Apagar
sudo shutdown now

# Ver temperatura
vcgencmd measure_temp

# Verificar SPI activo
ls /dev/spidev*

# Obtener EUI del concentrador LoRa
cd ~/sx1302_hal/util_chip_id/ && ./chip_id

# Escuchar paquetes LoRa raw
cd ~/sx1302_hal/libloragw/ && ./test_loragw_hal_rx -a 915.0 -b 915.0 -r 1250

# Correr packet forwarder US915
cd ~/sx1302_hal/packet_forwarder/ && ./lora_pkt_fwd

# Leer GPS con minicom
sudo minicom -D /dev/ttyS0 -b 9600

# Leer GPS con Python
python3 ~/test_gps.py
```
