# Lab 5 — Nessus Vulnerability Scanning

> Deploying, running, and interpreting authenticated and unauthenticated vulnerability scans against Azure lab VMs using **Nessus Essentials** — the full scan → find → prioritise → remediate → verify lifecycle.

![Status](https://img.shields.io/badge/status-complete-brightgreen)
![Tool](https://img.shields.io/badge/tool-Nessus%20Essentials-00619F)
![Cloud](https://img.shields.io/badge/cloud-Microsoft%20Azure-0078D4)
![Cert Alignment](https://img.shields.io/badge/aligns%20with-Security%2B%20%C2%B7%20CySA%2B%20%C2%B7%20PenTest%2B-red)
![Cost](https://img.shields.io/badge/cost-%240-success)

---

## Overview

Every network has vulnerabilities. The real question is whether you find them before an attacker does. This lab builds that muscle end to end: I stood up a Nessus scanner, ran both an unauthenticated discovery scan and a credentialed scan against a Windows Server target in Azure, interpreted the CVSS-scored findings, remediated a real weakness, verified the fix with a re-scan, and exported a stakeholder-ready report.

The `scan → find → prioritise → remediate → verify` loop practiced here is the same workflow enterprise security teams run continuously — the only difference at scale is automation and volume.

---

## Architecture

![image alt](https://github.com/mccraym7/ad-labset-005/blob/80526a7f05355dc7e3781763b687d06ffa605afb/Lab5-Architecture-Diagram%5B1%5D.png)

*Nessus scanner, targets, and the scan-to-report workflow, deployed inside a single Azure resource group.*

| Component | Role in the lab |
|---|---|
| **Nessus Scanner VM** | Hosts the Nessus Essentials engine and web UI on port `8834`. Initiates all scan traffic. Requires outbound HTTPS to Tenable for activation and plugin updates. |
| **DC01 (Target)** | Windows Server 2025 domain controller. Primary scan target. Remote Registry and SMB enabled so credentialed scans can inspect it from the inside. |
| **Other Lab VMs** | Optional additional targets (workstation, Splunk) kept within the free 5-IP licence limit. |
| **Tenable Plugin Feed** | External Tenable service reached over outbound HTTPS (`TCP 443`) for the activation code and for downloading/updating scan plugins. |

---

## Objectives

- [x] Deploy and configure Nessus Essentials on an Azure VM
- [x] Register the scanner via `nessuscli.exe fetch --register` and complete the plugin build
- [x] Run an unauthenticated **basic network scan** (the external attacker's view)
- [x] Run a **credentialed scan** (the internal, authenticated view)
- [x] Read and interpret **CVSS scores** and severity ratings
- [x] Prioritise findings by exploitability, asset criticality, and exposure
- [x] **Remediate** a finding, **re-scan**, and **verify** the fix
- [x] Export an **executive** and a **detailed** vulnerability report

---

## Tech Stack

| Category | Details |
|---|---|
| **Scanner** | Nessus Essentials (free, up to 5 IPs) |
| **Cloud platform** | Microsoft Azure (Free Account) — or local VirtualBox |
| **Target OS** | Windows Server 2025 |
| **Scripting** | PowerShell (activation, service config, remediation) |
| **Scoring standard** | CVSS v3.x |

---

## Key Concepts Demonstrated

### Basic vs. Credentialed Scanning

An unauthenticated scan of a Windows Server might surface ~10 findings. A credentialed scan authenticates to the host and inspects it from the inside — OS patch levels, installed software versions, local misconfigurations, and registry settings — typically surfacing **50–150 findings**. Credentialed scanning is the standard for internal vulnerability management.

### Severity Interpretation (CVSS)

| Severity | CVSS | Meaning | Example |
|---|---|---|---|
| **Critical** | 9.0–10.0 | Remotely exploitable, little/no auth. Fix immediately. | EternalBlue (MS17-010) RCE |
| **High** | 7.0–8.9 | Significant impact. Fix within 7–14 days. | Unpatched RDP vulnerability |
| **Medium** | 4.0–6.9 | Requires conditions to exploit. Fix within 30 days. | Expired SSL cert / weak cipher |
| **Low** | 0.1–3.9 | Minimal impact. Fix in routine cycles. | Missing security headers |
| **Info** | 0 | Not a vulnerability — system information. | Open port / OS version detected |

---

## Selected Commands

**Register the scanner (elevated PowerShell — note the `&` call operator):**
```powershell
cd "C:\Program Files\Tenable\Nessus"
& ".\nessuscli.exe" fetch --register XXXX-XXXX-XXXX-XXXX-XXXX
& ".\nessuscli.exe" update --all
Restart-Service "Tenable Nessus"
```

**Enable Remote Registry on the target for credentialed scanning:**
```powershell
Set-Service -Name RemoteRegistry -StartupType Automatic
Start-Service RemoteRegistry
```

**Sample remediation — disable SMBv1 (a frequent Nessus finding):**
```powershell
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
```

---

## Remediation Workflow

The single most important step in vulnerability management is **verification** — an unverified fix is not a fix.

```
Find  →  Prioritise  →  Remediate  →  Re-scan  →  Verify
 │           │              │            │           │
 CVSS      severity +     patch /      re-run      finding
 score     exposure       config       scan        no longer
                          change                    appears
```

In this lab I selected a Medium/High finding, applied the documented Solution, re-ran the scan, and confirmed the finding was gone from the results.

---

## Results & Verification

| Check | Verification |
|---|---|
| Basic scan completed | Scan returned findings (a patched server still returns Info-level findings) |
| Credentialed scan completed | Finding count significantly higher than the unauthenticated scan |
| Remediation verified | At least one original finding absent after remediation + re-scan |
| Report exported | PDF opens showing findings with CVSS scores |

---

## Repository Contents

| File | Description |
|---|---|
| [`Lab5-Nessus-Vulnerability-Scanning.docx`](./Lab5-Nessus-Vulnerability-Scanning.docx) | Full step-by-step lab documentation with screenshots, commands, and troubleshooting |
| [`Lab5-Architecture-Diagram.png`](./Lab5-Architecture-Diagram.png) | Reference architecture diagram |
| `README.md` | This file |

---

## Troubleshooting Notes

Real issues I worked through, captured for anyone reproducing this lab:

| Symptom | Cause & Fix |
|---|---|
| `fetch --register` hangs and never returns | Outbound `443` to Tenable blocked. Check the Azure NSG / proxy; confirm the VM has internet egress. |
| PowerShell: term `nessuscli` not recognised | Missing `&` call operator or wrong directory. `cd` into the Nessus folder and prefix the quoted path with `&`. |
| Credentialed scan finds no more than basic scan | Authentication failed. Verify username / password / domain and that **Remote Registry** is running on the target. |
| `localhost:8834` shows a certificate warning | Expected — Nessus uses a self-signed cert. Accept and proceed. |
| Plugin compilation stuck after install | First-run plugin build takes 10–20 min. Wait, then `Restart-Service "Tenable Nessus"`. |

---

## What I Learned

This lab mirrors the day-to-day of a **Vulnerability Analyst** and the foundational skillset for a **Cloud Security Engineer**. Cloud-native scanners like Microsoft Defender for Cloud and AWS Inspector automate much of this, but they rest on the same concepts practiced here: scan coverage, authenticated vs. unauthenticated visibility, CVSS-based prioritisation, and — most importantly — closing the loop by verifying that remediation actually reduced risk.

---

## Certification Alignment

**CompTIA Security+** · **CySA+** · **PenTest+** — vulnerability management, CVSS scoring, scan interpretation, and remediation prioritisation.

---

<sub>Part of my hands-on Cloud Security lab portfolio. Only scan systems you own or have explicit written permission to test.</sub>
