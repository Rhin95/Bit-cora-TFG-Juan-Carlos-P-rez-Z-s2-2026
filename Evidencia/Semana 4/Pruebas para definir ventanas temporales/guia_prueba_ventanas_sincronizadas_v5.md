---
created: 2026-08-31T20:17
updated: 2026-08-31T20:20
---
# Guía: Prueba de Ventanas de Tiempo Sincronizadas por GPS — LoRa (v5)

## 1. Objetivo

Validar que un nodo T3S3 (ESP32-S3 + SX1262) y un gateway (RPi CM4 + SX1303/SX1250) pueden coordinarse mediante ventanas temporales derivadas de la hora UTC del GPS, sin necesidad de encenderse juntos ni comunicarse previamente. Ambos comparten la misma tabla de schedule y la misma referencia de tiempo (UTC), lo que les permite calcular independientemente el mismo "punto de unión" dentro de un ciclo que se repite cada 104.4 segundos.

---

## 2. Hardware necesario

| Dispositivo | Componentes | Rol |
|---|---|---|
| **Nodo T3S3** | LilyGo T3 S3 (ESP32-S3 + SX1262) + GPS L76K integrado | Transmite y recibe según el schedule |
| **Gateway** | Raspberry Pi CM4 + HAT concentrador SX1303 con radios SX1250 + GPS L76K | Transmite y recibe (roles invertidos al T3S3) |

Ambos dispositivos necesitan antena GPS con vista al cielo (o ventana) y antena LoRa. Separar antenas LoRa al menos 1 metro.

---

## 3. Parámetros LoRa fijos

| Parámetro | T3S3 | Gateway |
|---|---|---|
| Bandwidth | 125 kHz | 125 kHz |
| Coding Rate | 4/6 | 4/6 |
| Sync Word | 0x34 | 0x34 (vía `lorawan_public=true`) |
| Header | Explícito | Explícito |
| CRC | ON | ON |
| Preámbulo | 8 símbolos | 16 símbolos |
| TX Power | −9 dBm | 14 dBm |
| Payload | 38 bytes fijos | 38 bytes fijos |
| Paquetes/ventana | 10 | 10 |
| Gap inter-paquete | 40 ms | 40 ms |
| LDRO | Explícito (`forceLDRO`) | Automático (el SX1302 lo activa solo en SF≥11) |

> **Nota sobre IQ:** El gateway TX usa IQ invertida (`invert_pol=true`) y su RX es siempre IQ normal (fijo en el HAL). Por lo tanto, el T3S3 debe usar IQ normal al transmitir e IQ invertida al recibir.

---

## 4. Tabla de schedule (idéntica en ambos dispositivos)

El ciclo dura **104 400 ms** (~1 min 44 s) y se repite indefinidamente. Los offsets son absolutos desde el inicio del ciclo (fase 0).

```
Slot  Offset(ms)  Dur(ms)  SF   Freq(MHz)  LDRO  T3S3    GW      Tipo
───── ────────── ──────── ──── ────────── ────── ─────── ─────── ──────────
  0        0      35000    12   902.0      ON    TX      RX      Activo
  1    35000        800     —     —         —    IDLE    IDLE    Guarda rol
  2    35800      35000    12   902.0      ON    RX      TX      Activo
  3    70800       2000     —     —         —    IDLE    IDLE    Guarda ciclo
  4    72800      10000    10   902.6      OFF   TX      RX      Activo
  5    82800        800     —     —         —    IDLE    IDLE    Guarda rol
  6    83600      10000    10   902.6      OFF   RX      TX      Activo
  7    93600       2000     —     —         —    IDLE    IDLE    Guarda ciclo
  8    95600       4000     8   903.2      OFF   TX      RX      Activo
  9    99600        800     —     —         —    IDLE    IDLE    Guarda rol
 10   100400       4000     8   903.2      OFF   RX      TX      Activo
```

Cada par de ventanas activas prueba un SF distinto a una frecuencia distinta:

- **Par 1 (slots 0–2):** SF12 a 902.0 MHz, ventanas de 35 s, LDRO ON
- **Par 2 (slots 4–6):** SF10 a 902.6 MHz, ventanas de 10 s, LDRO OFF
- **Par 3 (slots 8–10):** SF8 a 903.2 MHz, ventanas de 4 s, LDRO OFF

Las guardas de 800 ms separan el cambio de rol TX↔RX dentro del mismo par. Las guardas de 2000 ms separan pares con distinto SF/frecuencia.

---

## 5. Cómo funciona la sincronización (join point)

Ambos dispositivos ejecutan este mismo algoritmo de forma independiente:

1. **Obtener fix GPS** con hora UTC válida (status 'A' o 'D' en sentencia RMC).
2. **Calcular la fase** dentro del ciclo: `phase_ms = UTC_ms % 104400`. Esto dice en qué punto del ciclo de 104.4 s estamos ahora mismo.
3. **Buscar el próximo slot activo** (no guarda) cuyo offset sea mayor que la fase. Ese es el "join point". Si no queda ninguno en el ciclo actual, el join point es el slot 0 del próximo ciclo.
4. **Esperar** hasta el instante exacto en que arranca ese slot.
5. **Fijar el ancla t0:** se calcula hacia atrás el instante hipotético de fase 0 (`t0 = ahora − join_axis_ms`). A partir de aquí, todo el timing se mide contra este ancla.

Como ambos dispositivos usan la misma tabla y la misma hora UTC, calculan el mismo join point sin comunicarse. No importa en qué momento se encienda cada uno.

### Resincronización del reloj (solo T3S3)

El cristal del ESP32-S3 puede derivar con el tiempo. Al final de cada vuelta, el T3S3 lee una posición GPS fresca y corrige `t0` si el drift es menor a 5 s. Si el drift supera los 5 s, lo ignora (probablemente un fix malo).

### GPS en el gateway

El gateway usa el GPS solo para la sincronización inicial. Después de calcular el join point, cierra el puerto serie del GPS y todo el timing se basa en `CLOCK_MONOTONIC` del kernel Linux, que es suficientemente preciso.

---

## 6. Arquitectura del gateway (dual radio fijo)

A diferencia de un diseño donde se retunea el radio entre ciclos, este gateway configura **ambos radios SX1250 una sola vez al arrancar** y nunca los toca durante la ejecución:

| Radio | Frecuencia central (LO) | Canales multi-SF |
|---|---|---|
| Radio 0 | 902.3 MHz | 902.0 / 902.2 / 902.4 / 902.6 MHz |
| Radio 1 | 903.1 MHz | 902.8 / 903.0 / 903.2 / 903.4 MHz |

Los 8 canales multi-SF escuchan siempre en simultáneo y auto-detectan cualquier SF (5–12). No hay `lgw_stop`/`lgw_start` entre ventanas — el concentrador arranca una vez y queda corriendo.

Para transmitir, el gateway elige dinámicamente el radio cuyo LO está más cerca de la frecuencia del slot (902.0 y 902.6 MHz salen por radio 0; 903.2 MHz sale por radio 1).

---

## 7. Formato del paquete (38 bytes)

```
Byte(s)  Campo                              Descripción
──────── ──────────────────────────────────  ──────────────────────────────
  0      SF                                 Spreading factor (8, 10, 12)
  1–2    freq_field (uint16_t, LE)          freq_kHz − 900000
  3      Índice del paquete                 0–9
  4      Total de paquetes                  Siempre 10
  5–8    Timestamp relativo (uint32_t, LE)  ms desde inicio de la ventana
  9      Origen                             0xAA = T3S3, 0xBB = Gateway
 10–37   Padding                            0x55 (28 bytes)
```

Para reconstruir la frecuencia: `freq_MHz = (freq_field + 900000) / 1000.0`

---

## 8. Convención IQ (crítico)

| Dirección | Transmisor | IQ del TX | Receptor | IQ del RX |
|---|---|---|---|---|
| T3S3 → Gateway | T3S3 | Normal | Gateway | Normal (fijo en HAL) |
| Gateway → T3S3 | Gateway | Invertida | T3S3 | Invertida |

Si esta convención no se respeta, no se recibe nada o se obtienen errores CRC en cada paquete.

---

## 9. Ejecución del loop principal

Después del join point, ambos dispositivos entran en un loop infinito de "vueltas" (laps):

1. La primera vuelta puede ser parcial (arranca desde el slot del join point, no necesariamente el slot 0).
2. Para cada slot, se calcula su instante absoluto: `slot_start = lap × 104400 + offset_ms`.
3. Los slots de guarda son pausas donde se reconfigura el radio si el próximo slot activo usa distinto SF/frecuencia (solo en el T3S3 — el gateway no necesita reconfigurar).
4. Los slots activos ejecutan la ventana TX o RX correspondiente.
5. Al terminar el último slot (10), el T3S3 reconfigura el radio de vuelta a los parámetros del slot 0 (SF12, 902.0 MHz, LDRO ON) y hace resync GPS.
6. La siguiente vuelta arranca limpia desde el slot 0.

---

## 10. Procedimiento para ejecutar la prueba

### Preparación

1. Verificar que ambos GPS tienen vista al cielo y obtienen fix (el T3S3 imprime heartbeats cada 2 s mientras espera fix; el gateway imprime status de la sentencia RMC).
2. Conectar las antenas LoRa, separadas al menos 1 m.
3. Los dispositivos NO necesitan encenderse al mismo tiempo.

### En el gateway (RPi CM4)

1. Compilar: `make` (requiere que el HAL `sx1302_hal` esté compilado previamente).
2. Ejecutar:
   ```bash
   ./test_sync_windows 2>&1 | tee gw_log.txt
   ```
   Si el GPS no está en `/dev/serial0`, pasar el puerto como argumento.
3. Esperar a que aparezca `[SYNC] union alcanzada`.
4. Para detener: Ctrl+C (hace cleanup limpio y muestra resumen final).

### En el T3S3

1. Compilar y flashear con PlatformIO. El firmware usa el framework del repo LilyGo (LoRaBoards.h).
2. Abrir monitor serial a 115200 baud.
3. Esperar a que aparezca `[SYNC] JOIN alcanzado`.
4. A partir de ahí, los logs muestran cada ventana TX/RX con sus resultados.

### Observar la convergencia

Ambos dispositivos deben reportar que se unieron al **mismo slot** del schedule (pueden ser vueltas distintas si se encendieron en momentos diferentes, pero el slot y su SF/frecuencia deben coincidir). La primera evidencia de éxito es que las ventanas RX de un lado reciben los paquetes TX del otro.

---

## 11. Interpretación de los logs

### Tags principales

| Tag | Significado |
|---|---|
| `GPS-FIX` | Esperando/obtenido fix GPS |
| `SYNC` | Cálculo del join point, resincronización |
| `COUNTDOWN` | Cuenta regresiva al join point |
| `TX-START` / `TX` / `TX-END` | Ventana de transmisión |
| `RX-START` / `RX` / `RX-END` | Ventana de recepción |
| `RECONFIG` | Reconfiguración de SF/freq/LDRO (T3S3) |
| `LAP` | Inicio de nueva vuelta |
| `REPORT` | Resumen acumulado (gateway, al final de cada vuelta) |
| `TX-WARN` | Ventana TX agotada antes de enviar los 10 paquetes |

### Métricas clave por ventana RX

- **RSSI y SNR:** calidad de señal
- **PER (Packet Error Rate):** porcentaje de paquetes perdidos. Se calcula como `(10 − recibidos) / 10 × 100%`
- **freqErr:** error de frecuencia reportado por el SX1262 (solo T3S3)
- **CRC bad:** paquetes recibidos pero con CRC inválido

---

## 12. Criterios de éxito

| # | Criterio | PASS si... |
|---|---|---|
| C1 | Convergencia al mismo slot | Primer paquete RX dentro de la ventana esperada |
| C2 | Sin desborde de ventanas TX | 0 warnings `TX-WARN` |
| C3 | TX sin errores de radio | 60/60 paquetes TX exitosos por vuelta |
| C4 | Recepción ≥80% por ventana | ≥ 8/10 paquetes recibidos en cada ventana RX |
| C5 | Orden correcto | SF12 → SF10 → SF8 en cada vuelta |
| C6 | IQ correcta en ambas direcciones | T3S3 recibe del GW y viceversa |
| C7 | Resync GPS funcional | Drift reportado < 5 s entre vueltas |

---

## 13. Posibles problemas

| Síntoma | Causa probable | Solución |
|---|---|---|
| No se recibe nada en ninguna dirección | IQ invertida mal configurada | Verificar convención de la sección 8 |
| Gateway recibe 0 paquetes | Sync word diferente, o frecuencia mal | Verificar `0x34` y que la tabla sea idéntica |
| T3S3 no arranca | GPS sin fix | Mover antena GPS a ventana/exterior |
| `chars=0` en heartbeat del T3S3 | GPS no conectado o baud incorrecto | Verificar cableado y que se use `GPS_BAUD_RATE` |
| Solo CRC bad, nunca CRC ok | LDRO no coincide (SF12) | T3S3 debe usar `forceLDRO(true)` en SF12 |
| Dispositivos no convergen | Tabla de schedule difiere | Verificar offsets y `SCHEDULE_CYCLE_MS = 104400` |
| PER alto en una sola dirección | Potencia TX insuficiente o interferencia | Ajustar potencia o cambiar frecuencia |
| Drift acumulado entre vueltas | Cristal ESP32-S3 sin TCXO | El resync GPS al final de cada vuelta lo corrige |

---

## 14. Archivos asociados

Junto a esta guía deben estar:

1. `test_sync_windows.cpp` — firmware del T3S3 (PlatformIO/Arduino)
2. `test_sync_windows.c` — programa del gateway (C, compilar con el HAL sx1302)
3. `reset_lgw.sh` — script de reset GPIO del SX1302 (del directorio `libloragw` del HAL)


## 15. Resultados 


Los resultados fueron exitosos, ver los logs.