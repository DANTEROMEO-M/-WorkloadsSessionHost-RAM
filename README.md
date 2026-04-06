# WorkloadManager v4.3 — Fix for WorkloadsSessionHost.exe RAM Hoarding on Copilot+ PCs

**By Dante / SafeAutentic**

---

## The Problem

If you own a Copilot+ PC (AMD Ryzen AI, Intel Lunar Lake, or Snapdragon X) running Windows 11 24H2 or 25H2, you may have noticed multiple `WorkloadsSessionHost.exe` processes silently consuming anywhere from **2GB to 8GB of RAM** while doing absolutely nothing visible.

These processes are part of Microsoft's Windows AI infrastructure — the backend that powers Recall, Click to Do, Live Captions, and other AI features. Windows spawns between 5 and 9 instances at startup and keeps them loaded in RAM indefinitely, pre-warming AI models so they respond instantly when called.

The problem: **they never release that memory on their own**, even when idle. On machines with 16GB of RAM, this alone can push memory usage above 60% before you even open a browser.

### What makes it worse on AMD Ryzen AI

On AMD Ryzen AI 300 series (Strix Point, XDNA2 NPU), there is a confirmed bug where the NPU shows **0% compute usage** but keeps **3–4GB reserved in shared memory**. The AI models are being loaded into system RAM and executed on CPU instead of being offloaded to the XDNA2 engine — completely defeating the purpose of having a dedicated NPU.

---

## The Investigation

Standard approaches don't work:

- **Killing processes directly** — Windows respawns them immediately via `SystemEventsBroker`
- **Disabling WSAIFabricSvc** — kills all AI features including Click to Do
- **Registry tweaks from the internet** — most are fabricated keys that Windows never reads
- **Identifying processes by CommandLine** — all instances share the exact same arguments:
  ```
  -ServerName:Microsoft.Windows.Private.Workloads.SessionHost
  ```
  Making it impossible to distinguish which instance belongs to which feature

After deep investigation including process hierarchy analysis, module inspection, and handle enumeration, the conclusion was clear: **there is no native Windows setting to control this behavior**. Microsoft designed these processes to stay loaded indefinitely for low-latency AI responses.

---

## The Solution

**WorkloadManager** is a lightweight PowerShell daemon that runs silently in the background and enforces a **60-second lifetime on every WorkloadsSessionHost.exe process**.

### How it works

```
Windows spawns a new WorkloadsSessionHost.exe
             ↓
WorkloadManager detects the new PID within 3 seconds
             ↓
Starts an individual 60-second countdown for that PID
             ↓
At 60 seconds → Stop-Process -Force → RAM freed
             ↓
If Windows spawns another one → cycle repeats
```

Each PID gets its own independent timer. If Click to Do spawns a process to analyze your screen, it gets its 60 seconds to complete the task — then it's terminated. Windows will spawn a fresh instance the next time you use Click to Do.

### Key technical decisions

- Uses `System.Collections.Generic.Dictionary[string,datetime]` instead of a standard PowerShell hashtable to avoid type casting issues with WMI's `uint32` ProcessId
- Scans every 3 seconds to catch new processes quickly
- Logs every detection and termination to Windows Event Log (`Application → WorkloadManager`) including MB freed per process
- Does not touch `WSAIFabricSvc` — Click to Do remains fully functional
- Disables Recall permanently via Group Policy registry keys

### Results

In testing, a single cleanup cycle freed over **4,350 MB** across 7 processes:

```
PID 12736 → 3,546 MB freed   ← largest offender
PID 14912 →   524 MB freed
PID 6644  →   402 MB freed
PID 15056 →   399 MB freed
PID 13428 →   308 MB freed
PID 6272  →   144 MB freed
PID 14956 →    27 MB freed
```

---

## Installation

### Requirements
- Windows 11 24H2 or 25H2
- Copilot+ PC (AMD Ryzen AI, Intel Lunar Lake, Snapdragon X)
- PowerShell as Administrator

### Steps

1. Download all 3 files into the same folder (e.g. `C:\Scripts\`)
   - `WorkloadManager.ps1`
   - `Instalar-WorkloadManager.ps1`
   - `Desinstalar-WorkloadManager.ps1`

2. Open Terminal as Administrator (`Win + X` → Terminal (Admin)) and run:
```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
& "C:\Scripts\Instalar-WorkloadManager.ps1"
```

3. Done. The script starts immediately and auto-launches on every Windows startup via `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`.

### Verify it's working
```powershell
Get-EventLog -LogName Application -Source WorkloadManager -Newest 20 | Select-Object TimeCreated, Message
```

You should see PIDs being detected and terminated with MB freed.

### Uninstall
```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
& "C:\Scripts\Desinstalar-WorkloadManager.ps1"
```

---

## What it does and doesn't do

| ✅ Does | ❌ Does not |
|---|---|
| Kill RAM-hoarding WorkloadsSessionHost processes | Break Click to Do |
| Run silently in background | Require third-party software |
| Start automatically with Windows | Modify system files |
| Log all activity to Event Viewer | Disable WSAIFabricSvc |
| Disable Recall permanently | Affect gaming or GPU performance |
| Free 2–6GB of RAM within 60s of boot | Survive if you manually end the PowerShell process* |

*If the background PowerShell process is killed, simply re-run the installer or restart Windows.

---

## Compatibility

Tested on:
- AMD Ryzen AI 5 340 / Radeon 840M — Windows 11 25H2
- Driver: IPU/XDNA2 `IpuMcdmDriver` (ipustack.sys) v32.0.203.297

Should work on any Copilot+ PC running Windows 11 24H2 or 25H2 with the `WindowsWorkload.Manager` package installed.

---

## Background

This fix was developed after extensive live debugging — including WMI process enumeration, module inspection, named pipe analysis, PnP device property queries, and NPU driver investigation — to find the root cause of why 7 identical processes were consuming 4+ GB of RAM on a brand new Copilot+ laptop.

Microsoft has not provided a native way to control this behavior. This script fills that gap until they do.

---

*WorkloadManager is open and free to use, share, and modify.*
