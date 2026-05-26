# Strategic Proposal: AI-Assisted Penetration Test Report Generation via Wazuh Telemetry

---

## Executive Summary

Penetration testing is a discipline defined by technical depth, adversarial creativity, and exhaustive methodology. Yet the final deliverable — the report — is consistently the most painful and error-prone part of the engagement. Testers forget what they did, in what order, and at what time. Reconstructing a timeline after the fact from memory leads to thin, incomplete reports that fail to capture the full scope of the work performed.

This proposal outlines an architecture for a **browser extension that uses Wazuh as a passive, always-on telemetry backbone to draft penetration test reports in real time**, as the engagement happens. The tester does their work normally. As they execute attacks, they confirm each action with a tap on a structured attack dropdown. The AI writes the report as they go. By the time they click Stop, the report is largely complete — requiring only a brief review and approval rather than hours of retrospective documentation.

The operative framing is the same as any well-designed AI assist: **assistive, not authoritative.** The AI reduces documentation friction. It does not replace the tester's judgment, fabricate findings, or produce reports that can be treated as final without review.

---

## 1. The Problem Statement

Penetration test reports serve multiple audiences: technical teams who need to understand and reproduce findings, management who need to understand business risk, and compliance frameworks that require evidence of security validation. A good report demands precision — exact timestamps, attack sequences, outcomes, and analyst reasoning for every significant action.

The problem is structural. During an active engagement, a tester's cognitive resources are entirely consumed by the work itself: reconnaissance, enumeration, exploitation, lateral movement. Stopping to document breaks flow and misses context. Attempting to reconstruct everything afterward from memory is unreliable — actions blur together, timestamps are estimated, and the reasoning behind decisions is often lost entirely.

Critically, a report that cannot distinguish between a *confirmed successful exploit* and a *failed attempt* is dangerous in a compliance context. The tester always knows the outcome the moment it happens — the shell either opens or it doesn't. The challenge is capturing that knowledge at the exact moment it exists, without interrupting the work.

Existing tools do not solve this. Most AI-assisted pentest platforms are **agentic** — they drive the test themselves and report on what they did. These tools are useful for automated scans, but they cannot replicate the judgment of an experienced human tester working through complex, logic-based attack chains. For human-led engagements, the documentation problem remains entirely unsolved.

---

## 2. The Proposed Solution

A lightweight browser extension with a **Start / Stop session button** and a **structured attack dropdown panel**, overlaid on the Wazuh dashboard. When the tester clicks Start, the extension timestamps the beginning of the engagement session. From that point forward, Wazuh — already deployed on the pentester's machine and collecting telemetry continuously — serves as the single source of truth for what happened at the system level.

As the session progresses, the tester confirms their actions using the attack dropdown: selecting the attack they just executed and toggling its outcome. The extension pairs each confirmation with the corresponding Wazuh telemetry from that timestamp. The AI drafts the report continuously, building the narrative in real time — not from guesswork, but from the combination of tester confirmation and system telemetry.

When the tester clicks Stop, the draft report is complete. The tester reviews it, addresses any gaps, and approves. The time from Stop to deliverable-ready report is measured in minutes, not hours.

---

## 3. Design Philosophy: Assistive, Not Authoritative

This distinction is fundamental to the tool's value and its safe use.

The AI is a **live scribe**, not an analyst. Its job is to translate confirmed tester actions and supporting telemetry into coherent, structured prose. It does not invent findings. It does not assess severity without tester confirmation. It does not declare vulnerabilities confirmed unless the tester explicitly marked them as such.

The tester's confirmation via the attack dropdown is the authoritative signal. Wazuh telemetry is the corroborating evidence. The AI's job is to write the narrative that connects them.

The result is a collaborative workflow:

- Wazuh captures what happened at the system level — passively, automatically, without any tester effort.
- The attack dropdown captures what the tester intended and whether it succeeded — with minimal, two-second interactions.
- The AI handles all the writing, building the report section by section as confirmations come in.
- No finding is presented as confirmed without the tester having explicitly marked it so.

Every generated report section carries a consistent footer: *"This section is AI-drafted from tester confirmations and Wazuh telemetry. Review before submission."*

---

## 4. The Attack Dropdown: Core Interaction Model

The attack dropdown is the central UX innovation of this tool. It solves the fundamental problem of AI-inferred outcomes by replacing inference with direct tester confirmation — at minimal cost to engagement flow.

### How It Works

The extension's side panel displays a persistent dropdown organized by engagement phase. Each entry represents a common attack or test technique. The tester interacts with it in two steps:

1. **Tap to log Attempted** — the tester selects the attack they just executed. This stamps a timestamp and links the entry to current Wazuh telemetry.
2. **Tap again to confirm Success** — if the attack succeeded, the tester taps the entry a second time to toggle it to Confirmed. If they move on without a second tap, the entry remains as Attempted.

This two-state model is the compliance-critical distinction. Attempted and Confirmed are explicitly different report entries with different language, different severity weight, and different remediation implications. The tester never has to write that distinction — they just tap once or twice.

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

### What the AI Receives Per Entry

When the tester taps an attack entry, the extension packages and passes to the AI:

- Attack name and category
- Timestamp of execution
- Outcome state: Attempted or Confirmed
- Correlated Wazuh telemetry from the same timestamp window (process executions, network connections, tool signatures)
- Current engagement phase

The AI has everything it needs to write a precise, accurate report paragraph. No inference required for the outcome. No guessing about what tool was used — Wazuh confirms it. No reconstructing the timeline — it's timestamped to the second.

### The Report Update Loop

The tester sees the report draft update in the side panel in real time as they work through the dropdown. Tapping "SQL Injection — Confirmed" on the web application panel immediately produces a draft paragraph. They can see exactly what the AI is writing about what they just did, which serves as a lightweight QA pass throughout the engagement rather than a review dump at the end.

---

## 5. Why Wazuh on the Pentester's Machine is the Right Data Source

A common assumption is that useful pentest telemetry must come from the target. This proposal inverts that assumption deliberately.

Wazuh deployed on the **pentester's machine** captures a complete record of the tester's actions: process executions (Nmap, Metasploit, Hydra, Gobuster, etc.), network connections initiated, files written or modified, command-line activity, and tool output. This is precisely the data needed to corroborate what the tester did — not what the target experienced.

The attack dropdown tells the AI *what* the tester did and *whether it worked*. Wazuh tells the AI *exactly how* they did it — which tool, which arguments, which target, at what precise time. Together, they produce a report with a level of technical detail that retrospective writing from memory cannot match.

This architecture has two additional advantages. First, it requires no access to or modification of the target's infrastructure — a hard constraint in most real engagements. Second, Wazuh is already deployed across many security teams as part of their standard tooling. There is no new infrastructure to provision.

---

## 6. Architectural Blueprint

### Step 1: Session Scoping (Browser Extension)

The tester clicks **Start**, records the engagement name, target scope, and tester identity, and the session begins. The extension's side panel becomes active, showing the attack dropdown and the live report draft side by side.

Phase transitions — Reconnaissance, Enumeration, Exploitation, Post-Exploitation — are marked automatically based on which section of the dropdown the tester is working in. No manual phase tagging required.

When the tester clicks **Stop**, the session window closes and a final report assembly pass runs.

### Step 2: Attack Confirmation (Dropdown Interaction)

As the tester works through the engagement, they tap entries in the dropdown to log actions. Each tap:

- Timestamps the action to the second
- Links the entry to the active Wazuh telemetry window
- Sets initial state to Attempted
- Triggers an AI draft update for that section of the report

A second tap on the same entry confirms Success and updates the report paragraph accordingly. The tester can also add an optional one-line note — "got root via sudo misconfiguration", "credentials were admin/admin" — that the AI folds directly into the narrative without paraphrasing.

### Step 3: Telemetry Retrieval (Backend Service)

The extension communicates with a lightweight internal backend service. This service authenticates with the Wazuh API and retrieves events correlated to each dropdown action's timestamp. It applies pre-processing to:

- Match tool signatures to the corresponding dropdown entry for corroboration
- Deduplicate repeated identical events
- Strip noise events irrelevant to pentest activity
- Preserve forensically significant fields: timestamps, process names, command-line arguments, source and destination IPs, ports, file paths, rule IDs

### Step 4: Live Report Drafting (AI Layer)

The AI receives, per dropdown action: the attack name, outcome state, optional tester note, and correlated Wazuh telemetry. It produces a structured report paragraph for that action using language calibrated to the outcome state:

**Confirmed finding:**
> *SQL Injection — Confirmed [14:23] The tester identified and successfully exploited a SQL injection vulnerability in the login endpoint at /api/auth. Blind boolean-based injection was used to enumerate the users table. Sqlmap process execution and outbound connections to the target were corroborated by host telemetry. Tester note: extracted admin credentials from users table.*

**Attempted, not confirmed:**
> *Password Spray — Attempted [13:47] The tester conducted a password spray attack against the domain authentication service using a common credential list. No successful authentications were observed. Hydra process execution and repeated SMB connection attempts to the target were corroborated by host telemetry.*

The distinction in language is automatic and consistent. The tester never has to write "attempted" versus "confirmed" — the dropdown state handles it.

### Step 5: Report Assembly and Export

At session end, all drafted paragraphs are assembled into a structured report:

- Engagement metadata (scope, dates, tester, methodology)
- Phase-organized activity narrative
- Findings summary with tester-confirmed severity ratings
- Coverage matrix: what was tested, attempted, confirmed, and skipped
- Evidence appendix (corroborated tool executions, timestamps, optional screenshots)

Export formats: PDF, DOCX, and Markdown. Compliance-aligned templates (PTES, OWASP Testing Guide, NIST SP 800-115) are selectable at session start.

---

## 7. The Coverage Matrix

A section most pentest reports need and testers hate writing is generated automatically from the dropdown state at session end.

| Attack Category | Technique | Status | Timestamp |
| :--- | :--- | :--- | :--- |
| Web Application | SQL Injection | ✅ Confirmed | 14:23 |
| Web Application | XSS | ⚠️ Attempted | 14:31 |
| Credential Attacks | Password Spray | ⚠️ Attempted | 13:47 |
| Credential Attacks | Default Credentials | ✅ Confirmed | 13:52 |
| Post-Exploitation | Privilege Escalation | ✅ Confirmed | 15:10 |
| Post-Exploitation | Lateral Movement | — Skipped | — |

This table, auto-generated, tells the client exactly what was and wasn't tested. It provides audit evidence for compliance frameworks. It takes the tester zero additional effort to produce — it is a direct output of the dropdown interactions they already made during the engagement.

---

## 8. The Tester's Workflow

The total behavior change required from the tester is minimal by design:

1. **Click Start** when the engagement begins.
2. **Work normally.** Use whatever tools, techniques, and methodology the engagement requires.
3. **Tap the dropdown** as you execute attacks — once for Attempted, twice for Confirmed. Two seconds per action. Optional one-line note if the finding needs context.
4. **Click Stop** when the engagement concludes.
5. **Review the draft** in the side panel. Address any gaps. Confirm the coverage matrix is accurate.
6. **Export.** The report is done.

The post-session review step is estimated at 10 to 15 minutes for a typical engagement. The dropdown interactions during the engagement add approximately 2 seconds per action logged. The output quality is higher than retrospective writing because timestamps are exact, tool sequences are Wazuh-corroborated, and outcomes are tester-confirmed rather than AI-inferred.

---

## 9. Edge Cases and Failure Modes

| Scenario | Handling |
| :--- | :--- |
| **Wazuh telemetry missing for a confirmed action** | Report paragraph is written from the dropdown confirmation alone, flagged as "telemetry not corroborated — tester confirmation only." |
| **Attack not in the dropdown** | Tester uses a free-text entry field at the bottom of each phase section. Adds the attack name and outcome manually. AI drafts from the text input plus surrounding telemetry. |
| **Tester forgets to tap the dropdown during flow** | Session is fully timestamped. At Stop, the extension surfaces any Wazuh events not linked to a dropdown action and asks the tester to classify them. Maximum 2 minutes of cleanup for a typical session. |
| **Tester forgets to click Start** | Session can be manually backdated. Wazuh archives retain all events within the archive retention window. |
| **Sensitive client data in logs** | All processing occurs on internal infrastructure. No log data is transmitted to external APIs. Zero-exfiltration architecture. |
| **High-volume engagement** | Deduplication and clustering in the pre-processing layer reduce the event stream before AI processing. Report sections are generated incrementally. |

---

## 10. Compliance and Legal Considerations

Pentest reports in regulated environments carry compliance weight. Several design decisions address this directly.

The **two-state dropdown model** (Attempted / Confirmed) ensures that report language is always calibrated to what actually happened. No AI inference introduces false positives or overstates findings. Every confirmed finding has a timestamp, a tester confirmation, and Wazuh corroboration as an evidence trail.

All AI-generated sections are explicitly labeled as AI-drafted until tester-reviewed and approved. The audit trail shows which sections were modified by the tester versus accepted as-is, providing a clear record of human oversight — a direct response to client and legal pushback on AI-generated compliance documentation.

The coverage matrix provides evidence that specific attack categories were tested, supporting compliance frameworks that require documented scope coverage (PTES, OWASP Testing Guide, NIST SP 800-115).

---

## 11. Competitive Landscape

### What Already Exists

**Agentic pentest platforms** (Strix, PentAGI, XBOW, NodeZero): These tools drive the test themselves and report on what they did. They cannot document what a human tester did using their own judgment and tooling.

**AI report assistants** (PentestGPT, generic LLM wrappers): These take completed scan output or manually entered findings and generate report prose. They require the tester to have already collected and organized the information after the fact.

**PTaaS platforms** (Rapid7, NetSPI, Secureworks): Managed service platforms with integrated reporting. Designed for organizations buying pentest as a service, not for internal red teams or independent consultants.

### Where This Proposal is Different

| Dimension | Existing Solutions | This Proposal |
| :--- | :--- | :--- |
| **Who drives the test** | AI agent or human with no documentation assist | Human tester, fully in control |
| **Outcome accuracy** | AI-inferred, or tester-reconstructed from memory | Tester-confirmed at the moment of execution |
| **When documentation happens** | After the fact | Continuously, in real time |
| **Data source** | Manual tool output export | Wazuh telemetry + dropdown confirmations |
| **Behavior change required** | Significant | 2 seconds per action |
| **Coverage audit trail** | Manual | Automatic from dropdown state |
| **Target infrastructure access** | Sometimes required | Never |

### The Core Differentiator

No existing solution combines **passive telemetry corroboration** with **tester-confirmed attack outcomes** at the moment of execution. This is the gap. The dropdown is not a logging tool the tester fills in — it is a structured confirmation mechanism that captures knowledge that exists for approximately two seconds (the moment the tester sees whether an attack worked) before it begins to fade. Everything else in the product exists to make that two-second interaction produce a complete, accurate, compliance-ready report.

---

## 12. Market Context

The global penetration testing tools market was valued at approximately $2.24 billion in 2025, projected to reach $3.85 billion by 2034 at an 8.2% CAGR. AI integration and reporting automation are identified as primary growth drivers.

Despite this, a gap persists: the market is filling with tools that automate the testing, but the documentation burden on human testers conducting human-led engagements remains largely unaddressed. Human-led testing continues to uncover significantly more vulnerabilities than automated scans — particularly in API security, cloud configuration, and complex exploit chains — meaning the market for tools that support human testers, rather than replace them, is substantial and not yet well-served.

The natural buyers are internal red teams at mid-to-large enterprises, independent penetration testing consultancies, and Managed Security Service Providers (MSSPs) running high volumes of engagements where documentation overhead is a direct operational cost.

---

## 13. Implementation Strategy

**Phase 1 — Dropdown and Telemetry Validation:** Build the attack dropdown as a static prototype. Run a representative engagement, use the dropdown to log actions manually, export the Wazuh session logs, and feed both to an LLM with a structured prompt. Evaluate whether the combined input produces accurate, useful report paragraphs. This experiment takes one to two days and validates the core mechanism before any production engineering begins.

**Phase 2 — Backend Pipeline:** Build the backend service to authenticate with Wazuh, retrieve and pre-process session-scoped logs, and correlate telemetry to dropdown timestamps. Validate the tool recognition corroboration layer against a representative set of engagement recordings.

**Phase 3 — AI Drafting Layer:** Implement live report drafting from dropdown confirmations and correlated telemetry. Build the Attempted / Confirmed language calibration. Implement the coverage matrix auto-generation. Validate output quality with experienced testers reviewing for accuracy.

**Phase 4 — Browser Extension and UX:** Build the extension overlay with the Start/Stop controls, the attack dropdown panel, the real-time report draft view, and the optional one-line note field. Build report export with compliance-aligned templates. Pilot with 3 to 5 testers on real engagements.

**Phase 5 — Rollout and Iteration:** Deploy to a wider user base. Track post-session review time as the primary quality metric — target under 15 minutes. Expand the dropdown library based on attack techniques that testers are adding via the free-text field.

---

## 14. Conclusion

Penetration testing documentation is broken. The tools that could theoretically help either automate the test itself or assist with writing after the fact. Neither approach captures the one thing that makes a pentest report accurate: the tester's knowledge, in the moment of execution, of what they did and whether it worked.

This proposal captures that knowledge with a two-second tap. The tester confirms the attack. Wazuh corroborates the execution. The AI writes the report. By the time they stop working, the documentation is done.

The result is a tool that is genuinely useful to the people who need it most — not because it replaces their expertise, but because it captures it at the exact moment it exists, before memory degrades it into something less useful than the truth.
