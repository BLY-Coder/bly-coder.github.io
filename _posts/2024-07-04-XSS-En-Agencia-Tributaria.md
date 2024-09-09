---
layout: single
title: Vulnerabilidad XSS Reflejada Encontrada en el Sitio Web de la Agencia Tributaria
excerpt: Informe de Vulnerabilidad XSS en la Web de la Agencia Tributaria
date: 2024-09-04
classes: wide
header:
  teaser: /assets/images/xss-agencia/poc3.png
  teaser_home_page: true
  icon: /assets/images/xss-agencia/poc3.png
categories:
  - Review
  - Infosec
  - Vulns
tags:
  - Pentesting Web
  - Reflected XSS
---

# OverView

En el mundo de la ciberseguridad, uno de los problemas más comunes pero también peligrosos es el **Cross-Site Scripting** (XSS). Recientemente, durante un análisis rutinario de seguridad, me encontré con una vulnerabilidad de tipo **XSS reflejado** en el sitio web de la **Agencia Tributaria**. Esta vulnerabilidad podría permitir que un atacante ejecute código malicioso en los navegadores de usuarios autenticados, poniendo en peligro información sensible.

## ¿Qué es XSS Reflejado?

El **XSS reflejado** ocurre cuando los datos proporcionados por un usuario se envían a una página web sin ser correctamente validados o sanitizados. Como resultado, un atacante puede inyectar código JavaScript en el navegador del usuario, que se ejecuta cuando la víctima interactúa con la página afectada. Este tipo de vulnerabilidad puede utilizarse para robar cookies de sesión, redirigir a los usuarios a sitios maliciosos o realizar otras acciones sin el consentimiento del usuario.

## PoC

El sitio de la Agencia Tributaria contaba con una URL vulnerable:

```python
https://www12.agenciatributaria.gob.es/REDACTED=%3Cscript%3Ealert%28document.cookie%29%3C%2Fscript%3E
```

Al añadir un script malicioso en la URL, como el siguiente:

```html
<script>alert(document.cookie)</script>
```
el navegador de un usuario autenticado en el portal ejecuta este script, revelando información sensible como las **cookies de sesión**, que pueden contener datos valiosos para un atacante.

**PoC**
![](/assets/images/xss-agencia/poc1.png)

## ¿Por Qué es Peligroso?

Si bien ver una alerta con la cookie puede parecer inocuo, en manos equivocadas, esta vulnerabilidad puede llevar a ataques mucho más serios. Los atacantes pueden:

- Robar cookies de sesión para hacerse pasar por el usuario y acceder a su cuenta.
- Redirigir al usuario a sitios de phishing sin su consentimiento.
- Ejecutar scripts maliciosos que pueden hacer cambios en la cuenta de la víctima o extraer más datos.

## Proceso de Reproducción

Para reproducir este XSS reflejado, es necesario:

1. Estar autenticado en el portal de la Agencia Tributaria.
2. Acceder a la URL vulnerable con el código JavaScript inyectado.

Al hacerlo, el navegador del usuario ejecuta el script no autorizado, abriendo una ventana de alerta con las cookies de sesión del usuario.

## Recomendaciones para Mitigar el XSS

Para resolver este tipo de problemas, los desarrolladores del sitio web deben implementar medidas como:

- **Validación y Sanitización**: Es esencial validar todas las entradas del usuario y limpiar cualquier contenido que pueda contener código malicioso.
- **Uso de Escapes de Caracteres**: Escapar caracteres especiales en las entradas del usuario para evitar que se interpreten como código.
- **Encabezados de Seguridad**: Implementar encabezados HTTP, como `Content-Security-Policy`, para limitar la capacidad de ejecutar scripts no autorizados.
**En este momento este problema ya esta solventado**

## Mi Contacto con INCIBE

Tras descubrir esta vulnerabilidad, he tomado medidas responsables notificando al **Instituto Nacional de Ciberseguridad (INCIBE)**, para que puedan trabajar en conjunto con la Agencia Tributaria y mitigar este problema lo antes posible.

![](/assets/images/xss-agencia/poc2.png)
