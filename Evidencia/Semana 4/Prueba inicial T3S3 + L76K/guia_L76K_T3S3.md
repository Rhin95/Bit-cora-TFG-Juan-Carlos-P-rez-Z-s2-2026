---
created: 2026-08-29T16:56
updated: 2026-08-31T18:03
---
# Guía: L76K GPS en LilyGo T3-S3 V1.2 SX1262

## Contexto

El T3-S3 V1.2 SX1262 **no trae GPS integrado**. Esta guía documenta cómo conectar y configurar un módulo L76K externo usando el ejemplo `TinyGPS_Example` del repo [LilyGo LoRa Series](https://github.com/Xinyuan-LilyGO/LilyGo-LoRa-Series).

---

## 1. Conexión física

| L76K | T3-S3 GPIO | Dirección |
|------|-----------|-----------|
| TX   | GPIO 44   | GPS TX → Board RX |
| RX   | GPIO 43   | Board TX → GPS RX |
| VCC  | 3.3V      | Alimentación |
| GND  | GND       | Tierra |

> **Nota:** PPS no se conecta — los timestamps NMEA son suficientes para tracking marítimo.

---

## 2. Archivos modificados

Se trabaja sobre el repo LilyGo LoRa Series. Solo se tocan archivos del ejemplo GPS; el código de tesis (`examples/RadioLibExamples/Transmit_Interrupt`) queda intacto.

### 2.1 `examples/GPS/TinyGPS_Example/utilities.h`

Dentro del bloque `#ifdef LILYGO_T3_S3_V1_2_SX1262` (o el `#elif` correspondiente a esta placa), agregar:

```c
#define GPS_RX_PIN                  44   // board RX <- GPS TX
#define GPS_TX_PIN                  43   // board TX -> GPS RX
#define GPS_BAUD_RATE               9600
#define HAS_GPS
```

**¿Por qué?** Sin `HAS_GPS`, `setupBoards()` nunca inicializa el puerto serie del GPS — el `#ifdef HAS_GPS` lo excluye de compilación.

### 2.2 `examples/GPS/TinyGPS_Example/TinyGPS_Example.ino`

Agregar **justo después** de `setupBoards()`:

```cpp
SerialGPS.begin(GPS_BAUD_RATE, SERIAL_8N1, GPS_RX_PIN, GPS_TX_PIN);
```

**¿Por qué?** `setupBoards()` ejecuta una rutina de auto-detección diseñada para módulos UBlox (protocolo binario UBX). El L76K habla NMEA puro, así que la rutina:

1. No recibe los ACKs UBX que espera.
2. Barre varios baud rates buscando respuesta.
3. Termina dejando el puerto en **4800 baud** — incorrecto para el L76K (9600).

La línea extra fuerza el baud rate correcto antes de empezar a leer datos.

### 2.3 `platformio.ini`

Cambiar el `src_dir` activo para compilar el ejemplo GPS:

```ini
;src_dir = examples/RadioLibExamples/Transmit_Interrupt
src_dir = examples/GPS/TinyGPS_Example
```

> **Recordar:** revertir a `Transmit_Interrupt` al volver a pruebas de enlace LoRa.

---

## 3. Librerías involucradas

| Librería | Rol |
|----------|-----|
| **TinyGPS++** (`lib/TinyGPSPlus`) | Parsea sentencias NMEA ($GPRMC, $GPGGA, etc.) y expone `location`, `date`, `time` |
| **HardwareSerial** (Arduino-ESP32) | UART físico vía alias `SerialGPS` (= `Serial1`, definido en `LoRaBoards.h`) |
| **LoRaBoards.h/.cpp** | Infraestructura del repo — inicializa periféricos comunes de la placa |

---

## 4. Verificación

Al flashear y abrir el monitor serie (115200 baud), deberías ver datos GPS parseados: latitud, longitud, fecha, hora y número de satélites. Si solo ves ceros o `INVALID`, verificar:

1. **Cableado:** TX↔RX no estén cruzados al revés.
2. **Antena:** el L76K necesita cielo abierto para fix inicial (puede tardar 1-2 min en cold start).
3. **Baud rate:** confirmar que la línea `SerialGPS.begin(9600, ...)` está **después** de `setupBoards()`.

---

## 5. Resumen de cambios

```
repo LilyGo-LoRa-Series/
├── examples/GPS/TinyGPS_Example/
│   ├── utilities.h          ← +4 líneas (#define GPS pins + HAS_GPS)
│   └── TinyGPS_Example.ino  ← +1 línea (SerialGPS.begin)
└── platformio.ini            ← cambio de src_dir
```

Tres archivos, cinco líneas de código. Nada del código de tesis se toca.


# Resultados 


## Serial monitor 

![](imagenes/Log%20de%20prueba.png)




## Ensamble inicial

![](imagenes/Montaje%20físico.jpeg)


