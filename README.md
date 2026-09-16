# RemoteSW

RemoteSW is a **remote PC power control system built with the XIAO ESP32-C3 and a PhotoMOS relay**.

It electrically shorts the PC motherboard's power switch pins in the same way as a physical power button while keeping the ESP32 side electrically isolated.

An Android application is used for device configuration and control. Remote access can be provided through a FastAPI relay server running on a Raspberry Pi.

---

## Features

* Remote PC power control using XIAO ESP32-C3
* Electrically isolated PC power switch control using PhotoMOS
* Android-only Flutter application
* Initial device configuration over Bluetooth Low Energy
* Wi-Fi SSID / password configuration
* Persistent Wi-Fi configuration
* Configurable PhotoMOS GPIO
* User-defined device name
* Automatically generated unique Board ID based on ESP32-C3 eFuse MAC
* Current Wi-Fi IP address discovery
* PC power-state detection using USB connection state
* Direct LAN control
* Remote control through a FastAPI relay server
* Per-device HMAC-SHA256 authentication
* Raspberry Pi + Docker server deployment
* SQLite device and command storage
* Reduced idle power consumption using Wi-Fi modem sleep and limited BLE operation

---

## System Architecture

```text
                        Internet / LAN
                              │
                              ▼
                        Android App
                              │
                    REST API / HTTPS
                              │
                              ▼
                   FastAPI Relay Server
                    Raspberry Pi / Docker
                              │
                     board_id + HMAC
                              │
                              ▼
                    XIAO ESP32-C3
                              │
                              ▼
                         PhotoMOS
                              │
                              ▼
                      PC Power SW Pins
```

In relay mode, the ESP32 does not need to expose an HTTP server directly to the Internet.

Instead, the RemoteSW board operates as a **client** and initiates an outbound connection to the FastAPI relay server.

This eliminates the need to expose the ESP32 directly through router port forwarding.

---

## Board ID

Each RemoteSW board automatically generates a unique identifier from the ESP32-C3 eFuse MAC address.

Example:

```text
RSW-84FCE61234AB
```

The Board ID is automatically generated and remains unchanged for the lifetime of the board.

The relay server identifies devices using the Board ID rather than their IP address or user-defined device name.

Example:

```text
Board ID
RSW-84FCE61234AB
```

A separate human-readable name can be assigned by the user.

```text
Device Name
Lab PC
Home Desktop
Server PC
```

Changing the Device Name does not affect the Board ID.

---

## BLE Initial Configuration

BLE is temporarily enabled during the initial configuration process.

The Android application can configure:

* Wi-Fi SSID
* Wi-Fi password
* PhotoMOS GPIO
* Device name
* Relay server address
* Relay server port
* HTTP / HTTPS mode

The application can also retrieve:

* Board ID
* Device name
* Current Wi-Fi IP
* PhotoMOS GPIO
* PC power state

---

## GPIO Configuration

The GPIO controlling the PhotoMOS relay can be changed from the Android application.

Supported input formats may include:

```text
D1
D10
GPIO3
3
```

Arduino-style XIAO pin aliases are translated to their actual ESP32-C3 GPIO numbers.

When the output pin is changed, the previous GPIO is driven LOW and released before the new GPIO is configured as the output.

GPIOs that may interfere with the ESP32-C3 boot process can be restricted.

---

## PC Power State Detection

RemoteSW is designed for installations where the board remains connected to the PC through USB-C.

Instead of relying only on USB 5 V presence, the firmware can check whether the PC is operating as an active USB host.

The application displays states such as:

```text
ON
OFF / SLEEP
UNKNOWN
```

Depending on the motherboard BIOS and USB standby-power configuration, the firmware may not always be able to distinguish between sleep, hibernation and complete shutdown.

---

## Communication Modes

### Local Mode

When the Android device and RemoteSW board are on the same network, the application can communicate directly with the ESP32.

Default port:

```text
60553
```

Port 60553 is within the IANA Dynamic / Private port range.

---

### Relay Mode

For remote operation, both the Android application and RemoteSW board communicate through the FastAPI relay server.

```text
Android App
     │
     ▼
 FastAPI
     ▲
     │
 ESP32-C3
```

The Android application sends a command associated with a specific Board ID.

Example:

```text
RSW-84FCE61234AB
```

The relay server forwards the command only to the matching RemoteSW board.

---

## Security Architecture

User authentication and device authentication are separated.

### User Authentication

```text
Bearer Token
User PIN
```

### Device Authentication

Each board has an independent 256-bit device secret.

```text
Board ID
Device Secret
HMAC-SHA256
Nonce
```

Important ESP32-to-server messages are authenticated using HMAC-SHA256.

The device secret is unique to each board and is registered with the relay server during provisioning.

---

## FastAPI Relay Server

The relay server is designed to run inside Docker on a Raspberry Pi.

Current stack:

```text
FastAPI
Uvicorn
SQLite
Docker
Docker Compose
```

Example deployment:

```text
Raspberry Pi
└── Docker
    └── RemoteSW FastAPI
        └── 49384 -> 8000
```

Start the server:

```bash
docker compose up -d --build
```

Check container status:

```bash
docker compose ps
```

Check the API:

```bash
curl http://127.0.0.1:49384/health
```

---

## Automatic Startup

The Docker Compose configuration can use:

```yaml
restart: unless-stopped
```

Docker can also be enabled at system boot:

```bash
sudo systemctl enable docker
```

With this configuration, the RemoteSW relay server automatically starts again after a Raspberry Pi reboot.

---

## Repository Structure

```text
RemoteSW/
├── firmware/
│   └── xiao_esp32c3/
│
├── app/
│   └── android_flutter/
│
├── server/
│   └── fastapi/
│
├── docs/
│
└── hardware/
```

### firmware

Arduino firmware for XIAO ESP32-C3.

### app

Flutter-based Android RemoteSW application.

### server

FastAPI and Docker based relay server.

### docs

BLE protocol, relay protocol and architecture documentation.

### hardware

PhotoMOS circuit and future dedicated RemoteSW PCB design files.

---

## Technology Stack

### Embedded

* ESP32-C3
* Arduino ESP32 Core
* Bluetooth Low Energy
* Wi-Fi
* NVS
* FreeRTOS
* HMAC-SHA256
* USB Serial/JTAG

### Mobile

* Flutter
* Dart
* Android
* Bluetooth LE

### Backend

* Python
* FastAPI
* Uvicorn
* SQLite
* Docker
* Docker Compose

### Server

* Raspberry Pi 4B
* Linux
* Tailscale
* Docker

---

## Project Goals

RemoteSW is intended to be more than a simple Wi-Fi power switch.

The project focuses on:

* Remote access without exposing the ESP32 directly to the Internet
* Unique hardware identification
* Simple mobile-based provisioning
* Low MCU idle power consumption
* Server-side device authentication and access control
* Support for multiple RemoteSW devices
* Future integration into a dedicated PCB

---

## Roadmap

* HTTPS and reverse proxy support
* Server-side user account management
* Multi-user / multi-device mapping
* Push notifications
* PC power-state history
* OTA firmware updates
* Firmware version management
* Dedicated RemoteSW PCB
* Improved Android application UX

---

## Safety Notes

RemoteSW electrically shorts the motherboard Power Switch pins and therefore behaves like a physical PC power button.

Sending another power command while the PC is already running may initiate shutdown or forced power-off behavior depending on the operating system and motherboard configuration.

Always verify the motherboard Power SW pinout before connecting the hardware.

For production or Internet-facing deployments, the FastAPI server should not be exposed directly over unencrypted HTTP. HTTPS or a trusted VPN should be used.

---

## License

This project is licensed under the MIT License.

See the `LICENSE` file for details.

---

## Author

Graduate student personal embedded / IoT project.

RemoteSW was started as a personal project for developing practical experience across embedded systems, mobile applications, networking and backend infrastructure.
