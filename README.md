# ESP32-CAM Live Stream to Different Network

I'll provide you with code to stream from ESP32-CAM to your laptop over the internet. We'll use a method that works across different networks.

## Method: Using WebSocket Server (Easiest)

### ESP32-CAM Code

```cpp
#include "esp_camera.h"
#include <WiFi.h>
#include <WebSocketsServer.h>

// Replace with your network credentials
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// Camera pins for AI-Thinker ESP32-CAM
#define PWDN_GPIO_NUM     32
#define RESET_GPIO_NUM    -1
#define XCLK_GPIO_NUM      0
#define SIOD_GPIO_NUM     26
#define SIOC_GPIO_NUM     27
#define Y9_GPIO_NUM       35
#define Y8_GPIO_NUM       34
#define Y7_GPIO_NUM       39
#define Y6_GPIO_NUM       36
#define Y5_GPIO_NUM       21
#define Y4_GPIO_NUM       19
#define Y3_GPIO_NUM       18
#define Y2_GPIO_NUM        5
#define VSYNC_GPIO_NUM    25
#define HREF_GPIO_NUM     23
#define PCLK_GPIO_NUM     22

WebSocketsServer webSocket = WebSocketsServer(81);

void setupCamera() {
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;
  
  // Adjust frame size and quality
  if(psramFound()){
    config.frame_size = FRAMESIZE_VGA; // 640x480
    config.jpeg_quality = 10;
    config.fb_count = 2;
  } else {
    config.frame_size = FRAMESIZE_QVGA; // 320x240
    config.jpeg_quality = 12;
    config.fb_count = 1;
  }
  
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed with error 0x%x", err);
    return;
  }
}

void webSocketEvent(uint8_t num, WStype_t type, uint8_t * payload, size_t length) {
  switch(type) {
    case WStype_DISCONNECTED:
      Serial.printf("[%u] Disconnected!\n", num);
      break;
    case WStype_CONNECTED:
      {
        IPAddress ip = webSocket.remoteIP(num);
        Serial.printf("[%u] Connected from %d.%d.%d.%d\n", num, ip[0], ip[1], ip[2], ip[3]);
      }
      break;
  }
}

void setup() {
  Serial.begin(115200);
  Serial.println();
  
  // Initialize camera
  setupCamera();
  
  // Connect to Wi-Fi
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  Serial.print("IP Address: ");
  Serial.println(WiFi.localIP());
  
  // Start WebSocket server
  webSocket.begin();
  webSocket.onEvent(webSocketEvent);
  
  Serial.println("WebSocket server started on port 81");
}

void loop() {
  webSocket.loop();
  
  // Capture and send frame
  camera_fb_t * fb = esp_camera_fb_get();
  if (!fb) {
    Serial.println("Camera capture failed");
    return;
  }
  
  // Send to all connected clients
  webSocket.broadcastBIN(fb->buf, fb->len);
  
  esp_camera_fb_return(fb);
  
  delay(100); // Adjust for frame rate (~10 FPS)
}
```

### HTML Viewer (Save as `viewer.html` on your laptop)

```html
<!DOCTYPE html>
<html>
<head>
    <title>ESP32-CAM Live Stream</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            background-color: #1a1a1a;
            color: white;
            padding: 20px;
        }
        #container {
            max-width: 800px;
            width: 100%;
        }
        #stream {
            width: 100%;
            border: 3px solid #4CAF50;
            border-radius: 8px;
        }
        #controls {
            margin-top: 20px;
            padding: 15px;
            background-color: #2a2a2a;
            border-radius: 8px;
        }
        input {
            padding: 8px;
            margin: 5px;
            border-radius: 4px;
            border: 1px solid #555;
            background-color: #333;
            color: white;
        }
        button {
            padding: 10px 20px;
            margin: 5px;
            border: none;
            border-radius: 4px;
            background-color: #4CAF50;
            color: white;
            cursor: pointer;
            font-size: 16px;
        }
        button:hover {
            background-color: #45a049;
        }
        button:disabled {
            background-color: #666;
            cursor: not-allowed;
        }
        #status {
            margin-top: 10px;
            padding: 10px;
            border-radius: 4px;
        }
        .connected {
            background-color: #4CAF50;
        }
        .disconnected {
            background-color: #f44336;
        }
    </style>
</head>
<body>
    <div id="container">
        <h1>ESP32-CAM Live Stream</h1>
        
        <img id="stream" src="" alt="Waiting for stream...">
        
        <div id="controls">
            <div>
                <label>ESP32 IP/Domain:</label>
                <input type="text" id="ipAddress" placeholder="e.g., 192.168.1.100 or yourddns.com" value="">
            </div>
            <div>
                <button id="connectBtn" onclick="connect()">Connect</button>
                <button id="disconnectBtn" onclick="disconnect()" disabled>Disconnect</button>
            </div>
            <div id="status" class="disconnected">Status: Disconnected</div>
        </div>
    </div>

    <script>
        let ws;
        let imgElement = document.getElementById('stream');
        let statusElement = document.getElementById('status');
        let connectBtn = document.getElementById('connectBtn');
        let disconnectBtn = document.getElementById('disconnectBtn');

        function connect() {
            let ip = document.getElementById('ipAddress').value;
            if (!ip) {
                alert('Please enter ESP32 IP address or domain');
                return;
            }

            // Connect to WebSocket
            ws = new WebSocket(`ws://${ip}:81`);
            ws.binaryType = 'arraybuffer';

            ws.onopen = function() {
                statusElement.textContent = 'Status: Connected';
                statusElement.className = 'connected';
                connectBtn.disabled = true;
                disconnectBtn.disabled = false;
                console.log('WebSocket connected');
            };

            ws.onmessage = function(event) {
                // Convert binary data to image
                let arrayBuffer = event.data;
                let bytes = new Uint8Array(arrayBuffer);
                let blob = new Blob([bytes], {type: 'image/jpeg'});
                let url = URL.createObjectURL(blob);
                
                // Update image
                let oldUrl = imgElement.src;
                imgElement.src = url;
                
                // Clean up old URL to prevent memory leak
                if (oldUrl.startsWith('blob:')) {
                    URL.revokeObjectURL(oldUrl);
                }
            };

            ws.onerror = function(error) {
                console.error('WebSocket error:', error);
                statusElement.textContent = 'Status: Error - Check console';
                statusElement.className = 'disconnected';
            };

            ws.onclose = function() {
                statusElement.textContent = 'Status: Disconnected';
                statusElement.className = 'disconnected';
                connectBtn.disabled = false;
                disconnectBtn.disabled = true;
                console.log('WebSocket disconnected');
            };
        }

        function disconnect() {
            if (ws) {
                ws.close();
            }
        }
    </script>
</body>
</html>
```

## To Access Across Different Networks:

### Option 1: Port Forwarding (Recommended for home networks)

1. **Find ESP32's local IP** (shown in Serial Monitor)
2. **Router Settings:**
   - Login to your router (usually 192.168.1.1 or 192.168.0.1)
   - Find "Port Forwarding" section
   - Forward external port `81` to ESP32's IP port `81`
3. **Find your public IP:** Visit `whatismyip.com`
4. **Connect:** Use your public IP in the HTML viewer

### Option 2: Using ngrok (Easiest for testing)

```bash
# On a computer on the same network as ESP32:
# 1. Download ngrok from ngrok.com
# 2. Run:
ngrok tcp 192.168.1.XXX:81

# Use the provided URL in your HTML viewer
```

### Option 3: Free DDNS Service

Use services like:
- **No-IP** (noip.com)
- **DuckDNS** (duckdns.org)
- **Dynu** (dynu.com)

## Installation Instructions

### Arduino IDE Setup:

1. Install **ESP32 boards**:
   - File → Preferences → Additional Board Manager URLs
   - Add: `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`

2. Install **WebSockets library**:
   - Sketch → Include Library → Manage Libraries
   - Search: "WebSockets by Markus Sattler"
   - Install

3. **Select Board:**
   - Tools → Board → ESP32 Arduino → AI Thinker ESP32-CAM

4. **Upload:**
   - Connect ESP32-CAM with FTDI programmer
   - Connect GPIO 0 to GND for upload mode
   - Upload code
   - Disconnect GPIO 0 and reset

## Notes:

- **Frame rate:** Adjust `delay(100)` in loop() for different FPS
- **Quality:** Change `config.jpeg_quality` (lower = better quality, 10-63)
- **Resolution:** Modify `config.frame_size` (QVGA, VGA, SVGA, etc.)
- **Security:** Add authentication if exposing to internet

Let me know which method you'd like to use, and I can provide more specific instructions!
