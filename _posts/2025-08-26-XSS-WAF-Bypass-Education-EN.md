---
layout: single
title: "XSS Filter Bypass in Educational Web Application: SVG-Based Techniques"
excerpt: "Analysis of a successful XSS filter bypass in an educational web application using SVG elements with event handlers, demonstrating limitations in content filtering systems."
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

## Introduction: XSS Filter Analysis

During a security assessment, a **Cross-Site Scripting (XSS)** vulnerability was identified in an educational web application from the Junta de Andalucía that implemented a security filter to prevent such attacks. However, this filter had limitations that allowed bypass through specific techniques.

![PoC1](/assets/images/xss-educ/image.png)

## The XSS Filter Detection

The application implemented a filtering system that detected common XSS patterns. When attempting to inject malicious code, the system responded with a detailed error:

```
HTTP Status 500 – Internal Server Error
Type Exception Report
message An exception occurred processing JSP page

java.lang.SecurityException: Possible XSS attack attempt: "><img src=test >
```

### Filter System Analysis

The stack trace revealed valuable information about the implementation:

```java
sadiel.cec.xss.XSSManager.comprobarAtaqueXSS(XSSManager.java:56)
sadiel.cec.xss.ParametrosPeticionXSS.comprobarAtaque(ParametrosPeticionXSS.java:84)
sadiel.cec.xss.XSSFilter.doFilter(XSSFilter.java:52)
```

This information indicates that:
- The filter is implemented as a Java servlet `Filter`
- It uses an `XSSManager` component for detection
- It processes all HTTP request parameters

## Bypass Technique: SVG Elements

### The Successful Payload

The bypass was achieved using **SVG (Scalable Vector Graphics)** elements with unconventional event handlers:

```html
<svg onx=() onload=(confirm)(1)>
```

### Why Does This Bypass Work?

1. **SVG Element**: SVG elements are valid HTML5 and less common in blacklists
2. **Non-Standard Event Handler**: `onx=()` is an attribute that doesn't trigger events but confuses the parser
3. **Main Handler**: `onload=(confirm)(1)` uses valid but uncommon JavaScript syntax
4. **Parentheses vs Quotes**: Using parentheses instead of quotes can evade certain filters

### Technical Payload Analysis

```javascript
// Payload breakdown:
<svg           // Valid HTML5 element
onx=()         // Fictitious attribute that may confuse parsers
onload=        // Real event handler that executes on load
(confirm)(1)   // Valid JavaScript: executes confirm(1)
>              // Element closure
```

## Limitations of Blacklist-Based Filters

This case demonstrates common limitations of XSS filtering systems:

### 1. **Dependence on Known Patterns**
- The filter detected `<img src=` but not `<svg onload=`
- It focused on common HTML elements

### 2. **Incomplete Parsing**
- Did not properly analyze alternative JavaScript syntax
- `(confirm)(1)` vs `confirm(1)` - both valid syntax

### 3. **Lack of Context**
- Did not consider modern HTML5 context
- SVG introduced new attack vectors

## Security Recommendations

### For Developers:
1. **Whitelist over Blacklist**: Allow only specifically authorized content
2. **Context-Aware Encoding**: Encode according to context (HTML, JavaScript, CSS, etc.)
3. **Content Security Policy (CSP)**: Implement CSP to mitigate XSS
4. **Server-Side Validation**: Never rely solely on client-side validation

### For System Administrators:
1. **Updated WAF**: Keep Web Application Firewall rules updated
2. **Detailed Logging**: Log all bypass attempts for analysis
3. **Regular Testing**: Perform periodic penetration testing

## Impact and Responsibility

⚠️ **Important Note**: This analysis is presented for **educational and research** purposes only. The report was conducted following **responsible disclosure** principles and vulnerabilities were reported through official channels.

## Conclusions

This case illustrates the importance of:

1. **Implementing robust filters** that consider modern attack vectors
2. **Not relying solely on blacklists** for content filtering
3. **Keeping defenses updated** against new bypass techniques
4. **Performing regular audits** of security systems

XSS filters based on simple patterns are insufficient against attackers who understand the limitations of these systems. It is essential to adopt a defense-in-depth approach that combines multiple mitigation techniques.

---

*This analysis is published solely for educational and web security research purposes. All testing activities were conducted under ethical principles and responsible disclosure.*
