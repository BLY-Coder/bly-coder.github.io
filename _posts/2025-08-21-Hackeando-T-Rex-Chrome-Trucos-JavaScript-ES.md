---
layout: single
title: "Hackeando el T-Rex Runner en trex-runner.com: Entendiendo y Evadiendo el Sistema Anti-Trampas"
excerpt: Guía completa para manipular el juego T-Rex Runner en trex-runner.com usando JavaScript, incluyendo cómo establecer puntuaciones específicas, evitar el sistema anti-trampas y entender cómo el servidor detecta manipulaciones en las tablas de clasificación online.
date: 2025-08-23
classes: wide
header:
  teaser: /assets/images/trex-hack/logo.jpeg
  teaser_home_page: true
  icon: /assets/images/trex-hack/logo.jpeg
categories:
  - infosec
  - web-hacking
  - javascript
tags:  
  - JavaScript Hacking
  - T-Rex Game
  - Browser Exploitation
  - Game Manipulation
  - Anti-Cheat Systems
---

## Introducción: El T-Rex Runner en trex-runner.com

**¡Hola, entusiastas del hacking ético!** El **T-Rex Runner**, conocido como el juego del dinosaurio de Google Chrome, es un Easter Egg que aparece cuando no hay conexión a internet (`chrome://dino/`). Sin embargo, en [trex-runner.com](https://trex-runner.com/), puedes jugar una versión **hackeada** online que permite competir en tablas de clasificación globales, como se muestra en la siguiente tabla de puntuaciones diarias:



Este juego, ejecutado en JavaScript, es un excelente campo de pruebas para aprender sobre manipulación de código y seguridad web. En este post, exploraremos cómo hackear el juego para establecer puntuaciones específicas (como 10,000 puntos), entenderemos el **sistema anti-trampas** implementado en trex-runner.com y aprenderemos cómo evitar ser detectados como tramposos.

![T-Rex Game Hack](/assets/images/trex-hack/logo.jpeg)

### El Juego: T-Rex Runner Online

En **trex-runner.com**, el T-Rex Runner es una versión modificada del juego original de Chrome:
- **Mecánica**: Un dinosaurio corre automáticamente, saltando cactus y pterodáctilos con las teclas de flecha arriba (saltar) y abajo (agacharse).
- **Puntuación**: Basada en la distancia recorrida (`distanceRan`).
- **Online**: Envía puntuaciones al servidor para tablas de clasificación.
- **Anti-Trampas**: El servidor valida los datos enviados para detectar manipulaciones.

#### Fórmula de Puntuación
La puntuación se calcula con:
\[ \text{puntuación} = \text{Math.round}(\text{Math.ceil}(\text{distanceRan}) \times 0.025) \]
- `distanceRan`: Distancia recorrida.
- `Math.ceil`: Redondea hacia arriba.
- `0.025`: Factor de conversión.
- `Math.round`: Redondea al entero más cercano.

Para una puntuación de **10,000**:
\[ 10,000 = \text{Math.round}(\text{Math.ceil}(\text{distanceRan}) \times 0.025) \]
\[ \text{Math.ceil}(\text{distanceRan}) \approx \frac{10,000}{0.025} = 400,000 \]
\[ \text{distanceRan} \approx 399,999.999 \]

## Accediendo al Juego

### Método 1: Sin Conexión (Chrome Original)
1. Desconecta tu internet.
2. Intenta navegar a cualquier página en Chrome.
3. Presiona **Espacio** para empezar.

### Método 2: URL Directa
```
chrome://dino/
```

### Método 3: trex-runner.com
Visita [trex-runner.com](https://trex-runner.com/) para jugar online y competir en tablas de clasificación.

## Sistema Anti-Trampas Avanzado en trex-runner.com

La versión de [trex-runner.com](https://trex-runner.com/) implementa un **sistema anticheat sofisticado** que va mucho más allá del simple juego de Chrome. Cuando terminas una partida, el juego envía una solicitud POST a `/ajax/qufw/` con datos que el servidor valida exhaustivamente.

### **Request HTTP al Finalizar Partida**

```http
POST /ajax/qufw/ HTTP/2
Host: trex-runner.com
Content-Type: application/x-www-form-urlencoded

name=pikasito&points=10000&dist=400000&t=666667&token=0e5828d32be15905daad5b209d71a844796cdb8b99d35b2f962060448a0df88d
```

### **Parámetros Enviados**

| Parámetro | Descripción | Ejemplo |
|-----------|-------------|---------|
| `name` | Nombre del jugador | `pikasito` |
| `points` | Puntuación final calculada | `10000` |
| `dist` | Distancia recorrida | `400000` |
| `t` | Tiempo de juego (ms) | `666667` |
| `token` | Hash de verificación | `0e5828d32be...` |

### **Análisis del Código Anticheat**

Del código JavaScript proporcionado, el sistema anticheat funciona así:

```javascript
// Fragmento del código original
if (k > A) {
    var B = new XMLHttpRequest();
    B.open("GET", "/ajax/qufw/?points=" + k, true);
    B.onload = function() {
        if (B.readyState === B.DONE) {
            if (B.status === 200) {
                if (B.responseText != "") {
                    var D = "name=" + user_name + "&points=" + k + "&dist=" + C + "&t=" + z + "&token=" + document.getElementById("token").value;
                    B.open("POST", "/ajax/qufw/", true);
                    B.send(D);
                }
            }
        }
    };
}
```

#### **Variables Clave**
- `k = Math.round(this.highestScore * 0.025)` - Puntuación calculada
- `C = this.distanceRan` - Distancia recorrida
- `z = this.runningTime` - Tiempo de juego

### **Validaciones del Servidor**

#### **1. Fórmula Matemática**
El servidor verifica que:
\[ \text{points} = \text{Math.round}(\text{Math.ceil}(\text{dist}) \times 0.025) \]

**Ejemplo para 10,000 puntos:**
- `dist` debe ser ~400,000
- `Math.ceil(399,999.999) = 400,000`
- `400,000 × 0.025 = 10,000` ✅

#### **2. Coherencia Temporal**
La velocidad del T-Rex va de 6 a 13 unidades por frame (60 FPS):

\[ \text{velocidad promedio} \approx 10 \text{ unidades/frame} \]
\[ \text{dist} \approx 10 \times \left( \frac{\text{t}}{1000} \times 60 \right) = 0.6 \times \text{t} \]

**Para `dist=400,000`:**
\[ 400,000 = 0.6 \times \text{t} \]
\[ \text{t} = \frac{400,000}{0.6} = 666,667 \text{ ms} \approx 11.1 \text{ minutos} \]

#### **3. Token de Seguridad**
El `token` (ejemplo: `0e5828d32be15905daad5b209d71a844796cdb8b99d35b2f962060448a0df88d`) es un hash SHA-256 generado posiblemente con:
- Valores de `points`, `dist`, `t`
- Clave secreta del servidor
- Timestamp o session ID

### **Caso Real: Detección de Trampa**

En un intento previo:
```
points=7273, dist=290907.363, t=4760.3 ms
```

**Respuesta del servidor**: `"cheatin"`

**¿Por qué falló?**
- Velocidad calculada: \(\frac{290,907}{4.76} \div 60 \approx 1,018\) unidades/frame
- Límite del juego: ~13 unidades/frame máximo
- **Factor de error**: 78x más rápido que lo físicamente posible


## Técnicas de Hacking Avanzado

### 🚨 **Hack 1: Método de Puntuación Específica (Recomendado)**

Para lograr **cualquier puntuación** sin ser detectado:

```javascript
function setScore(targetScore) {
    // Calcular valores coherentes
    const distance = (targetScore / 0.025) - 0.001;
    const time = Math.round(distance / 0.6); // Velocidad promedio realista
    
    // Establecer valores
    Runner.instance_.distanceRan = distance;
    Runner.instance_.runningTime = time;
    Runner.instance_.highestScore = Math.ceil(distance);
    
    // Finalizar juego
    Runner.instance_.gameOver();
    
    console.log(`✅ Configurado: ${targetScore} pts, dist: ${distance}, tiempo: ${time}ms`);
}

// Ejemplo: 10,000 puntos
setScore(10000);
```

#### **Matemáticas del Hack**
Para cualquier puntuación `P`:
\[ \text{distanceRan} = \frac{P}{0.025} - 0.001 \]
\[ \text{runningTime} = \frac{\text{distanceRan}}{0.6} \]

**Ejemplos de puntuaciones populares:**
- **1,000 pts**: `distanceRan=39,999.999`, `time=66,667ms` (~1.1 min)
- **5,000 pts**: `distanceRan=199,999.999`, `time=333,333ms` (~5.6 min)
- **10,000 pts**: `distanceRan=399,999.999`, `time=666,667ms` (~11.1 min)

### 🚨 **Hack 2: Invencibilidad + Auto-Accumulate**

Para jugar "legítimamente" pero sin morir:

```javascript
// Desactivar game over
Runner.prototype.gameOver = function(){};

// Auto-play inteligente
function smartAutoPlay() {
    if (Runner.instance_.crashed) return;
    
    const obstacles = Runner.instance_.horizon.obstacles;
    const tRex = Runner.instance_.tRex;
    
    if (obstacles.length > 0) {
        const nextObstacle = obstacles[0];
        const distance = nextObstacle.xPos - tRex.xPos;
        
        // Saltar cuando el obstáculo esté cerca
        if (distance <= 100 && distance > 50 && !tRex.jumping) {
            tRex.startJump();
        }
        
        // Agacharse para pterodáctilos altos
        if (nextObstacle.typeConfig && nextObstacle.typeConfig.type === 'PTERODACTYL') {
            if (nextObstacle.yPos < 80 && distance <= 80 && !tRex.ducking) {
                tRex.setDuck(true);
                setTimeout(() => tRex.setDuck(false), 500);
            }
        }
    }
    
    setTimeout(smartAutoPlay, 20);
}

smartAutoPlay();
console.log("🤖 Auto-play inteligente activado");
```

### 🚨 **Hack 3: Bypass Completo del Anticheat**

El método más sofisticado combina timing realista con automatización:

```javascript
function antiCheatBypass() {
    console.log("🔓 Iniciando Bypass Anticheat...");
    
    // Configuración realista
    const targetScore = parseInt(prompt("Puntuación objetivo:", "5000"));
    const baseDistance = (targetScore / 0.025) - 0.001;
    const baseTime = baseDistance / 0.6;
    
    // Agregar variabilidad realista
    const variance = 1 + (Math.random() * 0.1 - 0.05); // ±5% variación
    const finalDistance = baseDistance * variance;
    const finalTime = baseTime * variance;
    
    // Ejecutar hack gradual
    let currentDist = Runner.instance_.distanceRan;
    let currentTime = Runner.instance_.runningTime;
    
    const interval = setInterval(() => {
        if (currentDist >= finalDistance) {
            Runner.instance_.distanceRan = finalDistance;
            Runner.instance_.runningTime = Math.round(finalTime);
            Runner.instance_.highestScore = Math.ceil(finalDistance);
            Runner.instance_.gameOver();
            clearInterval(interval);
            console.log(`✅ Bypass completado: ${targetScore} puntos`);
            return;
        }
        
        // Incremento gradual
        const increment = (finalDistance - currentDist) / 100;
        const timeIncrement = (finalTime - currentTime) / 100;
        
        currentDist += increment;
        currentTime += timeIncrement;
        
        Runner.instance_.distanceRan = currentDist;
        Runner.instance_.runningTime = Math.round(currentTime);
    }, 50);
    
    console.log(`🎯 Objetivo: ${targetScore} pts en ${Math.round(finalTime/1000)}s`);
}

antiCheatBypass();
```

### 🚨 **Hack 4: Monitor de Sistema Anticheat**

Script para monitorear las validaciones del servidor:

```javascript
// Interceptar requests XMLHttpRequest
const originalXHR = XMLHttpRequest.prototype.send;
XMLHttpRequest.prototype.send = function(data) {
    console.log("📡 Request interceptada:", data);
    
    // Analizar datos enviados
    if (data && data.includes('points=')) {
        const params = new URLSearchParams(data);
        const points = params.get('points');
        const dist = params.get('dist');
        const time = params.get('t');
        const token = params.get('token');
        
        console.log("🔍 Análisis de coherencia:");
        console.log(`Points: ${points}`);
        console.log(`Distance: ${dist}`);
        console.log(`Time: ${time}ms (${(time/1000/60).toFixed(2)} min)`);
        console.log(`Token: ${token.substring(0,16)}...`);
        
        // Validar fórmula
        const expectedPoints = Math.round(Math.ceil(parseFloat(dist)) * 0.025);
        const speedCheck = (parseFloat(dist) / (parseFloat(time) / 1000)) / 60;
        
        console.log(`✓ Fórmula correcta: ${points == expectedPoints ? 'SÍ' : 'NO'}`);
        console.log(`✓ Velocidad realista: ${speedCheck <= 13 ? 'SÍ' : 'NO'} (${speedCheck.toFixed(2)} u/f)`);
    }
    
    // Interceptar respuesta
    this.addEventListener('load', function() {
        if (this.responseText.includes('cheatin')) {
            console.log("🚨 ANTICHEAT ACTIVADO - Trampa detectada");
        } else if (this.responseText === '1') {
            console.log("✅ Puntuación aceptada por el servidor");
        }
    });
    
    return originalXHR.call(this, data);
};

console.log("🕵️ Monitor anticheat activado");
```


## Script Completo: T-Rex Ultimate Hacker

### **Herramienta Todo-en-Uno**

```javascript
(function() {
    const TRexHacker = {
        // Configuración
        config: {
            minSpeed: 6,
            maxSpeed: 13,
            frameRate: 60,
            scoreFactor: 0.025
        },
        
        // Calcular valores coherentes
        calculateValues: function(targetScore) {
            const distance = (targetScore / this.config.scoreFactor) - 0.001;
            const avgSpeed = (this.config.minSpeed + this.config.maxSpeed) / 2;
            const time = Math.round(distance / (avgSpeed * this.config.frameRate / 1000));
            
            return {
                distance: distance,
                time: time,
                score: targetScore,
                highScore: Math.ceil(distance)
            };
        },
        
        // Establecer puntuación específica
        setScore: function(score) {
            if (typeof Runner === 'undefined') {
                alert('❌ Ve a trex-runner.com primero');
                return;
            }
            
            const values = this.calculateValues(score);
            
            Runner.instance_.distanceRan = values.distance;
            Runner.instance_.runningTime = values.time;
            Runner.instance_.highestScore = values.highScore;
            Runner.instance_.gameOver();
            
            console.log(`✅ Score: ${score}, Dist: ${values.distance}, Time: ${values.time}ms`);
        },
        
        // Modo invencible
        godMode: function() {
            Runner.prototype.gameOver = function(){};
            console.log("🛡️ Invencibilidad activada");
        },
        
        // Auto-play inteligente
        autoPlay: function() {
            function play() {
                if (Runner.instance_.crashed) return;
                
                const obstacles = Runner.instance_.horizon.obstacles;
                const tRex = Runner.instance_.tRex;
                
                if (obstacles.length > 0) {
                    const obstacle = obstacles[0];
                    const dist = obstacle.xPos - tRex.xPos;
                    
                    if (dist <= 100 && dist > 50) {
                        if (obstacle.typeConfig.type === 'PTERODACTYL' && obstacle.yPos > 75) {
                            if (!tRex.ducking) tRex.setDuck(true);
                            setTimeout(() => tRex.setDuck(false), 300);
                        } else if (!tRex.jumping) {
                            tRex.startJump();
                        }
                    }
                }
                setTimeout(play, 30);
            }
            play();
            console.log("🤖 Auto-play activado");
        },
        
        // Monitor del sistema
        monitorAntiCheat: function() {
            const original = XMLHttpRequest.prototype.send;
            XMLHttpRequest.prototype.send = function(data) {
                if (data && data.includes('points=')) {
                    const params = new URLSearchParams(data);
                    console.log("📊 Datos enviados al servidor:");
                    console.table({
                        'Points': params.get('points'),
                        'Distance': params.get('dist'),
                        'Time (ms)': params.get('t'),
                        'Time (min)': (params.get('t')/60000).toFixed(2),
                        'Token': params.get('token')?.substring(0,16) + '...'
                    });
                }
                
                this.addEventListener('load', function() {
                    if (this.responseText.includes('cheatin')) {
                        console.log("🚨 Servidor respondió: TRAMPA DETECTADA");
                    } else {
                        console.log("✅ Puntuación aceptada");
                    }
                });
                
                return original.call(this, data);
            };
            console.log("🕵️ Monitor anticheat iniciado");
        }
    };
    
    // Menú interactivo
    console.log("🦕 T-REX ULTIMATE HACKER CARGADO 🦕");
    console.log("Comandos disponibles:");
    console.log("TRexHacker.setScore(10000) - Establecer puntuación");
    console.log("TRexHacker.godMode() - Invencibilidad");
    console.log("TRexHacker.autoPlay() - Juego automático");
    console.log("TRexHacker.monitorAntiCheat() - Monitor del sistema");
    
    // Hacer global
    window.TRexHacker = TRexHacker;
    
    // Activar monitor por defecto
    TRexHacker.monitorAntiCheat();
})();
```

#### **Instrucciones de Uso**

1. **Ir al juego**: Abre [trex-runner.com](https://trex-runner.com/)
2. **Abrir DevTools**: Presiona `F12` > `Console`
3. **Cargar el script**: Pega el código completo y presiona Enter
4. **Usar comandos**:
   ```javascript
   TRexHacker.setScore(8500);        // Puntuación específica
   TRexHacker.godMode();             // Invencibilidad
   TRexHacker.autoPlay();            // Juego automático
   ```

![Result](/assets/images/trex-hack/trex1.png)

## Análisis Profundo del Código Anticheat

### **Decompilación del Sistema**

El código JavaScript minificado revela el mecanismo completo:

```javascript
// Código original extraído
k = Math.round(this.highestScore * 0.025);  // Puntuación

// Validación de nuevo récord
if (k > A) {  // A = puntuación anterior
    var B = new XMLHttpRequest();
    B.open("GET", "/ajax/qufw/?points=" + k, true);
    
    // Solicitud POST con datos completos
    var D = "name=" + user_name + "&points=" + k + "&dist=" + C + "&t=" + z + "&token=" + document.getElementById("token").value;
    B.open("POST", "/ajax/qufw/", true);
    B.send(D);
}
```

### **Flujo Completo del Anticheat**

1. **Cálculo Local**: `points = Math.round(highestScore * 0.025)`
2. **Verificación GET**: `/ajax/qufw/?points=X` - Valida si es nuevo récord
3. **Envío POST**: Datos completos con token de seguridad
4. **Validación Servidor**: Múltiples checks de coherencia
5. **Respuesta**: "1" (éxito) o "cheatin" (trampa detectada)

### **Generación del Token**

El token es crítico y probablemente se genera con:

```javascript
// Posible algoritmo del token (especulación)
token = SHA256(points + dist + time + secretKey + sessionId)
```

**Estrategias para evadir:**
- **No modificar directamente**: Usar valores que generen tokens válidos
- **Respetar timing**: El token puede incluir timestamp
- **Calcular coherentemente**: Todos los valores deben ser matemáticamente consistentes

## Técnicas de Evasión Avanzadas

### **Método 1: Slow Burn (Recomendado)**

```javascript
function slowBurnHack(targetScore, durationMinutes = 10) {
    const values = TRexHacker.calculateValues(targetScore);
    const targetTime = durationMinutes * 60 * 1000; // Convertir a ms
    
    // Ajustar velocidad promedio para el tiempo deseado
    const requiredSpeed = values.distance / (targetTime / 1000 * 60);
    
    if (requiredSpeed > 13) {
        console.log(`⚠️ Velocidad requerida (${requiredSpeed.toFixed(2)}) > límite (13)`);
        console.log(`💡 Aumenta duración a ${Math.ceil(values.distance/(13*60*1000/1000)/60)} minutos`);
        return;
    }
    
    console.log(`🎯 Configurando ${targetScore} pts en ${durationMinutes} min`);
    console.log(`📈 Velocidad promedio: ${requiredSpeed.toFixed(2)} u/f`);
    
    // Aplicar hack
    Runner.instance_.distanceRan = values.distance;
    Runner.instance_.runningTime = targetTime;
    Runner.instance_.highestScore = values.highScore;
    
    setTimeout(() => {
        Runner.instance_.gameOver();
        console.log("✅ Slow burn completado");
    }, 1000);
}

// Ejemplo: 5000 puntos en 8 minutos
slowBurnHack(5000, 8);
```

### **Método 2: Token Preservation**

```javascript
function preserveTokenHack(targetScore) {
    // Guardar token original
    const originalToken = document.getElementById("token").value;
    
    // Hacer pequeños ajustes que no afecten dramáticamente el token
    const originalDist = Runner.instance_.distanceRan;
    const originalTime = Runner.instance_.runningTime;
    
    const values = TRexHacker.calculateValues(targetScore);
    
    // Aplicar cambios gradualmente
    const steps = 20;
    const distStep = (values.distance - originalDist) / steps;
    const timeStep = (values.time - originalTime) / steps;
    
    let step = 0;
    const interval = setInterval(() => {
        if (step >= steps) {
            clearInterval(interval);
            Runner.instance_.gameOver();
            return;
        }
        
        Runner.instance_.distanceRan += distStep;
        Runner.instance_.runningTime += timeStep;
        step++;
    }, 100);
    
    console.log("🔐 Hack con preservación de token iniciado");
}
```

## Consideraciones Éticas

### ⚠️ **Notas Importantes**
- **Solo propósito educativo** - No hagas trampa en leaderboards reales
- **Respeta términos de servicio** - Úsalo responsablemente
- **Testing local recomendado** - chrome://dino/ es más seguro

### **Para Desarrolladores**
Validaciones clave del servidor: consistencia de fórmulas, límites de velocidad y verificación de tokens.

## Conclusiones

### 🎓 **Lecciones Clave**

Esta aventura de hackear el T-Rex nos enseñó principios fundamentales de ciberseguridad:

#### **"Nunca Confíes en el Cliente"**
- **Validación del cliente** = teatro de seguridad
- **Validación del servidor** = seguridad real  
- Todo lo enviado desde navegadores puede ser manipulado

#### **Las Matemáticas Derrotan los Hacks**
- Velocidad = Distancia ÷ Tiempo (¡la física se aplica al hacking!)
- La **consistencia** en números es clave tanto para saltarse COMO para detectar
- Los servidores nos pillaron con velocidades sobrehumanas imposibles

#### **Aplicaciones del Mundo Real**
Estos no son solo trucos de juegos - se aplican a:
- **Apps bancarias** (mismos problemas de confianza)
- **Comercio electrónico** (prevención de manipulación de precios)  
- **APIs** (rate limiting, validación)
- **Sistemas de autenticación** (validación de tokens)

### 🔬 **Habilidades Técnicas Adquiridas**
- **Competencia en JavaScript** (manipulación DOM, debugging)
- **Análisis de redes** (peticiones HTTP, manejo de respuestas)
- **Ingeniería inversa** (análisis de código, comprensión de sistemas)
- **Mentalidad de seguridad** (pensar como atacante Y defensor)

### 🎯 **Reflexiones Finales**

Google intencionalmente mantiene chrome://dino/ hackeable por **valor educativo**. La verdadera victoria no fue conseguir puntuaciones altas - fue entender cómo funcionan los sistemas por dentro.

**Recuerda**: Todo experto comenzó como un principiante curioso. La diferencia entre sombrero blanco y negro no es la habilidad - es **cómo eliges usar ese conocimiento**.

---

*Este análisis se publica con fines educativos para enseñar conceptos de seguridad web y programación en JavaScript.*
