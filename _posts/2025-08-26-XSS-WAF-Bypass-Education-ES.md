---
layout: single
title: "Bypass de XSS Filter en Aplicación de Educación: Técnicas con SVG"
excerpt: "Análisis de un bypass exitoso de filtro XSS en una aplicación web de educación utilizando elementos SVG con handlers de eventos, demostrando limitaciones en sistemas de filtrado de contenido."
date: 2025-08-26
classes: wide
header:
  teaser: /assets/images/xss-educ/logo.jpeg
  teaser_home_page: true
  icon: /assets/images/xss-educ/logo.jpeg
categories:
  - infosec
  - web-hacking
  - xss
tags:  
  - XSS
  - WAF Bypass
  - SVG Injection
  - Web Security
  - Penetration Testing
---

## Introducción: Análisis de Filtros XSS

Durante una evaluación de seguridad, se identificó una vulnerabilidad de **Cross-Site Scripting (XSS)** en una aplicación web educativa de la Junta de Andalucía que implementaba un filtro de seguridad para prevenir este tipo de ataques. Sin embargo, este filtro presentaba limitaciones que permitían su bypass mediante técnicas específicas.

![PoC1](/assets/images/xss-educ/image.png)

## El Filtro XSS Detectado

La aplicación implementaba un sistema de filtrado que detectaba patrones comunes de XSS. Cuando se intentaba inyectar código malicioso, el sistema respondía con un error detallado:

```
Estado HTTP 500 – Internal Server Error
Tipo Informe de Excepción 
mensaje Ha sucedido una excepción al procesar la página JSP

java.lang.SecurityException: Posible intento ataque XSS: "><img src=test >
```

### Análisis del Sistema de Filtrado

El stack trace reveló información valiosa sobre la implementación:

```java
sadiel.cec.xss.XSSManager.comprobarAtaqueXSS(XSSManager.java:56)
sadiel.cec.xss.ParametrosPeticionXSS.comprobarAtaque(ParametrosPeticionXSS.java:84)
sadiel.cec.xss.XSSFilter.doFilter(XSSFilter.java:52)
```

Esta información indica que:
- El filtro está implementado como un `Filter` de servlet Java
- Utiliza un componente `XSSManager` para la detección
- Procesa todos los parámetros de la petición HTTP

## Técnica de Bypass: Elementos SVG

### El Payload Exitoso

El bypass se logró utilizando elementos **SVG (Scalable Vector Graphics)** con event handlers no convencionales:

```html
<svg onx=() onload=(confirm)(1)>
```

### ¿Por Qué Funciona Este Bypass?

1. **Elemento SVG**: Los elementos SVG son HTML5 válido y menos común en listas negras
2. **Event Handler No Estándar**: `onx=()` es un atributo que no dispara eventos, pero confunde al parser
3. **Handler Principal**: `onload=(confirm)(1)` utiliza sintaxis de JavaScript válida pero poco común
4. **Parentheses vs Quotes**: El uso de paréntesis en lugar de comillas puede evadir ciertos filtros

### Análisis Técnico del Payload

```javascript
// Payload desglosado:
<svg           // Elemento HTML5 válido
onx=()         // Atributo ficticio que puede confundir parsers
onload=        // Event handler real que se ejecuta al cargar
(confirm)(1)   // JavaScript válido: ejecuta confirm(1)
>              // Cierre del elemento
```

## Limitaciones de los Filtros Basados en Blacklist

Este caso demuestra las limitaciones comunes de los sistemas de filtrado XSS:

### 1. **Dependencia de Patrones Conocidos**
- El filtro detectaba `<img src=` pero no `<svg onload=`
- Se enfocaba en elementos HTML comunes

### 2. **Parsing Incompleto**
- No analizaba correctamente la sintaxis JavaScript alternativa
- `(confirm)(1)` vs `confirm(1)` - ambas sintaxis válidas

### 3. **Falta de Contexto**
- No consideraba el contexto HTML5 moderno
- SVG introdujo nuevos vectores de ataque

## Recomendaciones de Seguridad

### Para Desarrolladores:
1. **Whitelist sobre Blacklist**: Permitir solo contenido específicamente autorizado
2. **Context-Aware Encoding**: Codificar según el contexto (HTML, JavaScript, CSS, etc.)
3. **Content Security Policy (CSP)**: Implementar CSP para mitigar XSS
4. **Validación del Lado Servidor**: Nunca confiar solo en validación cliente

### Para Administradores de Sistemas:
1. **WAF Actualizado**: Mantener reglas de Web Application Firewall actualizadas
2. **Logging Detallado**: Registrar todos los intentos de bypass para análisis
3. **Testing Regular**: Realizar pruebas de penetración periódicas

## Impacto y Responsabilidad

⚠️ **Nota Importante**: Este análisis se presenta con fines **educativos y de investigación** únicamente. El reporte fue realizado siguiendo principios de **divulgación responsable** y las vulnerabilidades fueron reportadas a través de canales oficiales.

## Conclusiones

Este caso ilustra la importancia de:

1. **Implementar filtros robustos** que consideren vectores de ataque modernos
2. **No depender únicamente de blacklists** para filtrado de contenido
3. **Mantener actualizadas las defensas** contra nuevas técnicas de bypass
4. **Realizar auditorías regulares** de sistemas de seguridad

Los filtros XSS basados en patrones simples son insuficientes contra atacantes que comprenden las limitaciones de estos sistemas. Es fundamental adoptar un enfoque de defensa en profundidad que combine múltiples técnicas de mitigación.

---

*Este análisis se publica únicamente para fines educativos y de investigación en seguridad web. Toda actividad de testing fue realizada bajo principios éticos y de divulgación responsable.*
