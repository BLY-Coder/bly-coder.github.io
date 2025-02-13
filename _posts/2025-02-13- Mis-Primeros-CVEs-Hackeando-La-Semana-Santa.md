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

# 🎯 Mis Primeros CVEs, hackeando la Semana Santa.

## 🔍 Introducción

> *Siempre me ha llamado la atención cómo se gestionan los sistemas informáticos en la iglesia, cofradías, etc. Así que me decidí a investigar y descubrir vulnerabilidades.*

Recientemente, he descubierto una vulnerabilidad de seguridad en la aplicación web **H6Web**, ampliamente utilizada por Cofradías en España y otras organizaciones similares. Esta vulnerabilidad se clasifica como **Insecure Direct Object Reference (IDOR)** y permisos inseguros, permitiendo el acceso no autorizado a información confidencial de los usuarios.

Este hallazgo ha sido oficialmente reconocido y se le han asignado dos CVEs (**CVE-2025-1270** y **CVE-2025-1271**), lo que marca un hito importante en mi carrera en ciberseguridad.

## 💡 ¿Qué es un CVE?

**CVE** (*Common Vulnerabilities and Exposures*) es un identificador único asignado a una vulnerabilidad de seguridad descubierta en software o hardware. Su objetivo es proporcionar un estándar para que investigadores y organizaciones puedan rastrear y referenciar vulnerabilidades de manera uniforme en diversas bases de datos y sistemas de seguridad.

Cada CVE incluye una descripción detallada de la vulnerabilidad, su impacto y posibles soluciones o mitigaciones. Es gestionado por MITRE Corporation en colaboración con la comunidad de seguridad global.


## 🔐 Detalles del CVE y Enlaces Oficiales

Según ha confirmado INCIBE, se han asignado dos CVE a estas vulnerabilidades en H6Web:

- **CVE-2025-1270**: [Informe de INCIBE](https://www.incibe.es/en/incibe-cert/notices/aviso/multiple-vulnerabilities-anapi-group-h6web)
- **CVE-2025-1271**: [Informe de INCIBE](https://www.incibe.es/en/incibe-cert/notices/aviso/multiple-vulnerabilities-anapi-group-h6web)


## 🔐 Detalles de la Vulnerabilidad

| Campo | Detalle |
|-------|---------|
| **Aplicación Afectada** | H6Web |
| **Tipo de Vulnerabilidad** | <span style="color: #ff0000;">Incorrect Access Control / Insecure Permissions (IDOR), Cross-Site Scripting (XSS)</span> |
| **Tipo de Ataque** | Remoto |
| **Impacto** | ⚠️ Acceso no autorizado a información sensible de usuarios autenticados y posible inyección de JavaScript |

## 🛠️ Descripción Técnica

### 🔍 CVE-2025-1270: Vulnerabilidad IDOR

Se ha identificado una vulnerabilidad de IDOR en el endpoint:

```
/h6web/ha_datos_hermano.php
```

La aplicación no valida si el usuario autenticado tiene permisos para acceder a los datos asociados con el parámetro `pkrelacionado` en una solicitud POST. Esto permite a un atacante modificar este parámetro y acceder a información de otros usuarios sin restricciones.


### ⚡ Prueba de Concepto (PoC)

Imagen de los datos expuestos:

![PoC](/assets/images/CVEs/poc1.png)


#### 🚨 Datos Expuestos y Prueba de Concepto

A continuación, se muestra un ejemplo de los datos que pueden ser expuestos a través de esta vulnerabilidad.

| Field | Value |
|-------|-------|
| Brother No. | [HIDDEN] |
| Real No. | [HIDDEN] |
| Name | [HIDDEN] |
| Surnames | [HIDDEN] |
| Registration Date | [HIDDEN] |
| Birth Date | [HIDDEN] |
| ID Number | [HIDDEN] |
| Address | [HIDDEN] |
| City | [HIDDEN] |
| Postal Code | [HIDDEN] |
| Province | [HIDDEN] |
| Phone | [HIDDEN] |
| Mobile | [HIDDEN] |
| Email | [HIDDEN] |
| Pending Payments | [HIDDEN] |
| Payment Method | [HIDDEN] |
| Bank Account | [HIDDEN] |
| Periodicity | [HIDDEN] |

### 🛡️ Recomendaciones de Seguridad

Para mitigar esta vulnerabilidad, se recomienda implementar las siguientes medidas:

1. 🔒 **Validación de permisos**: Verificar que el usuario autenticado tiene derechos sobre el recurso solicitado.
2. 👥 **Uso de autenticación basada en roles (RBAC)**: Implementar un sistema de control de acceso basado en roles.
3. 📝 **Registro de actividad**: Monitorizar y registrar solicitudes inusuales o sospechosas.
4. 🔍 **Pruebas de seguridad regulares**: Realizar auditorías de código y pruebas de penetración.

## ✅ Solución

El equipo de Grupo Anapi ha implementado las siguientes medidas:

- 🚫 La vulnerabilidad IDOR ha sido completamente deshabilitada
- 🔧 La vulnerabilidad XSS ha sido corregida en la última versión
- ⬆️ Se recomienda a todos los usuarios actualizar a la última versión

## 🎉 Conclusión

Este hallazgo destaca la importancia de implementar controles de acceso adecuados en las aplicaciones web. La asignación de dos CVEs a estas vulnerabilidades refuerza la necesidad de seguir investigando y reportando problemas de seguridad para hacer de Internet un lugar más seguro.

> *¡Mis primeros CVEs, y muchos más por venir! 🚀*
