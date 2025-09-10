---
layout: single
title: "MCPwn: Herramienta Avanzada de Pruebas de Penetración para Servidores Model Context Protocol"
excerpt: "Descubre MCPwn, una herramienta integral de pruebas de seguridad diseñada para auditar servidores Model Context Protocol. Aprende cómo identificar vulnerabilidades, realizar evaluaciones automatizadas y explotar fallas de seguridad comunes en implementaciones MCP."
date: 2025-09-08
classes: wide
header:
  teaser: /assets/images/mcpwn/logo.png
  icon: /assets/images/mcpwn/logo.pngeg
  teaser_home_page: true
  
categories:
  - Seguridad
  - Pruebas de Penetración
  - MCP
tags:
  - MCP
  - Pruebas de Seguridad
  - Pentesting
  - Model Context Protocol
  - Evaluación de Vulnerabilidades
---

## Introducción

A medida que la adopción del Model Context Protocol (MCP) crece en aplicaciones de IA, la necesidad de herramientas especializadas de pruebas de seguridad se vuelve crítica. **MCPwn** es una tool diseñada específicamente para auditar servidores MCP, identificar vulnerabilidades y evaluar la postura de seguridad. 

## ¿Qué es MCPwn?

MCPwn es una herramienta especializada de pruebas de seguridad construida para servidores Model Context Protocol. Combina detección semi-automatizada de vulnerabilidades con capacidades de pruebas manuales, facilitando la identificación de fallas de seguridad comunes como bypasses de control de acceso, vulnerabilidades de path traversal y problemas de divulgación de información.

## Características Principales

MCPwn ofrece un conjunto integral de características para pruebas de seguridad MCP:

- **Descubrimiento Automatizado de Servidores**: Enumeración rápida de herramientas, recursos y capacidades MCP
- **Auditoría de Seguridad**: Evaluación de riesgos integrada con puntuación de vulnerabilidades
- **Soporte para Pruebas Manuales**: Ejecución directa de herramientas y pruebas de acceso a recursos
- **Integración con Proxies**: Integración perfecta con Burp Suite y otros proxies de seguridad
- **Soporte Multi-objetivo**: Pruebas concurrentes de múltiples servidores MCP
- **Análisis Asistido por LLM**: Priorización de vulnerabilidades y reportes asistidos por IA
- **Comparación de Línea Base**: Rastrea cambios de seguridad a lo largo del tiempo

## Instalación y Configuración

Comenzar con MCPwn es sencillo:

```bash
git clone https://github.com/BLY-Coder/MCPwn.git
cd MCPwn
pip install -r requirements.txt
```

Para análisis mejorado asistido por IA, configura tu clave API de Anthropic:

```bash
echo "ANTHROPIC_API_KEY=tu_clave_api_aqui" > .env
```

## Uso Básico y Enumeración de Servidores

La función principal de MCPwn es enumerar y evaluar servidores MCP. Comencemos con reconocimiento básico:

```bash
python3 MCPwn.py localhost:9003
```

Este comando proporciona una visión general completa del servidor objetivo, mostrando herramientas disponibles, recursos, prompts y posibles preocupaciones de seguridad en un formato organizado:

```bash
┌──────────────────────────────────────────────────────────────────────────────┐
│ HOST: http://localhost:9003                                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│ Server Name: Challenge 3 - Excessive Permission Scope                       │
└──────────────────────────────────────────────────────────────────────────────┘

┌─ TOOLS (2) ───────────────────────────────────────────────────────────────────┐
│  1. Read a file from the public directory.                                   │
│     Args: filename: Name of the file to read (e.g., 'welcome.txt')           │
│     → read_file '{"filename": ""}'                                           │
│                                                                               │
│  2. Search for files containing a specific keyword in the public directory.  │
│     Args: keyword: The keyword to search for                                 │
│     → search_files '{"keyword": ""}'                                         │
└──────────────────────────────────────────────────────────────────────────────┘

┌─ RESOURCES (2) ────────────────────────────────────────────────────────────────┐
│  1. get_public_files (List of public files available to all users)           │
│     → files://public                                                         │
│  2. get_private_files (RESTRICTED: List of confidential files - Admin acc... │
│     → internal://credentials                                                 │
└──────────────────────────────────────────────────────────────────────────────┘

┌─ PROMPTS ─────────────────────────────────────────────────────────────────────┐
│ No prompts available                                                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Auditoría de Seguridad Automatizada

MCPwn incluye un modo de auditoría integral que identifica automáticamente problemas de seguridad:

```bash
python3 MCPwn.py --audit localhost:9003
```

```bash
python3 MCPwn.py localhost:9010 --audit                                                        

=== MCP Quick Audit: http://localhost:9010 ===
Server: Challenge 10 - Multi-Vector Attack | Version: 1.13.1
Tools: 8 | Resources: 3 | Prompts: 0

Timings (ms): init=28 tools=6 resources=3 prompts=1

-- Tools (top risks) --
* malicious_check_system_status: score=5 risky_terms=['system'] schema_anomaly=True poison=True
* get_user_profile: score=4 risky_terms=['system'] schema_anomaly=False poison=True
* check_system_status: score=3 risky_terms=['system'] schema_anomaly=True poison=False
* get_config: score=2 risky_terms=['system'] schema_anomaly=False poison=False
* run_system_diagnostic: score=2 risky_terms=['run', 'system'] schema_anomaly=False poison=False

-- Error Probing Findings --
* get_config: Accepted invalid input without error
* get_config: Accepted invalid input without error
* process_user_input: Accepted invalid input without error
* process_user_input: Accepted invalid input without error
* authenticate: Accepted invalid input without error
* authenticate: Accepted invalid input without error
* get_user_profile: Accepted invalid input without error
* get_user_profile: Accepted invalid input without error
* run_system_diagnostic: Accepted invalid input without error
* check_system_status: Accepted invalid input without error
* malicious_check_system_status: Accepted invalid input without error
* analyze_log_file: Accepted invalid input without error
* analyze_log_file: Accepted invalid input without error
```

La función de auditoría realiza:
- **Evaluación de Riesgos**: Analiza nombres y descripciones de herramientas para operaciones peligrosas
- **Análisis de Permisos**: Identifica recursos inapropiadamente expuestos
- **Validación de Esquemas**: Verifica esquemas de entrada malformados
- **Sondeo de Errores**: Prueba divulgación de información a través de mensajes de error

## Pruebas Manuales de Vulnerabilidades

Más allá del escaneo automatizado, MCPwn sobresale en pruebas manuales de seguridad. Aquí hay escenarios de ataque comunes:

### Probando Controles de Acceso a Recursos

MCPwn puede probar directamente el acceso a recursos para identificar bypasses de autorización:

```bash
python3 MCPwn.py localhost:9003 internal://credentials
```

En implementaciones vulnerables, esto podría devolver información sensible:

```bash
┌─ RESOURCE CONTENT ────────────────────────────────────────────────────────────┐
│ [1] Text Content                                                           │
│     Type: text/plain                                                         │
│     Content:                                                                 │
│     Private Files (RESTRICTED):                                             │
│     employee_salaries.txt                                                   │
│     system_credentials.txt                                                  │
│     acquisition_plans.txt                                                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Pruebas de Path Traversal

MCPwn puede probar herramientas para vulnerabilidades de path traversal:

```bash
python3 MCPwn.py localhost:9003 read_file '{"filename": "../private/employee_salaries.txt"}'
```

La explotación exitosa podría revelar:

```bash
┌─ TOOL RESPONSE ───────────────────────────────────────────────────────────────┐
│ [1] Text Output                                                             │
├──────────────────────────────────────────────────────────────────────────────┤
│ CONFIDENTIAL: Employee Salary Information                                   │
│ CEO: $1,200,000/year                                                        │
│ CTO: $950,000/year                                                          │
│ Senior Engineers: $180,000-$250,000/year                                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

Estos ejemplos demuestran cómo MCPwn puede identificar rápidamente vulnerabilidades críticas como bypasses de control de acceso y fallas de path traversal.



## Análisis de Seguridad Asistido por LLM

MCPwn puede aprovechar IA para análisis mejorado de vulnerabilidades:

```bash
python3 MCPwn.py --audit --audit-llm localhost:9003
```

```bash
python3 MCPwn.py localhost:9003 --audit --audit-llm

=== MCP Quick Audit: http://localhost:9003 ===
Server: Challenge 3 - Excessive Permission Scope | Version: 1.13.1
Tools: 2 | Resources: 2 | Prompts: 0

Timings (ms): init=22 tools=4 resources=6 prompts=2

-- Tools (top risks) --

-- Error Probing Findings --
* read_file: Accepted invalid input without error
* read_file: Accepted invalid input without error
* search_files: Accepted invalid input without error
* search_files: Accepted invalid input without error


--- LLM Summary ---
## High-Signal Security Findings

### **CRITICAL Risk**
1. **Excessive Permission Scope** - Server exposes both public AND private credential resources
   - `files://public` (appropriate for tools)
   - `internal://credentials` marked "RESTRICTED: Admin access only"
   - **Impact**: Potential unauthorized access to confidential data

### **MEDIUM Risk**
2. **Poor Input Validation** - Tools accept invalid inputs without proper sanitization
   - Missing required fields only trigger Pydantic validation errors
   - No business logic validation for file access patterns
   - **Impact**: Potential for injection attacks or unintended file access

### **LOW Risk**
3. **Information Disclosure** - Error messages reveal internal structure
   - File system paths and validation schema exposed
   - **Impact**: Aids reconnaissance for attackers

## Prioritized Next Steps

1. **IMMEDIATE**: Remove `internal://credentials` resource or implement proper access controls
2. **HIGH**: Audit file access permissions - ensure tools can only access intended public directory
3. **MEDIUM**: Implement input sanitization beyond basic validation (path traversal protection)
4. **LOW**: Sanitize error messages to avoid information leakage

## Key Question
Why does a "public directory" server have access to restricted credential files? This suggests a fundamental architecture flaw.
-------------------
```

Esta característica proporciona:
- **Priorización Inteligente de Riesgos**: Evaluación de severidad asistida por IA
- **Guía de Explotación**: Recomendaciones específicas para vulnerabilidades identificadas
- **Reportes Integrales**: Análisis detallado de seguridad con pasos de remediación

## Integración con Proxies para Pruebas Avanzadas

MCPwn puede enrutar tráfico a través de Burp Suite para análisis detallado:

```bash
python3 MCPwn.py --burp localhost:9003 read_file '{"filename": "../private/employee_salaries.txt"}'
```

Esto te permite:
- Interceptar y modificar requests
- Analizar el tráfico HTTP/JSON-RPC subyacente
- Probar payloads adicionales a través del Repeater de Burp
- Documentar hallazgos con capturas de pantalla



## Conclusión

MCPwn representa un avance significativo en las capacidades de pruebas de seguridad MCP. A medida que la adopción del Model Context Protocol continúa creciendo en aplicaciones de IA, tener herramientas especializadas para evaluación integral de seguridad se vuelve cada vez más crítico.

La combinación de la herramienta de detección automatizada de vulnerabilidades, capacidades de pruebas manuales y análisis asistido por IA la convierte en una adición invaluable al toolkit de cualquier profesional de seguridad. Desde enumeración básica hasta ataques multi-vector complejos, MCPwn proporciona el framework integral necesario para identificar y abordar vulnerabilidades de seguridad en implementaciones MCP.

Beneficios clave de usar MCPwn incluyen:

- **Cobertura Integral**: Prueba todo el espectro de vulnerabilidades específicas de MCP
- **Facilidad de Uso**: Interfaz intuitiva adecuada tanto para principiantes como expertos
- **Características Avanzadas**: Análisis asistido por IA e integración con proxies para pruebas sofisticadas
- **Escalabilidad**: Soporte multi-objetivo para evaluaciones a nivel empresarial

Ya seas un tester de penetración, investigador de seguridad o desarrollador trabajando con implementaciones MCP, MCPwn proporciona las herramientas especializadas necesarias para asegurar que estos sistemas permanezcan seguros y confiables. A medida que la tecnología MCP continúa evolucionando, herramientas como MCPwn jugarán un papel crucial en mantener la postura de seguridad de aplicaciones impulsadas por IA.

## Recursos

- **Repositorio GitHub**: [https://github.com/BLY-Coder/MCPwn](https://github.com/BLY-Coder/MCPwn)
- **Especificación Model Context Protocol**: [https://spec.modelcontextprotocol.io/](https://spec.modelcontextprotocol.io/)

