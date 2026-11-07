# AI-Assisted Recon-to-Report Pentest Automation Pipeline

An automated red team reconnaissance pipeline that chains **Nmap**, **n8n**, and a **local LLM (Ollama)** to turn raw network scan output into an AI-generated, report-ready risk assessment — running entirely offline on consumer hardware (8GB RAM).

Built as the offensive-security counterpart to a Suricata + n8n + Ollama blue team detection pipeline, demonstrating both sides of AI-assisted security operations: automated **detection** and automated **reconnaissance analysis**.

---

## Why this project

Manual penetration testing involves a lot of repetitive, time-consuming steps: run a scan, read the output, decide what to try next, then write it all up in a report. This project automates the first and last of those steps — scanning and reporting — while keeping exploitation decisions in human hands, where they belong.

This mirrors a growing trend in the security industry: using AI to accelerate recon triage and report drafting, not to replace the tester's judgment.

---

## Architecture

```
┌─────────────────┐     ┌──────────────┐     ┌───────────────────┐     ┌──────────────────┐
│  Schedule        │────▶│  Nmap Scan   │────▶│  Ollama (local LLM)│────▶│  Report saved to  │
│  Trigger (n8n)   │     │  via SSH     │     │  risk analysis     │     │  Kali Desktop     │
└─────────────────┘     └──────────────┘     └───────────────────┘     └──────────────────┘
```

1. **n8n Schedule Trigger** kicks off the workflow automatically (e.g., every 30 minutes)
2. **SSH node** connects to the Kali attacker VM, runs an Nmap service scan against the target, and reads the raw output
3. **Ollama node** (running `llama3.2:3b` locally) analyzes the scan output and returns:
   - Top 3 highest-risk open ports and why
   - Suggested CVEs / exploit techniques per risky port
   - A concise, report-ready summary
4. **SSH node** writes the AI's analysis to a timestamped report file on the Kali Desktop

No cloud APIs, no external AI calls — everything runs locally.

---

## Lab Environment

Isolated, fully virtualized lab (VMware Workstation) — no production networks touched:

| VM | Role | Network |
|---|---|---|
| Kali Linux | Attacker / automation host (Nmap, SSH target for n8n) | VMnet2 |
| Metasploitable 2 | Intentionally vulnerable target | VMnet3 |
| pfSense | Firewall / routing between segments (also runs Suricata for the companion blue team pipeline) | VMnet2 ↔ VMnet3 |
| Windows Host | Runs n8n + Ollama | VMnet2 (via host-only adapter) |

**Tool stack:**
- **Nmap** — service/version reconnaissance
- **n8n** — workflow orchestration (self-hosted, local)
- **Ollama** (`llama3.2:3b`) — local LLM inference, chosen specifically to run on 8GB RAM
- **Metasploit Framework** — manual exploitation (used to generate ground-truth findings documented below)

---

## Manual Findings (Validation Targets)

Before automating analysis, two real vulnerabilities were manually exploited on Metasploitable 2 to validate the lab and provide ground-truth data for the AI pipeline:

### Finding 1 — vsftpd 2.3.4 Backdoor
- **CVE:** CVE-2011-2523
- **Module:** `exploit/unix/ftp/vsftpd_234_backdoor`
- **Result:** Remote root shell (`uid=0(root) gid=0(root)`)
- **Evidence:** `id`, `uname -a`, `/etc/passwd` captured via Meterpreter session

### Finding 2 — Samba `usermap_script` Command Injection
- **CVE:** CVE-2007-2447
- **Module:** `exploit/multi/samba/usermap_script`
- **Result:** Remote command shell as root via shell metacharacter injection in the `username map script` config parameter

Both exploits were run manually first, specifically so the AI pipeline could later be tested against **real scan data from an actually-compromised host**, not synthetic data.

---

## Sample AI Output

> **Risk Assessment Report**
>
> Based on the provided Nmap scan output, the top 3 most risky open ports are:
>
> 1. **1524/tcp** — This port is open and advertising a Metasploitable root shell, making it a high-risk target for exploitation. Potential exploit techniques to try include Metasploit modules such as `exploit/multi/http/metasploit` or `exploit/linux/local/privs`.
> 2. **23/tcp** — The telnet service is open, which can be a significant risk if an attacker gains access to the system. Potential exploit techniques include brute-force attacks or using known vulnerabilities like `CVE-2019-220...`
> 3. *(truncated — full reports saved per-run on Kali Desktop)*

---

## Setup

### Prerequisites
- VMware Workstation (or similar hypervisor)
- Kali Linux VM with SSH enabled
- n8n (self-hosted — `npm install n8n` or Docker)
- Ollama installed with a pulled model (`ollama pull llama3.2:3b`)

### n8n Workflow Nodes
1. **Schedule Trigger** — interval-based (e.g., every 30 min)
2. **SSH — Execute Command:**
   ```bash
   nmap -sV -oN /home/kali/Desktop/scan_latest.txt <target_ip> && cat /home/kali/Desktop/scan_latest.txt
   ```
3. **Ollama — Message a Model** (`llama3.2:3b`), prompt:
   ```
   You are a penetration testing assistant. Below is an Nmap scan output.
   Analyze it and respond with:
   1. Top 3 most risky open ports and why
   2. For each risky port, suggest what CVE or exploit technique to try
   3. Keep the response concise and professional, suitable for a report

   Nmap output:
   {{ $json.stdout }}
   ```
4. **SSH — Execute Command** (save report):
   ```bash
   echo "{{ $json.content.replace(/"/g, '\\"') }}" > /home/kali/Desktop/ai_report_$(date +%Y%m%d_%H%M%S).txt
   ```

### Credentials
- SSH credential pointed at the Kali VM (host-only adapter IP, not NAT — see Troubleshooting)
- Ollama credential: `http://127.0.0.1:11434` (see Troubleshooting for why not `localhost`)

---

## Troubleshooting Notes (real issues hit while building this)

A few non-obvious issues came up building this in an isolated VM lab — documenting them here since they're common gotchas:

- **SSH "timed out waiting for handshake" from host to an isolated VMnet:** The Windows host had no route into the VM's host-only network by default. Fixed by enabling *"Connect a host virtual adapter to this network"* in VMware's Virtual Network Editor for that VMnet — gives the host a local IP on the same subnet without bridging it to the real LAN.
- **SSH "all authentication methods failed" despite correct password:** Root login was restricted by default SSH config (`PermitRootLogin prohibit-password`). Fixed in `/etc/ssh/sshd_config`, or simpler — just used a non-root user account instead.
- **Ollama reachable via `curl`/browser but n8n gets `ECONNREFUSED`:** Classic `localhost` vs `127.0.0.1` resolution mismatch (IPv6 `::1` vs IPv4). Fixed by using `http://127.0.0.1:11434` explicitly in the n8n credential instead of `localhost`.
- **System crash running too many VMs + local LLM inference simultaneously on 8GB RAM:** Only the VMs actually needed for a given test should run at once — pfSense/Metasploitable can stay off while testing the pure recon pipeline against saved scan data.

---

## What this demonstrates

- End-to-end workflow automation (trigger → action → AI analysis → output), not just a single API call
- Practical constraint-driven engineering: building a working local-AI pipeline within strict 8GB RAM limits
- Real debugging of networking, auth, and resource issues across a multi-VM isolated lab — not just following a tutorial
- Manual exploitation skills (CVE-2011-2523, CVE-2007-2447) used to generate and validate the automation against real findings, not synthetic test data

---

## Companion Project

This pipeline pairs with a **Blue Team detection automation lab**: `Suricata (pfSense) → n8n → Ollama → severity/verdict → action`, using the same tool stack from the opposite side of the security operations cycle.

---

## Disclaimer

This lab is fully isolated (host-only VMware networks, no bridged/internet-facing interfaces for target VMs). All testing was performed against intentionally vulnerable lab VMs (Metasploitable 2) owned and controlled by the author. Nothing in this repository should be used against systems you do not own or have explicit written authorization to test.
