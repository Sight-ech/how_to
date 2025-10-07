# How to Secure a Web App

## 🧭 Introduction

In this tutorial, we’ll secure a simple web application hosted with **Nginx** on a Linux VPS.
The goal is to progressively add security layers — from reactive IP bans to HTTPS and network-level protection — and understand what each one defends against.

We’ll work with a **static site (HTML + Nginx)** so that all focus stays on the infrastructure and configuration.

**Main objectives:**
- Harden the Nginx web stack (TLS, headers, limits).
- Detect and ban abusive clients.
- Restrict system permissions.
- Protect against common attacks (XSS, SQLi, brute-force).
- Monitor and validate security events.

> 💡 The philosophy: *Defense in depth* — several lightweight protections combined are stronger than one heavy solution.

---
## Table of Contents

- [How to Secure a Web App](#how-to-secure-a-web-app)
  - [🧭 Introduction](#-introduction)
  - [Table of Contents](#table-of-contents)
  - [1. 🧱 Security Layers Overview](#1--security-layers-overview)
    - [🔍 Defense Chain](#-defense-chain)


---

## 1. 🧱 Security Layers Overview

Web application security is built in layers.
Each layer answers a specific question:
> “What if someone tries to attack me **here**?”

| Layer | Tool(s) | Goal |
|--------|----------|------|
| Reactive Ban | Fail2ban | Detect & block repetitive abusive IPs (SSH, Nginx logs). |
| Shared Reputation | CrowdSec | Faster bans + community threat intelligence. |
| Application Firewall | ModSecurity + OWASP CRS | Filter malicious payloads (SQLi, XSS, scanners). |
| App-Level Limits | Nginx rate limiting, connection limits | Prevent brute-force or flood from one IP. |
| System Hardening | SELinux, firewall rules | Restrict Nginx privileges, block floods early. |
| Transport Security | Certbot, HTTPS, headers | Encrypt and hide information. |
| Network Edge | Provider firewall, CDN | Drop large-scale attacks before reaching the VM. |

---

### 🔍 Defense Chain

```mermaid
flowchart TB
    subgraph WebApp["🕸️ Web Application (Nginx + HTML)"]
        Nginx -->|Logs| Fail2ban
        Nginx --> ModSecurity
        Nginx -->|CrowdSec Agent| CrowdSec
    end

    subgraph System["💻 System Layer"]
        CrowdSec --> nftables
        Fail2ban --> iptables
        Nginx --> SELinux
    end

    subgraph Network["🌐 Network Edge"]
        nftables --> ProviderFirewall
        ProviderFirewall --> Internet
    end

    Internet --> ProviderFirewall --> Nginx
````

We’ll start from the **innermost layer (reactive ban)** and move outward to the **network edge**, adding one layer of protection at a time.

Each section will be introduced by one or more guiding questions:

* *How can I detect brute-force attacks?*
* *How can I block bad IPs faster?*
* *How can I protect against injections or floods?*

Then we’ll answer with a short explanation, followed by practical commands and configuration examples.

---

Next: **Part 1 – Reactive Ban (Fail2ban setup and explanation)**
