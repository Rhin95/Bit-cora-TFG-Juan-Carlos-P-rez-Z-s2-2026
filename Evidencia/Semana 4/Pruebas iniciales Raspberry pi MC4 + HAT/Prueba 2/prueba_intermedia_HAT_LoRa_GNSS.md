---
created: 2026-08-27T19:11
updated: 2026-08-27T19:26
---
# Prueba Intermedia: Validación del HAT SX1303 + L76K GNSS

**Objetivo:** Confirmar que el Waveshare SX1303 915M LoRaWAN Gateway HAT está correctamente conectado y funcionando, tanto en su parte LoRa (chip SX1303) como en su parte GNSS (módulo L76K), **sin necesidad de dispositivos externos** como el LilyGo T3 S3.

**Prerrequisito:** Prueba 1 (Hola Mundo con la CM4) completada con éxito. Estás conectado por SSH a la CM4.

**Sistema validado en:** Debian GNU/Linux 13 (Trixie), kernel 6.18, Raspberry Pi OS Lite 64-bit.

---

## Parte A: Preparación del hardware

### Paso A.1 — Apagar la CM4 antes de montar el HAT

**Nunca conectes o desconectes el HAT con la CM4 encendida.** Podés dañar el chip.

Desde la sesión SSH:

```bash
sudo shutdown -h now
```

Esperá ~30 segundos y desconectá el USB-C de alimentación.

### Paso A.2 — Montar el HAT sobre la CM4-IO-BASE-A

1. Alineá los 40 pines del conector GPIO del HAT con el header GPIO de la CM4-IO-BASE-A.
2. Presioná firme y de forma pareja hasta que quede completamente asentado (los pines no deben quedar visibles).
3. Si el HAT trae separadores plásticos (standoffs) y tornillos, atornillalos para mayor estabilidad.
4. Conectá una **antena de 915 MHz** al conector U.FL o SMA marcado como "LoRa" en el HAT.
   - ⚠️ **Importante:** nunca energices el HAT sin antena conectada — puede dañar el chip de RF.
5. Opcional pero recomendado: conectá también la antena GPS al conector "GNSS" (necesaria para que el L76K obtenga fix).

### Paso A.3 — Encender y reconectar por SSH

Reconectá la alimentación USB-C. Esperá ~60 segundos y conectate por SSH como antes:

```bash
ssh jcperezz@cm4-jk-gateway.local
```

---

## Parte B: Habilitar interfaces SPI, UART e I2C

El HAT usa **SPI** para comunicarse con el chip SX1303, **UART** para el GNSS L76K, e **I2C** para leer el sensor de temperatura interno (necesario para calibración RF). Las tres interfaces están desactivadas por defecto.

### Paso B.1 — Abrir `raspi-config`

```bash
sudo raspi-config
```

### Paso B.2 — Habilitar SPI

1. Navegá a: **3 Interface Options** → **I4 SPI** → **Yes** (habilitar) → OK
2. Volvés al menú principal.

### Paso B.3 — Habilitar UART (Serial Port)

1. Navegá a: **3 Interface Options** → **I5 Serial Port**
2. Pregunta: *"Would you like a login shell to be accessible over serial?"* → **No**
3. Pregunta: *"Would you like the serial port hardware to be enabled?"* → **Yes**
4. OK.

### Paso B.4 — Habilitar I2C

El SX1303 tiene un sensor de temperatura interno accesible por I2C (dirección `0x39`). Sin I2C habilitado, el receptor LoRa falla al arrancar con `ERROR: failed to open I2C for temperature sensor`.

1. Navegá a: **3 Interface Options** → **I5 I2C** → **Yes** (habilitar) → OK
2. Volvés al menú principal.

### Paso B.5 — Salir y reiniciar

Elegí **Finish** en el menú principal. Si pregunta si querés reiniciar, decí **Yes**. Si no pregunta, reiniciá manualmente:

```bash
sudo reboot
```

Esperá ~60 segundos y reconectate por SSH.

### Paso B.6 — Verificar que SPI está activo

```bash
ls /dev/spi*
```

Deberías ver algo como:

```
/dev/spidev0.0  /dev/spidev0.1
```

Si no aparecen, SPI no se habilitó bien — repetí el paso B.2.

### Paso B.7 — Verificar que UART está disponible

```bash
ls -l /dev/serial*
```

Deberías ver un enlace simbólico como `/dev/serial0` apuntando a `ttyS0` o `ttyAMA0`.

### Paso B.8 — Verificar que I2C está activo y detecta el sensor

```bash
ls /dev/i2c*
```

Deberías ver `/dev/i2c-1`. Después instalá las herramientas de I2C y escaneá el bus:

```bash
sudo apt install -y i2c-tools
sudo i2cdetect -y 1
```

Deberías ver una matriz de direcciones. Buscá específicamente el `39` en la fila `30:`, que es el sensor de temperatura del HAT:

```
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- 39 -- -- -- -- -- --
```

Si `39` no aparece, revisá que el HAT esté bien asentado y que I2C esté habilitado (paso B.4).

---

## Parte C: Instalar y compilar sx1302_hal (fork de Waveshare)

El **sx1302_hal** es la librería oficial de Semtech para operar los chips SX1302/SX1303. **Waveshare mantiene su propio fork** con ajustes específicos para su HAT, disponible en la rama `ws-dev`.

### Paso C.1 — Actualizar el sistema

```bash
sudo apt update
sudo apt upgrade -y
```

Esto puede tardar varios minutos.

### Paso C.2 — Instalar herramientas de compilación

```bash
sudo apt install -y git build-essential
```

### Paso C.3 — Clonar el fork oficial de Waveshare

```bash
cd ~
git clone https://github.com/siuwahzhong/sx1302_hal.git
cd sx1302_hal
```

### Paso C.4 — Cambiar a la rama `ws-dev`

Esta rama contiene los ajustes específicos que Waveshare aplica sobre el HAL de Semtech.

```bash
git checkout ws-dev
```

Deberías ver: `Switched to a new branch 'ws-dev'`.

Confirmá que estás en la rama correcta:

```bash
git branch
```

Debería aparecer un asterisco al lado de `ws-dev`:

```
  master
* ws-dev
```

### Paso C.5 — Compilar

```bash
make clean all
make all
```

La compilación tarda entre 2 y 5 minutos. Al final no debería haber errores rojos.

### Paso C.6 — Copiar el script de reset a las carpetas donde se ejecutan los tests

```bash
cp tools/reset_lgw.sh util_chip_id/
cp tools/reset_lgw.sh packet_forwarder/
cp tools/reset_lgw.sh libloragw/
```

### Paso C.7 — Verificar que se generaron los binarios de prueba

```bash
ls libloragw/
```

Deberías ver varios `test_loragw_*` entre los archivos.

---

## Parte D: Adaptar reset_lgw.sh para Debian Trixie / kernel 6.x

**Este paso es crítico** en versiones modernas de Raspberry Pi OS (Bookworm y Trixie con kernel 6.x). El script `reset_lgw.sh` que trae Waveshare usa el método viejo `/sys/class/gpio/` (sysfs GPIO), **que ya no funciona** en el kernel 6.x — fue reemplazado por `gpiod` / `pinctrl`.

### Paso D.1 — Verificar que `pinctrl` está disponible

`pinctrl` es la herramienta oficial de Raspberry Pi para manipular GPIOs en kernel moderno. Viene preinstalada.

```bash
which pinctrl
```

Debería devolver algo como `/usr/bin/pinctrl`.

### Paso D.2 — Hacer respaldo del script original

```bash
cd ~/sx1302_hal/libloragw
cp reset_lgw.sh reset_lgw.sh.original
```

### Paso D.3 — Crear el reset_lgw.sh adaptado a `pinctrl`

Copiá y pegá este bloque **completo** (incluyendo `EOF` al final) en la terminal:

```bash
cat > reset_lgw.sh << 'EOF'
#!/bin/sh
# reset_lgw.sh adaptado para Raspberry Pi OS Bookworm/Trixie (kernel 6.x)
# Usa pinctrl en lugar del sysfs GPIO obsoleto

SX1302_RESET_PIN=23
SX1302_POWER_EN_PIN=18
SX1261_RESET_PIN=22
AD5338R_RESET_PIN=13

WAIT_GPIO() {
    sleep 0.1
}

init() {
    pinctrl set $SX1302_RESET_PIN op
    pinctrl set $SX1261_RESET_PIN op
    pinctrl set $SX1302_POWER_EN_PIN op
    pinctrl set $AD5338R_RESET_PIN op
}

reset() {
    echo "CoreCell reset through GPIO$SX1302_RESET_PIN..."
    echo "SX1261 reset through GPIO$SX1261_RESET_PIN..."
    echo "CoreCell power enable through GPIO$SX1302_POWER_EN_PIN..."
    echo "CoreCell ADC reset through GPIO$AD5338R_RESET_PIN..."

    pinctrl set $SX1302_POWER_EN_PIN dh; WAIT_GPIO

    pinctrl set $SX1302_RESET_PIN dh; WAIT_GPIO
    pinctrl set $SX1302_RESET_PIN dl; WAIT_GPIO

    pinctrl set $SX1261_RESET_PIN dl; WAIT_GPIO
    pinctrl set $SX1261_RESET_PIN dh; WAIT_GPIO

    pinctrl set $AD5338R_RESET_PIN dl; WAIT_GPIO
    pinctrl set $AD5338R_RESET_PIN dh; WAIT_GPIO
}

term() {
    :  # nada que hacer con pinctrl, los pines quedan como están
}

case "$1" in
    start)
        init
        reset
        ;;
    stop)
        term
        ;;
    *)
        init
        reset
        ;;
esac

exit 0
EOF
chmod +x reset_lgw.sh
```

### Paso D.4 — Copiar también a las otras carpetas de test

```bash
cp reset_lgw.sh ~/sx1302_hal/util_chip_id/
cp reset_lgw.sh ~/sx1302_hal/packet_forwarder/
cp reset_lgw.sh ~/sx1302_hal/tools/
```

---

## Parte E: Prueba D.1 — Comunicación SPI con el SX1303

Este test **no transmite ni recibe LoRa** — solo verifica que la CM4 puede "hablar" con el chip SX1303 por SPI. Si esto falla, ninguna prueba de LoRa va a funcionar después.

### Paso E.1 — Ejecutar test_loragw_reg

```bash
cd ~/sx1302_hal/libloragw
sudo ./test_loragw_reg
```

**Nota:** `sudo` es necesario porque el programa accede directo al hardware SPI/GPIO.

### Paso E.2 — Interpretar el resultado

Si el HAT está bien montado y todo funciona, vas a ver:

```
CoreCell reset through GPIO23...
SX1261 reset through GPIO23...
CoreCell power enable through GPIO18...
CoreCell ADC reset through GPIO13...
Opening SPI communication interface
Note: chip version is 0x12 (v1.2)
## TEST#1: read all registers and check default value for non-read-only registers
------------------
 TEST#1 PASSED
------------------

## TEST#2: read/write test on all non-read-only, non-pulse, non-w0clr, non-w1clr registers
------------------
 TEST#2 PASSED
------------------

Closing SPI communication interface
```

**"chip version is 0x12"** confirma que el SX1303 está siendo detectado. Los dos tests PASSED confirman que la comunicación SPI está limpia.

**Detener con:** `Ctrl+C` si el programa queda esperando.

---

## Parte F: Prueba D.2 — Verificar que el receptor LoRa "escucha"

Ahora que confirmamos que la CM4 puede hablar con el SX1303, vamos a poner el receptor en modo escucha. Como no tenemos transmisor externo, **no vamos a recibir paquetes**, pero podemos verificar que el receptor arranca sin errores y queda a la espera.

### Paso F.1 — Iniciar el receptor en 915 MHz

```bash
cd ~/sx1302_hal/libloragw
sudo ./test_loragw_hal_rx -a 915.0 -b 915.0 -r 1250
```

**Parámetros:**
- `-a 915.0`: frecuencia central del radio A en MHz (banda US915 para Costa Rica).
- `-b 915.0`: frecuencia central del radio B.
- `-r 1250`: tipo de radio (1250 = SX1250, que es el que usa el Waveshare SX1303 HAT).

### Paso F.2 — Interpretar el resultado

Si el receptor arrancó bien, vas a ver algo como:

```
INFO: Configuring SPI
INFO: chip version is 0x12
INFO: Radio A: SX1250 detected
INFO: Radio B: SX1250 detected
INFO: Concentrator started, waiting for packets...
Nb valid packets received: 0 CRC OK, 0 CRC ERROR, 0 NO CRC
```

Y cada pocos segundos va a repetir la línea de conteo. **Esto es lo que queremos ver** — el receptor está funcionando y esperando paquetes. Como no hay transmisor cerca, el conteo se queda en 0.

**Detener con:** `Ctrl+C`.

---

## Parte G: Prueba D.3 — Leer datos del GNSS L76K

El módulo L76K se comunica con la CM4 por **UART a 9600 baud** y envía sentencias NMEA continuamente. Vamos a leerlas directamente para confirmar que el módulo está funcionando.

### Paso G.1 — Lectura rápida con `cat`

La forma más simple de ver que el GPS está funcionando:

```bash
cat /dev/serial0
```

Cada segundo aproximadamente vas a ver texto tipo:

```
$GNRMC,183012.00,A,0954.80471,N,08405.27618,W,2.13,127.94,270826,,,A,V*15
$GNVTG,127.94,T,,M,2.13,N,3.95,K,A*25
$GNGGA,183012.00,0954.80471,N,08405.27618,W,1,05,2.4,975.6,M,11.4,M,,*57
$GPTXT,01,01,01,ANTENNA OK*35
```

**Cómo saber si tiene fix:**
- `$GNRMC,...,A,...` → la **A** (Active) significa que tiene fix (ubicó satélites)
- `$GNRMC,...,V,...` → la **V** (Void) significa sin fix todavía
- `ANTENNA OK` → confirma que la antena GPS está bien conectada

El L76K puede tardar entre 30 segundos y varios minutos en tener fix la primera vez, especialmente en interior — necesita ver cielo.

**Salir de `cat`:** presioná `Ctrl+C`.


### Paso G.3 (opcional) — Usar minicom para una interfaz más ordenada

Si preferís ver los datos en una terminal serial dedicada:

```bash
sudo apt install -y minicom
sudo minicom -D /dev/serial0 -b 9600
```

Salir de minicom: `Ctrl+A`, luego `X`, y confirmá con Enter.

### Paso G.4 (opcional) — Prueba con el programa oficial

sx1302_hal también incluye un test dedicado al GNSS:

```bash
cd ~/sx1302_hal/libloragw
sudo ./test_loragw_gps
```

Esto lee el GPS y muestra información parseada. Salí con `Ctrl+C`.

# Resultado de las pruebas 

1. Prueba test_lora_reg

![](imagenes/Pasted%20image%2020260827191821.png)


2. Prueba  test_loragw_hal_rx

![](imagenes/Pasted%20image%2020260827191958.png)

3. Prueba GNSS
![](imagenes/Pasted%20image%2020260827192539.png)
## ⚠️ Notas importantes: cosas que NO sirven y precauciones

Durante el desarrollo de esta guía encontramos varios detalles que vale la pena documentar para futuras referencias o para reproducir el proyecto en otro entorno.

### 1. El repositorio oficial de Semtech (Lora-net) NO funciona directamente con el HAT de Waveshare

El repo genérico `https://github.com/Lora-net/sx1302_hal.git` compila, pero al ejecutar los tests da:

```
ERROR: failed to reset SX1302, check your reset_lgw.sh script
```

**Motivo:** el script `reset_lgw.sh` del repo de Semtech está pensado para la placa CoreCell de referencia, pero además usa GPIOs que Waveshare respeta. El verdadero problema es otro (ver punto 3), pero conviene igual usar el fork de Waveshare porque tiene ajustes menores adicionales.

**Solución:** usar el fork `siuwahzhong/sx1302_hal`, rama `ws-dev`.

### 2. Los pines GPIO no cambian entre Semtech y Waveshare

Aunque uno esperaría que el fork de Waveshare cambiara los números de pines, **son los mismos** (RESET=23, POWER_EN=18, SX1261_RESET=22, AD5338R_RESET=13). Waveshare diseñó su HAT siguiendo el pinout de referencia del CoreCell de Semtech.

Verificalo con:
```bash
cat ~/sx1302_hal/tools/reset_lgw.sh | head -20
```

### 3. El script original de reset NO funciona en Raspberry Pi OS Bookworm/Trixie (kernel 6.x)

El `reset_lgw.sh` original (tanto de Semtech como de Waveshare) usa el método viejo de sysfs GPIO:

```bash
echo 23 > /sys/class/gpio/export
```

**Este método está deprecado en kernel 6.x** y produce errores como:

```
tee: /sys/class/gpio/export: Invalid argument
./reset_lgw.sh: cannot create /sys/class/gpio/gpio23/direction: Directory nonexistent
```

Verificalo así:
```bash
uname -r        # si empieza con 6.x, sysfs GPIO está deprecado
ls /sys/class/gpio/    # ahora solo aparecen chip0, chip1, gpiochip512, etc.
```

**Solución:** reemplazar el script por una versión que use `pinctrl` (paso D.3 de esta guía). `pinctrl` viene preinstalado en Raspberry Pi OS Bookworm y Trixie.

### 4. `test_loragw_reg` puede pasar aunque el reset falle

Curiosamente, después del boot el SX1303 arranca en un estado válido por sí solo, y los tests de lectura/escritura de registros funcionan **sin necesidad del reset por hardware**. Por eso, aunque el script original de reset arroje errores, `test_loragw_reg` igual puede reportar `TEST#1 PASSED` y `TEST#2 PASSED`.

**No te confíes** — para tests reales de RX/TX (como `test_loragw_hal_rx`), el reset por hardware SÍ es necesario. Si no lo arreglás, esos tests pueden fallar silenciosamente o comportarse errático.

### 5. Nunca desconectar el HAT con la CM4 encendida

Siempre apagá con `sudo shutdown -h now` y desconectá alimentación antes de mover el HAT. El SPI y las líneas GPIO se pueden dañar por conexión en caliente.

### 6. Nunca energizar el HAT sin antena conectada

El chip SX1303 puede dañarse si intenta transmitir sin una carga (antena) apropiada. Conectá la antena de 915 MHz **antes** de encender.

### 7. Costa Rica usa la banda US915

Para las pruebas de LoRa se usan frecuencias en **915 MHz** (`-a 915.0 -b 915.0`), correspondiente a la banda US915 asignada a Región 2 ITU. Nunca uses 868 MHz (EU868) porque:
- Costa Rica no autoriza esa banda para operación abierta
- El HAT que compraste ("SX1303 915M") solo tiene componentes de RF sintonizados para 915 MHz

### 8. El HAT requiere I2C habilitado (no solo SPI)

Aunque la comunicación principal con el SX1303 es por SPI, el chip incluye un **sensor de temperatura interno en I2C (dirección 0x39)** que se usa para calibración RF. Si I2C no está habilitado, `test_loragw_hal_rx` falla al arrancar con:

```
ERROR: failed to open I2C for temperature sensor on port 0x39
ERROR: failed to start the gateway
```

**Detalle importante:** este error NO aparece en `test_loragw_reg` (que solo prueba SPI), así que uno puede pensar erróneamente que el HAT está listo cuando en realidad falta el I2C. Siempre habilitá I2C junto con SPI y UART en el paso B.

### 9. El GNSS puede tardar en obtener fix

Es normal que el L76K muestre `V` (void, sin fix) durante los primeros minutos, especialmente en interior o cerca de ventanas con poco cielo visible. Para acelerar el fix:
- Sacá la antena GPS a una ventana o al exterior
- Esperá al menos 3-5 minutos en el primer arranque (cold start)
- Los siguientes arranques (warm start) son mucho más rápidos si no movés mucho la antena
