---
layout: single
title: Hacking de Microscopio IoT - WiFi Abierta y Stream de Video sin Cifrar
excerpt: Descubrimiento de múltiples vulnerabilidades de seguridad en un microscopio IoT con cámara IP incorporada.
date: 2026-01-08
classes: wide
header:
  teaser: /assets/images/microscope/logo.png
  teaser_home_page: true
  icon: /assets/images/microscope/logo.png
categories:
  - Review
  - Infosec
  - Vulns
  - Hacking
  - IoT Security
tags:
  - IoT
  - Network Security
  - WiFi Security
  - Video Streaming
  - Privacy
---

# 🔬 Un Regalo para Mi Padre

Le regalaron a mi padre un microscopio IoT, de esos modernos que tienen cámara incorporada. La idea es genial: enciendes el microscopio, este crea su propia red WiFi, te conectas con el móvil, abres una app y listo, puedes ver en la pantalla lo que hay bajo el objetivo. Perfecto para examinar cualquier cosa pequeña sin tener que estar pegado al ocular.

Cuando empezamos a usarlo me di cuenta de algo raro: la red WiFi estaba abierta, sin contraseña. Y claro, siendo yo como soy, esto me hizo pensar... ¿qué más estará mal configurado? Lo que empezó como echarle un vistazo rápido acabó siendo todo un descubrimiento de agujeros de seguridad.

![Micro](/assets/images/microscope/micro.jpg)

## 📡 Cómo Funciona (o Debería Funcionar)

Antes de entrar en materia, dejo claro cómo va el tema:

1. Enciendes el microscopio y crea una red WiFi
2. La red es una 192.168.34.0/24, donde el propio microscopio está en la 192.168.34.1
3. Tu móvil se conecta y le asignan la IP 192.168.34.3
4. Con la app "Wifi Check" del fabricante ves el video
5. El microscopio tiene los puertos 8080 y 8081 abiertos, y por el 8080 va el video en H.264

![IP](/assets/images/microscope/2026-01-08_09-55.png)

Hasta aquí todo bien, ¿no? Bueno, no tanto.


## 🚨 WiFi Sin Contraseña: El Primer Problema

Lo primero que salta a la vista: **la WiFi está abierta de par en par**. Nada de contraseña, ni WPA, ni nada. Cualquiera que pase cerca con su móvil puede conectarse.

![Red WiFi del microscopio](/assets/images/microscope/2026-01-08_09-36.png)

Y aquí viene el problema gordo. Imagínate que usas esto en un hospital para ver muestras de pacientes, o en un laboratorio de investigación, o en control de calidad de una empresa. Pues resulta que cualquier vecino con un móvil puede:
- Conectarse sin que te enteres
- Ver el video en directo
- Echar un vistazo a lo que estés mirando

Vamos, que si estás examinando algo sensible, puedes tener a alguien mirando por encima de tu hombro sin que te des cuenta.

## 🔍 Echando un Vistazo a la Red

Vale, me conecto y lo primero que hago es un escaneo rápido a ver qué tiene abierto:

**Lo que encontré:**
- **Puerto 8080**: Por aquí va el video en H.264
- **Puerto 8081**: Otro servicio (supongo que para controlar el micro)

Y lo mejor de todo: el puerto 8080 está abierto para quien quiera verlo. Sin usuario, sin contraseña, sin nada.

## 📹 Viendo el Video (Sin App, Sin Nada)

Con la app "Wifi Check" que trae el fabricante, el video se ve de maravilla...

Pero es que ni siquiera hace falta la app. El video va por sin cifrar y en formato H.264. 

**Para ver el video solo tienes que:**
1. Conectarte a la WiFi abierta
2. Abrir por tcp `192.168.34.1:8080` en cualquier reproductor
3. Listo, ya estás viendo lo mismo que el dueño del microscopio

Así de simple. VLC, ffplay, o lo que sea que reproduzca H.264.

![Stream](/assets/images/microscope/2026-01-08_09-54.png)

## 📡 Lo Peor: Ni Siquiera Hace Falta Conectarse

Pero espera, que aún hay más. Como la WiFi está abierta y el video va sin cifrar, ni siquiera necesitas conectarte a la red para ver lo que están haciendo.

### Modo Monitor: Espiando Sin Dejar Rastro

Con un adaptador WiFi en modo monitor y herramientas como Wireshark o airodump-ng puedes capturar todo el tráfico de la red sin que nadie se entere. Es como escuchar una conversación a través de la pared.

**Cómo se hace:**

Primero pones tu tarjeta WiFi en modo monitor:

```bash
sudo airmon-ng start wlan0
```
![Monitor mode](/assets/images/microscope/2026-01-08_09-56.png)

Luego capturas todo lo que pasa por el aire:

```bash
sudo airodump-ng \
  --bssid 50:9B:94:CD:71:AF \
  -c 10 \
  -w cam \
  wlan0mon
```
![Capturing data](/assets/images/microscope/2026-01-08_10-01.png)


Y después lo abres en Wireshark para ver qué hay:

![Wireshark](/assets/images/microscope/2026-01-08_10-02.png)


Si Seguimos la secuencia de los paquetes TCP y los guardamos en raw en formato .h264 podemos obtener el video

![Captura de Wireshark](/assets/images/microscope/2026-01-08_10-10.png)

Como el video va en texto plano, los paquetes llevan el video entero. Ádemas nadie sabe que estás ahí. Es completamente pasivo.

![Frames de video reconstruidos](/assets/images/microscope/2026-01-08_10-13.png)


## 💭 Resumiendo los Fallos

### 1. WiFi Abierta
**El problema:** No hay contraseña, ni WPA, ni nada
**Qué pasa:** Cualquiera se conecta sin más

### 2. Video Sin Cifrar
**El problema:** El H.264 va en texto plano por el aire
**Qué pasa:** Puedes capturarlo y verlo sin que nadie se entere

### 3. Sin Autenticación
**El problema:** El puerto 8080 está abierto a quien quiera
**Qué pasa:** Si estás en la red, ves el video directamente


### 4. Espionaje Pasivo Posible
**El problema:** WiFi abierta + video sin cifrar = desastre
**Qué pasa:** Alguien puede estar grabando todo desde su coche sin que sepas que está ahí


## 🎓 Qué Podemos Aprender

### Si Eres Fabricante:

1. **Pon contraseña a la WiFi**: Por favor, aunque sea por defecto, algo. No la dejes abierta.
2. **Cifra el video**: HTTPS/TLS existe por algo, úsalo.
3. **Pide autenticación**: Aunque sea usuario y contraseña básicos, pero pide algo.
4. **Seguridad por defecto**: Que venga seguro de fábrica, no que el usuario tenga que configurarlo.
5. **Piensa en qué se va a usar**: Un microscopio puede acabar en un hospital o laboratorio. No vale cualquier cosa.

### Si Eres Usuario:

1. **No te fíes de que funcione**: Que se vea bonito no significa que sea seguro.
2. **Piensa en qué estás mirando**: Si es algo sensible, quizás ese micro IoT barato no sea la mejor idea.
3. **Aísla los IoT**: Si puedes, estos trastos en su propia red, lejos de tus cosas importantes.
4. **Quéjate**: Si compras algo así y ves que es un desastre de seguridad, díselo al fabricante. A ver si se enteran.

## 🚀 Conclusión

Lo que iba a ser ayudar a mi padre a montar su microscopio nuevo acabó siendo descubrir un agujero de seguridad tras otro. Este cacharro, que perfectamente podría estar en un hospital, un laboratorio de investigación, una universidad o una fábrica, no tiene ni lo más básico en seguridad.

WiFi abierta + video sin cifrar = cualquiera que pase cerca puede ver lo que estás mirando. Ya sea conectándose directamente o capturando el tráfico desde su coche.

### Dónde Puede Acabar Esto Mal:

- **Hospitales**: Muestras de pacientes al alcance de cualquiera
- **Laboratorios**: Tu investigación super secreta, no tan secreta
- **Empresas**: Control de calidad que puede ver la competencia
- **Universidades**: Trabajos de estudiantes que cualquier gracioso puede copiar

Otro ejemplo más de que en IoT la seguridad se la piensan al final, si es que se la piensan. La tecnología mola, sí, pero de qué sirve si cualquiera puede espiar lo que haces.

---

*Nota: Este análisis se realizó en un dispositivo propiedad de mi familia en un entorno controlado. No se accedió a datos de terceros. El propósito es puramente educativo para crear conciencia sobre problemas de seguridad IoT.*

