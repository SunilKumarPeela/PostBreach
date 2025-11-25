# 🔐 EntangleX – Post-Breach Data Control (Classical Prototype)

> _“What if, even after a data breach, you could still control what the attacker sees?”_

EntangleX is a **quantum-inspired, classical prototype** that demonstrates
**post-breach data control** across three systems:

- 🪟 **Windows A** – where sensitive PII initially lives  
- 🐧 **Ubuntu A** – EntangleX core server + Entangle B watcher  
- 🐧 **Ubuntu B** – isolated **vault node** for the original data  

When a user **double-clicks** a sensitive file on Windows A:

1. The original file is **vaulted** one-way to Ubuntu B  
2. The file on Windows A is replaced with an **Entangle A** representation  
3. A replica **Entangle B** lives on Ubuntu A  
4. Edits to Entangle B automatically propagate back to Entangle A  
5. A live dashboard shows the entire flow in real time  

This repository contains the full classical prototype that backs the paper:

> **“A Quantum-Inspired Cybersecurity System for Post-Breach Data Control”**  
> (LaTeX source: `docs/paper/entanglex_ieee_paper.tex`)

---

## 🧩 High-Level Architecture

```text
          ┌────────────────────┐
          │      Windows A     │
          │  (Client Endpoint) │
          │  10.0.2.15         │
          └────────┬───────────┘
                   │ double-click
                   ▼
         Open-Entangle-Double.bat/.ps1
                   │ HTTP
                   ▼
          ┌────────────────────┐
          │      Ubuntu A      │
          │  EntangleX Core    │
          │  + B Watcher       │
          │ 10.255.84.134      │
          └────────┬───────────┘
                   │ one-way scp (data diode)
                   ▼
          ┌────────────────────┐
          │      Ubuntu B      │
          │    Vault Node      │
          │ 10.255.65.149      │
          └────────────────────┘

Entangle A – authoritative, on Windows A (but content controlled)

Entangle B – attacker-facing replica on Ubuntu A

Vault – original file safely stored on Ubuntu B
```

---

# ⚙️ EntangleX – Complete Setup Guide

This guide walks through deploying the full EntangleX prototype across:

- **Windows A** – Client endpoint where sensitive files reside  
- **Ubuntu A** – Core Server + Entangle B Watcher  
- **Ubuntu B** – Secure Vault Node (one-way storage)  

---

## 🖥️ 1. Ubuntu A — Core Server Setup

Clone the repository, then:

```bash
cd server
python3 -m venv venv
source venv/bin/activate
python entanglex_core_server.py

```
🔗 Default Bind Address
http://0.0.0.0:5000

📊 Dashboard URL
http://<Ubuntu-A-IP>:5000/dashboard

Example (your deployment):
http://10.255.84.134:5000/dashboard


