# 📧 Email Analysis — Automated Email Security Scanner (n8n Workflow)

An n8n automation that monitors an inbox via IMAP, extracts metadata/attachments/URLs from incoming emails, scans them against **VirusTotal**, and sends a consolidated security report to **Telegram**.

![n8n](https://img.shields.io/badge/n8n-workflow-orange) ![VirusTotal](https://img.shields.io/badge/VirusTotal-API-blue) ![Telegram](https://img.shields.io/badge/Telegram-Bot-2CA5E0)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Workflow Breakdown](#workflow-breakdown)
  - [1. Trigger & Metadata Extraction](#1-trigger--metadata-extraction)
  - [2. Attachment Branch](#2-attachment-branch)
  - [3. Sender IP Branch](#3-sender-ip-branch)
  - [4. URL Branch](#4-url-branch)
  - [5. Merge & Report Delivery](#5-merge--report-delivery)
---

## Overview

`Email Analysis` is an n8n workflow that acts as an automated first-line triage tool for incoming email. On every new message, it:

1. Parses headers, SPF/DKIM/DMARC results, body text/HTML, and any URLs.
2. If an attachment is present, uploads it to VirusTotal and polls until the scan completes.
3. Extracts the sender's IP from the email headers and checks its reputation via VirusTotal.
4. If URLs are present in the body, submits each to VirusTotal for scanning.
5. Merges the three result branches (attachment, IP, URL) by position.
6. Builds a formatted Markdown report and sends it to a Telegram chat.

---

## Architecture

```
Email Trigger (IMAP)
        │
        ▼
Extract Email Metadata
        │
   ┌────┴──────────────────────┐
   ▼                            ▼
If attatchment exists       If URLs exist
   │ (true)                     │ (true)
   ├──────────────┐             ▼
   ▼              ▼        Split out the URLs
Extract        Check              │
Sender IP    attachment           ▼
   │         (upload to VT)   Scan URL (POST /urls)
   ▼              │               │
Check Sender IP   ▼               ▼
(GET /ip_addr) Get result    Get URL scan result
   │           scan (poll)       │
   ▼              │              ▼
Extract IP        ▼        Extract URL scan result
Information   Extract              │
   │          Attachment           │
   │          result                │
   │              │                 │
   └──────┬───────┴────────┬────────┘
          ▼                ▼
              Merge (combine by position, 3 inputs)
                       │
                       ▼
              Report Preparing (Code)
                       │
                       ▼
              Send a text message (Telegram)
```

**Merge input mapping (combine by position):**
| Input | Source node |
|---|---|
| Input 0 | `Extract Attachment result` |
| Input 1 | `Extract IP Information` |
| Input 2 | `Extract URL scan result` |

---

## Prerequisites

| Requirement | Purpose |
|---|---|
| n8n instance (self-hosted or cloud) | Runs the workflow |
| Email account with IMAP access | `Email Trigger (IMAP)` node |
| [VirusTotal API key](https://www.virustotal.com/gui/join-us) | File upload/scan, IP reputation, URL scanning |
| Telegram Bot Token (via [@BotFather](https://core.telegram.org/bots#botfather)) | Sends the final report |
| Telegram Chat ID | Destination for the report |

**VirusTotal free-tier limits:** 4 requests/minute, 500/day — be mindful of this if your inbox receives high mail volume.

---

## Workflow Breakdown

### 1. Trigger & Metadata Extraction

| Node | Type | Purpose |
|---|---|---|
| `Email Trigger (IMAP)` | `emailReadImap` | Fires on new incoming email; tracks last message ID so it doesn't reprocess mail |
| `Extract Email Metadata` | Code | Parses sender/recipient, subject, body text & HTML, extracts and dedupes URLs via regex, parses SPF/DKIM/DMARC out of the `Authentication-Results` header, and passes through any binary attachments unchanged |

This node's output (and the passed-through binary data) feeds both `If attatchment exists` and `If URLs exist`.

**Output fields:** `message_id`, `sender`, `sender_name`, `recipient`, `subject`, `body_text`, `body_html`, `urls[]`, `has_urls`, `date`, `return_path`, `authentication_results`, `spf`, `dkim`, `dmarc`, `received`

---

### 2. Attachment Branch

| Node | Type | Purpose |
|---|---|---|
| `If attatchment exists` | IF | Checks `Object.keys($binary ?? {}).length > 0` — true if any binary attachment is present |
| `Extract Sender IP` | Code | *(runs on the same true branch)* — extracts IPv4 addresses from `authentication_results` + `received` headers via regex |
| `Check attachment` | HTTP Request | `POST https://www.virustotal.com/api/v3/files` — uploads the binary field `attachment_0` as multipart form data |
| `Get result scan` | HTTP Request | `GET {{ $json.data.links.self }}` — polls the analysis link returned by the upload |
| `Extract Attachment result` | Code | Normalizes the VirusTotal response into a clean verdict object |

> ⚠️ Note: `Extract Sender IP` is wired off the **true** branch of `If attatchment exists`, not off `Extract Email Metadata` directly. This means if an email has **no attachment**, the sender-IP branch (and therefore IP reputation checking) never runs. See [Known Limitations](#known-limitations--improvement-ideas).

**`Extract Attachment result` output:**
```json
{
  "verdict": "CLEAN",
  "status": "completed",
  "is_malicious": false,
  "file": { "name": "Uploaded Attachment", "type": "Binary/Document", "size_bytes": 945166 },
  "stats": { "malicious": 0, "suspicious": 0, "harmless": 0, "undetected": 62, "total_scanned": 62 },
  "hashes": { "sha256": "...", "sha1": "...", "md5": "..." },
  "file_id": "9cd54ce1d5271abe..."
}
```

---

### 3. Sender IP Branch

| Node | Type | Purpose |
|---|---|---|
| `Check Sender IP` | HTTP Request | `GET https://www.virustotal.com/api/v3/ip_addresses/{{ $json.sender_ip }}` |
| `Extract IP Information` | Code | Normalizes country, ASN, owner, reputation score, and vote counts |

**Output:**
```json
{
  "ip": "209.85.220.41",
  "country": "US",
  "asn": 15169,
  "as_owner": "Google LLC",
  "network": "209.85.128.0/17",
  "registry": "ARIN",
  "reputation": -32,
  "analysis": { "malicious": 0, "suspicious": 0, "harmless": 54, "undetected": 37, "timeout": 0 },
  "votes": { "harmless": 11, "malicious": 46 }
}
```

---

### 4. URL Branch

| Node | Type | Purpose |
|---|---|---|
| `If URLs exist` | IF | Checks boolean `has_urls` from metadata extraction |
| `Split out the URLs` | Split Out | Splits the `urls` array into individual items |
| `Scan URL` | HTTP Request | `POST https://www.virustotal.com/api/v3/urls` (form-urlencoded, `url={{ $json.urls }}`) |
| `Get URL scan result` | HTTP Request | `GET {{ $json.data.links.self }}` — retrieves the scan analysis |
| `Extract URL scan result` | Code | Normalizes verdict, category, and stats |

**Output:**
```json
{
  "url": "https://saeed8elfiky.github.io/portfolio",
  "title": "N/A",
  "verdict": "Clean",
  "malicious_count": 0,
  "suspicious_count": 0,
  "harmless_count": 0,
  "undetected_count": 0,
  "categories": [],
  "scan_date": "2026-09-02T12:27:32.000Z"
}
```

> Note: only the **first** URL in the email is scanned end-to-end per execution, since `Split out the URLs` creates one item per URL but the rest of the branch and the Merge node (combine-by-position, 3 inputs) aren't designed for multiple URL items in parallel. See [Known Limitations](#known-limitations--improvement-ideas).

---

### 5. Merge & Report Delivery

| Node | Type | Purpose |
|---|---|---|
| `Merge` | Merge — **combine by position**, 3 inputs | Combines Attachment (input 0), IP (input 1), and URL (input 2) results into one item |
| `Report Preparing` | Code | Builds the final Markdown report string |
| `Send a text message` | Telegram | Sends to a fixed `chatId`, with `parse_mode: Markdown` |

---
