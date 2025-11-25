# EntangleX: Post-Breach Data Control (Classical Prototype)

This repository contains the **classical prototype** of **EntangleX**, a
quantum-inspired cybersecurity system for **post-breach data control**.

The system demonstrates how a sensitive file on **Windows A** can be:

1. Vaulted to an isolated **Ubuntu B** node via a one-way transfer,
2. Replaced locally with an **Entangle A** representation,
3. Mirrored as **Entangle B** on **Ubuntu A**,
4. Kept in sync via one-way **B→A** updates,
5. Visualized in real time via a live dashboard.

The prototype matches the implementation described in the paper:

> _"A Quantum-Inspired Cybersecurity System for Post-Breach Data Control"_

---

## Topology

- **Windows 11 (A)** — Client Endpoint  
  - IP: `10.0.2.15`  
  - Holds original PII file initially  
  - Runs `Open-Entangle-Double.ps1` via `Open-Entangle-Double.bat`

- **Ubuntu 3 (A)** — EntangleX Core  
  - IP: `10.255.84.134`  
  - Runs Flask `entanglex_core_server.py`  
  - Hosts:
    - EntangleX API
    - Entangle B repo
    - Live dashboard
    - B→A synchronization endpoint

- **Ubuntu 5 (B)** — Vault Node  
  - IP: `10.255.65.149`  
  - Receives one-way copies of original sensitive files  
  - Acts as an air-gapped **vault**

---

## Components

### 1. Core Server (Ubuntu A)

Located in: `server/entanglex_core_server.py`

Responsibilities:

- Accept original file uploads from Windows A
- Vault the original to Ubuntu B (via `scp`)
- Generate Entangle B files
- Serve Entangle A content dynamically
- Expose endpoints:
  - `POST /entangle`
  - `GET  /download_a`
  - `GET  /check_update`
  - `POST /update_from_b`
  - `GET  /metrics`
  - `GET  /events`
  - `GET  /dashboard`

### 2. Entangle B Watcher (Ubuntu A)

Located in: `watcher/entanglex_b_watcher.py`

Responsibilities:

- Watch `~/entangler/repo/B` for changes to `*_B.*`
- Debounce rapid edits
- Send updated B files to:
  - `POST /update_from_b` on the core server

### 3. Windows Double-Click Integration

Located in: `windows/`  
Files:

- `Open-Entangle-Double.ps1`  
  - Handles first-time entanglement and polling
  - Replaces the original file with Entangle A
  - Polls `/check_update` and pulls new versions of A

- `Open-Entangle-Double.bat`  
  - Used as the default **“Open with”** action for the target file type
  - Launches the PowerShell script hidden

---

## Setup Instructions

### 0. Clone the Repo

```bash
git clone https://github.com/<your-username>/entanglex-post-breach-data-control.git
cd entanglex-post-breach-data-control
