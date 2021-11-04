---
layout: post
title:  "Reprogramando switch Sonoff para MQTT local"
date:   2017-01-27
image: "images/reprogramar-sonoff-switch-mqtt/sonoff.jpg"
description: "Modificación del firmware de los switches wifi Sonoff para utilizarlos con un servidor MQTT local."
tags: iot things
---
Los dispositivos Sonoff de ITead nos permiten crear una _"smart home"_ con un coste realmente bajo y de forma muy sencilla, sin embargo, la gestión de estos dispositivos se hace a través de los propios servidores de Sonoff, por lo que estarán constanmente enviando y recibiendo las órdenes desde un lugar que no está bajo nuestro control, y mejor evitar esto, ¿no?

En este caso vamos a ver como reprogramar el [Sonoff Wifi Smart Switch](https://www.itead.cc/smart-home/sonoff-wifi-wireless-switch.html), (el cual tiene un coste de 4,85$ (~4,50€)) para que utilice nuestro propio servidor MQTT y todo se gestione desde nuestra red local.

![Switch Sonoff](/images/reprogramar-sonoff-switch-mqtt/sonoff.jpg)

Para poder reprogramar el dispositivo necesitaremos un adaptador USB a serial-TTL (yo utilizo [este](https://www.amazon.es/gp/product/B01C2P9GD2)), y cuatro pines para soldarlos a la placa del switch.

Al lado del pulsador del switch hay cinco agujeros previstos para soldar una hilera de pines. Simplemente tendremos que soldar una hilera de cuatro pines en estos agujeros (el quinto no nos hace falta), tal y como está en la siguiente imagen:
![Pines](/images/reprogramar-sonoff-switch-mqtt/pines2.png)

Una vez hecho esto tendremos que conectar el switch al adaptador USB serial-TTL de la siguiente forma:
- El primer pin (empezamos por el que tenía el slot cuadrado, el más próximo al pulsador) a la salida 3,3v del adaptador USB.
- El segundo pin es RX, lo conectaremos al TX del adaptador.
- El tercer pin es TX, lo conectaremos al RX del adaptador.
- El cuarto pin es GND, lo conectaremos a un GND del adaptador.

Una vez lo tengamos todo conectado, ya podemos pasar a programar el switch. Para encender el Sonoff en modo programación tendremos que mantener pulsado el pulsador antes de conectar el adaptador al PC. 

Cuando al fin estemos en modo programación, podremos subir nuestro propio sketch al switch. La configuración para cargar correctamente el sketch en la placa es la siguiente:
- Board: Generic ESP8266 Module
- Flash mode: DIO
- Flash size: 1M (64K SPIFFS)
- Reset method: ck

Además necesitaremos usar un fork de la librería PubSubClient que tiene soporte para MQTT, disponible [aquí](https://github.com/Imroy/pubsubclient).

Tras estos pasos ya podremos cargar el nuevo sketch en la placa sin ningún problema:

{% highlight CPP linenos %}
//Based on a KmanOz work: https://github.com/KmanOz/Sonoff-HomeAssistant/blob/master/arduino/ESPsonoff-v1.01p/ESPsonoff-v1.01p.ino

//Needs PubSubClient with MQTT support: https://github.com/Imroy/pubsubclient

#include <EEPROM.h>
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <Ticker.h>

#define BUTTON          0
#define RELAY           12
#define LED             13

#define WIFI_SSID       "SSID"
#define WIFI_PASS       "PASSWORD"

#define MQTT_CLIENT     "MQTT_CLIENT"
#define MQTT_SERVER     "MQTT_SERVER_IP"
#define MQTT_PORT       1883
#define MQTT_TOPIC      "mqtt/topic"

#define MQTT_AUTH       true
#define MQTT_USER       "user"
#define MQTT_PASS       "pass"

//logic
bool sendStatus = true;
bool restart = false;
int lastState = true;
unsigned long TTasks;
unsigned long count = 0;

extern "C" {
  #include "user_interface.h"
}

WiFiClient wifiClient;
PubSubClient mqttClient(wifiClient, MQTT_SERVER, MQTT_PORT);
Ticker ticker;

void callback(const MQTT::Publish& pub) {
  if(pub.payload_string() == "stat") {
  } else if(pub.payload_string() == "on") {
    digitalWrite(LED, LOW);
    digitalWrite(RELAY, HIGH);
  } else if(pub.payload_string() == "off") {
    digitalWrite(LED, HIGH);
    digitalWrite(RELAY, LOW);
  } else if(pub.payload_string() == "reset") {
    restart = true;
  }
  sendStatus = true;
}

void setup() {
  pinMode(LED, OUTPUT);
  pinMode(RELAY, OUTPUT);
  pinMode(BUTTON, INPUT);
  digitalWrite(LED, HIGH);
  digitalWrite(RELAY, LOW);
  Serial.begin(115200);
  EEPROM.begin(8);
  lastState = EEPROM.read(0);
  if(lastState == 1) {
    digitalWrite(LED, LOW);
    digitalWrite(RELAY, HIGH);
  }
  ticker.attach(0.05, button);
  mqttClient.set_callback(callback);
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while(WiFi.status() != WL_CONNECTED) {
    delay(500);
  }
  if(WiFi.status() == WL_CONNECTED) {
    Serial.println("IP: " + WiFi.localIP());
    if(MQTT_AUTH) {
      while(!mqttClient.connect(MQTT::Connect(MQTT_CLIENT).set_keepalive(90).set_auth(MQTT_USER, MQTT_PASS))) {
        delay(1000);
      }
    } else {
      while(!mqttClient.connect(MQTT::Connect(MQTT_CLIENT).set_keepalive(90))) {
        delay(1000);
      }
    }
    if(mqttClient.connected()) {
      mqttClient.subscribe(MQTT_TOPIC);
      if(digitalRead(RELAY) == HIGH)  {
        digitalWrite(LED, LOW);
      } else {
        digitalWrite(LED, HIGH);
      }
    }
  }
}

void loop() {
  mqttClient.loop();
  timedTasks();
  checkStatus();
}

void button() {
  if(!digitalRead(BUTTON)) {
    count++;
  } else {
    if(count > 1 && count <= 40) {
      digitalWrite(LED, !digitalRead(LED));
      digitalWrite(RELAY, !digitalRead(RELAY));
      sendStatus = true;
    } else if(count > 40) {
      restart = true;
    }
    count = 0;
  }
}

void checkConnection() {
  if(WiFi.status() == WL_CONNECTED) {
    if(!mqttClient.connected()) {
      restart = true;    
    }
  } else {
    restart = true;
  }
}

void checkStatus() {
  if(sendStatus) {
    if(digitalRead(LED) == LOW) {
      EEPROM.write(0, 1);
      mqttClient.publish(MQTT::Publish(MQTT_TOPIC"/stat", "on").set_retain().set_qos(1));
    } else {
      EEPROM.write(0, 0);
      mqttClient.publish(MQTT::Publish(MQTT_TOPIC"/stat", "off").set_retain().set_qos(1));
    }
    EEPROM.commit();
    sendStatus = false;
  }
  if(restart) {
    ESP.restart();
  }
}

void timedTasks() {
  if ((millis() > TTasks + (60000)) || (millis() < TTasks)) { 
    TTasks = millis();
    checkConnection();
  }
}
{% endhighlight %}

Para terminar, reiniciaremos el switch para comprobar que se conecta automáticamente a la red wifi y al servidor MQTT indicados, quitándonos así los servidores de Sonoff de en medio :P

Si todo funciona correctamente, ya podemos conectarlo a la corriente alterna para darle el uso que teníamos previsto.