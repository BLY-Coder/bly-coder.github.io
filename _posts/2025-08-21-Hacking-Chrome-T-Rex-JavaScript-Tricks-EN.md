---
layout: single
title: "Hacking T-Rex Runner on trex-runner.com: Understanding and Bypassing Anti-Cheat Systems"
excerpt: Complete guide to manipulating the T-Rex Runner game on trex-runner.com using JavaScript, including how to set specific scores, bypass anti-cheat systems, and understand how servers detect manipulations in online leaderboards.
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

## Introduction: T-Rex Runner on trex-runner.com

**Hello, ethical hacking enthusiasts!** The **T-Rex Runner**, known as Google Chrome's dinosaur game, is an Easter Egg that appears when there's no internet connection (`chrome://dino/`). However, on [trex-runner.com](https://trex-runner.com/), you can play a **hacked** online version that allows you to compete on global leaderboards, as shown in the following daily score table:



This game, executed in JavaScript, is an excellent testing ground for learning about code manipulation and web security. In this post, we'll explore how to hack the game to set specific scores (like 10,000 points), understand the **anti-cheat system** implemented in trex-runner.com, and learn how to avoid being detected as cheaters.

![T-Rex Game Hack](/assets/images/trex-hack/logo.jpeg)

### The Game: T-Rex Runner Online

On **trex-runner.com**, T-Rex Runner is a modified version of Chrome's original game:
- **Mechanics**: A dinosaur runs automatically, jumping over cacti and pterodactyls with the up arrow key (jump) and down arrow key (duck).
- **Score**: Based on distance traveled (`distanceRan`).
- **Online**: Sends scores to the server for leaderboards.
- **Anti-Cheat**: The server validates sent data to detect manipulations.

#### Score Formula
The score is calculated with:
\[ \text{score} = \text{Math.round}(\text{Math.ceil}(\text{distanceRan}) \times 0.025) \]
- `distanceRan`: Distance traveled.
- `Math.ceil`: Rounds up.
- `0.025`: Conversion factor.
- `Math.round`: Rounds to the nearest integer.

For a score of **10,000**:
\[ 10,000 = \text{Math.round}(\text{Math.ceil}(\text{distanceRan}) \times 0.025) \]
\[ \text{Math.ceil}(\text{distanceRan}) \approx \frac{10,000}{0.025} = 400,000 \]
\[ \text{distanceRan} \approx 399,999.999 \]

## Accessing the Game

### Method 1: No Connection (Original Chrome)
1. Disconnect your internet.
2. Try to navigate to any page in Chrome.
3. Press **Space** to start.

### Method 2: Direct URL
```
chrome://dino/
```

### Method 3: trex-runner.com
Visit [trex-runner.com](https://trex-runner.com/) to play online and compete on leaderboards.

## Advanced Anti-Cheat System on trex-runner.com

The [trex-runner.com](https://trex-runner.com/) version implements a **sophisticated anticheat system** that goes far beyond Chrome's simple game. When you finish a game, it sends a POST request to `/ajax/qufw/` with data that the server validates exhaustively.

### **HTTP Request at Game End**

```http
POST /ajax/qufw/ HTTP/2
Host: trex-runner.com
Content-Type: application/x-www-form-urlencoded

name=pikasito&points=10000&dist=400000&t=666667&token=0e5828d32be15905daad5b209d71a844796cdb8b99d35b2f962060448a0df88d
```

### **Sent Parameters**

| Parameter | Description | Example |
|-----------|-------------|---------|
| `name` | Player name | `pikasito` |
| `points` | Final calculated score | `10000` |
| `dist` | Distance traveled | `400000` |
| `t` | Game time (ms) | `666667` |
| `token` | Verification hash | `0e5828d32be...` |

### **Anti-Cheat Code Analysis**

From the provided JavaScript code, the anti-cheat system works as follows:

```javascript
// Fragment from original code
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

#### **Key Variables**
- `k = Math.round(this.highestScore * 0.025)` - Calculated score
- `C = this.distanceRan` - Distance traveled
- `z = this.runningTime` - Game time

### **Server Validations**

#### **1. Mathematical Formula**
The server verifies that:
\[ \text{points} = \text{Math.round}(\text{Math.ceil}(\text{dist}) \times 0.025) \]

**Example for 10,000 points:**
- `dist` must be ~400,000
- `Math.ceil(399,999.999) = 400,000`
- `400,000 × 0.025 = 10,000` ✅

#### **2. Temporal Consistency**
T-Rex speed ranges from 6 to 13 units per frame (60 FPS):

\[ \text{average speed} \approx 10 \text{ units/frame} \]
\[ \text{dist} \approx 10 \times \left( \frac{\text{t}}{1000} \times 60 \right) = 0.6 \times \text{t} \]

**For `dist=400,000`:**
\[ 400,000 = 0.6 \times \text{t} \]
\[ \text{t} = \frac{400,000}{0.6} = 666,667 \text{ ms} \approx 11.1 \text{ minutes} \]

#### **3. Security Token**
The `token` (example: `0e5828d32be15905daad5b209d71a844796cdb8b99d35b2f962060448a0df88d`) is a SHA-256 hash possibly generated with:
- Values of `points`, `dist`, `t`
- Server secret key
- Timestamp or session ID

### **Real Case: Cheat Detection**

In a previous attempt:
```
points=7273, dist=290907.363, t=4760.3 ms
```

**Server response**: `"cheatin"`

**Why did it fail?**
- Calculated speed: \(\frac{290,907}{4.76} \div 60 \approx 1,018\) units/frame
- Game limit: ~13 units/frame maximum
- **Error factor**: 78x faster than physically possible


## Advanced Hacking Techniques

### 🚨 **Hack 1: Specific Score Method (Recommended)**

To achieve **any score** without being detected:

```javascript
function setScore(targetScore) {
    // Calculate coherent values
    const distance = (targetScore / 0.025) - 0.001;
    const time = Math.round(distance / 0.6); // Realistic average speed
    
    // Set values
    Runner.instance_.distanceRan = distance;
    Runner.instance_.runningTime = time;
    Runner.instance_.highestScore = Math.ceil(distance);
    
    // End game
    Runner.instance_.gameOver();
    
    console.log(`✅ Set: ${targetScore} pts, dist: ${distance}, time: ${time}ms`);
}

// Example: 10,000 points
setScore(10000);
```

#### **Hack Mathematics**
For any score `P`:
\[ \text{distanceRan} = \frac{P}{0.025} - 0.001 \]
\[ \text{runningTime} = \frac{\text{distanceRan}}{0.6} \]

**Examples of popular scores:**
- **1,000 pts**: `distanceRan=39,999.999`, `time=66,667ms` (~1.1 min)
- **5,000 pts**: `distanceRan=199,999.999`, `time=333,333ms` (~5.6 min)
- **10,000 pts**: `distanceRan=399,999.999`, `time=666,667ms` (~11.1 min)

### 🚨 **Hack 2: Invincibility + Auto-Accumulate**

To play "legitimately" but without dying:

```javascript
// Disable game over
Runner.prototype.gameOver = function(){};

// Smart auto-play
function smartAutoPlay() {
    if (Runner.instance_.crashed) return;
    
    const obstacles = Runner.instance_.horizon.obstacles;
    const tRex = Runner.instance_.tRex;
    
    if (obstacles.length > 0) {
        const nextObstacle = obstacles[0];
        const distance = nextObstacle.xPos - tRex.xPos;
        
        // Jump when obstacle is near
        if (distance <= 100 && distance > 50 && !tRex.jumping) {
            tRex.startJump();
        }
        
        // Duck for high pterodactyls
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
console.log("🤖 Smart auto-play activated");
```

### 🚨 **Hack 3: Complete Anti-Cheat Bypass**

The most sophisticated method combines realistic timing with automation:

```javascript
function antiCheatBypass() {
    console.log("🔓 Starting Anti-Cheat Bypass...");
    
    // Realistic configuration
    const targetScore = parseInt(prompt("Target score:", "5000"));
    const baseDistance = (targetScore / 0.025) - 0.001;
    const baseTime = baseDistance / 0.6;
    
    // Add realistic variability
    const variance = 1 + (Math.random() * 0.1 - 0.05); // ±5% variation
    const finalDistance = baseDistance * variance;
    const finalTime = baseTime * variance;
    
    // Execute gradual hack
    let currentDist = Runner.instance_.distanceRan;
    let currentTime = Runner.instance_.runningTime;
    
    const interval = setInterval(() => {
        if (currentDist >= finalDistance) {
            Runner.instance_.distanceRan = finalDistance;
            Runner.instance_.runningTime = Math.round(finalTime);
            Runner.instance_.highestScore = Math.ceil(finalDistance);
            Runner.instance_.gameOver();
            clearInterval(interval);
            console.log(`✅ Bypass completed: ${targetScore} points`);
            return;
        }
        
        // Gradual increment
        const increment = (finalDistance - currentDist) / 100;
        const timeIncrement = (finalTime - currentTime) / 100;
        
        currentDist += increment;
        currentTime += timeIncrement;
        
        Runner.instance_.distanceRan = currentDist;
        Runner.instance_.runningTime = Math.round(currentTime);
    }, 50);
    
    console.log(`🎯 Target: ${targetScore} pts in ${Math.round(finalTime/1000)}s`);
}

antiCheatBypass();
```

### 🚨 **Hack 4: Anti-Cheat System Monitor**

Script to monitor server validations:

```javascript
// Intercept XMLHttpRequest requests
const originalXHR = XMLHttpRequest.prototype.send;
XMLHttpRequest.prototype.send = function(data) {
    console.log("📡 Request intercepted:", data);
    
    // Analyze sent data
    if (data && data.includes('points=')) {
        const params = new URLSearchParams(data);
        const points = params.get('points');
        const dist = params.get('dist');
        const time = params.get('t');
        const token = params.get('token');
        
        console.log("🔍 Consistency analysis:");
        console.log(`Points: ${points}`);
        console.log(`Distance: ${dist}`);
        console.log(`Time: ${time}ms (${(time/1000/60).toFixed(2)} min)`);
        console.log(`Token: ${token.substring(0,16)}...`);
        
        // Validate formula
        const expectedPoints = Math.round(Math.ceil(parseFloat(dist)) * 0.025);
        const speedCheck = (parseFloat(dist) / (parseFloat(time) / 1000)) / 60;
        
        console.log(`✓ Formula correct: ${points == expectedPoints ? 'YES' : 'NO'}`);
        console.log(`✓ Realistic speed: ${speedCheck <= 13 ? 'YES' : 'NO'} (${speedCheck.toFixed(2)} u/f)`);
    }
    
    // Intercept response
    this.addEventListener('load', function() {
        if (this.responseText.includes('cheatin')) {
            console.log("🚨 ANTI-CHEAT ACTIVATED - Cheat detected");
        } else if (this.responseText === '1') {
            console.log("✅ Score accepted by server");
        }
    });
    
    return originalXHR.call(this, data);
};

console.log("🕵️ Anti-cheat monitor activated");
```


## Complete Hacking Script: T-Rex Ultimate Hacker

### **All-in-One Tool**

```javascript
(function() {
    const TRexHacker = {
        // Configuration
        config: {
            minSpeed: 6,
            maxSpeed: 13,
            frameRate: 60,
            scoreFactor: 0.025
        },
        
        // Calculate coherent values
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
        
        // Set specific score
        setScore: function(score) {
            if (typeof Runner === 'undefined') {
                alert('❌ Go to trex-runner.com first');
                return;
            }
            
            const values = this.calculateValues(score);
            
            Runner.instance_.distanceRan = values.distance;
            Runner.instance_.runningTime = values.time;
            Runner.instance_.highestScore = values.highScore;
            Runner.instance_.gameOver();
            
            console.log(`✅ Score: ${score}, Dist: ${values.distance}, Time: ${values.time}ms`);
        },
        
        // Invincible mode
        godMode: function() {
            Runner.prototype.gameOver = function(){};
            console.log("🛡️ Invincibility activated");
        },
        
        // Smart auto-play
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
            console.log("🤖 Auto-play activated");
        },
        
        // System monitor
        monitorAntiCheat: function() {
            const original = XMLHttpRequest.prototype.send;
            XMLHttpRequest.prototype.send = function(data) {
                if (data && data.includes('points=')) {
                    const params = new URLSearchParams(data);
                    console.log("📊 Data sent to server:");
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
                        console.log("🚨 Server responded: CHEAT DETECTED");
                    } else {
                        console.log("✅ Score accepted");
                    }
                });
                
                return original.call(this, data);
            };
            console.log("🕵️ Anti-cheat monitor started");
        }
    };
    
    // Interactive menu
    console.log("🦕 T-REX ULTIMATE HACKER LOADED 🦕");
    console.log("Available commands:");
    console.log("TRexHacker.setScore(10000) - Set score");
    console.log("TRexHacker.godMode() - Invincibility");
    console.log("TRexHacker.autoPlay() - Automatic play");
    console.log("TRexHacker.monitorAntiCheat() - System monitor");
    
    // Make global
    window.TRexHacker = TRexHacker;
    
    // Activate monitor by default
    TRexHacker.monitorAntiCheat();
})();
```

#### **Usage Instructions**

1. **Go to the game**: Open [trex-runner.com](https://trex-runner.com/)
2. **Open DevTools**: Press `F12` > `Console`
3. **Load the script**: Paste the complete code and press Enter
4. **Use commands**:
   ```javascript
   TRexHacker.setScore(8500);        // Specific score
   TRexHacker.godMode();             // Invincibility
   TRexHacker.autoPlay();            // Automatic play
   ```

![Result](/assets/images/trex-hack/trex1.png)

## Deep Analysis of Anti-Cheat Code

### **System Decompilation**

The minified JavaScript code reveals the complete mechanism:

```javascript
// Original extracted code
k = Math.round(this.highestScore * 0.025);  // Score

// New record validation
if (k > A) {  // A = previous score
    var B = new XMLHttpRequest();
    B.open("GET", "/ajax/qufw/?points=" + k, true);
    
    // POST request with complete data
    var D = "name=" + user_name + "&points=" + k + "&dist=" + C + "&t=" + z + "&token=" + document.getElementById("token").value;
    B.open("POST", "/ajax/qufw/", true);
    B.send(D);
}
```

### **Complete Anti-Cheat Flow**

1. **Local Calculation**: `points = Math.round(highestScore * 0.025)`
2. **GET Verification**: `/ajax/qufw/?points=X` - Validates if new record
3. **POST Submission**: Complete data with security token
4. **Server Validation**: Multiple consistency checks
5. **Response**: "1" (success) or "cheatin" (cheat detected)

### **Token Generation**

The token is critical and likely generated with:

```javascript
// Possible token algorithm (speculation)
token = SHA256(points + dist + time + secretKey + sessionId)
```

**Evasion strategies:**
- **Don't modify directly**: Use values that generate valid tokens
- **Respect timing**: Token may include timestamp
- **Calculate consistently**: All values must be mathematically consistent

## Advanced Evasion Techniques

### **Method 1: Slow Burn (Recommended)**

```javascript
function slowBurnHack(targetScore, durationMinutes = 10) {
    const values = TRexHacker.calculateValues(targetScore);
    const targetTime = durationMinutes * 60 * 1000; // Convert to ms
    
    // Adjust average speed for desired time
    const requiredSpeed = values.distance / (targetTime / 1000 * 60);
    
    if (requiredSpeed > 13) {
        console.log(`⚠️ Required speed (${requiredSpeed.toFixed(2)}) > limit (13)`);
        console.log(`💡 Increase duration to ${Math.ceil(values.distance/(13*60*1000/1000)/60)} minutes`);
        return;
    }
    
    console.log(`🎯 Setting ${targetScore} pts in ${durationMinutes} min`);
    console.log(`📈 Average speed: ${requiredSpeed.toFixed(2)} u/f`);
    
    // Apply hack
    Runner.instance_.distanceRan = values.distance;
    Runner.instance_.runningTime = targetTime;
    Runner.instance_.highestScore = values.highScore;
    
    setTimeout(() => {
        Runner.instance_.gameOver();
        console.log("✅ Slow burn completed");
    }, 1000);
}

// Example: 5000 points in 8 minutes
slowBurnHack(5000, 8);
```

### **Method 2: Token Preservation**

```javascript
function preserveTokenHack(targetScore) {
    // Save original token
    const originalToken = document.getElementById("token").value;
    
    // Make small adjustments that don't dramatically affect token
    const originalDist = Runner.instance_.distanceRan;
    const originalTime = Runner.instance_.runningTime;
    
    const values = TRexHacker.calculateValues(targetScore);
    
    // Apply changes gradually
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
    
    console.log("🔐 Token preservation hack initiated");
}
```

## Ethical Considerations

### ⚠️ **Important Notes**
- **Educational purpose only** - Don't cheat on real leaderboards
- **Respect terms of service** - Use responsibly 
- **Local testing recommended** - chrome://dino/ is safer

### **For Developers**
Key server validations: formula consistency, speed limits, and token verification.

## Conclusions

### 🎓 **Key Lessons**

This T-Rex hacking adventure taught us fundamental cybersecurity principles:

#### **"Never Trust the Client"**
- **Client-side validation** = security theater  
- **Server-side validation** = real security
- Everything sent from browsers can be manipulated

#### **Mathematics Defeats Hacks**  
- Speed = Distance ÷ Time (physics applies to hacking!)
- **Consistency** in numbers is key to both bypassing AND detecting
- Servers caught us with impossible superhuman speeds

#### **Real-World Applications**
These aren't just game tricks - they apply to:
- **Banking apps** (same trust issues)
- **E-commerce** (price manipulation prevention)
- **APIs** (rate limiting, validation)
- **Authentication systems** (token validation)

### 🔬 **Technical Skills Gained**
- **JavaScript proficiency** (DOM manipulation, debugging)
- **Network analysis** (HTTP requests, response handling)  
- **Reverse engineering** (code analysis, system understanding)
- **Security mindset** (thinking like attacker AND defender)

### 🎯 **Final Thoughts**

Google intentionally keeps chrome://dino/ hackable for **educational value**. The real victory wasn't getting high scores - it was understanding how systems work under the hood.

**Remember**: Every expert started as a curious beginner. The difference between white hat and black hat isn't skill - it's **how you choose to use that knowledge**.

---

*This analysis is published for educational purposes to teach web security concepts and JavaScript programming.*
