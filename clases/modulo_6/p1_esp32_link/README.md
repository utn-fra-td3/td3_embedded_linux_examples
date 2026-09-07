# Práctica 1 — UART real con `serdev` y protocolo con el ESP32-S3

Driver que reemplaza la búsqueda manual de `of_find_compatible_node` (Módulos 4-5) por el
mecanismo real de binding automático, aplicado a un periférico real: un UART secundario
dedicado de la Raspberry Pi. En vez de exponer un `/dev/ttyAMAx` genérico, el driver se registra
como cliente del bus `serdev` y es dueño exclusivo del puerto, hablando un protocolo de texto
simple (`set`/`get`/`exec`) con una placa ESP32-S3 corriendo FreeRTOS. Expone `/dev/esp32link`
con `read`/`write`/`ioctl`.

## Archivos

| Archivo                     | Descripción                                                        |
|-------------------------------|-----------------------------------------------------------------------|
| `esp32_link.c`               | Driver: bus `serdev`, `kfifo` (productor-consumidor), `ioctl` con `completion` |
| `esp32_link.h`               | Comandos `ioctl` compartidos con `../p2_esp32_ctl/`                |
| `esp32-link-overlay.dts`     | Overlay: describe al ESP32-S3 como nodo hijo del UART              |
| `Makefile`                   | Invoca Kbuild contra las cabeceras del kernel                      |

## Requisitos previos

### Conexionado

- Pi `GPIO4` (TXD3, pin físico 7) → RX del ESP32-S3.
- Pi `GPIO5` (RXD3, pin físico 29) ← TX del ESP32-S3.
- GND común entre las dos placas.
- Ambas trabajan a 3.3V: conexión directa, sin traductor de niveles.

Se eligió **UART3** (`GPIO4`/`GPIO5`) en vez de UART2 (`GPIO0`/`GPIO1`, reservado para la
detección de HATs con EEPROM) o el UART primario (consola serie del sistema).

### Habilitar el UART y aplicar el overlay

```bash
# Habilitar el UART3 (overlay oficial de la Raspberry Pi Foundation)
sudo dtoverlay uart3

# Compilar y aplicar nuestro overlay (agrega el nodo hijo esp32-link)
dtc -@ -I dts -O dtb -o esp32-link.dtbo esp32-link-overlay.dts
sudo dtoverlay -d . esp32-link

# Verificar que el nodo quedo en el arbol en vivo
ls /sys/firmware/devicetree/base/soc*/serial*/esp32-link/ 2>/dev/null || \
    find /sys/firmware/devicetree/base -name esp32-link
```

## El protocolo

Texto plano, una línea por mensaje, terminada en `\n`:

| Verbo | Formato | Respuesta esperada |
|-------|---------|---------------------|
| `set` | `set VARIABLE VALOR\n` | ninguna |
| `get` | `get VARIABLE\n` | `VARIABLE=VALOR\n` |
| `exec` | `exec ACCION\n` | ninguna |

El firmware del ESP32-S3 que implementa el lado que responde **no se provee acá** -- cada
alumno tiene el suyo. Lo único fijo es el formato de arriba, en particular la respuesta a `get`.

## Probar sin un ESP32-S3: una terminal serie en la PC

Como el protocolo es texto plano sobre UART, se puede probar el driver completo (`kfifo`,
`ioctl` con timeout, la CLI) sin tener el firmware del ESP32-S3 listo -- alcanza con hacer de
cuenta que la PC es el ESP32-S3, contestando a mano:

1. Conectar un adaptador USB-serie **de 3.3V** (no de 5V) a la PC.
2. Cablear cruzado contra los mismos pines de la Pi: TX del adaptador → `GPIO5` (RX de la
   Pi), RX del adaptador ← `GPIO4` (TX de la Pi), GND común.
3. Abrir una terminal serie en la PC, por ejemplo:
   ```bash
   picocom -b 115200 /dev/ttyUSB0
   # o: minicom -D /dev/ttyUSB0 -b 115200
   # o: screen /dev/ttyUSB0 115200
   ```
4. Cuando desde la Pi se haga `get temp` (por `ioctl` o por `esp32_ctl`), en la terminal de la
   PC va a aparecer la línea `get temp`. Escribir a mano `temp=234` y Enter -- eso completa el
   `ioctl(ESP32_GET, ...)` que está bloqueado del lado de la Pi.
5. `set`/`exec` se ven aparecer igual en la terminal, sin que haga falta contestar nada.

Esto sirve para validar el driver de forma independiente de tener el firmware FreeRTOS
terminado, y para depurar el protocolo desde los dos lados por separado.

## Compilación, carga y prueba

```bash
# Compilar el modulo
make

# Cargar el modulo (el overlay ya debe estar aplicado)
sudo insmod build/esp32_link.ko

# Verificar que el nodo se creo solo
ls -l /dev/esp32link

# exec y monitoreo crudo con herramientas genericas
echo "exec toggle" | sudo tee /dev/esp32link
cat /dev/esp32link &

# Los comandos ioctl (ver ../p2_esp32_ctl/)
sudo ../p2_esp32_ctl/esp32_ctl

# Ver los mensajes del modulo
dmesg | tail -10

# Descargar el modulo
sudo rmmod esp32_link
```

## Nota

Si el driver `ledbtn` de los Módulos 4-5 sigue cargado, `class_create("td3")` va a fallar acá
porque ese nombre de clase ya está tomado en `sysfs` -- hace falta `sudo rmmod ledbtn` antes de
cargar `esp32_link`, o cambiar `CLASS_NAME` en el código.
