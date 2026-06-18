# Enterprise Email Forensics: Operation "Phantom Invoice" (BEC Investigation)

## Executive Summary

During a 3-day window, a multi-stage Business Email Compromise (BEC) and financial invoice fraud campaign targeted the Accounts Payable department at **Nexus Logistics Group**. The threat actor concurrently impersonated the company's CFO (James Hargrove) and an external vendor (Meridian Freight Solutions) to create a high-pressure, layered social engineering trap. The attack attempted to authorize a fraudulent wire transfer of **$47,000 USD**. The campaign was successfully detected, triaged, and neutralized by IT Security before any financial exfiltration occurred.

| Metric | Target Details |
| :--- | :--- |
| **Attack Type** | Business Email Compromise (BEC) & Invoice Fraud |
| **Target Vector** | Accounts Payable Department |
| **Potential Loss** | $47,000 USD |
| **Impersonated Entities** | James Hargrove (CFO) & Meridian Freight Solutions (Vendor) |
| **Final Status** | **Mitigated** — Zero Financial Loss |

> Note: Supporting documentation and artefacts can be found in `docs/PhantomInvoice_Report.pdf` and `images/` respectively.
---

## Technical Analysis & Forensic Artifacts

### 1. The Homoglyph Domain Attack (Typosquatting)

The threat actor registered a lookalike domain specifically designed to bypass human visual inspection. By replacing a lowercase `l` with an uppercase `I`, the attacker matched the font geometry of the legitimate corporate domain.

- **Legitimate Production Domain:** `nexuslogistics.com`
- **Attacker Infrastructure Domain:** `nexus-Iogistics.com` (uses uppercase `I`)

### 2. Header Discrepancies & The Reply-To Trap

A deep-dive review of the raw transmission headers revealed an intentional disconnect between the outbound display fields and the inbound reply vectors.

```yaml
From: "James Hargrove CFO" <j.hargrove@nexus-Iogistics.com>  # Spoofed Lookalike Domain
Reply-To: <j.hargrove.cfo@gmail.com>                         # Attacker-Controlled Personal Inbox
Return-Path: <bounce@smtp.webmailpro.xyz>                    # Malicious SMTP Relay Server
```

**The Mechanism:** When the victim clicks "Reply," the mail client automatically overrides the corporate `From:` display and routes the transmission directly to the attacker's personal Gmail workspace, breaking internal corporate visibility.

---

## Email Infrastructure Triage

### Email 1: CFO Initial Request (March 11, 2024)

The campaign begins with the impersonated CFO demanding an immediate payment for a vendor while claiming to be unavailable in an off-site board meeting.

```
Delivered-To: sarah.okonkwo@nexuslogistics.com
From: "James Hargrove CFO" <j.hargrove@nexus-Iogistics.com>
Reply-To: j.hargrove.cfo@gmail.com
Return-Path: <bounce@smtp.webmailpro.xyz>
Authentication-Results: mx.nexuslogistics.com;
  spf=fail (sender IP is 185.234.219.101)
  dkim=none (no signature)
  dmarc=fail action=none header.from=nexus-Iogistics.com;
```

**Key Finding:** The originating IP address `185.234.219.101` traces back to a generic VPS provider hosted in Austria, showing zero geographic or infrastructure alignment with real Nexus Logistics corporate operations.

### Email 2: Fake Vendor Confirmation (March 12, 2024)

The attacker switches identities to the vendor account (`billing@meridianfreight-solutions.net`) to push a fraudulent invoice matching the previous request.

**Key Finding:** The attacker injects a forged historical correspondence block at the bottom of the email to make it appear as an authentic ongoing thread. No actual thread hijacking took place.

### Email 3: The Urgency Escalation (March 13, 2024)

The threat actor switches back to the fake CFO profile, applying severe psychological authority and artificial urgency ("FINAL NOTICE") to force the wire transfer execution.

---

## Email Authentication Failures (SPF / DKIM / DMARC)

All three core email security frameworks threw hard technical alerts across the lifecycle of this campaign:

```
SPF:   ❌ FAIL  (185.234.219.101 is not an authorized sender for the domain)
DKIM:  ❌ NONE / FAIL  (No cryptographic signature present to verify integrity)
DMARC: ❌ FAIL  (Strict domain misalignment detected between From header and envelope)
```

---

## Threat Intelligence & Behavioral Mapping

### Psychological Manipulation Levers

This campaign deployed a zero-payload strategy. By omitting malicious attachments (`.exe`/`.lnk`) or suspicious links, the actor completely bypassed standard Secure Email Gateway (SEG) sandboxes and signature-based Antivirus (AV) detection loops, relying entirely on human exploitation.

- **Urgency:** "Process this NOW", "FINAL NOTICE"
- **Isolation:** Instructions explicitly telling the victim: "Do NOT call anyone in my office"
- **Fear:** Manufacturing a scenario where a fictitious $380,000 contract was at risk

---

## MITRE ATT&CK Tracking Matrix

| Tactic | Technique ID | Technique Name | Applied Evidence |
| :--- | :--- | :--- | :--- |
| Resource Development | T1583.001 | Acquire Infrastructure: Domains | Registration of the `nexus-Iogistics.com` homoglyph domain |
| Resource Development | T1586.002 | Compromise Accounts: Email | Use of `smtp.webmailpro.xyz` as an attacker-controlled mail relay |
| Initial Access | T1566.001 | Phishing: Spearphishing Attachment | Highly targeted text-based spearphishing to Accounts Payable |
| Defense Evasion | T1036.005 | Masquerading: Match Legitimate Name | Character spoofing visually indistinguishable from legitimate brand fonts |
| Impact | T1657 | Financial Theft | Attempted unauthorized wire transfer of $47,000 USD |

---

## Strategic Remediation & Hardening Controls

### Technical Engineering Defense

- **DMARC Enforcement:** Transition internal DMARC records from `p=none` (monitoring) to a strict `p=reject` configuration to drop unauthorized senders automatically.
- **Defensive Domain Registration:** Proactively acquire common homoglyph permutations and lookalike domains associated with corporate brands to keep them out of attacker hands.
- **External Mail Banner Tagging:** Configure mail gateway rules to append an explicit warning banner to any inbound traffic originating from external networks mimicking internal display names.

### Administrative Process Policy

- **Out-of-Band (OOB) Verification Protocol:** Mandate a strict secondary confirmation step via a known, trusted corporate phone directory number before executing any change to vendor banking routing data or high-value payment requests.
- **Multi-Authorized Payment Structures:** Implement dual-custody approval gates requiring secondary executive authorization tokens for transfers exceeding specific dollar thresholds.
