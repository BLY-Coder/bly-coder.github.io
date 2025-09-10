---
layout: single
title: "MCPwn: Advanced Penetration Testing Tool for Model Context Protocol Servers"
excerpt: "Discover MCPwn, a comprehensive security testing tool designed for auditing Model Context Protocol servers. Learn how to identify vulnerabilities, perform automated assessments, and exploit common security flaws in MCP implementations."
date: 2025-09-08
classes: wide
header:
  teaser: /assets/images/xss-mcpwn/logo.jpeg
  icon: /assets/images/xss-mcpwn/logo.jpeg
  teaser_home_page: true
categories:
  - Security
  - Penetration Testing
  - MCP
tags:
  - MCP
  - Security Testing
  - Penetration Testing
  - Model Context Protocol
  - Vulnerability Assessment
---

## Introduction

As Model Context Protocol (MCP) adoption grows in AI applications, the need for specialized security testing tools becomes critical. **MCPwn** is a tool specifically designed to audit MCP servers, identify vulnerabilities, and assess security posture.

## What is MCPwn?

MCPwn is a specialized security testing tool built for Model Context Protocol servers. It combines semi-automated vulnerability detection with manual testing capabilities, making it easy to identify common security flaws such as access control bypasses, path traversal vulnerabilities, and information disclosure issues.

## Key Features

MCPwn offers a comprehensive set of features for MCP security testing:

- **Automated Server Discovery**: Quick enumeration of MCP tools, resources, and capabilities
- **Security Auditing**: Built-in risk assessment with vulnerability scoring
- **Manual Testing Support**: Direct tool execution and resource access testing
- **Proxy Integration**: Seamless integration with Burp Suite and other security proxies
- **Multi-target Support**: Concurrent testing of multiple MCP servers
- **LLM-Assisted Analysis**: AI-powered vulnerability prioritization and reporting
- **Baseline Comparison**: Track security changes over time

## Installation and Setup

Getting started with MCPwn is straightforward:

```bash
git clone https://github.com/BLY-Coder/MCPwn.git
cd MCPwn
pip install -r requirements.txt
```

For enhanced AI-assisted analysis, configure your Anthropic API key:

```bash
echo "ANTHROPIC_API_KEY=your_api_key_here" > .env
```

## Basic Usage and Server Enumeration

MCPwn's primary function is to enumerate and assess MCP servers. Let's start with basic reconnaissance:

```bash
python3 MCPwn.py localhost:9003
```

This command provides a comprehensive overview of the target server, displaying available tools, resources, prompts, and potential security concerns in an organized format:

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

## Automated Security Auditing

MCPwn includes a comprehensive audit mode that automatically identifies security issues:

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

The audit feature performs:
- **Risk Assessment**: Analyzes tool names and descriptions for dangerous operations
- **Permission Analysis**: Identifies improperly exposed resources
- **Schema Validation**: Checks for malformed input schemas
- **Error Probing**: Tests for information disclosure through error messages

## Manual Vulnerability Testing

Beyond automated scanning, MCPwn excels at manual security testing. Here are common attack scenarios:

### Testing Resource Access Controls

MCPwn can directly test resource access to identify authorization bypasses:

```bash
python3 MCPwn.py localhost:9003 internal://credentials
```

In vulnerable implementations, this might return sensitive information:

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

### Path Traversal Testing

MCPwn can test tools for path traversal vulnerabilities:

```bash
python3 MCPwn.py localhost:9003 read_file '{"filename": "../private/employee_salaries.txt"}'
```

Successful exploitation might reveal:

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

These examples demonstrate how MCPwn can quickly identify critical vulnerabilities like access control bypasses and path traversal flaws.

## LLM-Assisted Security Analysis

MCPwn can leverage AI for enhanced vulnerability analysis:

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

This feature provides:
- **Intelligent Risk Prioritization**: AI-powered severity assessment
- **Exploitation Guidance**: Specific recommendations for identified vulnerabilities
- **Comprehensive Reporting**: Detailed security analysis with remediation steps

## Proxy Integration and Traffic Analysis

MCPwn seamlessly integrates with security testing proxies:

```bash
# Route traffic through Burp Suite
python3 MCPwn.py --burp localhost:9003

# Use custom proxy configuration
python3 MCPwn.py --proxy http://127.0.0.1:8080 localhost:9003
```

Proxy integration enables:
- **Traffic Interception**: Capture and analyze MCP protocol communications
- **Request Modification**: Test custom payloads and edge cases
- **Session Analysis**: Understand authentication and state management
- **Documentation**: Generate detailed reports with traffic evidence

## Conclusion

MCPwn represents a significant advancement in MCP security testing capabilities. As Model Context Protocol adoption continues to grow across AI applications, having specialized tools for comprehensive security assessment becomes increasingly critical.

The tool's combination of automated vulnerability detection, manual testing capabilities, and AI-assisted analysis makes it an invaluable addition to any security professional's toolkit. From basic enumeration to complex multi-vector attacks, MCPwn provides the comprehensive framework needed to identify and address security vulnerabilities in MCP implementations.

Key benefits of using MCPwn include:

- **Comprehensive Coverage**: Tests for the full spectrum of MCP-specific vulnerabilities
- **Ease of Use**: Intuitive interface suitable for both beginners and experts
- **Advanced Features**: AI-powered analysis and proxy integration for sophisticated testing
- **Scalability**: Multi-target support for enterprise-level assessments

Whether you're a penetration tester, security researcher, or developer working with MCP implementations, MCPwn provides the specialized tools needed to ensure these systems remain secure and trustworthy. As MCP technology continues to evolve, tools like MCPwn will play a crucial role in maintaining the security posture of AI-powered applications.

## Resources

- **GitHub Repository**: [https://github.com/BLY-Coder/MCPwn](https://github.com/BLY-Coder/MCPwn)
- **Model Context Protocol Specification**: [https://spec.modelcontextprotocol.io/](https://spec.modelcontextprotocol.io/)
