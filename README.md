# RemoteSW

RemoteSW는 **XIAO ESP32-C3와 PhotoMOS 릴레이를 이용해 PC의 전원 스위치 핀을 원격으로 제어하는 시스템**입니다.

Android 앱을 통해 보드를 설정하고, 같은 네트워크에서는 직접 제어할 수 있으며, FastAPI 기반 중계 서버를 이용하면 외부에서도 특정 RemoteSW 보드를 식별하여 안전하게 전원 명령을 전달할 수 있습니다.

---

## 주요 기능

* XIAO ESP32-C3 기반 원격 PC 전원 제어
* PhotoMOS를 이용한 메인보드 Power SW 핀 절연 및 쇼트
* Android 전용 Flutter 앱
* BLE를 이용한 초기 보드 설정
* Wi-Fi SSID / Password 설정 및 저장
* PhotoMOS 출력 GPIO 동적 설정
* 사용자 지정 보드 이름 지원
* ESP32 eFuse MAC 기반 고유 Board ID 자동 생성
* 현재 Wi-Fi IP 조회
* USB 연결 상태를 이용한 PC 전원 상태 확인
* LAN 환경에서 보드 직접 제어
* FastAPI Relay Server를 통한 원격 제어
* 보드별 HMAC-SHA256 인증
* Raspberry Pi + Docker 기반 서버 운영
* SQLite 기반 보드 및 명령 상태 관리
* 저전력 동작을 위한 Wi-Fi Modem Sleep 및 BLE 비활성화 정책

---

## 시스템 구조

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

RemoteSW 보드는 서버에서 명령을 기다리는 HTTP 서버가 아니라, FastAPI 서버에 직접 접속하는 **클라이언트 방식**으로 동작할 수 있습니다.

이를 통해 외부 인터넷에 ESP32 포트를 직접 노출하거나 공유기 포트포워딩을 구성하지 않아도 됩니다.

---

## Board ID

각 RemoteSW 보드는 ESP32-C3의 eFuse MAC을 기반으로 고유 ID를 자동 생성합니다.

예:

```text
RSW-84FCE61234AB
```

Board ID는 사용자가 직접 설정하지 않으며, 보드가 바뀌지 않는 이상 항상 동일하게 유지됩니다.

서버는 보드 이름이나 IP 주소가 아니라 이 Board ID를 기준으로 장치를 식별합니다.

```text
Board ID
RSW-84FCE61234AB
```

사용자가 설정하는 이름은 별도로 관리됩니다.

```text
Device Name
연구실 PC
집 데스크탑
Server PC
```

Device Name을 변경해도 Board ID에는 영향을 주지 않습니다.

---

## BLE 초기 설정

보드가 처음 부팅되면 일정 시간 BLE 설정 모드가 활성화됩니다.

Android 앱을 통해 다음 항목을 설정할 수 있습니다.

* Wi-Fi SSID
* Wi-Fi Password
* PhotoMOS GPIO
* 보드 이름
* Relay Server 주소
* Relay Server Port
* HTTP / HTTPS 여부

앱은 BLE 연결을 통해 다음 정보도 확인할 수 있습니다.

* Board ID
* Device Name
* 현재 Wi-Fi IP
* PhotoMOS Pin
* PC 상태

---

## GPIO 설정

PhotoMOS를 제어하는 GPIO는 앱에서 변경할 수 있습니다.

예:

```text
D1
D10
GPIO3
3
```

XIAO ESP32-C3의 Arduino 핀 별칭을 실제 GPIO 번호로 변환하여 사용합니다.

핀을 변경하면 기존 출력 핀은 LOW 상태로 만든 후 INPUT으로 해제하고, 새로운 핀만 OUTPUT으로 활성화합니다.

ESP32-C3의 부팅에 영향을 줄 수 있는 일부 Strapping Pin은 설정하지 못하도록 제한할 수 있습니다.

---

## PC 전원 상태 확인

RemoteSW는 항상 PC와 USB-C로 연결되는 환경을 기준으로 설계되었습니다.

단순한 USB 5V 존재 여부가 아니라 ESP32-C3의 USB Host 연결 상태를 이용하여 PC가 USB Host로 활성화되어 있는지를 확인합니다.

표시 상태:

```text
ON
OFF / SLEEP
UNKNOWN
```

메인보드의 BIOS 설정이나 USB 대기전원 정책에 따라 절전 상태와 완전 종료 상태를 완전히 구분하지 못할 수 있습니다.

---

## 통신 방식

### Local Mode

같은 네트워크에서는 Android 앱이 ESP32에 직접 연결할 수 있습니다.

기본 포트:

```text
60553
```

60553은 IANA Dynamic / Private Port 범위에 포함되는 포트입니다.

---

### Relay Mode

외부에서는 Android 앱과 ESP32가 모두 FastAPI Relay Server에 접속합니다.

```text
Android
   │
   ▼
FastAPI
   ▲
   │
ESP32
```

앱은 명령을 보낼 때 Board ID를 지정합니다.

```text
RSW-84FCE61234AB
```

FastAPI 서버는 해당 Board ID의 보드에만 명령을 전달합니다.

---

## 보안 구조

사용자와 보드 인증을 서로 분리합니다.

### 사용자 인증

```text
Bearer Token
User PIN
```

### 보드 인증

각 보드는 별도의 256-bit Device Secret을 사용합니다.

```text
Board ID
Device Secret
HMAC-SHA256
Nonce
```

ESP32와 FastAPI 서버 사이의 주요 요청은 HMAC-SHA256을 이용해 검증됩니다.

Device Secret은 보드마다 다르며 서버에 최초 등록된 후 장치 인증에 사용됩니다.

---

## FastAPI Server

FastAPI 서버는 Raspberry Pi에서 Docker로 실행할 수 있습니다.

현재 구성:

```text
FastAPI
Uvicorn
SQLite
Docker
Docker Compose
```

기본 예시:

```text
Raspberry Pi
└── Docker
    └── RemoteSW FastAPI
        └── 49384 -> 8000
```

실행:

```bash
docker compose up -d --build
```

상태 확인:

```bash
docker compose ps
```

서버 확인:

```bash
curl http://127.0.0.1:49384/health
```

---

## 자동 실행

Docker Compose 설정에서는 다음 정책을 사용할 수 있습니다.

```yaml
restart: unless-stopped
```

Docker 서비스도 부팅 시 자동 실행되도록 설정합니다.

```bash
sudo systemctl enable docker
```

이렇게 설정하면 Raspberry Pi가 재부팅되어도 RemoteSW FastAPI 서버가 자동으로 시작됩니다.

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

XIAO ESP32-C3용 Arduino 펌웨어

### app

Flutter 기반 Android RemoteSW 앱

### server

FastAPI / Docker 기반 Relay Server

### docs

BLE 및 Relay Protocol 문서

### hardware

PhotoMOS 및 향후 RemoteSW 전용 PCB 관련 자료

---

## 주요 기술

### Embedded

* ESP32-C3
* Arduino ESP32 Core
* BLE
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

## 개발 목표

RemoteSW는 단순한 Wi-Fi 전원 스위치가 아니라 다음과 같은 구조를 목표로 개발하고 있습니다.

* 별도의 공유기 포트포워딩 없이 원격 접근
* 보드별 고유 ID 기반 관리
* 모바일 앱에서 간단한 초기 설정
* 낮은 MCU 대기전력
* 서버를 통한 장치 인증 및 접근 제어
* 여러 RemoteSW 보드 관리
* 향후 전용 PCB로 통합

---

## 향후 계획

* HTTPS / Reverse Proxy 적용
* 서버 사용자 계정 관리
* 여러 사용자와 여러 보드 매핑
* Push Notification
* PC 상태 변화 기록
* OTA Firmware Update
* 서버 기반 Firmware Version 관리
* 전용 RemoteSW PCB 제작
* Android 앱 UX 개선

---

## 주의사항

RemoteSW는 PC 메인보드의 Power Switch 핀을 전기적으로 쇼트하여 실제 전원 버튼과 동일한 동작을 수행합니다.

이미 PC가 켜진 상태에서 다시 전원 명령을 보내면 운영체제 또는 메인보드 설정에 따라 종료 또는 강제 종료 동작이 발생할 수 있습니다.

실제 하드웨어에 연결하기 전에 사용하는 메인보드의 Power SW 핀 구성을 반드시 확인하십시오.

또한 실제 운영 환경에서는 FastAPI 서버를 인터넷에 HTTP 상태로 직접 노출하지 말고 HTTPS 또는 VPN 환경을 사용하는 것을 권장합니다.

---

## License

This project is licensed under the MIT License.

자세한 내용은 `LICENSE` 파일을 참고하십시오.

---

## Author

Graduate student personal embedded / IoT project.

RemoteSW는 개인 연구 및 임베디드 시스템 개발 경험을 목적으로 시작된 프로젝트입니다.
