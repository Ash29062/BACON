# ESP32 MQTT Publisher - Node-RED MQTT Subscriber

## Requirements

1. Mosquitto Broker installed (running on your local machine)
2. Node-RED installed and running
3. ESP32 development board
4. Potentiometer
5. LED + current-limiting resistor
6. Arduino IDE (or similar) for uploading code to ESP32

---

## Overview

In this project, we use an ESP32 to:

* **Read an analog value** from a potentiometer
* **Publish** this value via MQTT to a **local Mosquitto broker**
* **Subscribe** to a binary topic (`/led/control`) to **control an LED**

Node-RED acts as the frontend. It:

* Subscribes to the potentiometer values (`/pot/value`)
* Displays them on a dashboard gauge
* Lets the user set a threshold setpoint
* Publishes `1` to `/led/control` when the potentiometer value exceeds the setpoint — which turns ON the LED on the ESP32

---

## Wiring Diagram

| Component                | ESP32 Pin    |
| ------------------------ | ------------ |
| Potentiometer            | GPIO 36 (A0) |
| LED (with 220Ω resistor) | GPIO 2       |
| VCC / GND                | 3.3V / GND   |

---

## Setting up the ESP32

### 📋 Arduino Code

Upload the following code to your ESP32:

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "YourWiFiSSID";
const char* password = "YourWiFiPassword";
const char* mqtt_server = "192.168.X.X"; // Replace with your PC's IP address
// You can find your PC's IP address by running ipconfig in terminal and copying the IPv4 address

WiFiClient espClient;
PubSubClient client(espClient);

const int potPin = 36;
const int ledPin = 2;

int potValue = 0;

void callback(char* topic, byte* payload, unsigned int length) {
  payload[length] = '\0';
  String value = String((char*)payload);

  if (value == "1") {
    digitalWrite(ledPin, HIGH);
  } else {
    digitalWrite(ledPin, LOW);
  }
}

void setup_wifi() {
  Serial.print("Connecting to WiFi");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println(" connected!");
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Connecting to MQTT...");
    if (client.connect("ESP32Client")) {
      Serial.println(" connected!");
      client.subscribe("/led/control");
    } else {
      Serial.print(" failed, rc=");
      Serial.print(client.state());
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(ledPin, OUTPUT);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
}

void loop() {
  if (!client.connected()) reconnect();
  client.loop();

  potValue = analogRead(potPin);
  int scaledValue = map(potValue, 0, 4095, 0, 100);

  char valueStr[5];
  itoa(scaledValue, valueStr, 10);
  client.publish("/pot/value", valueStr);

  delay(1000);
}
```

---

## Setting up Node-RED

### Logic

* Subscribes to `/pot/value`
* Displays the value on a gauge
* Accepts a user-defined **setpoint** via dashboard
* Compares the latest potentiometer value to the setpoint
* Publishes `1` or `0` to `/led/control` depending on the comparison

### Required Nodes

Make sure the following are installed via Node-RED Palette Manager:

* `node-red-dashboard`
* `mqtt` node (comes by default)

### Import This Flow

1. Open [http://localhost:1880](http://localhost:1880)
2. Go to the menu (☰) → Import
3. Paste the following flow JSON:
```json
[
  {
    "id": "mqtt-in-pot",
    "type": "mqtt in",
    "z": "mqtt-flow",
    "name": "Potentiometer",
    "topic": "/pot/value",
    "broker": "mqtt-local",
    "x": 120,
    "y": 100,
    "wires": [["gauge", "compare-setpoint"]]
  },
  {
    "id": "gauge",
    "type": "ui_gauge",
    "z": "mqtt-flow",
    "name": "Potentiometer Gauge",
    "group": "group",
    "order": 1,
    "width": 6,
    "height": 3,
    "gtype": "gage",
    "title": "Potentiometer",
    "label": "%",
    "format": "{{value}}",
    "min": 0,
    "max": 100,
    "colors": ["#00b500", "#e6e600", "#ca3838"],
    "seg1": "",
    "seg2": "",
    "x": 340,
    "y": 60,
    "wires": []
  },
  {
    "id": "ui-setpoint",
    "type": "ui_numeric",
    "z": "mqtt-flow",
    "name": "Setpoint",
    "label": "Setpoint",
    "tooltip": "",
    "group": "group",
    "order": 2,
    "width": 3,
    "height": 1,
    "passthru": true,
    "topic": "setpoint",
    "format": "{{value}}",
    "min": 0,
    "max": 100,
    "step": 1,
    "x": 120,
    "y": 160,
    "wires": [["store-setpoint"]]
  },
  {
    "id": "store-setpoint",
    "type": "function",
    "z": "mqtt-flow",
    "name": "Store Setpoint",
    "func": "flow.set(\"setpoint\", msg.payload);\nreturn null;",
    "outputs": 0,
    "noerr": 0,
    "x": 330,
    "y": 160,
    "wires": []
  },
  {
    "id": "compare-setpoint",
    "type": "function",
    "z": "mqtt-flow",
    "name": "Compare",
    "func": "let setpoint = flow.get(\"setpoint\") || 50;\nlet val = parseInt(msg.payload);\n\nmsg.topic = \"/led/control\";\nmsg.payload = val >= setpoint ? \"1\" : \"0\";\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 360,
    "y": 100,
    "wires": [["mqtt-out-led"]]
  },
  {
    "id": "mqtt-out-led",
    "type": "mqtt out",
    "z": "mqtt-flow",
    "name": "LED Control",
    "topic": "",
    "qos": "",
    "retain": "",
    "broker": "mqtt-local",
    "x": 560,
    "y": 100,
    "wires": []
  },
  {
    "id": "mqtt-local",
    "type": "mqtt-broker",
    "name": "Mosquitto Local",
    "broker": "localhost",
    "port": "1883",
    "clientid": "",
    "usetls": false,
    "keepalive": "60",
    "cleansession": true
  },
  {
    "id": "group",
    "type": "ui_group",
    "name": "Control",
    "tab": "tab",
    "order": 1,
    "disp": true,
    "width": "6"
  },
  {
    "id": "tab",
    "type": "ui_tab",
    "name": "ESP32 Demo",
    "icon": "dashboard",
    "order": 1
  }
]
```
4. Deploy, and open [http://localhost:1880/ui](http://localhost:1880/ui) to view the dashboard

## Testing the System
Connect your ESP32 to your PC and upload the code.

Open http://localhost:1880/ui

Adjust the potentiometer.

Set a threshold setpoint on the dashboard.

When potentiometer value ≥ setpoint, the LED connected to ESP32 will light up.

## Notes

1. Ensure Mosquitto is running before launching Node-RED and ESP32.

2. If you’re using a firewall, allow traffic on port 1883 (MQTT).

3. Replace 192.168.X.X in the ESP32 code with your actual IP (use ipconfig to find it).

