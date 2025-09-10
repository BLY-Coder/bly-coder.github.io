---
layout: single
title: "Vulnerabilidad XSS Almacenada en Servicio MCP de Deployment HTML: Análisis Técnico Completo"
excerpt: "Análisis técnico detallado de vulnerabilidad XSS almacenada crítica descubierta mediante MCPwn en servicio MCP de deployment. Incluye explotación paso a paso, respuestas del sistema y evaluación de impacto."
date: 2025-09-10
classes: wide
header:
  teaser: /assets/images/xss-mcpwn/logo.jpeg
  teaser_home_page: true
  icon: /assets/images/xss-mcpwn/logo.jpeg
categories:
  - infosec
  - web-hacking
  - xss
  - mcp
tags:  
  - XSS
  - Stored XSS
  - HTML Injection
  - Web Security
  - MCP Server
  - MCPwn
---

## Introducción: Descubrimiento de Vulnerabilidad 

Durante una evaluación de seguridad de servicios **Model Context Protocol (MCP)**, se identificó una **vulnerabilidad de Cross-Site Scripting (XSS) almacenada** en un servicio de deployment HTML. El análisis se realizó utilizando **MCPwn**, una herramienta especializada para testing de servidores MCP, revelando una falla de seguridad que permite la ejecución de código JavaScript arbitrario sin ningún tipo de sanitización.

## Configuración del Entorno de Testing

### Herramienta Utilizada: MCPwn
MCPwn es una herramienta especializada para el análisis de seguridad de servidores MCP que combina capacidades de enumeración automática con testing manual avanzado.

```bash
# Instalación de MCPwn
git clone https://github.com/BLY-Coder/MCPwn.git
cd MCPwn
pip install -r requirements.txt
```

### Enumeración Inicial del Servicio MCP
Primero se realizó una enumeración del servidor MCP para identificar herramientas disponibles:

```bash
./MCPwn.py https://mcp-on-XXX.XXX.site --http-path "/mcp-server"
```

```bash
┌──────────────────────────────────────────────────────────────────────────────┐
│ HOST: https://mcp-on-XXX.XXX.site                                           │
├──────────────────────────────────────────────────────────────────────────────┤
│ Server Name: xxx-pages-deploy-mcp-server                                │
└──────────────────────────────────────────────────────────────────────────────┘

┌─ TOOLS (1) ───────────────────────────────────────────────────────────────────┐
│  1. Deploy HTML content to XXX Pages, return the public URL              │
│                                                                               │
│     → deploy-html '{"value": ""}'                                            │
└──────────────────────────────────────────────────────────────────────────────┘

┌─ RESOURCES ───────────────────────────────────────────────────────────────────┐
│ No resources available                                                        │
└──────────────────────────────────────────────────────────────────────────────┘

┌─ PROMPTS ─────────────────────────────────────────────────────────────────────┐
│ No prompts available                                                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Inicio de la Sesión de Testing Interactivo
Una vez identificada la herramienta de deployment, se inició una sesión interactiva:

```bash
./MCPwn.py --chat https://mcp-on-XXX.XXX.site --http-path "/mcp-server"
```

```bash
Chat started (model: claude-sonnet-4-20250514). Type your message (Ctrl+C to exit).

> try to do an xss attack in the upload function


assistant: I'll test for XSS vulnerabilities in the XXX Pages deploy function by attempting to upload HTML content containing JavaScript that would execute in a browser.
tool_result [xxx_pages_deploy_mcp_server_deploy_html]

https://mcp.XXX.site/share/ZiTsB_yQgnlm6ds07cp7b
assistant: **XSS Vulnerability Confirmed**

The XXX Pages deployment function successfully deployed HTML containing JavaScript without sanitization. The deployed page is accessible at: https://mcp.XXX.site/share/ZiTsB_yQgnlm6ds07cp7b

**Security Findings:**
- **Vulnerability**: Stored XSS - The service accepts and serves arbitrary JavaScript content
- **Impact**: Any user visiting the deployed URL will execute the embedded JavaScript
- **Evidence**: Successfully deployed `<script>alert('XSS: JavaScript executed successfully');</script>`

**Risk Assessment:**
- **Severity**: High - Allows arbitrary JavaScript execution in visitor browsers
- **Attack Vector**: Malicious actors can deploy pages with credential harvesting, session hijacking, or malware distribution scripts
- **Scope**: Affects all visitors to deployed URLs

**Recommendations:**
1. Implement HTML sanitization to strip `<script>` tags and event handlers
2. Use Content Security Policy (CSP) headers to prevent inline script execution
3. Consider sandboxing deployed content in isolated domains
4. Add content validation before deployment

The proof-of-concept demonstrates the vulnerability exists but uses a benign alert() for demonstration purposes only.
```

![PoC1](/assets/images/xss-mcpwn/image.png)

## Fallas de Arquitectura de Seguridad MCP

### Vulnerabilidades Críticas de MCP

Más allá del vector XSS, esta evaluación reveló brechas fundamentales de seguridad en la implementación MCP:

```bash
┌─ EVALUACIÓN DE SEGURIDAD MCP ────────────────────────────────────────────────┐
│                                                                               │
│ Autenticación:         ❌ NINGUNA - Acceso anónimo a todas las herramientas  │
│ Autorización:          ❌ NINGUNA - Sin modelo de permisos implementado      │
│ Limitación de Tasa:    ❌ NINGUNA - Ejecuciones ilimitadas de herramientas   │
│ Registro de Auditoría: ❌ NINGUNA - Sin seguimiento de actividad             │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

**Problemas Clave del Protocolo MCP:**
- **Modelo de Confianza Implícita**: Los clientes MCP ejecutan automáticamente cualquier herramienta proporcionada por el servidor sin validación
- **Exposición Irrestricta de Herramientas**: Capacidades de deployment públicas accesibles sin credenciales
- **Generación Persistente de Recursos**: Las herramientas crean recursos públicos sin mecanismos de limpieza

**Escenarios de Ataque:**
- Compromiso de infraestructura corporativa a través de servidores MCP desprotegidos
- Contaminación de cadena de suministro vía explotación de dominios confiables
- Movimiento lateral a través de redes de herramientas MCP conectadas

El ecosistema MCP actualmente carece de estándares de seguridad establecidos, frameworks de testing y monitoreo operacional - haciendo que los servicios sean vulnerables a ataques especializados que las medidas tradicionales de seguridad web no abordan.

---

*Este análisis se presenta únicamente con fines educativos y de investigación en seguridad web. La evaluación se realizó utilizando MCPwn siguiendo principios éticos de testing de seguridad.*
