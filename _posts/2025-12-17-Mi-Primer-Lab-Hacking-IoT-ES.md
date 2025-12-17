---
layout: single
title: "Mi primer lab de hacking IoT en casa"
excerpt: "Montando un laboratorio práctico de pentesting IoT con ESP32, sensores DHT11 y MQTT para reproducir escenarios reales de auditoría, desde la extracción de firmware hasta ataques de spoofing de datos."
date: 2025-17-12
classes: wide
header:
  teaser: /assets/images/iot-lab/esp32-lab.png 
  teaser_home_page: true
  icon: /assets/images/iot-lab/esp32-lab.png
categories:
  - IoT Security
  - Hardware Hacking
  - Pentesting
tags:
  - IoT
  - ESP32
  - MQTT
  - Firmware Analysis
  - Hardware Security
  - Pentesting
---

## Mi primer lab de hacking IoT en casa

Desde hace tiempo tenía claro que quería aprender pentesting IoT de forma práctica, no solo leyendo papers o haciendo CTFs, sino tocando hardware real y reproduciendo escenarios que se ven en auditorías.

Así que monté mi primer laboratorio de hacking IoT en casa, intentando seguir el mismo flujo que usaría un pentester en un entorno real:
- primero entender el dispositivo,
- luego extraer el firmware,
- y finalmente atacar la comunicación.

## El dispositivo IoT: hardware y funcionamiento

El dispositivo que monté es algo muy típico en el mundo IoT:

**Componentes principales:**

- **ESP32** como microcontrolador
- **Sensor DHT11** para temperatura y humedad
- **LCD 16x2 por I2C** como interfaz visible
- **WiFi + MQTT** para comunicación con un backend

En la práctica, esto se parece mucho a:
- termostatos
- estaciones meteorológicas
- sensores domésticos o industriales

### Conexiones principales

**DHT11:**
- VCC → 3.3V
- GND → GND
- DATA → GPIO 14

**LCD I2C:**
- VCC → 3.3V
- GND → GND
- SDA → GPIO 21
- SCL → GPIO 22

![ESP32 con sensor y LCD](/assets/images/iot-lab/esp32-connections.jpeg)

## Qué hace el dispositivo (comportamiento normal)

El flujo normal del dispositivo es el siguiente:

1. Se conecta a la red WiFi.
2. Se conecta a un servidor MQTT (backend IoT).
3. Lee temperatura y humedad del sensor.
4. Muestra los valores en el LCD.
5. Escucha mensajes MQTT para recibir datos remotos.

El punto interesante es que **el dispositivo confía en los datos que recibe por MQTT** y los trata como válidos.

**Codigo Vulnerable**

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include "DHT.h"

// ---- WIFI ----
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// ---- MQTT ----
const char* mqtt_server = "YOUR_MQTT_SERVER";
const char* topic = "casa/salon/temp";

// ---- LCD ----
LiquidCrystal_I2C lcd(0x27, 16, 2);

// ---- DHT ----
#define DHTPIN 14
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

// ---- MQTT CLIENT ----
WiFiClient espClient;
PubSubClient client(espClient);

// ---- STATE ----
String tempStr = "";
float hum = 0;
bool spoofActive = false;
unsigned long spoofTimestamp = 0;

void callback(char* t, byte* payload, unsigned int length) {
  char msg[32];
  if (length > 31) length = 31;
  memcpy(msg, payload, length);
  msg[length] = '\0';
  
  tempStr = String(msg); // dato no confiable (texto)
  spoofActive = true;
  spoofTimestamp = millis();
}

void reconnect() {
  while (!client.connected()) {
    client.connect("esp32-client");
    client.subscribe(topic);
  }
}

void setup() {
  Serial.begin(115200);
  
  Wire.begin(21, 22);
  lcd.init();
  lcd.backlight();
  dht.begin();
  
  lcd.print("WiFi...");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  
  lcd.clear();
  lcd.print("MQTT...");
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  reconnect();
  
  lcd.clear();
}

void loop() {
  if (!client.connected()) reconnect();
  client.loop();
  
  // --- READ SENSOR ---
  if (!spoofActive || millis() - spoofTimestamp > 10000) {
    float t = dht.readTemperature();
    hum = dht.readHumidity();
    tempStr = String(t); // convertimos float real a texto
    spoofActive = false;
  }
  
  // --- DISPLAY ---
  lcd.setCursor(0, 0);
  lcd.print("TEMP: ");
  lcd.print(tempStr);
  lcd.print(" ");  // limpia restos
  
  lcd.setCursor(0, 1);
  lcd.print("HUM: ");
  lcd.print(hum);
  lcd.print("% ");
  
  delay(2000);
}
```

## Paso 1: extracción del firmware (reconocimiento)

Antes de atacar nada, lo primero fue extraer el firmware del dispositivo, simulando un escenario muy realista:
- acceso físico autorizado
- laboratorio
- auditoría interna
- dispositivo de pruebas

Usando `esptool.py`:

```bash
esptool.py --port /dev/ttyUSB0 read_flash 0x000000 0x400000 firmware_dump.bin
```

> 0x400000 hace referencia a los 4MB del dispositivo

Esto genera una copia completa del firmware del ESP32.

```bash
esptool.py --port /dev/ttyUSB0 read_flash 0x000000 0x400000 firmware_dump.bin
esptool.py v3.3.3
Serial port /dev/ttyUSB0
Connecting.......
Detecting chip type... Unsupported detection protocol, switching and trying again...
Connecting....
Detecting chip type... ESP32
Chip is ESP32-D0WD-V3 (revision v3.1)
Features: WiFi, BT, Dual Core, 240MHz, VRef calibration in efuse, Coding Scheme None
Crystal is 40MHz
MAC: 5c:01:3b:9c:9f:08
Uploading stub...
Running stub...
Stub running...
4194304 (100 %)
4194304 (100 %)
Read 4194304 bytes at 0x0 in 380.9 seconds (88.1 kbit/s)...
Hard resetting via RTS pin...
```

## Paso 2: análisis del firmware y extracción de información

Con el firmware extraído, el siguiente paso fue buscar strings interesantes:

```bash
strings firmware_dump.bin | grep -i mqtt
```

Aquí aparecen datos clave:
- dirección del servidor MQTT
- topics utilizados
- mensajes internos del programa

Por ejemplo:

```
"Wifi Pass": XXX
"Wifi SSID": XXX
casa/salon/temp
```

Esto es muy importante desde el punto de vista del pentesting IoT:

**El atacante no adivina el topic MQTT, lo extrae del firmware.**

Este patrón es muy común en dispositivos reales, donde los topics y endpoints están hardcodeados.

## Paso 3: ataque MQTT (spoofing de datos)

Una vez conocido el topic, el ataque es trivial.

El dispositivo se suscribe a:

```
casa/salon/temp
```

Cualquier cliente con acceso al servidor MQTT puede publicar en ese topic.

Ejemplo:

```bash
mosquitto_pub -h <mqtt_server> -t casa/salon/temp -m "IOT PWNED"
```

**Resultado:**
- El ESP32 acepta el mensaje.
- El LCD muestra 35°C.
- El sensor real queda ignorado durante unos segundos.

![LCD mostrando temperatura falsa](/assets/images/iot-lab/fake-temperature.jpg)

Esto es un **Sensor Data Spoofing** clásico:
- sin exploits
- sin ejecución de código
- sin autenticación
- pero con impacto visible y real

## El fallo de diseño

El problema no es el protocolo, sino la lógica:

```cpp
tempStr = String(msg); // dato no confiable
```

El dispositivo:
- no valida el origen
- no valida el tipo
- no valida el rango
- confía ciegamente en el backend

Este tipo de fallo se ve constantemente en auditorías IoT.

## Qué demuestra este laboratorio

Aunque es un lab casero, reproduce problemas reales:

- **MQTT sin autenticación**: cualquiera puede publicar en los topics
- **Confianza excesiva en el backend**: los datos no se validan
- **Secretos visibles en firmware**: topics y configuraciones hardcodeadas
- **Impacto físico**: alteración visible del comportamiento (LCD)

## Conclusiones

Este laboratorio demuestra que:

1. **La extracción de firmware es fundamental**: proporciona información crítica sobre el funcionamiento interno del dispositivo.

2. **MQTT sin autenticación es un riesgo grave**: permite ataques triviales con impacto real.

3. **La validación de datos es esencial**: nunca confiar en datos externos sin validación adecuada.

**¡Gracias por leer!**

