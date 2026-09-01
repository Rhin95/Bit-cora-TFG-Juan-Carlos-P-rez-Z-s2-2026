---
created: 2026-07-30T01:49
updated: 2026-07-30T09:42
---
```cpp
Serial.printf("RX,OK,%u,%lu,%.2f,%.2f,%lu\n",
              (unsigned)index, (unsigned long)lastPacketMillis, rssiVal, snrVal, (unsigned long)turnaroundUs);
```

Tomando la línea `01:34:18.904 > RX,OK,1,16452,-9.00,7.50,38657`, cada campo significa:

| Campo          | Valor ejemplo      | Significado                                                                                                                                                                                                                                      |
| -------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `01:34:18.904` | timestamp          | Hora agregada por el monitor `log2file` de PlatformIO al capturar la línea (no viene del firmware)                                                                                                                                               |
| `RX`           | `RX`               | Tag fijo: línea generada por el receptor                                                                                                                                                                                                         |
| `OK`           | `OK`               | Estado del paquete: recibido y con CRC válido. Las otras variantes que emite el mismo bloque de código son `CRC_FAIL` (línea 540, paquete recibido pero corrupto) y `ERR,<state>` (línea 556, error de radio distinto a CRC, ej. timeout de SPI) |
| `1`            | índice             | Byte de índice del paquete (1-100), extraído del payload                                                                                                                                                                                         |
| `16452`        | `lastPacketMillis` | `millis()` del ESP32 en el momento en que se procesó el paquete (referencia de tiempo del propio RX, no sincronizada con el TX)                                                                                                                  |
| `-9.00`        | `rssiVal`          | RSSI en dBm reportado por el SX1262 para ese paquete                                                                                                                                                                                             |
| `7.50`         | `snrVal`           | SNR en dB reportado por el SX1262 para ese paquete                                                                                                                                                                                               |
| `38657`        | `turnaroundUs`     | Microsegundos entre la interrupción de recepción (IRQ) y que `radio.startReceive()` vuelve a dejar el radio armado — el turnaround que se pensaba usar luego para dimensionar el margen de guarda, según el diseño en [[project_lora_link_test]] |

Nota: en `CRC_FAIL` el orden de campos es igual (índice, millis, rssi, snr, turnaround), pero en `ERR` solo aparecen 3 campos (`state`, millis, turnaround) porque no hay RSSI/SNR válidos ni índice confiable.


Condiciones


En el **código TX actual** ([Transmit_Interrupt.ino](vscode-webview://1acgrh7nl055et4a9uvc024f0hd011374k9kv7ujdo0mvdm5u8s8/examples/RadioLibExamples/Transmit_Interrupt/Transmit_Interrupt.ino)):

| Parámetro           | Valor                       | ¿Dónde se define?                                                                                                                                                                                                             |
| ------------------- | --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Coding Rate**     | `6` (CR = 4/6)              | [línea 267](vscode-webview://1acgrh7nl055et4a9uvc024f0hd011374k9kv7ujdo0mvdm5u8s8/examples/RadioLibExamples/Transmit_Interrupt/Transmit_Interrupt.ino#L267): `radio.setCodingRate(6)`                                         |
| **Header**          | No se define explícitamente | No hay llamada a `setExplicitHeader()` / `setImplicitHeader()` en el archivo. RadioLib deja el header en modo **explícito por defecto** (implícito solo se activa si lo pides).                                               |
| **Payload size**    | `38` bytes                  | [línea 155](vscode-webview://1acgrh7nl055et4a9uvc024f0hd011374k9kv7ujdo0mvdm5u8s8/examples/RadioLibExamples/Transmit_Interrupt/Transmit_Interrupt.ino#L155): `#define TEST_PAYLOAD_SIZE 38` (1 byte índice + 37 bytes random) |
| **Preamble length** | `16` símbolos               | [línea 314](vscode-webview://1acgrh7nl055et4a9uvc024f0hd011374k9kv7ujdo0mvdm5u8s8/examples/RadioLibExamples/Transmit_Interrupt/Transmit_Interrupt.ino#L314): `radio.setPreambleLength(16)`                                    |


## Estimaciones con la calculadora


![](calculadora%20lora%20param.png)

![](calculadora%20lora%20data.png)
![](calculadora%20lora%20estimaciones.png)

## Datos experimentales 


01:34:24.686 > RX link test summary - SF5

01:34:24.686 >   Valid packets    : 50 / 100 (50.0% PDR)

01:34:24.686 >   CRC failures     : 0

01:34:24.686 >   Other errors     : 0

01:34:24.686 >   Session duration : 3.825 s

01:34:24.686 >   RSSI avg/min/max : -8.98 / -9.00 / -8.00 dBm

01:34:24.693 >   SNR  avg/min/max : 8.16 / 7.00 / 9.50 dB

01:34:24.693 >   Turnaround max   : 38825 us


## Error 

$|(3.2126 - 3.825)|/ 3.825 * 100 = 16.01045 $  
