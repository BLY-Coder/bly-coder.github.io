---
layout: single
title: Hacking 80% of Sushi Buffets - Critical Vulnerabilities in Management Systems
excerpt: Discovery of critical vulnerabilities in the shared infrastructure of multiple sushi restaurants that expose sensitive customer data and allow manipulation of operations.
date: 2025-10-29
classes: wide
header:
  teaser: /assets/images/shushi/shushi.png
  teaser_home_page: true
  icon: /assets/images/shushi/shushi.png
categories:
  - Review
  - Infosec
  - Vulns
  - Hacking
  - Pentesting
tags:
  - Web Pentesting
  - API Security
  - Swagger
  - Broken Access Control
  - OWASP Top 10
---

# 🔍 The Initial Discovery

I was enjoying my meal when I decided to take a look at the URL of the restaurant's ordering system. Once home, my curiosity led me to investigate further. I started with something basic: **subdomain enumeration**.

To my surprise, I discovered something fascinating: **all the restaurants from different chains were under the same domain**. But not only that, I found multiple subdomains with the `xxapi.*` pattern that immediately caught my attention.

![API subdomains found](/assets/images/shushi/apisubdomains.png)

## 🚨 Exposed Swagger: The Gateway

When accessing these API subdomains, I found something that should never be public in production: **fully exposed Swagger UI** or at least in this context.

> For those who don't know, Swagger is an extremely useful API documentation tool for developers... but one that should never be publicly accessible in a production environment, especially without authentication. (obviously depends on context)

![Exposed Swagger UI](/assets/images/shushi/Swagger.png)

And this is where things got really interesting.

## What could be done without authentication?

### 1. 📊 List All Tables and Their Sensitive Data

The `/admins/Table/GetTables` endpoint allowed obtaining complete information of all restaurant tables:

```
POST /admins/Table/GetTables
```

![GetTables endpoint response](/assets/images/shushi/GetTables.png)

**Exposed information:**
- Unique table and area identifiers
- Room names and locations
- Table QR codes
- **Total spending of each table in real-time**
- **Products ordered by each customer**
- **Payment methods used**
- Account information and passwords

Imagine the implications: an attacker could know exactly how much each table is spending, what they're eating, and plan targeted attacks.

### 2. 📱 Manipulate Messages on Tablets

The endpoint allowed modifying messages that appear on customer tablets. This could be used for:
- Targeted phishing to customers in the restaurant
- Creating panic or confusion
- Redirecting to malicious sites
- Displaying fake offers

### 3. 💸 Close Tables (Eat for Free!)

One of the most critical findings: it was possible to close tables remotely without paying. An attacker could:
1. Sit at a table
2. Order whatever they wanted
3. Close the bill remotely using the API
4. Leave without paying

This represents a **direct financial risk** for establishments.

### 4. 💰 Modify Prices and Menus

The exposed endpoints also allowed:
- Changing product prices
- Modifying menu descriptions
- Adding or removing products
- Altering the entire restaurant menu

## 🗺️ Scope of the Problem

The most concerning aspect of all this is the scope. During my investigation, I discovered that:

- **Multiple restaurants** use the same infrastructure
- **All shared the same vulnerabilities**
- The API was accessible from anywhere with an Internet connection
- There were no apparent intrusion detection systems
- No rate limiting existed

This means an attacker could:
- Extract data from all restaurants en masse
- Automate attacks against multiple establishments
- Cause significant financial losses
- Compromise the privacy of thousands of customers


## 🛡️ Responsible Disclosure

**It's important to note that this vulnerability has been responsibly reported** to the software provider. As an ethical security researcher, my goal is never to cause harm, but to help improve system security.

## 💭 Final Thoughts

This case is a **perfect example of how development convenience can seriously compromise security**. Leaving Swagger exposed in production is like leaving your house blueprint on the front door with all the locks marked.

### Lessons Learned:

1. **Security must be a priority from design**: It's not something added later.
2. **API documentation is for development, not production**: Tools like Swagger should be disabled in production environments.
3. **Every endpoint must have authentication**: Especially those handling sensitive operations.
4. **The principle of defense in depth is crucial**: A single layer of security is not enough.

## 🚀 Conclusion

What started as simple curiosity during a sushi meal turned into the discovery of critical vulnerabilities affecting a significant portion of the industry. This finding demonstrates that:

- **Modern POS (Point of Sale) systems** are as vulnerable as any other web application
- **API security** remains a pending subject in many organizations
- **Responsible disclosure** is essential to improve security for everyone

And remember: **next time you order sushi from a tablet, think about everything behind that simple interface**. 🍣🔒

---

*Note: All images have been edited to protect the identity of the affected establishments. Specific information that could identify the companies has been redacted.*

*This article is published for educational and security awareness purposes. No information is included that would facilitate the exploitation of these vulnerabilities.*

