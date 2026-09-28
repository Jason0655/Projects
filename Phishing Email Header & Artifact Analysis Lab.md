# 🎣 Phishing Email Header & Artifact Analysis Lab

A hands-on SOC (Security Operations Center) analyst lab demonstrating end-to-end triage of a suspicious email — from raw header inspection to indicator extraction, threat intelligence lookups, and formal incident documentation.

![Status](https://img.shields.io/badge/status-complete-brightgreen)
![Focus](https://img.shields.io/badge/focus-Blue%20Team%20%7C%20SOC%20Analysis-blue)
![Tools](https://img.shields.io/badge/tools-VirusTotal%20%7C%20URLScan.io%20%7C%20AbuseIPDB-orange)

## 📋 Overview

This project simulates the workflow of a **Tier-1 SOC Analyst** performing phishing email triage. It walks through safely acquiring a real-world phishing sample, parsing its headers for authentication and routing data, extracting and defanging indicators of compromise (IoCs), and cross-referencing those indicators against open-source threat intelligence platforms — culminating in a full incident report and remediation playbook.

**Goal:** Build repeatable, safe procedures for identifying spoofed senders, failed email authentication, and malicious artifacts in suspicious `.eml`/`.msg` files, and translate findings into an actionable incident response record.

## 🧰 Environment & Tooling

| Category | Tool(s) |
|---|---|
| Analyst Workstation | Isolated host OS / VM |
| Header Parsing | Google Admin Toolbox Messageheader, MXToolbox Email Header Analyzer |
| Reputation & Sandboxing | VirusTotal, URLScan.io, AbuseIPDB |
| Safe Artifact Inspection | VS Code / Notepad++ (plaintext view only) |
| Sample Source | Public phishing datasets (e.g. `rf-peixoto/phishing_pot`) |

## 🔬 Methodology

### 1. Safely Acquire a Sample
Raw `.eml`/`.msg` files are **never** opened in an active email client (Outlook, Apple Mail) to avoid triggering tracking pixels or embedded scripts. Samples are pulled from public phishing archives and opened only in a plaintext editor.

### 2. Inspect Raw Headers
The header block (`Received:`, `From:`, `To:`, etc.) is copied into a header analyzer (Google Messageheader / MXToolbox) to extract:
- **Return-Path** vs. `From:` address — checked for domain spoofing
- **Originating IP** — the earliest `Received: from` hop in the chain
- **Authentication results** — SPF / DKIM / DMARC pass-fail status

### 3. Extract & Defang Artifacts
All URLs and attachment hashes are defanged (`http` → `hXXp`, `.` → `[.]`) before submission to any third-party service, and attachment hashes are generated locally:

```powershell
# Windows
Get-FileHash -Algorithm SHA256 .\suspicious_attachment.pdf
```
```bash
# Linux / macOS
sha256sum suspicious_attachment.pdf
```

### 4. Threat Intelligence & Reputation Lookup
- **IP reputation** checked against AbuseIPDB and VirusTotal (ISP, geolocation, abuse confidence score)
- **URLs** detonated in URLScan.io's sandbox (DOM structure, HTTP request tree, screenshot) with no risk to the local machine
- **File hashes** checked against VirusTotal's multi-engine AV detection ratio

### 5. Document & Respond
Findings are compiled into a standardized incident report and mapped to concrete remediation steps.

## 🧪 Case Study: Sample Analysis

| Field | Value |
|---|---|
| **Incident ID** | INC-20260923-01 |
| **Severity** | 🔴 HIGH — financial impersonation / advance-fee fraud |
| **Subject Line** | `Re: $12,000,000.00 USD Payment from Royal Bank of Canada` |
| **Claimed Sender** | "Royal Bank Of Canada" `<RoyalBanOfCannada@hotmail.com>` |
| **Actual Sender** | `oblszn57.ru` (via Postfix) |
| **SPF** | SoftFail |
| **DKIM** | None (unsigned) |
| **DMARC** | Fail |
| **Sending IP** | `62.33.7.21` — AS20485, TransTeleCom, Russian Federation |
| **Reply-To (social engineering vector)** | `rev.innocent-johnson27@outlook.com` |
| **Payload SHA-256** | `b632ba99fa67c19c3ec0f1a803bd9e2c2b424992ae00b89379732f212226814f` |

**Summary:** Header analysis confirmed domain spoofing — the message claimed to originate from Hotmail/RBC but actually routed through a Russian-hosted Postfix server. All three authentication mechanisms (SPF/DKIM/DMARC) failed or were absent. No malicious links were embedded; the attack relied entirely on social engineering, directing victims to reply to a lookalike address to advance the fraud.

### Recommended Remediation
1. **Network layer:** Block the originating IP at the firewall / mail gateway
2. **Email gateway layer:** Blocklist the sending domain and reply-to address
3. **Tenant containment:** Purge all matching messages org-wide
4. **User & endpoint remediation:** Identify responders, isolate endpoints, force credential resets, flag accounts for monitoring

## 🛠️ Troubleshooting Notes

| Issue | Cause | Resolution |
|---|---|---|
| Headers missing in plaintext view | Editor auto-formatting / stripping raw data | Use Notepad++ / VS Code with UTF-8 encoding; never open in a full email client |
| No embedded links/attachments | Pure social-engineering / advance-fee fraud technique | Shift focus to header authentication (SPF/DKIM/DMARC) and IP reputation lookups |
| Ambiguous DMARC result | `From:` display domain ≠ connecting server domain | If domains don't align, treat DMARC as **Fail** regardless of individual SPF result |

## 📚 References

- [Google Admin Toolbox Messageheader](https://toolbox.googleapps.com/apps/messageheader/)
- [MXToolbox Email Header Analyzer](https://mxtoolbox.com/EmailHeaders.aspx)
- [URLScan.io](https://urlscan.io/)
- [VirusTotal](https://www.virustotal.com/)
- [Phishing Pot Sample Repository](https://github.com/rf-peixoto/phishing_pot)
- [CISA: Email Authentication Standards](https://www.cisa.gov/)

## ⚠️ Disclaimer

This lab was conducted in an isolated environment for educational purposes using publicly available phishing samples. No live malware was executed on a production or personal system. All IOCs are shared for detection/blocklist purposes only.

---

**Author:** Oluwatoni Oderinlo
