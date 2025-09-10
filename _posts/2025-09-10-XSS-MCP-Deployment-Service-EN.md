---
layout: single
title: "Stored XSS Vulnerability in MCP HTML Deployment Service: Technical Analysis"
excerpt: "Detailed technical analysis of critical stored XSS vulnerability discovered using MCPwn in MCP deployment service. Includes step-by-step exploitation, system responses, and impact assessment."
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

## Introduction: Vulnerability Discovery

During a security assessment of **Model Context Protocol (MCP)** services, a **Stored Cross-Site Scripting (XSS) vulnerability** was identified in an HTML deployment service. The analysis was conducted using **MCPwn**, a specialized tool for testing MCP servers, revealing a security flaw that allows execution of arbitrary JavaScript code without any sanitization.

## Testing Environment Setup

### Tool Used: MCPwn
MCPwn is a specialized tool for security analysis of MCP servers that combines automatic enumeration capabilities with advanced manual testing.

```bash
# MCPwn Installation
git clone https://github.com/BLY-Coder/MCPwn.git
cd MCPwn
pip install -r requirements.txt
```

### Initial MCP Service Enumeration
First, enumeration of the MCP server was performed to identify available tools:

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
│  1. Deploy HTML content to XXXX Pages, return the public URL              │
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

### Interactive Testing Session Initiation
Once the deployment tool was identified, an interactive session was started:

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

## MCP Security Architecture Flaws

### Critical MCP Vulnerabilities

Beyond the XSS vector, this assessment revealed fundamental security gaps in the MCP implementation:

```bash
┌─ MCP SECURITY ASSESSMENT ────────────────────────────────────────────────────┐
│                                                                               │
│ Authentication:        ❌ NONE - Anonymous access to all tools               │
│ Authorization:         ❌ NONE - No permission model implemented             │
│ Rate Limiting:         ❌ NONE - Unlimited tool executions                   │
│ Audit Logging:         ❌ NONE - No activity tracking                        │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

**Key MCP Protocol Issues:**
- **Implicit Trust Model**: MCP clients automatically execute any server-provided tool without validation
- **Unrestricted Tool Exposure**: Public deployment capabilities accessible without credentials  
- **Persistent Resource Generation**: Tools create public resources without cleanup mechanisms

**Attack Scenarios:**
- Corporate infrastructure compromise through unprotected MCP servers
- Supply chain contamination via trusted domain exploitation
- Lateral movement through connected MCP tool networks

The MCP ecosystem currently lacks established security standards, testing frameworks, and operational monitoring - making services vulnerable to specialized attacks that traditional web security measures don't address.

---

*This analysis is presented solely for educational and web security research purposes. The evaluation was conducted using MCPwn following ethical security testing principles.*
