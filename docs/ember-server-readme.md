# EMBER Server

Hub central para la simulación sísmica EMBER.  
Conecta la cabina (ESP32 vía WebSocket) con Roblox (vía HTTP polling).

---

## Requisitos

- Node.js >= 16
- npm

---

## Instalación y uso

```bash
npm install
npm start
```

Variables de entorno opcionales:

```bash
WS_PORT=8080    # Puerto WebSocket para la cabina (default: 8080)
REST_PORT=3000  # Puerto REST para Roblox y control (default: 3000)
DASH_MS=400     # Refresh del dashboard en ms (default: 400)
```

---

## Endpoints REST

| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/state` | Estado actual (Roblox hace polling aquí) |
| GET | `/health` | Health check |
| POST | `/simulation/start` | Iniciar simulación `{"mode":"earthquake"\|"fire"}` |
| POST | `/simulation/stop` | Detener simulación |
| POST | `/actuator` | Controlar actuador `{"action":"motor_on"\|"motor_off"\|"heater_on"\|"heater_off"\|"led_red"\|"led_green"\|"led_blue"\|"led_earthquake"\|"led_off"}` |

---

## Protocolo WebSocket (ESP32 → Server)

### Registro (obligatorio al conectar)
```json
{"type": "register", "role": "cabina"}
```

### Movimiento
```json
{"type": "movement_start", "direction": "forward"}
{"type": "movement_start", "direction": "backward"}
{"type": "movement_start", "direction": "left"}
{"type": "movement_start", "direction": "right"}
{"type": "movement_stop"}
```

### Datos IMU crudos (opcional, no loggea)
```json
{"type": "imu_data", "ax": 0.1, "ay": -0.05, "az": 9.8}
```

### Confirmación de actuador desde cabina
```json
{"type": "actuator_ack", "action": "motor_on"}
```

---

## Protocolo WebSocket (Server → Cabina)

```json
{"type": "event",    "event":  "earthquake_start"}
{"type": "event",    "event":  "fire_start"}
{"type": "event",    "event":  "simulation_stop"}
{"type": "actuator", "action": "motor_on"}
{"type": "actuator", "action": "motor_off"}
{"type": "actuator", "action": "heater_on"}
{"type": "actuator", "action": "led_earthquake"}
```

---

## Código Roblox (HTTP polling)

```lua
local HttpService = game:GetService("HttpService")
local RunService  = game:GetService("RunService")

local SERVER_URL = "https://tu-tunnel.trycloudflare.com"  -- o URL de cloudflared fija
local POLL_RATE  = 0.1  -- segundos (10 req/seg)

local lastMovement = "idle"

local function applyMovement(direction)
    local character = game.Players.LocalPlayer.Character
    if not character then return end
    local hrp = character:FindFirstChild("HumanoidRootPart")
    local humanoid = character:FindFirstChild("Humanoid")
    if not hrp or not humanoid then return end

    local cam = workspace.CurrentCamera
    local moveVec = Vector3.zero

    if direction == "forward"  then moveVec =  cam.CFrame.LookVector
    elseif direction == "backward" then moveVec = -cam.CFrame.LookVector
    elseif direction == "left"     then moveVec = -cam.CFrame.RightVector
    elseif direction == "right"    then moveVec =  cam.CFrame.RightVector
    end

    humanoid:Move(moveVec, false)
end

-- Polling loop
task.spawn(function()
    while true do
        local ok, response = pcall(function()
            return HttpService:RequestAsync({
                Url    = SERVER_URL .. "/state",
                Method = "GET"
            })
        end)

        if ok and response.StatusCode == 200 then
            local data = HttpService:JSONDecode(response.Body)

            -- Movimiento
            if data.movement ~= lastMovement then
                lastMovement = data.movement
                applyMovement(data.movement)
            end

            -- Evento de simulación
            if data.event == "earthquake_start" then
                -- activar efectos visuales de sismo
                print("[EMBER] Sismo iniciado")
            elseif data.event == "simulation_stop" then
                print("[EMBER] Simulación detenida")
            end
        end

        task.wait(POLL_RATE)
    end
end)
```

---

## Código Arduino ESP32 (sketch base)

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <ArduinoWebsockets.h>
#include <ArduinoJson.h>

using namespace websockets;

const char* SSID     = "TU_SSID";
const char* PASSWORD = "TU_PASSWORD";
const char* WS_URL   = "ws://192.168.1.X:8080";  // IP de la laptop

WebsocketsClient wsClient;
unsigned long lastReconnect = 0;

void onMessage(WebsocketsMessage msg) {
  StaticJsonDocument<256> doc;
  deserializeJson(doc, msg.data());

  const char* type = doc["type"];
  if (strcmp(type, "event") == 0) {
    const char* event = doc["event"];
    Serial.printf("[SERVER] Evento: %s\n", event);
    // Activar actuadores según evento
    if (strcmp(event, "earthquake_start") == 0) {
      // activar motor, LED earthquake
    }
  }
  if (strcmp(type, "actuator") == 0) {
    const char* action = doc["action"];
    Serial.printf("[SERVER] Actuador: %s\n", action);
  }
}

void connectWS() {
  wsClient.onMessage(onMessage);
  bool connected = wsClient.connect(WS_URL);
  if (connected) {
    Serial.println("[WS] Conectado");
    // Registrarse
    wsClient.send("{\"type\":\"register\",\"role\":\"cabina\"}");
  } else {
    Serial.println("[WS] Fallo conexión");
  }
}

void sendMovement(const char* type, const char* direction = nullptr) {
  StaticJsonDocument<128> doc;
  doc["type"] = type;
  if (direction) doc["direction"] = direction;
  char buf[128];
  serializeJson(doc, buf);
  wsClient.send(buf);
}

void setup() {
  Serial.begin(115200);
  WiFi.begin(SSID, PASSWORD);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  Serial.println("[WiFi] Conectado");
  connectWS();
}

void loop() {
  if (!wsClient.available()) {
    if (millis() - lastReconnect > 5000) {
      lastReconnect = millis();
      Serial.println("[WS] Reconectando...");
      connectWS();
    }
    return;
  }
  wsClient.poll();

  // Aquí va la lógica de detección de movimiento con IMU
  // sendMovement("movement_start", "forward");
  // sendMovement("movement_stop");
}
```

---

## Cloudflare Tunnel (para Roblox)

```bash
# Instalar cloudflared
# https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/

# Tunnel temporal (para pruebas):
cloudflared tunnel --url http://localhost:3000

# Tunnel nombrado (recomendado para defensa — URL fija):
cloudflared tunnel create ember
cloudflared tunnel route dns ember ember.tudominio.com
cloudflared tunnel run ember
```

El WebSocket de la cabina va **directo a IP local**, no necesita tunnel.  
Solo el REST para Roblox necesita exposición pública.
