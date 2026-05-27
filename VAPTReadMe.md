# Strategic Proposal: AI-Assisted Penetration Test Report Generation via Wazuh Telemetry

---

## Executive Summary

Penetration testing is a discipline defined by technical depth, adversarial creativity, and exhaustive methodology. Yet the final deliverable — the report — is consistently the most painful and error-prone part of the engagement. Testers forget what they did, in what order, and at what time. Reconstructing a timeline after the fact from memory leads to thin, incomplete reports that fail to capture the full scope of the work performed.

This proposal outlines an architecture for a **browser extension that uses Wazuh as a passive, always-on telemetry backbone to draft penetration test reports in real time**, as the engagement happens. The tester works normally. As they execute attacks, they confirm each action with a tap on a structured attack dropdown. The AI writes the report as they go. By the time they click Stop, the report is largely complete — requiring only a brief review before export.

The operative framing is the same as any well-designed AI assist: **assistive, not authoritative.** The AI reduces documentation friction. It does not replace the tester's judgment, fabricate findings, or produce reports that can be treated as final without review.

---

## 1. The Problem Statement

Penetration test reports serve multiple audiences: technical teams who need to understand and reproduce findings, management who needs to understand business risk, and compliance frameworks that require evidence of security validation. A good report demands precision — exact timestamps, attack sequences, outcomes, and analyst reasoning for every significant action.

The problem is structural. During an active engagement, a tester's cognitive resources are entirely consumed by the work itself: reconnaissance, enumeration, exploitation, lateral movement. Stopping to document breaks flow and misses context. Attempting to reconstruct everything afterward from memory is unreliable — actions blur together, timestamps are estimated, and the reasoning behind decisions is often lost entirely.

Critically, a report that cannot distinguish between a *confirmed successful exploit* and a *failed attempt* is dangerous in a compliance context. The tester always knows the outcome the moment it happens — the shell either opens or it doesn't. The challenge is capturing that knowledge at the exact moment it exists, without interrupting the work.

Existing tools do not solve this. Most AI-assisted pentest platforms are **agentic** — they drive the test themselves and report on what they did. For human-led engagements, the documentation problem remains entirely unsolved.

---

## 2. The Proposed Solution

A lightweight browser extension overlaid on the Wazuh dashboard with three core controls: **Start**, **Stop**, and a **structured attack dropdown panel**. When the tester clicks Start, the session begins. As they work through the engagement, they confirm each action in the dropdown — selecting the attack they just executed and marking it Attempted or Confirmed. When they click Stop, the full session data is sent to the backend, the SLM generates a structured Markdown report, and the report is returned to the extension for display and download.

Wazuh — deployed on the pentester's machine and collecting telemetry continuously — serves as the corroborating source of truth. The dropdown tells the SLM what the tester did and whether it worked. Wazuh tells the SLM exactly how they did it.

---

## 3. Design Philosophy: Assistive, Not Authoritative

The AI is a **live scribe**, not an analyst. Its job is to translate confirmed tester actions and supporting telemetry into coherent, structured prose. It does not invent findings. It does not assess severity without tester confirmation. It does not declare vulnerabilities confirmed unless the tester explicitly marked them as such.

The tester's confirmation via the attack dropdown is the authoritative signal. Wazuh telemetry is the corroborating evidence. The SLM's job is to write the narrative that connects them.

No finding is presented as confirmed without the tester having explicitly toggled it so. Every generated report carries a consistent footer: *"This report is AI-drafted from tester confirmations and Wazuh telemetry. Tester review required before submission."*

A **per-section confidence score** derived from token log probabilities is displayed on each report section. See Section 5 for the full specification.

---

## 4. The Attack Dropdown: Core Interaction Model

The attack dropdown is the central UX innovation of this tool. It replaces AI inference of attack outcomes with direct tester confirmation at minimal cost to engagement flow.

### How It Works

The extension's side panel displays a persistent dropdown organized by engagement phase. Each entry represents a common attack technique. The tester interacts in two steps:

1. **Tap once — Attempted:** The tester selects the attack they just executed. Timestamp A is saved. The entry is linked to the current Wazuh telemetry window.
2. **Tap again — Confirmed:** If the attack succeeded, the tester taps the entry a second time. Timestamp B is saved. The entry state is toggled to Confirmed.

If the tester moves to the next action without a second tap, the entry remains as Attempted. The distinction is automatic, requires no typing, and takes under two seconds per action.

An optional one-line note field is available on each entry — "got root via sudo misconfiguration," "credentials were admin/admin" — which the SLM folds directly into the report narrative without paraphrasing.

### Timestamp Pair Architecture

Each dropdown entry produces a **Timestamp A / Timestamp B pair**: the moment the tester began the action and the moment they marked the outcome. This pair defines the Wazuh query window for that specific action. The backend queries Wazuh for events within that precise window, corroborating exactly what tools ran and what connections were made during that attack — not across the entire session.

This is a more precise data shape than session-wide telemetry. The SLM receives a clean, scoped event set per action rather than hours of mixed noise.

### Dropdown Structure

Organized by phase to mirror natural engagement flow:

**Reconnaissance**
- Passive OSINT gathering
- DNS enumeration
- WHOIS / certificate transparency lookup
- Subdomain discovery
- Employee / email harvesting

**Network Scanning**
- Port scan (TCP SYN)
- Service version detection
- OS fingerprinting
- Network topology mapping
- Firewall / IDS evasion probe

**Enumeration**
- SMB enumeration
- LDAP enumeration
- SNMP enumeration
- NFS share discovery
- Web directory enumeration

**Web Application**
- SQL injection
- Cross-site scripting (XSS)
- Insecure direct object reference (IDOR)
- Authentication bypass
- File inclusion (LFI / RFI)
- Server-side request forgery (SSRF)
- XML external entity injection (XXE)
- Business logic abuse
- API endpoint fuzzing
- Session token analysis

**Credential Attacks**
- Password spray
- Credential stuffing
- Brute force
- Default credentials
- Hash cracking
- Kerberoasting / AS-REP roasting

**Exploitation**
- Known CVE exploitation
- Misconfiguration abuse
- Unpatched service exploitation
- Buffer overflow
- Deserialization attack

**Post-Exploitation**
- Privilege escalation (local)
- Lateral movement
- Persistence mechanism
- Credential harvesting
- Data exfiltration
- Defense evasion
- Domain privilege escalation

A free-text entry field at the bottom of each phase section allows the tester to log attacks not in the dropdown. These are included in the session payload and the SLM drafts from the text input plus surrounding telemetry.

---

## 5. Confidence Scoring via Token Log Probabilities

The confidence score is a first-class feature of both this tool and the Log Analyzer. The canonical specification is documented in the Log Analyzer proposal (Section 5) and applies identically here. A summary follows.

### Why Logprobs, Not Self-Assessment

Asking the SLM to rate its own confidence produces a number generated by the same model that may be hallucinating — unreliable by design. Token log probabilities are a mathematical property of the model's output: the probability distribution assigned to each generated token. High mean logprob means the model generated those tokens with high certainty. Low mean logprob means the model was uncertain. This is an honest signal independent of the model's self-awareness.

### Computation and Display

The backend computes mean log probability per report section. Scores are converted to a 0–100 range and displayed as color-coded badges on each section header in the report viewer.

| Score Range | Badge Color | Tester Guidance |
| :--- | :--- | :--- |
| 80–100 | Green | High token certainty. Standard review applies. |
| 50–79 | Yellow | Meaningful uncertainty. Verify before submitting. |
| 0–49 | Red | Significant uncertainty. Manual rewrite recommended. |

### Inference Engine Requirement

Logprob support is a **hard requirement** for SLM inference engine selection. The exact model is undecided pending evaluation. vLLM is the leading candidate. Any alternative must be confirmed to expose per-token logprobs before adoption.

### Purpose

The confidence score serves three audiences: the pentester (signal on which sections need closest review), the tool developer (empirical data on model performance to guide fine-tuning), and compliance documentation (evidence that AI-assisted findings were evaluated with awareness of model uncertainty).

---

## 6. Report Output Format

The SLM produces the report in **Markdown**. This is a deliberate design choice:

- Markdown's constrained syntax reduces token consumption relative to prose-heavy formats
- A rigid section skeleton passed to the SLM as a template constrains output shape and reduces hallucination risk — the model fills defined sections rather than generating structure freely
- Markdown diffs are human-readable if the tester edits sections post-generation
- The same source file converts cleanly to PDF and DOCX without information loss

The report skeleton passed to the SLM:

```md
# Penetration Test Report
## Engagement Details
<!-- Fill: client, date, tester, scope, methodology -->

## Findings Summary
<!-- Fill: table of confirmed findings with severity -->

## Coverage Matrix
<!-- Fill: all dropdown entries with Attempted / Confirmed / Skipped status -->

## [Phase]: Reconnaissance
<!-- Fill: operations in this phase with timestamps and outcomes -->

## [Phase]: Enumeration
...

## [Phase]: Exploitation
...

## [Phase]: Post-Exploitation
...

## Appendix: Telemetry Corroboration
<!-- Fill: Wazuh-corroborated tool executions per confirmed finding -->
```

The skeleton acts as both a token budget and a hallucination fence. The SLM cannot invent new sections or reorder findings because the structure is predetermined.

---

## 7. The User Experience Workflow

### During the Engagement

1. **Click Start.** Enter engagement name, target scope, tester identity. Session begins, timestamp recorded.
2. **Work normally.** Use whatever tools and methodology the engagement requires. Nothing about the testing process changes.
3. **Tap the dropdown** as attacks are executed. Once for Attempted (Timestamp A), again for Confirmed (Timestamp B). Two seconds per action. Optional one-line note if context is needed.
4. **Click Stop.** Session ends. All dropdown entries — attack names, timestamps, outcomes, notes — are packaged with session metadata and sent to the backend.

### At Session End (Prototype: Batch Processing)

The backend receives the full session payload, queries Wazuh for each Timestamp A/B window, pre-processes the correlated telemetry, and sends the complete structured input to the SLM in a single batch call. The SLM generates the Markdown report. The report is returned to the extension.

*Note: The prototype uses batch processing at session end for simplicity. A subsequent iteration will shift to per-operation processing — where the SLM drafts each paragraph immediately after the tester taps the dropdown — enabling a real-time report preview as the engagement progresses. Batch processing validates the full pipeline end-to-end first.*

### Reviewing and Exporting

The completed report is displayed in the extension's inline Markdown viewer with confidence score badges on each section. The tester expands to a full browser tab for comfortable review, edits any sections as needed, and downloads in their preferred format:

- **Markdown (.md)** — native format, suitable for version control
- **PDF** — for formal client delivery
- **DOCX** — for integration into existing Word-based reporting workflows

Post-session review is estimated at 10 to 15 minutes for a typical engagement, compared to 2 to 4 hours of retrospective writing from memory.

---

## 8. The Coverage Matrix

A section most pentest reports need and testers universally avoid writing is generated automatically from the dropdown state at session end:

| Phase | Technique | Status | Timestamp A | Timestamp B |
| :--- | :--- | :--- | :--- | :--- |
| Web Application | SQL Injection | ✅ Confirmed | 14:23:04 | 14:26:11 |
| Web Application | XSS | ⚠️ Attempted | 14:31:09 | — |
| Credential Attacks | Password Spray | ⚠️ Attempted | 13:47:22 | — |
| Credential Attacks | Default Credentials | ✅ Confirmed | 13:52:01 | 13:52:44 |
| Post-Exploitation | Privilege Escalation | ✅ Confirmed | 15:10:33 | 15:14:02 |
| Post-Exploitation | Lateral Movement | — Skipped | — | — |

This table is a direct output of the dropdown interactions the tester already made. It takes zero additional effort to produce and provides clients and compliance auditors with an explicit record of what was and was not tested.

---

## 9. Why Wazuh on the Pentester's Machine

Wazuh deployed on the **pentester's machine** captures a complete record of tester actions: process executions (Nmap, Metasploit, Hydra, Gobuster, etc.), network connections initiated, files written, command-line activity. This is the attacker-side record — precisely what is needed to corroborate what the tester did.

The dropdown tells the SLM what happened and whether it worked. Wazuh confirms how it happened — which tool, which arguments, which target, at which precise second. Together they produce a report with technical detail that retrospective writing from memory cannot match.

This architecture requires no access to or modification of the target's infrastructure — a hard constraint in most engagements where the tester does not control the client's environment. Wazuh is also already deployed across many security teams, meaning no new infrastructure is required.

---

## 10. Architectural Blueprint

### Backend Pre-Processing

The backend receives the session payload, queries Wazuh for each Timestamp A/B window, and pre-processes the correlated telemetry:

- Match tool signatures to the corresponding dropdown entry for corroboration
- Deduplicate repeated identical events within each action window
- Strip noise events irrelevant to the action
- Preserve forensically significant fields: timestamps, process names, command-line arguments, IPs, ports, file paths, rule IDs

**Note on shared pre-processing:** The pre-processing pipeline is shared between the Pentest Report Generator and the Log Analyzer in the initial implementation. As both tools mature, the pipeline will be forked and tuned independently — pentest and incident investigation data have different noise characteristics that benefit from different handling.

### SLM Report Generation

The structured input — dropdown entries, timestamp pairs, outcomes, notes, correlated Wazuh telemetry — is passed to the SLM with the Markdown skeleton template. Logprobs are requested alongside generated tokens. The backend computes per-section confidence scores and appends them to the report as metadata.

### Tool Recognition Corroboration

For each dropdown entry, the backend attempts to match the Wazuh telemetry from the corresponding time window to known tool signatures. Matched tools are cited as corroboration in the report appendix. Unmatched windows are noted as "telemetry not corroborated — tester confirmation only."

| Tool | Wazuh Signal | Corroboration Confidence |
| :--- | :--- | :--- |
| Nmap | Process execution with characteristic arguments (-sS, -sV, -p, -A) | High |
| Metasploit | msfconsole process; network connections to target on exploit ports | High |
| Hydra / Medusa | Process execution with wordlist arguments; rapid repeated connections | High |
| Gobuster / Feroxbuster | Process execution; high-volume sequential HTTP requests | High |
| SQLMap | Process execution with URL arguments | High |
| Burp Suite | Java process; proxy port activity | Medium |
| Netcat / Socat | Network connection establishment; common shell port activity | Medium |
| Custom scripts | Unrecognized process | Low — noted in report |

---

## 11. Edge Cases and Failure Modes

| Scenario | Handling |
| :--- | :--- |
| **Wazuh telemetry missing for a confirmed action** | Report paragraph written from dropdown confirmation alone, flagged as "telemetry not corroborated — tester confirmation only." |
| **Attack not in dropdown** | Free-text entry field used. SLM drafts from the text input and surrounding telemetry. |
| **Tester forgets to tap dropdown during flow** | At Stop, the extension surfaces Wazuh events not linked to any dropdown entry and prompts the tester to classify them. Estimated cleanup: under 2 minutes. |
| **Tester forgets to click Start** | Session can be manually backdated. Wazuh archives retain all events within the retention window. |
| **Sensitive client data in logs** | All processing occurs on internal infrastructure. No log data transmitted to external APIs. Zero-exfiltration architecture. |
| **High-volume engagement** | Pre-processing deduplication reduces event volume. Report sections generated incrementally if context limits are approached. |

---

## 12. Compliance and Legal Considerations

The **two-state dropdown model** (Attempted / Confirmed) ensures report language is always calibrated to what actually happened. No AI inference introduces false positives or overstates findings. Every confirmed finding has a tester confirmation, a timestamp pair, and Wazuh corroboration as an evidence trail.

All AI-generated sections are explicitly labeled as AI-drafted until tester-reviewed. The audit trail shows which sections were modified by the tester versus accepted as-is — a direct response to client and legal pushback on AI-generated compliance documentation.

Per-section confidence scores are included in the audit record, providing evidence that the tester was given uncertainty signals and made an informed decision to accept or modify each section.

Report templates aligned to PTES, OWASP Testing Guide, and NIST SP 800-115 are selectable at session start.

---

## 13. Competitive Landscape

| Dimension | Existing Solutions | This Proposal |
| :--- | :--- | :--- |
| **Who drives the test** | AI agent or human with no documentation assist | Human tester, fully in control |
| **Outcome accuracy** | AI-inferred, or tester-reconstructed from memory | Tester-confirmed at moment of execution via dropdown |
| **When documentation happens** | After the fact | At session end from live-collected data |
| **Data source** | Manual tool output export | Wazuh telemetry correlated to dropdown timestamp pairs |
| **Confidence signal** | None | Token logprob-derived per-section score |
| **Behavior change required** | Significant | 2 seconds per action |
| **Coverage audit trail** | Manual | Automatic from dropdown state |
| **Target infrastructure access** | Sometimes required | Never |

### The Core Differentiator

No existing solution combines **passive telemetry corroboration** with **tester-confirmed attack outcomes at the moment of execution** and **honest per-section uncertainty quantification**. The dropdown captures knowledge that exists for approximately two seconds — the moment the tester sees whether an attack worked — before it begins to fade. Everything else in the product exists to make that two-second interaction produce a complete, accurate, compliance-ready report.

---

## 14. Implementation Strategy

**Phase 1 — Dropdown and Telemetry Validation:** Build the attack dropdown as a static prototype. Run a representative engagement, use the dropdown to log actions manually, export the Wazuh session logs segmented by timestamp pairs, and feed the combined input to an SLM. Evaluate whether the output produces accurate, useful report paragraphs. This validates the core mechanism before any production engineering begins.

**Phase 2 — Backend Pipeline:** Build the backend service to authenticate with Wazuh, receive session payloads, query Wazuh per Timestamp A/B window, pre-process correlated telemetry, and pass structured input to the SLM. Validate the tool recognition corroboration layer. Implement logprob extraction and per-section confidence score computation.

**Phase 3 — SLM Report Generation:** Implement report drafting from the structured session input and Markdown skeleton. Validate Attempted / Confirmed language calibration. Implement coverage matrix generation. Validate output quality with experienced testers reviewing for accuracy.

**Phase 4 — Browser Extension and UX:** Build the extension overlay with Start/Stop controls, the attack dropdown panel, and the inline Markdown viewer with confidence score badges. Build the full-tab expansion view and download options (MD, PDF, DOCX). Pilot with 3 to 5 testers on real engagements.

**Phase 5 — Per-Operation Processing:** Upgrade from batch-at-session-end to per-operation SLM drafting, enabling a real-time report preview as the engagement progresses. This is a backend and UX upgrade layered on the validated Phase 2–4 architecture.

**Phase 6 — Rollout and Iteration:** Deploy to a wider user base. Track post-session review time as the primary quality metric — target under 15 minutes. Expand the dropdown library based on techniques testers are adding via the free-text field.

---

## 15. Conclusion

Penetration testing documentation is broken. The tools that could theoretically help either automate the test itself or assist with writing after the fact — neither captures the one thing that makes a pentest report accurate: the tester's knowledge, in the moment of execution, of what they did and whether it worked.

This proposal captures that knowledge with a two-second tap. The tester confirms the attack. Wazuh corroborates the execution. The SLM writes the report. Confidence scores tell the tester exactly which sections to scrutinize. By the time they stop working, the documentation is done.

The result is a tool that is genuinely useful to the people who need it most — not because it replaces their expertise, but because it captures it at the exact moment it exists, before memory degrades it into something less useful than the truth.