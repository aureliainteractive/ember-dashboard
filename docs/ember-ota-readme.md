# 🔥 EMBER OTA — Sistema de actualización de firmware local

Sistema completo de actualización Over-The-Air (OTA) para dispositivos ESP32 en red local.  
Diseñado para el proyecto EMBER · UETS · Cuenca, Ecuador.

---

## Arquitectura del sistema

```
┌─────────────────────────────────────────────────────────┐
│                       RED LOCAL (WiFi)                   │
│                                                          │
│  ┌──────────────┐     GET /version     ┌──────────────┐ │
│  │   ESP32 #1   │ ──────────────────▶  │              │ │
│  │  cabina-01   │ ◀────────────────── │  Node.js OTA │ │
│  └──────────────┘   {ver, url, flag}   │    Server    │ │
│                                        │  :3000       │ │
│  ┌──────────────┐   GET /firmware.bin  │              │ │
│  │   ESP32 #2   │ ──────────────────▶  │              │ │
│  │  cabina-02   │ ◀────────────────── │              │ │
│  └──────────────┘    (binary stream)   └──────┬───────┘ │
│                                               │         │
│  ┌──────────────┐                    ┌────────▼──────┐  │
│  │   ESP32 #3   │                    │  Dashboard    │  │
│  │  sensor-01   │                    │  :3000/       │  │
│  └──────────────┘                    └───────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## Estructura del proyecto

```
ota-system/
├── server/
│   ├── index.js          ← Servidor Express (OTA + API)
│   ├── package.json
│   ├── .env.example
│   ├── data/
│   │   ├── version.json  ← Versión activa (auto-generado)
│   │   ├── devices.json  ← Registro de dispositivos (auto-generado)
│   │   └── activity.log  ← Log de actividad (auto-generado)
│   └── firmware/
│       └── firmware.bin  ← Firmware activo (se sube via dashboard)
│
├── esp32/
│   └── ember_ota/
│       ├── ember_ota.ino ← Sketch principal Arduino
│       └── config.h      ← ⚡ Configuración por dispositivo
│
├── dashboard/
│   └── index.html        ← Interfaz web (servida por Express)
│
└── README.md             ← Este archivo
```

---

## 1. Configurar e iniciar el servidor

### Requisitos
- Node.js 16 o superior → https://nodejs.org
- npm (incluido con Node.js)

### Instalación

```bash
cd ota-system/server
npm install
```

### Iniciar el servidor

```bash
npm start
```

El servidor queda escuchando en el puerto 3000 en todas las interfaces de red.

```
🔥 EMBER OTA Server running on port 3000
   Dashboard : http://localhost:3000
   Version   : http://localhost:3000/version
   Firmware  : http://localhost:3000/firmware.bin
```

### Encontrar la IP local del servidor

Los ESP32 necesitan la IP LAN de la computadora donde corre el servidor.

**Windows:**
```cmd
ipconfig
```
Busca "IPv4 Address" en el adaptador WiFi. Ejemplo: `192.168.1.100`

**Linux / Mac:**
```bash
ip a        # o
ifconfig
```
Busca la IP en `wlan0` o `en0`.

---

## 2. Configurar el ESP32

### Instalar herramientas

1. Instala **Arduino IDE 2.x** → https://www.arduino.cc/en/software
2. En Arduino IDE → Preferences → "Additional boards manager URLs":
   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
3. Tools → Board → Boards Manager → busca **esp32** → instala "esp32 by Espressif Systems"
4. Instala la librería **ArduinoJson** (Benoit Blanchon) desde Library Manager

### Editar `config.h`

Abre `esp32/ember_ota/config.h` y edita:

```cpp
#define WIFI_SSID       "NombreDeTuRed"     // ← tu red WiFi
#define WIFI_PASSWORD   "ClaveDeWiFi"       // ← clave WiFi
#define OTA_SERVER_IP   "192.168.1.100"     // ← IP de tu laptop (ver paso 1)
#define OTA_SERVER_PORT 3000
#define DEVICE_ID       "cabina-01"         // ← ID único por dispositivo
#define FIRMWARE_VERSION "1.0.0"            // ← versión inicial
```

> ⚠️ **Para cada ESP32 físico**, cambia `DEVICE_ID` a algo único:  
> `cabina-01`, `cabina-02`, `sensor-piso-1`, etc.

### Flashear el firmware inicial

1. Conecta el ESP32 por USB
2. En Arduino IDE: Tools → Board → selecciona tu modelo (ej: ESP32 Dev Module)
3. Tools → Port → selecciona el puerto COM/ttyUSB del ESP32
4. Click **Upload** (botón →)

El ESP32 arranca, se conecta a WiFi y consulta el servidor inmediatamente.

---

## 3. Subir firmware actualizado

### Opción A — Dashboard web (recomendado)

1. Abre el navegador en `http://localhost:3000`
2. En "Subir nueva versión":
   - Ingresa la versión (ej: `1.0.1`)
   - Escribe notas del release
   - Arrastra o selecciona el archivo `.bin`
3. Clic en **Publicar firmware**

El firmware queda disponible inmediatamente. Los ESP32 lo detectarán en el próximo boot.

### Opción B — API REST (curl)

```bash
curl -X POST http://localhost:3000/upload \
  -F "firmware=@/ruta/al/firmware.bin" \
  -F "version=1.0.1" \
  -F "notes=Fix motor control timing"
```

### ¿Cómo obtener el `.bin` de Arduino IDE?

En Arduino IDE 2.x:
- Sketch → **Export Compiled Binary** (Ctrl+Alt+S)
- El `.bin` aparece en la carpeta del sketch

---

## 4. Cómo los dispositivos se actualizan automáticamente

```
ESP32 arranca
    │
    ▼
Conecta a WiFi
    │
    ▼
GET /version?current=1.0.0  ──▶  Servidor responde:
                                  { version: "1.0.1",
                                    url: "http://192.168.1.100:3000/firmware.bin",
                                    update_available: true }
    │
    ▼ (si update_available = true)
Verifica semver localmente (1.0.1 > 1.0.0 ✓)
    │
    ▼
GET /firmware.bin  ──▶  Descarga binario (HTTP stream)
    │
    ▼
Update.begin() + Update.writeStream()
    │
    ▼
Update.end() → flash completo
    │
    ▼
Guarda nueva versión en NVS (flash no-volátil)
    │
    ▼
ESP.restart() → arranca con nuevo firmware
```

En el próximo boot, el ESP32 reporta `current=1.0.1` y el servidor responde `update_available: false`. No vuelve a actualizar.

---

## 5. API Reference

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET`  | `/version?current=X.Y.Z` | Versión activa + flag de actualización |
| `GET`  | `/firmware.bin` | Descarga el binario activo |
| `POST` | `/upload` | Sube nuevo firmware (multipart: `firmware`, `version`, `notes`) |
| `GET`  | `/devices` | JSON con todos los dispositivos registrados |
| `GET`  | `/logs?n=100` | Últimas N líneas del log de actividad |
| `GET`  | `/status` | Health check + versión + conteo de dispositivos |
| `GET`  | `/` | Dashboard web |

### Headers que el ESP32 envía
```
x-device-id: cabina-01   ← identifica al dispositivo en el servidor
```

---

## 6. Seguridad OTA (anti-brick)

El sistema tiene múltiples capas de protección:

1. **Verificación semver doble**: el servidor compara versiones Y el ESP32 verifica localmente antes de flashear.
2. **Validación de tamaño**: si el binario no cabe en flash, el update aborta limpiamente.
3. **Verificación de stream**: si se descargan menos bytes de los esperados, se aborta.
4. **`Update.end()` check**: el ESP32 no reinicia si la escritura en flash no completó.
5. **Reintento controlado**: si un update falla, espera `OTA_RETRY_DELAY_S` segundos y reintenta (máx `OTA_MAX_RETRIES` veces).
6. **Versión guardada en NVS** antes del reboot: si por alguna razón el nuevo firmware falla al arrancar y el usuario reflashea manualmente, el NVS simplemente se sobreescribe.

---

## 7. Múltiples dispositivos

Para 3–5 ESP32, solo cambia `DEVICE_ID` y `FIRMWARE_VERSION` en `config.h` de cada uno antes de flashear. El servidor gestiona todos bajo el mismo endpoint — el dashboard muestra cada dispositivo por separado con su versión actual.

---

## 8. Troubleshooting

| Síntoma | Causa probable | Solución |
|---------|---------------|----------|
| `[WiFi] ✗ Failed to connect` | SSID/pass incorrectos o señal débil | Verifica `config.h` |
| `Version check failed — HTTP 0` | IP del servidor incorrecta o server no corre | Verifica `OTA_SERVER_IP` y `npm start` |
| `Not enough space` | Partición OTA no habilitada | Selecciona "Default 4MB with spiffs" o "Minimal SPIFFS" en Tools → Partition Scheme |
| Update falla pero descarga OK | Binario corrupto o generado para otra placa | Recompila con la placa correcta seleccionada |
| Dispositivos no aparecen en dashboard | Firewall bloqueando puerto 3000 | Abre el puerto 3000 en el firewall de la laptop |

---

*EMBER OTA System · UETS · 2025–2026*
