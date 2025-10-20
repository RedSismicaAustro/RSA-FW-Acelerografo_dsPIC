# RSA-FW-Acelerografo_dsPIC

Sistema de adquisición de datos sísmicos basado en microcontrolador **dsPIC33EP256MC202** para la Red Sísmica del Austro (RSA).

> **⚠️ PROYECTO LEGACY:** Este repositorio contiene el firmware original del acelerógrafo RSA, donde un microcontrolador dsPIC actúa como intermediario entre el acelerómetro ADXL355 y una Raspberry Pi. Incluye los scripts en C necesarios para la comunicación entre ambos dispositivos.
El software complementario para la gestión de archivos y comunicaciones, que se ejecuta en la Raspberry Pi, se ha trasladado al proyecto [RSA-Acelerografo](https://github.com/Red-Sismica-del-Austro/RSA-Acelerografo).

## Descripción

El sistema adquiere datos de aceleración triaxial (X, Y, Z) desde un acelerómetro ADXL355 a 250 Hz, sincronizados con GPS/RTC, y los transmite a una Raspberry Pi vía comunicación SPI en tramas binarias de 2506 bytes por segundo.

### Componentes Hardware

- **Microcontrolador:** dsPIC33EP256MC202 @ 80 MHz
- **Acelerómetro:** ADXL355 (resolución 20 bits, 3 ejes)
- **Reloj en Tiempo Real:** DS3234 (RTC con cristal de alta precisión)
- **Módulo GPS:** Sincronización temporal mediante señal PPS (Pulse Per Second)
- **Comunicación:** SPI esclavo hacia Raspberry Pi

## Arquitectura del Sistema

```
┌─────────────┐         SPI          ┌──────────────┐
│   ADXL355   │────────────────────→ │   dsPIC33    │
│ Acelerómetro│                      │  EP256MC202  │
└─────────────┘                      │              │
                                     │  - Timer1    │
┌─────────────┐         SPI          │  - UART GPS  │         SPI
│   DS3234    │────────────────────→ │  - SPI Slave │────────────────→ Raspberry Pi
│     RTC     │────────────────────→ │              │     (2506 bytes/s)
└─────────────┘      SQW (1Hz)       └──────────────┘
                                           ↑ ↑
┌─────────────┐         UART               │ │
│    Módulo   │────────────────────────────┘ │
│     GPS     │──────────────────────────────┘
└─────────────┘      PPS (1Hz)
          
```

## Formato de Datos Binarios

Cada trama contiene **2506 bytes** correspondientes a 1 segundo de datos:

### Estructura de la Trama

**Bytes 0-2499:** Datos de aceleración (250 muestras)
- Cada muestra: 10 bytes = `[#muestra (1B)] + [X (3B)] + [Y (3B)] + [Z (3B)]`
- Numeración de muestras: 0 a 249
- Cada eje: entero con signo de 20 bits almacenado en 3 bytes

**Bytes 2500-2505:** Marca de tiempo
- `[año, mes, día, hora, minuto, segundo]`

### Conversión de Datos

Algoritmo para convertir los 3 bytes de cada eje a aceleración en gales (cm/s²):

```python
import numpy as np

# Leer 3 bytes de un eje (ejemplo: X)
datosX = [byte1, byte2, byte3]

# Extraer valor de 20 bits
xValue = ((datosX[0] << 12) & 0xFF000) + \
         ((datosX[1] << 4) & 0xFF0) + \
         ((datosX[2] >> 4) & 0xF)

# Aplicar complemento a 2 si es negativo
if xValue >= 0x80000:
    xValue = xValue & 0x7FFFF  # Descartar bit de signo
    xValue = -1 * (((~xValue) + 1) & 0x7FFFF)

# Convertir a gales
aceleracion_gals = xValue * (980 / 2**18)
```

Ver [Documentacion/Protocolo/Formato de datos de registro continuo.txt](Documentacion/Protocolo/Formato%20de%20datos%20de%20registro%20continuo.txt) para la especificación completa.

## Protocolo de Comunicación SPI

El dsPIC actúa como **esclavo SPI** y la Raspberry Pi como **maestro**. Las operaciones se estructuran como:

```
0xA[N] (inicio) → [transferencia de datos] → 0xF[N] (fin)
```

### Comandos Principales

| Comando | Descripción |
|---------|-------------|
| `0xA0-0xF0` | Solicitar tipo de operación del dsPIC |
| `0xA1-0xF1` | Iniciar/detener muestreo |
| `0xA2-0xF2` | Inicializar GPS |
| `0xA3-0xF3` | Leer trama de datos (2506 bytes) |
| `0xA4-0xF4` | Enviar hora desde RPi al dsPIC (sincronizar RTC) |
| `0xA5-0xF5` | Solicitar hora local del dsPIC |
| `0xA6-0xF6` | Solicitar fuente de tiempo (GPS o RTC) |

### Pines de Interrupción

- **P1 (pin A4):** El dsPIC genera interrupción en RPi cuando hay datos listos
- **P2 (pin B4):** Reservado para uso futuro

## Estructura del Repositorio

```
RSA-FW-Acelerografo_dsPIC/
├── Firmware/
│   ├── Acelerografo/              # Firmware principal del dsPIC
│   │   ├── Acelerografo.c         # Programa principal
│   │   ├── Acelerografo.hex       # Firmware compilado
│   │   └── ...                    # Archivos de proyecto mikroC
│   ├── Librerias firmware/        # Librerías del dsPIC
│   │   ├── ADXL355_SPI.c/h        # Driver del acelerómetro
│   │   ├── TIEMPO_GPS.c/h         # Procesamiento de tramas GPS
│   │   ├── TIEMPO_RPI.c/h         # Interfaz de tiempo con RPi
│   │   └── TIEMPO_RTC.c/h         # Driver del RTC DS3234
│   └── Datos pruebas/             # Datos de pruebas de laboratorio
│
├── Software/
│   ├── RPi/                       # Software para Raspberry Pi (legacy)
│   │   ├── C/                     # Programas en C
│   │   │   ├── Control/           # Control de adquisición
│   │   │   ├── Conversion/        # Conversores binario→MiniSEED
│   │   │   └── Pruebas/           # Programas de prueba
│   │   ├── Python/                # Utilidades en Python
│   │   │   ├── BinarioToMiniSeed_V12.py       # Conversor a MiniSEED
│   │   │   ├── ComprobarTrama_V2.1.py         # Validación de tramas
│   │   │   ├── SubirArchivoDrive.py           # Subida a Google Drive
│   │   │   ├── GraficarEventoBinario.py       # Visualización de eventos
│   │   │   └── LimpiarArchivosRegistro.py     # Limpieza de logs
│   │   └── Scripts/               # Scripts shell y configuraciones crontab
│   │
│   └── W10/                       # Utilidades para Windows 10
│       ├── Convertidor de formato/    # Conversor y extractor de eventos
│       ├── Visualizador de eventos/   # GUI para visualizar eventos
│       ├── API Google Drive/          # Scripts de subida a Drive
│       ├── Bot Telegram/              # Bot de notificaciones (experimental)
│       └── Extractor de eventos/      # Extracción de ventanas de eventos
│
├── Documentacion/
│   ├── Protocolo/                 # Especificación del protocolo de datos
│   ├── Esquema/                   # Diagramas esquemáticos (PDFs)
│   └── Software/                  # Guías de instalación
│
├── CLAUDE.md                      # Guía para Claude Code
└── README.md                      # Este archivo
```

## Desarrollo del Firmware

### Entorno de Desarrollo

- **IDE:** mikroC PRO for dsPIC (propietario, solo Windows)
- **Compilador:** Compilador integrado de mikroC
- **Programador:** PICkit3, ICD3, u otro compatible con dsPIC33EP

### Compilación

1. Abrir el proyecto `Firmware/Acelerografo/Acelerografo.mcpds` en mikroC PRO
2. Compilar el proyecto (Build → Build)
3. El archivo `.hex` se genera automáticamente en el mismo directorio

### Programación del Microcontrolador

Usar el programador PICkit3 desde mikroC PRO o MPLAB IPE para cargar el archivo `.hex` al dsPIC.

### Componentes Clave del Firmware

**Programa principal:** [Firmware/Acelerografo/Acelerografo.c](Firmware/Acelerografo/Acelerografo.c)
- Inicializa periféricos (SPI1 esclavo, SPI2 maestro, UART1, GPIO, Timers)
- Maneja interrupciones SPI para comunicación con RPi
- Gestiona ciclos de muestreo a 250 Hz
- Sincroniza con señal PPS del GPS o SQW del RTC (1 Hz)

**Librerías:**
- `ADXL355_SPI.h`: Lectura del FIFO, configuración de tasa de muestreo, control de energía
- `TIEMPO_GPS.h`: Parseo de sentencias GPRMC, extracción de fecha/hora
- `TIEMPO_RPI.h`: Conversión de formato de tiempo desde RPi
- `TIEMPO_RTC.h`: Interfaz SPI con DS3234, establecer/leer tiempo

## Software para Raspberry Pi (Legacy)

> **Nota:** Estos scripts son versiones antiguas. Las versiones actualizadas están en el proyecto [RSA-Acelerografo](https://github.com/Red-Sismica-del-Austro/RSA-Acelerografo).

### Instalación de Librerías

```bash
# Librería bcm2835 (comunicación SPI)
wget http://www.airspayce.com/mikem/bcm2835/bcm2835-1.58.tar.gz
tar zxvf bcm2835-1.58.tar.gz
cd bcm2835-1.58
./configure
make
sudo make install

# Librería WiringPi (control GPIO)
sudo apt-get install wiringpi
```

Ver [Documentacion/Software/Instalacion librerias.txt](Documentacion/Software/Instalacion%20librerias.txt) para más detalles.

### Configuración de Tareas Cron

Ejemplo de configuración en [Software/RPi/Scripts/crontab.txt](Software/RPi/Scripts/crontab.txt):

```cron
# Reiniciar registro a medianoche
59 23 * * * /usr/local/bin/registrocontinuo stop
0 0 * * * /usr/local/bin/registrocontinuo start

# Al arrancar: resetear dsPIC, iniciar GPS, comenzar registro
@reboot sleep 30 && /usr/local/bin/resetmaster
@reboot sleep 60 && /usr/local/bin/iniciargps
@reboot sleep 90 && /usr/local/bin/registrocontinuo start
```

### Conversión a Formato MiniSEED

El script [Software/RPi/Python/BinarioToMiniSeed_V12.py](Software/RPi/Python/BinarioToMiniSeed_V12.py) convierte archivos binarios al estándar MiniSEED usando la librería ObsPy.

Características:
- Manejo de muestras faltantes (gaps)
- Validación de marcas de tiempo
- Compresión STEIM1

## Utilidades para Windows 10

Localizadas en [Software/W10/](Software/W10/):

- **Convertidor de formato:** Conversor binario a MiniSEED con GUI para extracción de eventos
- **Visualizador de eventos:** Interfaz gráfica para visualizar eventos registrados
- **API Google Drive:** Scripts para subida automática con archivos de configuración
- **Bot Telegram:** Integración experimental con bot de Telegram
- **Extractor de eventos:** Programas en C para extraer ventanas de tiempo con eventos

## Migración al Sistema Actual

Este sistema basado en dsPIC ha sido **reemplazado** por [RSA-Acelerografo](../RSA-Acelerografo/) que:

✅ Elimina el microcontrolador dsPIC
✅ Conecta el ADXL355 directamente a la Raspberry Pi vía SPI
✅ Usa programas en C compilados con GCC en RPi (no mikroC)
✅ Implementa el mismo formato binario para compatibilidad
✅ Añade telemetría MQTT y gestión mejorada de archivos
✅ Simplifica el hardware y reduce costos

### ¿Cuándo usar este repositorio?

Este repositorio se mantiene para:
- **Referencia del formato de datos binarios** (estándar establecido)
- **Depuración de estaciones legacy** que aún usan dsPIC
- **Documentación de hardware** y esquemas eléctricos
- **Historial del desarrollo** del sistema RSA

**Para desarrollo activo de acelerógrafos, usar [RSA-Acelerografo](https://github.com/Red-Sismica-del-Austro/RSA-Acelerografo).**

## Fuentes de Reloj

El sistema soporta múltiples fuentes de tiempo con prioridad:

| Código | Fuente | Descripción |
|--------|--------|-------------|
| `0` | RPi | Tiempo del sistema de la Raspberry Pi |
| `1` | GPS | GPS con fix válido (mayor precisión) |
| `2` | RTC | Reloj DS3234 (backup si no hay GPS) |
| `3` | GPS (E3) | GPS sin fix válido |
| `4` | RTC (E4) | RTC por error en cabecera GPS |
| `5` | RTC (E5) | RTC por timeout de GPS |

## Documentación

- **Protocolo de datos:** [Documentacion/Protocolo/](Documentacion/Protocolo/)
- **Esquemas de hardware:** [Documentacion/Esquema/](Documentacion/Esquema/) (archivos PDF)
- **Instalación de software:** [Documentacion/Software/](Documentacion/Software/)
- **Guía para Claude Code:** [CLAUDE.md](CLAUDE.md)

## Historial del Proyecto

**Autor:** Milton Muñoz
**Email:** miltonrodrigomunoz@gmail.com
**Fecha de creación:** 14 de marzo de 2019
**Última actualización importante:** 20 de septiembre de 2023

### Hitos Principales

- **Marzo 2019:** Implementación inicial con dsPIC33EP256MC202
- **2019-2023:** Correcciones de sincronización GPS/RTC, mejoras en conversor MiniSEED
- **2024+:** Migración al sistema RSA-Acelerografo (interface directa RPi-ADXL355)

## Licencia

Este proyecto fue desarrollado para la **Red Sísmica del Austro (RSA)** de la Universidad de Cuenca, Ecuador.

---

**Red Sísmica del Austro (RSA)**
Universidad de Cuenca
Cuenca, Ecuador
