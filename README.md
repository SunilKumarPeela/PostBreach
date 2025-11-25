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

---

### 🔁 2. Ubuntu A — Entangle B Watcher

The watcher syncs modifications from Entangle B → Entangle A → Windows A.

Start it using the same virtual environment:

```bash
cd watcher
source ../server/venv/bin/activate
python entanglex_b_watcher.py
```
🧠 Watcher Responsibilities

Monitors: ~/entangler/repo/B

Detects changes to: *_B.*

Debounces rapid edits (VSCode, nano, notepad temp writes)

Pushes updates to:
```
POST /update_from_b
```

---

### 🔐 3. Ubuntu B — Vault Node Setup

Create the secure vault directory:
```
sudo mkdir -p /srv/entangle-vault
sudo chown vaultbot:vaultbot /srv/entangle-vault
```
🔑 Configure One-Way SSH (Ubuntu A → Ubuntu B)

Ensure Ubuntu A can securely push files into the vault:
```
ssh -i ~/.ssh/entangle_b_key vaultbot@10.255.65.149

```
✔️ This is required for pushing the original sensitive file into the encrypted vault.
---
### 🪟 4. Windows A — Double-Click Activation

Copy the entire windows/ folder into:
```
C:\Entangle\
```

Set Open-Entangle-Double.bat as the default “Open with” handler for your file type:

Example:
Right-click → Open With → Choose Another App → Browse… →
select:

C:\Entangle\Open-Entangle-Double.bat
---

---

## 📜 Core Scripts in This Prototype

This section describes the four main scripts that power the EntangleX classical prototype:

- 🧠 EntangleX Core API + Dashboard  
- 👁️ Entangle B Watcher  
- 🪟 Windows PowerShell integration  
- 🪟 Windows Batch double-click launcher  

Each script lives in its own folder and has a clear, single responsibility.

---
###1 entanglex_core_server.py
```bash
#!/usr/bin/env python3
# entangle_core.py  — core + live dashboard (SSE + D3)
from flask import Flask, request, send_file, jsonify, Response, render_template_string
from flask_cors import CORS
from werkzeug.utils import secure_filename
import os, io, json, time, uuid, secrets, string, subprocess, threading
from glob import glob
from queue import Queue, Empty

# --------------------------- fan-out targets (Ubuntu B, etc.) ---------------------------
KEEP_LOCAL_VAULT_COPY = False
VAULT_TARGETS = [
    {
        "enabled": True,
        "host":   "10.255.69.149",                 # Ubuntu B
        "user":   "vaultbot",
        "destdir":"/srv/entangle-vault",
        "ssh_key":os.path.expanduser("~/.ssh/entangle_b_key"),
        "label":  "Ubuntu B (Vault)"
    },
    # Add more vaults if you want
    # {"enabled": True, "host":"10.0.0.10", "user":"vaultbot", "destdir":"/srv/entangle-vault", "ssh_key": "~/.ssh/key2", "label":"Vault-C"}
]

# -------------------------------- directories --------------------------------
BASE  = os.path.expanduser("~/entangler")
VAULT = os.path.join(BASE, "vault")
REPOB = os.path.join(BASE, "repo", "B")
os.makedirs(VAULT, exist_ok=True)
os.makedirs(REPOB, exist_ok=True)

# --------------------------------- app ---------------------------------------
app = Flask(__name__)
CORS(app)

# ------------------------------- file meta utils -----------------------------
def meta_path(fid: str) -> str:
    return os.path.join(BASE, f"{fid}.meta.json")

def write_meta(fid: str, m: dict) -> None:
    with open(meta_path(fid), "w", encoding="utf-8") as f:
        json.dump(m, f, indent=2)

def read_meta(fid: str) -> dict | None:
    p = meta_path(fid)
    if os.path.exists(p):
        with open(p, "r", encoding="utf-8") as f:
            return json.load(f)
    return None

def list_all_meta() -> list[dict]:
    items = []
    for p in glob(os.path.join(BASE, "*.meta.json")):
        try:
            with open(p, "r", encoding="utf-8") as f:
                m = json.load(f)
                items.append(m)
        except Exception:
            pass
    # newest first
    items.sort(key=lambda x: x.get("updated", 0), reverse=True)
    return items

# ------------------------------- data helpers --------------------------------
def make_random_creds(count=20, uname_len=8, pwd_len=12) -> str:
    import secrets, string
    alph = string.ascii_lowercase
    pwd_chars = string.ascii_letters + string.digits + "!@#$%^&*()-_=+"
    lines = []
    for _ in range(count):
        uname = ''.join(secrets.choice(alph) for _ in range(uname_len))
        pwd   = ''.join(secrets.choice(pwd_chars) for _ in range(pwd_len))
        lines.append(f"{uname}:{pwd}")
    return "\n".join(lines) + "\n"

def scp_one(local_path: str, dest_name: str, t: dict) -> str | None:
    if not t.get("enabled", True):
        return None
    try:
        remote = f'{t["user"]}@{t["host"]}:{t["destdir"]}/{dest_name}'
        cmd = [
            "scp", "-i", os.path.expanduser(t["ssh_key"]),
            "-o", "StrictHostKeyChecking=accept-new",
            local_path, remote
        ]
        subprocess.check_call(cmd)
        app.logger.info(f"[vault] pushed {dest_name} -> {t['host']}:{t['destdir']}")
        return f'{t["destdir"]}/{dest_name}'
    except Exception as e:
        app.logger.warning(f"[vault] SCP to {t.get('host')} failed: {e}")
        return None

def fanout_to_vaults(local_path: str, dest_name: str) -> list[str]:
    results = []
    for t in VAULT_TARGETS:
        r = scp_one(local_path, dest_name, t)
        if r: results.append(r)
    return results

def render_A_text(meta: dict) -> str:
    """A := current B (stream on demand; we do not store A)."""
    b_path = meta.get("b_path")
    if b_path and os.path.exists(b_path):
        with open(b_path, "r", encoding="utf-8", errors="ignore") as f:
            return f.read()
    # fallback for very first time
    return make_random_creds()

# ------------------------------- SSE bus -------------------------------------
clients: list[Queue] = []

def push_event(kind: str, payload: dict):
    ev = {"kind": kind, "ts": int(time.time()), **payload}
    for q in list(clients):
        try:
            q.put_nowait(ev)
        except Exception:
            pass

@app.get("/events")
def sse_stream():
    q = Queue(maxsize=100)
    clients.append(q)

    def gen():
        # hello burst: dashboard can render immediately
        yield f"event: hello\ndata: {json.dumps({'ok': True, 'now': int(time.time())})}\n\n"
        try:
            while True:
                try:
                    ev = q.get(timeout=30)
                    yield f"event: {ev['kind']}\ndata: {json.dumps(ev)}\n\n"
                except Empty:
                    # keep-alive
                    yield f"event: keepalive\ndata: {{\"ts\": {int(time.time())}}}\n\n"
        finally:
            try: clients.remove(q)
            except ValueError: pass

    return Response(gen(), mimetype="text/event-stream")

# --------------------------------- API ---------------------------------------
@app.get("/health")
def health():
    return jsonify({"ok": True})

@app.post("/entangle")
def entangle():
    if "file" not in request.files:
        return jsonify({"ok": False, "error": "no file"}), 400

    up = request.files["file"]
    client_path = request.form.get("client_path", "")
    client_filename = secure_filename(up.filename or "file")
    name, ext = os.path.splitext(client_filename)
    if not ext: ext = ".txt"

    fid = str(uuid.uuid4())
    ts  = int(time.time())

    # 1) Vault original locally
    vault_name = f"{name}_{ts}{ext}"
    local_vault_path = os.path.join(VAULT, vault_name)
    up.save(local_vault_path)

    # 1b) Fan-out to vaults
    vault_paths = fanout_to_vaults(local_vault_path, vault_name)
    if KEEP_LOCAL_VAULT_COPY or not vault_paths:
        vault_paths.insert(0, local_vault_path)
    else:
        try: os.remove(local_vault_path)
        except OSError: pass

    # 2) Create ONLY B (A is streamed from B)
    b_path = os.path.join(REPOB, f"{fid}_B{ext}")
    with open(b_path, "w", encoding="utf-8") as wf:
        wf.write(make_random_creds())

    meta = {
        "file_id": fid,
        "client_path": client_path,
        "client_filename": client_filename,
        "ext": ext,
        "b_path": b_path,
        "vault_paths": vault_paths,
        "version": 1,
        "updated": ts,
    }
    write_meta(fid, meta)

    # push to dashboard
    push_event("entangled", {
        "file_id": fid,
        "filename": client_filename,
        "client_path": client_path,
        "b_path": b_path,
        "vault_paths": vault_paths,
        "version": 1
    })

    return jsonify(meta)

@app.get("/download_a")
def download_a():
    fid = request.args.get("file_id", "")
    m = read_meta(fid)
    if not m:
        return jsonify({"ok": False, "error": "unknown file_id"}), 404
    text = render_A_text(m)
    bio  = io.BytesIO(text.encode("utf-8"))
    return send_file(
        bio, mimetype="text/plain", as_attachment=False,
        download_name=m.get("client_filename", "entangle_A.txt")
    )

@app.get("/check_update")
def check_update():
    fid = request.args.get("file_id", "")
    client_seen = int(request.args.get("last_updated", "0"))
    m = read_meta(fid)
    if not m:
        return jsonify({"ok": False, "error": "unknown file_id"}), 404
    return jsonify({"ok": True, "update": m["updated"] > client_seen, "updated": m["updated"]})

@app.post("/update_from_b")
def update_from_b():
    fid = request.form.get("file_id", "")
    m = read_meta(fid)
    if not m:
        return jsonify({"ok": False, "error": "unknown file_id"}), 404
    if "new_b_file" not in request.files:
        return jsonify({"ok": False, "error": "no B file"}), 400

    up  = request.files["new_b_file"]
    b_path = m["b_path"]
    tmp = b_path + ".tmp"
    up.save(tmp)
    os.replace(tmp, b_path)

    m["version"] = int(m.get("version", 1)) + 1
    m["updated"] = int(time.time())
    write_meta(fid, m)

    # live pulse
    push_event("b_updated", {
        "file_id": fid,
        "version": m["version"],
        "b_path": b_path
    })

    return jsonify({"ok": True, "file_id": fid, "new_version": m["version"]})

@app.get("/metrics")
def metrics():
    items = list_all_meta()
    return jsonify({
        "ok": True,
        "count": len(items),
        "files": [{
            "file_id": m.get("file_id"),
            "client_filename": m.get("client_filename"),
            "client_path": m.get("client_path"),
            "version": m.get("version"),
            "updated": m.get("updated"),
            "b_path": m.get("b_path"),
            "vault_paths": m.get("vault_paths", [])
        } for m in items]
    })

# ------------------------------ Dashboard UI --------------------------------
DASHBOARD_HTML = r"""
<!doctype html>
<html>
<head>
  <meta charset="utf-8"/>
  <title>Entangle Live</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdn.jsdelivr.net/npm/dayjs@1/dayjs.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/dayjs@1/plugin/relativeTime.js"></script>
  <script src="https://d3js.org/d3.v7.min.js"></script>
  <style>
    .pulse { filter: drop-shadow(0 0 6px #22d3ee); }
    .rowflash { animation: flash 1s ease; }
    @keyframes flash { 0%{background:#0ea5e94d;} 100%{background:transparent;} }
  </style>
</head>
<body class="bg-slate-900 text-slate-100">
  <div class="max-w-7xl mx-auto p-6">
    <div class="flex items-center justify-between mb-4">
      <h1 class="text-3xl font-bold tracking-tight">Entangle — Live Flow</h1>
      <span id="status" class="text-sm text-slate-400">connecting…</span>
    </div>

    <!-- Graph -->
    <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
      <div class="bg-slate-800/60 rounded-2xl p-4 ring-1 ring-white/10">
        <h2 class="font-semibold mb-2">Live Topology</h2>
        <svg id="graph" class="w-full" style="height:420px"></svg>
        <p class="text-xs text-slate-400 mt-2">Glowing pulses show B→A updates and Vault fan-out.</p>
      </div>

      <!-- Table -->
      <div class="bg-slate-800/60 rounded-2xl p-4 ring-1 ring-white/10">
        <div class="flex items-center justify-between mb-2">
          <h2 class="font-semibold">Files</h2>
          <button id="refresh" class="px-3 py-1 rounded-md bg-cyan-500/20 hover:bg-cyan-500/30 ring-1 ring-cyan-400/40 text-cyan-200 text-sm">Refresh</button>
        </div>
        <div class="overflow-auto max-h-[380px]">
          <table class="w-full text-sm">
            <thead class="text-slate-400">
              <tr>
                <th class="text-left py-1">File</th>
                <th class="text-left">file_id</th>
                <th class="text-left">Version</th>
                <th class="text-left">Last update</th>
              </tr>
            </thead>
            <tbody id="rows"></tbody>
          </table>
        </div>
      </div>
    </div>
  </div>

<script>
dayjs.extend(dayjs_plugin_relativeTime)

const rowsEl = document.querySelector('#rows')
const statusEl = document.querySelector('#status')

// --- Graph setup
const svg = d3.select('#graph')
const width = document.querySelector('#graph').clientWidth
const height = 420

const nodes = [
  {id: 'Windows A', type: 'client'},
  {id: 'Ubuntu A (B)', type: 'server'},
  {% for v in vaults %}
  {id: '{{v}}', type: 'vault'},
  {% endfor %}
]
const links = [
  {source: 'Windows A', target: 'Ubuntu A (B)'},
  {% for v in vaults %}
  {source: 'Ubuntu A (B)', target: '{{v}}'},
  {% endfor %}
]

const link = svg.append('g').selectAll('line')
  .data(links).enter().append('line')
  .attr('stroke', '#64748b').attr('stroke-width', 1.5)

const node = svg.append('g').selectAll('circle')
  .data(nodes).enter().append('circle')
  .attr('r', d => d.type==='client' ? 14 : 10)
  .attr('fill', d => d.type==='client' ? '#22d3ee' : d.type==='server' ? '#a78bfa' : '#34d399')

const label = svg.append('g').selectAll('text')
  .data(nodes).enter().append('text')
  .attr('fill', '#e2e8f0').attr('font-size', 12).attr('dy', -18)
  .text(d => d.id)

const sim = d3.forceSimulation(nodes)
  .force('link', d3.forceLink(links).id(d=>d.id).distance(150))
  .force('charge', d3.forceManyBody().strength(-300))
  .force('center', d3.forceCenter(width/2, height/2))
  .on('tick', () => {
    link.attr('x1', d=>d.source.x).attr('y1', d=>d.source.y)
        .attr('x2', d=>d.target.x).attr('y2', d=>d.target.y)
    node.attr('cx', d=>d.x).attr('cy', d=>d.y)
    label.attr('x', d=>d.x).attr('y', d=>d.y)
  })

function pulse(from, to){
  link.filter(d=> (d.source.id===from && d.target.id===to))
    .attr('stroke', '#22d3ee').attr('stroke-width', 5).classed('pulse', true)
    .transition().duration(800)
    .attr('stroke', '#64748b').attr('stroke-width', 1.5).on('end', function(){ d3.select(this).classed('pulse', false) })
}

// --- Table
function upsertRow(f){
  const id = `row-${f.file_id}`
  let tr = document.getElementById(id)
  const ago = dayjs(f.updated*1000).fromNow()
  if(!tr){
    tr = document.createElement('tr')
    tr.id = id
    tr.innerHTML = `
      <td class="py-1 pr-2">${f.client_filename||'(unnamed)'}</td>
      <td class="pr-2 text-slate-400 text-xs">${f.file_id}</td>
      <td class="pr-2" data-v="v"></td>
      <td class="text-slate-300" data-v="u"></td>`
    rowsEl.appendChild(tr)
  }
  tr.querySelector('[data-v="v"]').textContent = f.version
  tr.querySelector('[data-v="u"]').textContent = ago
  tr.classList.remove('rowflash'); void tr.offsetWidth; tr.classList.add('rowflash')
}

async function loadMetrics(){
  const res = await fetch('/metrics'); const j = await res.json()
  if(j.ok){
    rowsEl.innerHTML = ''
    j.files.forEach(upsertRow)
  }
}
document.querySelector('#refresh').onclick = loadMetrics
loadMetrics(); setInterval(loadMetrics, 10000)

// --- Live SSE
const es = new EventSource('/events')
es.addEventListener('hello', e => { statusEl.textContent = 'live' })
es.addEventListener('b_updated', e => {
  const d = JSON.parse(e.data)
  pulse('Ubuntu A (B)', 'Windows A')
  {% for v in vaults %}
  pulse('Ubuntu A (B)', '{{v}}')
  {% endfor %}
  // optimistic row bump
  upsertRow({file_id:d.file_id, version:d.version, updated: Math.floor(Date.now()/1000)})
})
es.addEventListener('entangled', e => {
  const d = JSON.parse(e.data)
  pulse('Windows A', 'Ubuntu A (B)')
  {% for v in vaults %}
  pulse('Ubuntu A (B)', '{{v}}')
  {% endfor %}
  upsertRow({file_id:d.file_id, client_filename:d.filename, version:d.version, updated: Math.floor(Date.now()/1000)})
})
es.addEventListener('keepalive', e => { /* no-op */ })
es.onerror = () => statusEl.textContent = 'reconnecting…'
</script>
</body>
</html>
"""

@app.get("/dashboard")
def dashboard():
    # pass vault labels to render nodes
    vault_labels = [t.get("label") or f"{t['host']} (Vault)" for t in VAULT_TARGETS if t.get("enabled", True)]
    return render_template_string(DASHBOARD_HTML, vaults=vault_labels)

# --------------------------------- main --------------------------------------
if __name__ == "__main__":
    # Run Flask (dev)
    app.run(host="0.0.0.0", port=5000, threaded=True)

```
---
###2  entanglex_b_watcher.py
```bash
#!/usr/bin/env python3
"""
Watches ~/entangler/repo/B for edits to any *_B.* file and pushes the
new content to the entangler core so Windows A updates in place.

Deps: pip install watchdog requests
Run : python3 ~/entangler/watcher.py
"""

import os, re, time, threading, requests
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

# --- CONFIG ---
ENTANGLER_URL = os.environ.get("ENTANGLER_URL", "http://127.0.0.1:5000")
WATCH_DIR     = os.path.expanduser("~/entangler/repo/B")
DEBOUNCE_SEC  = 1.2

# ignore temp/swap files editors create
IGNORE_PATTERNS = (".swp", ".swx", "~", ".tmp", ".partial", ".crdownload")

# file id extractor: 36-char uuid then "_B"
FID_RE = re.compile(r"(?P<fid>[0-9a-fA-F-]{36})_B", re.IGNORECASE)

# timers per-path for debounce
_timers: dict[str, threading.Timer] = {}

def extract_fid(path: str) -> str | None:
    m = FID_RE.search(os.path.basename(path))
    return m.group("fid") if m else None

def should_ignore(path: str) -> bool:
    name = os.path.basename(path)
    if not name: return True
    if "_B" not in name: return True
    return any(name.endswith(sfx) for sfx in IGNORE_PATTERNS)

def push_update(b_path: str, fid: str) -> None:
    """POST the changed B to /update_from_b"""
    try:
        with open(b_path, "rb") as f:
            files = {"new_b_file": f}
            data  = {"file_id": fid}
            r = requests.post(f"{ENTANGLER_URL}/update_from_b",
                              data=data, files=files, timeout=15)
        if r.ok:
            print(f"[B→A] OK  fid={fid} -> {r.status_code} {r.json().get('new_version')=}")
        else:
            print(f"[B→A] ERR fid={fid} -> {r.status_code} {r.text[:200]}")
    except Exception as e:
        print(f"[B→A] EXC fid={fid} path={b_path}: {e}")

class BHandler(FileSystemEventHandler):
    def _schedule(self, path: str):
        if should_ignore(path): return
        fid = extract_fid(path)
        if not fid: return
        # debounce this file
        if (t := _timers.get(path)): t.cancel()
        _timers[path] = threading.Timer(DEBOUNCE_SEC, push_update, args=(path, fid))
        _timers[path].start()
        print(f"[WATCH] scheduled push for {fid} in {DEBOUNCE_SEC}s")

    def on_created(self, e):
        if not e.is_directory: self._schedule(e.src_path)

    def on_modified(self, e):
        if not e.is_directory: self._schedule(e.src_path)

    def on_moved(self, e):
        if not e.is_directory: self._schedule(e.dest_path)

def main():
    os.makedirs(WATCH_DIR, exist_ok=True)
    print(f"[INIT] Watching all B files in {WATCH_DIR}")
    print(f"[INIT] Sending updates to {ENTANGLER_URL}/update_from_b")
    obs = Observer()
    obs.schedule(BHandler(), WATCH_DIR, recursive=False)
    obs.start()
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        obs.stop()
    obs.join()

if __name__ == "__main__":
    main()
```
---
###3 Open-Entangle-Double.ps1
```bash
param(
  [Parameter(Mandatory=$true)][string]$Path,
  [string]$EntanglerUrl = "http://10.255.84.134:5000"  # <-- set to Ubuntu A IP:port
)

function AtomicReplace([string]$Target, [string]$Tmp) {
  if (Test-Path $Tmp) {
    if (Test-Path $Target) { Remove-Item $Target -Force }
    Move-Item -Path $Tmp -Destination $Target -Force
  }
}

if (-not (Test-Path $Path)) {
  Write-Host "[ENTANGLE-DOUBLE] File not found: $Path" -ForegroundColor Red
  exit 1
}

$metaPath = "$Path.entangle.json"
$meta = $null
if (Test-Path $metaPath) {
  try { $meta = Get-Content $metaPath | ConvertFrom-Json } catch { $meta = $null }
}

# 1) First-time entangle: upload and replace with A
if (-not $meta) {
  Write-Host "[ENTANGLE-DOUBLE] First-time entangle: $Path"
  Add-Type -AssemblyName System.Net.Http
  $client = New-Object System.Net.Http.HttpClient
  $mp = New-Object System.Net.Http.MultipartFormDataContent
  $fs = [System.IO.File]::OpenRead($Path)
  $filePart = New-Object System.Net.Http.StreamContent($fs)
  $filePart.Headers.ContentType = [System.Net.Http.Headers.MediaTypeHeaderValue]::Parse("application/octet-stream")
  $mp.Add($filePart, "file", (Split-Path $Path -Leaf))
  $mp.Add((New-Object System.Net.Http.StringContent($Path)), "client_path")

  try {
    $resp = $client.PostAsync("$EntanglerUrl/entangle", $mp).Result
    if (-not $resp.IsSuccessStatusCode) { throw "Entangler: $($resp.StatusCode) $($resp.ReasonPhrase)" }
    $meta = $resp.Content.ReadAsStringAsync().Result | ConvertFrom-Json
  } finally {
    $fs.Dispose(); $mp.Dispose(); $client.Dispose()
  }

  $tmp = [System.IO.Path]::GetTempFileName()
  Invoke-WebRequest -Uri "$EntanglerUrl/download_a?file_id=$($meta.file_id)" -OutFile $tmp -UseBasicParsing
  AtomicReplace -Target $Path -Tmp $tmp
  @{ file_id = $meta.file_id; last_updated = $meta.updated } | ConvertTo-Json | Set-Content $metaPath -Encoding UTF8
} else {
  Write-Host "[ENTANGLE-DOUBLE] Already entangled: $Path"
}

# 2) Open and POLL while Notepad is open (no background jobs required)
$notepad = Start-Process notepad.exe -ArgumentList @("$Path") -PassThru

while (-not $notepad.HasExited) {
  try {
    $m = Get-Content $metaPath | ConvertFrom-Json
    $file_id = $m.file_id
    $last    = [int64]$m.last_updated

    $check = Invoke-RestMethod -Uri "$EntanglerUrl/check_update?file_id=$file_id&last_updated=$last" -Method Get -TimeoutSec 8
    if ($check.ok -and $check.update) {
      $tmp2 = [System.IO.Path]::GetTempFileName()
      Invoke-WebRequest -Uri "$EntanglerUrl/download_a?file_id=$file_id" -OutFile $tmp2 -UseBasicParsing
      AtomicReplace -Target $Path -Tmp $tmp2
      $m.last_updated = $check.updated
      $m | ConvertTo-Json | Set-Content $metaPath -Encoding UTF8
      Write-Host "[ENTANGLE-DOUBLE] Pulled update at $(Get-Date -f HH:mm:ss)"
    }
  } catch {
    # ignore transient errors
  }
  Start-Sleep -Seconds 3
}

# optional: one last quick check after Notepad closes
try {
  $m = Get-Content $metaPath | ConvertFrom-Json
  $check = Invoke-RestMethod -Uri "$EntanglerUrl/check_update?file_id=$($m.file_id)&last_updated=$([int64]$m.last_updated)" -Method Get -TimeoutSec 8
  if ($check.ok -and $check.update) {
    $tmp3 = [System.IO.Path]::GetTempFileName()
    Invoke-WebRequest -Uri "$EntanglerUrl/download_a?file_id=$($m.file_id)" -OutFile $tmp3 -UseBasicParsing
    AtomicReplace -Target $Path -Tmp $tmp3
    $m.last_updated = $check.updated
    $m | ConvertTo-Json | Set-Content $metaPath -Encoding UTF8
  }
} catch {}
```
---
###4 Open-Entangle-Double.bat
```bash
@echo off
setlocal
if "%~1"=="" exit /b 1

rem --- absolute PowerShell path ---
set "PS=%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe"

rem --- polling script that opens Notepad and pulls updates while it’s open ---
set "SCRIPT=C:\Entangle\Open-Entangled-Double.ps1"

rem --- launch hidden; PS script handles first-time upload + continuous polling ---
start "" "%PS%" -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden ^
  -File "%SCRIPT%" -Path "%~f1" -EntanglerUrl "http://10.255.84.134:5000"

exit /b
```
