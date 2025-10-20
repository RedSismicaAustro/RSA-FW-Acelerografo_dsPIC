# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Legacy firmware and software** for the RSA (Red Sísmica del Austro) accelerograph system based on **dsPIC33EP256MC202** microcontroller. This is the predecessor to the current [RSA-Acelerografo](../RSA-Acelerografo/) project which runs directly on Raspberry Pi.

The system acquires 3-axis acceleration data from an ADXL355 accelerometer at 250 Hz, synchronized with GPS/RTC, and transmits binary data frames to a Raspberry Pi via SPI communication.

**Status:** This is a legacy project. Active development has moved to RSA-Acelerografo which eliminates the dsPIC and interfaces the ADXL355 directly to the Raspberry Pi.

## Architecture

### Hardware Components
- **Microcontroller:** dsPIC33EP256MC202 @ 80 MHz
- **Accelerometer:** ADXL355 (20-bit resolution, 3 axes)
- **RTC:** DS3234 (Real-Time Clock)
- **GPS Module:** For time synchronization via PPS (Pulse Per Second) signal
- **Communication:** SPI slave to Raspberry Pi

### Communication Protocol

**SPI Communication (dsPIC as slave, RPi as master)**

Command structure: `0xA[N]` (start) → data transfer → `0xF[N]` (end)

Key operations:
- `0xA0-0xF0`: Request operation type from dsPIC
- `0xA1-0xF1`: Start/stop sampling
- `0xA2-0xF2`: Initialize GPS
- `0xA3-0xF3`: Read accelerometer data frame (2506 bytes)
- `0xA4-0xF4`: Send time from RPi to dsPIC (set RTC)
- `0xA5-0xF5`: Request local time from dsPIC
- `0xA6-0xF6`: Request time reference (GPS or RTC)

**Interrupt pins:**
- `P1` (pin A4): dsPIC triggers interrupt on RPi to signal data ready or events
- `P2` (pin B4): Reserved for future use

### Binary Data Format

**Frame structure:** 2506 bytes per second
- Bytes 0-2499: Acceleration data (250 samples × 10 bytes per sample)
  - Sample format: `[sample_number (1B)] [X (3B)] [Y (3B)] [Z (3B)]`
  - Each axis: 20-bit signed integer in 24-bit container
- Bytes 2500-2505: Timestamp `[year, month, day, hour, minute, second]`

**Sample numbering:** 0-249 (one per sample at 250 Hz)

**Acceleration conversion:**
```python
# Extract 20-bit value from 3 bytes
xValue = ((datosX[0] << 12) & 0xFF000) + ((datosX[1] << 4) & 0xFF0) + ((datosX[2] >> 4) & 0xF)

# Two's complement for negative values
if xValue >= 0x80000:
    xValue = xValue & 0x7FFFF
    xValue = -1 * (((~xValue) + 1) & 0x7FFFF)

# Convert to gals (cm/s²)
aceleracion_gals = xValue * (980 / 2**18)
```

See [Documentacion/Protocolo/Formato de datos de registro continuo.txt](Documentacion/Protocolo/Formato de datos de registro continuo.txt) for complete format specification.

## Directory Structure

```
RSA-FW-Acelerografo_dsPIC/
├── Firmware/
│   ├── Acelerografo/           # Main dsPIC firmware (mikroC PRO)
│   │   ├── Acelerografo.c      # Main program
│   │   ├── Acelerografo.hex    # Compiled firmware
│   │   └── Acelerografo.*      # Build artifacts
│   └── Librerias firmware/     # dsPIC libraries
│       ├── ADXL355_SPI.c/h     # ADXL355 accelerometer driver
│       ├── TIEMPO_GPS.c/h      # GPS time parsing
│       ├── TIEMPO_RPI.c/h      # RPi time handling
│       └── TIEMPO_RTC.c/h      # DS3234 RTC driver
├── Software/
│   ├── RPi/                    # Raspberry Pi software (legacy)
│   │   ├── C/                  # C programs for RPi
│   │   │   ├── Control/        # Recording control programs
│   │   │   ├── Conversion/     # Binary to MiniSEED converters
│   │   │   └── Pruebas/        # Test programs
│   │   ├── Python/             # Python utilities
│   │   │   ├── BinarioToMiniSeed*.py      # Binary to MiniSEED converters
│   │   │   ├── ComprobarTrama*.py         # Frame validation
│   │   │   ├── SubirArchivoDrive.py       # Google Drive upload
│   │   │   ├── GraficarEventoBinario.py   # Event visualization
│   │   │   └── LimpiarArchivosRegistro.py # Log file cleanup
│   │   └── Scripts/            # Shell scripts and crontab configs
│   └── W10/                    # Windows 10 utilities
│       ├── Convertidor de formato/  # Format converters
│       ├── Visualizador de eventos/ # Event viewer GUI
│       └── API Google Drive/        # Drive upload scripts
├── Documentacion/
│   ├── Protocolo/              # Communication protocol docs
│   ├── Esquema/                # Hardware schematics (PDFs)
│   └── Software/               # Installation guides
└── README.md                   # (Empty - documentation in CLAUDE.md)
```

## Firmware Development

### Build Environment
- **IDE:** mikroC PRO for dsPIC (proprietary, Windows-only)
- **Compiler:** Built-in mikroC compiler
- **Output:** [Firmware/Acelerografo/Acelerografo.hex](Firmware/Acelerografo/Acelerografo.hex)

### Key Firmware Components

**Main program:** [Firmware/Acelerografo/Acelerografo.c](Firmware/Acelerografo/Acelerografo.c)
- Initializes peripherals (SPI1 slave, SPI2 master, UART1, GPIO, Timers)
- Handles SPI interrupts for RPi communication
- Manages sampling cycles at 250 Hz
- Synchronizes with GPS PPS or RTC SQW (1 Hz square wave)

**Libraries:**
- `ADXL355_SPI.h`: Read FIFO, configure sampling rate, power control
- `TIEMPO_GPS.h`: Parse GPRMC sentences, extract time/date
- `TIEMPO_RPI.h`: Convert RPi timestamp format
- `TIEMPO_RTC.h`: DS3234 SPI interface, set/get time

**Sampling Strategy:**
- ADXL355 stores samples in 96-sample FIFO
- Timer1 interrupts every 100ms to read FIFO (25 samples × 3 axes)
- After 10 Timer1 cycles (1 second), sends complete frame to RPi

**Clock Sources:**
- `fuenteReloj = 0`: RPi system time
- `fuenteReloj = 1`: GPS (valid fix)
- `fuenteReloj = 2`: RTC
- `fuenteReloj = 3`: GPS (error E3 - invalid fix)
- `fuenteReloj = 4`: RTC (error E4 - GPS header mismatch)
- `fuenteReloj = 5`: RTC (error E5 - GPS timeout)

## Raspberry Pi Software (Legacy)

### Setup Requirements

**Libraries:**
```bash
# bcm2835 library (for SPI communication)
wget http://www.airspayce.com/mikem/bcm2835/bcm2835-1.58.tar.gz
tar zxvf bcm2835-1.58.tar.gz
cd bcm2835-1.58
./configure && make && sudo make install

# WiringPi library (for GPIO)
sudo apt-get install wiringpi
```

### Cron Jobs

From [Software/RPi/Scripts/crontab.txt](Software/RPi/Scripts/crontab.txt):
```cron
# Restart recording at midnight
59 23 * * * /usr/local/bin/registrocontinuo stop
0 0 * * * /usr/local/bin/registrocontinuo start

# On boot: reset dsPIC, init GPS, start recording
@reboot sleep 30 && /usr/local/bin/resetmaster
@reboot sleep 60 && /usr/local/bin/iniciargps
@reboot sleep 90 && /usr/local/bin/registrocontinuo start
```

### Python Utilities

**Binary to MiniSEED Conversion:**
- [Software/RPi/Python/BinarioToMiniSeed_V12.py](Software/RPi/Python/BinarioToMiniSeed_V12.py) - Latest version
- Handles missing samples (gaps), invalid timestamps
- Uses ObsPy library for MiniSEED generation

**Frame Validation:**
- [Software/RPi/Python/ComprobarTrama_V2.1.py](Software/RPi/Python/ComprobarTrama_V2.1.py) - Validates binary frame structure

**Data Management:**
- [Software/RPi/Python/SubirArchivoDrive.py](Software/RPi/Python/SubirArchivoDrive.py) - Upload to Google Drive
- [Software/RPi/Python/LimpiarArchivosRegistro.py](Software/RPi/Python/LimpiarArchivosRegistro.py) - Clean old logs

**Visualization:**
- [Software/RPi/Python/GraficarEventoBinario.py](Software/RPi/Python/GraficarEventoBinario.py) - Plot events from binary
- [Software/RPi/Python/GraficarTiempoReal.py](Software/RPi/Python/GraficarTiempoReal.py) - Real-time plotting

## Windows 10 Utilities

Located in [Software/W10/](Software/W10/):
- **Convertidor de formato:** Binary to MiniSEED converter, event extraction GUI
- **Visualizador de eventos:** GUI for visualizing recorded events
- **API Google Drive:** Upload scripts with configuration files
- **Bot Telegram:** Telegram bot integration (experimental)
- **Extractor de eventos:** C programs to extract event windows from continuous data

## Migration to Current System

This legacy dsPIC-based system has been **replaced** by [RSA-Acelerografo](../RSA-Acelerografo/) which:
- Eliminates the dsPIC microcontroller
- Connects ADXL355 directly to Raspberry Pi via SPI
- Uses C programs compiled with GCC on RPi (not mikroC)
- Implements the same binary format for backward compatibility
- Adds MQTT telemetry and improved file management

**If working on active accelerograph development, use RSA-Acelerografo instead.**

This repository is maintained for:
- Reference for the binary data format
- Debugging legacy deployed stations still using dsPIC
- Hardware documentation and schematics

## Important Notes

- **mikroC PRO is Windows-only** - firmware compilation requires Windows or Wine
- **Binary frame format is standardized** - maintained in RSA-Acelerografo for compatibility
- **Python scripts are outdated** - modern versions are in RSA-Acelerografo repository
- **No automatic deployment** - firmware must be manually programmed via PICkit3/ICD3
- **GPS synchronization is critical** - PPS signal must be connected to INT2 (pin B14)

## Documentation

- Protocol specification: [Documentacion/Protocolo/](Documentacion/Protocolo/)
- Hardware schematics: [Documentacion/Esquema/](Documentacion/Esquema/) (PDF files)
- Software installation: [Documentacion/Software/Instalacion librerias.txt](Documentacion/Software/Instalacion%20librerias.txt)

## Project History

**Author:** Milton Muñoz (miltonrodrigomunoz@gmail.com)
**Created:** March 14, 2019
**Last major update:** September 20, 2023 (commit c619bf9)

Key milestones:
- 2019-03: Initial dsPIC33EP256MC202 implementation
- 2023-09: GPS/RTC synchronization fixes, MiniSEED converter improvements
- 2024+: System migrated to direct RPi interface (RSA-Acelerografo)
