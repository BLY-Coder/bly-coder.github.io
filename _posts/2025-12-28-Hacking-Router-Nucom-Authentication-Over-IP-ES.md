---
layout: single
title: Hacking de Router Nucom NJ-3R - Autenticación Basada en IP sin Validación
excerpt: Descubrimiento de múltiples vulnerabilidades en router Nucom NJ-3R durante unas vacaciones.
date: 2025-12-28
classes: wide
header:
  teaser: /assets/images/Router-ClickTel/logo.png
  teaser_home_page: true
  icon: /assets/images/Router-ClickTel/logo.png
categories:
  - Review
  - Infosec
  - Vulns
  - Hacking
  - IoT Security
tags:
  - Router Security
  - Authentication Bypass
  - IoT
  - Network Security
---

# 🏡 Vacaciones y Curiosidad

Estoy de viaje en una casa rural, disfrutando de unos días de descanso. Al llegar, nos proporcionaron la contraseña del WiFi para conectarnos, algo completamente normal. Pero como buen investigador de seguridad, mi curiosidad no descansa ni en vacaciones.

Mientras navegaba conectado a la red, decidí echar un vistazo al router. Lo que comenzó como una simple curiosidad se convirtió en el descubrimiento de múltiples vulnerabilidades críticas que comprometen completamente la seguridad de este dispositivo.

## 🔍 Primer Vistazo: Admin/Admin

El router en cuestión es un **Nucom NJ-3R**, distribuido por ClickTel. Lo primero que intenté fue acceder a la interfaz de administración en `192.168.0.253`.

Para mi sorpresa, las credenciales por defecto `admin/admin` funcionaron a la perfección. Esto ya es un problema en sí mismo, pero lo que descubrí después fue mucho más grave.

![Login con credenciales por defecto](/assets/images/Router-ClickTel/login.png)

## 🚨 Sin Tokens, Sin Cookies, Sin Autenticación Real

Mientras investigaba el panel de administración, abrí las herramientas de desarrollador para analizar las peticiones HTTP. Inmediatamente noté algo extraño: **no había tokens de sesión, no había cookies, no había ningún mecanismo tradicional de autenticación**.

Al hacer login, la petición `POST /goform/setAuth` simplemente enviaba las credenciales:

```
loginUser=admin&loginPass=admin
```

Y la respuesta era un simple redirect a `/adm/settings.asp`. Pero aquí viene lo interesante: **después del login, ninguna petición subsiguiente incluía información de autenticación**.

![Request sin headers de autenticación](/assets/images/Router-ClickTel/2025-12-28_17-20.png)

Cuando accedí a `/adm/settings.asp` después del login, la petición era un simple `GET` sin ningún tipo de autenticación. Entonces, ¿cómo sabe el router quién está autenticado?

## 💡 Autenticación Basada en IP

La respuesta es simple y aterradora: **el router utiliza autenticación basada únicamente en la dirección IP del cliente**. Una vez que te autentticas desde una IP, esa IP queda "autorizada" para acceder al panel de administración.

### Prueba de Concepto

Para confirmar mi teoría, realicé la siguiente prueba:

1. Me autentiqué normalmente en el router desde mi dispositivo (IP: 192.168.0.194)
2. Abrí una ventana de incógnito en el navegador
3. Sin introducir credenciales, accedí directamente a `192.168.0.253/adm/settings.asp`

**Resultado**: Acceso completo al panel de administración sin necesidad de autenticación.

![Acceso en modo incógnito sin autenticación](/assets/images/Router-ClickTel/2025-12-28_17-27.png)

Esto confirma que el router simplemente comprueba la IP de origen, no si hay una sesión válida.

## 🔓 Bypass de "Autenticación" Cambiando IP

Pero la cosa se pone peor. Si la autenticación se basa solo en la IP, ¿qué pasa si simplemente cambio mi IP a una que esté autenticada?

### Proceso de Explotación

1. **Dispositivo 1** (192.168.0.194): Me autentico normalmente con admin/admin
2. **Dispositivo 2** (192.168.0.200): Sin autenticación previa
3. En el Dispositivo 2, cambio manualmente mi IP a 192.168.0.194

![Configuración de IP manual en Dispositivo 2](/assets/images/Router-ClickTel/device2-ip-config.png)

4. Accedo a la interfaz de administración del router desde el Dispositivo 2

**Resultado**: El router me otorga acceso completo sin validar la dirección MAC del dispositivo. Solo comprueba la IP.

![Acceso desde Dispositivo 2 con IP suplantada](/assets/images/Router-ClickTel/device2-router-access.png)

Lo más preocupante es que el router **no valida la dirección MAC** del dispositivo. Como podemos ver en la lista de clientes DHCP del router, hay dos dispositivos diferentes con MACs distintas (9c:58:84:64:fb:aa y 0a:e1:94:cd:86:aa), pero ambos tienen acceso a la administración cuando usan la misma IP autenticada:

![Lista DHCP mostrando dos dispositivos con MACs diferentes](/assets/images/Router-ClickTel/dhcp-client-list-different-macs.png)

Esto significa que cualquier dispositivo en la red puede hacerse pasar por otro simplemente cambiando su IP, y obtener acceso completo al router si esa IP estaba previamente autenticada.

### Implicaciones de Seguridad

Esta vulnerabilidad permite:

- **IP Spoofing dentro de la LAN**: Cualquier dispositivo puede suplantar la IP del administrador
- **Acceso no autorizado**: No se requiere conocer las credenciales si alguien ya se autenticó desde esa IP
- **Persistencia sin credenciales**: Una vez autenticada una IP, cualquiera puede usarla
- **Sin validación de dispositivo**: La dirección MAC no se verifica

## 📂 Exposición de Configuración en Texto Plano

Como si las vulnerabilidades de autenticación no fueran suficientes, descubrí que el router expone toda su configuración en texto plano a través del endpoint `/cgi-bin/export_settings.cgi`.

Este endpoint devuelve un archivo de configuración completo que incluye:

![Configuración del router en texto plano](/assets/images/Router-ClickTel/2025-12-28_17-23.png)

**Información expuesta:**
- `Login=admin`
- `Password=admin` (en texto plano)
- `HostName=NuCom_B16A53`
- Configuración completa de red (IPs, máscaras, gateways)
- Configuración de WAN (usuario y contraseña PPPoE)

Pero espera, hay más. El archivo también incluye toda la configuración WiFi:

![Credenciales WiFi expuestas](/assets/images/Router-ClickTel/2025-12-28_17-25.png)

**Configuración WiFi expuesta:**
- `WscSSID=` (nombre de la red WiFi)
- `WPAPSK1=` (contraseña de la red WiFi en texto plano)
- Tipo de encriptación y método de autenticación
- Claves WPA completas

Esto significa que con un simple `GET` request a `/cgi-bin/export_settings.cgi` desde una IP "autenticada", un atacante puede obtener:

- Credenciales del router
- Credenciales del WiFi
- Configuración completa de red
- Credenciales PPPoE del ISP
- Y mucho más...

![Interface mostrando opción de backup](/assets/images/Router-ClickTel/2025-12-28_17-20_1.png)

## 🎯 Cadena de Ataque Completa

Un atacante en la misma red WiFi podría:

1. Esperar a que el administrador legítimo se autentique en el router
2. Identificar la IP del administrador (escaneando la red o monitorizando tráfico)
3. Cambiar su propia IP a la del administrador
4. Acceder al panel de administración sin credenciales
5. Descargar la configuración completa con todas las credenciales
6. Comprometer completamente la red

Alternativamente, si el atacante ya tiene acceso a la red (lo cual es fácil en una casa rural donde se comparte el WiFi), podría:

1. Probar las credenciales por defecto `admin/admin`
2. Si funcionan, descargar inmediatamente la configuración completa
3. Obtener todas las contraseñas en texto plano

## 🚀 Conclusión

Lo que comenzó como una simple curiosidad durante unas vacaciones se convirtió en el descubrimiento de múltiples vulnerabilidades críticas en un router ampliamente desplegado. 

Este router, que probablemente esté en miles de hogares y negocios en España, tiene problemas de seguridad fundamentales que ponen en riesgo la privacidad y seguridad de sus usuarios.


---

*Nota: Este análisis se realizó en un entorno controlado durante una estancia legal en una casa rural. No se accedió a información de terceros ni se realizaron modificaciones maliciosas. El propósito es puramente educativo.*

