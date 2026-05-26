# Strategic Proposal: AI-Assisted Penetration Test Report Generation via Wazuh Telemetry

---

## Executive Summary

Penetration testing is a discipline defined by technical depth, adversarial creativity, and exhaustive methodology. Yet the final deliverable — the report — is consistently the most painful and error-prone part of the engagement. Testers forget what they did, in what order, and at what time. Reconstructing a timeline after the fact from memory leads to thin, incomplete reports that fail to capture the full scope of the work performed.

This proposal outlines an architecture for a **browser extension that uses Wazuh as a passive, always-on telemetry backbone to draft penetration test reports in real time**, as the engagement happens. The tester does their work normally. The AI writes as they go. By the time they click Stop, the report is largely complete — requiring only a brief review and approval rather than hours of retrospective documentation.

The operative framing is the same as any well-designed AI assist: **assistive, not authoritative.** The AI reduces documentation friction. It does not replace the tester's judgment, fabricate findings, or produce reports that can be treated as final without review.

---

## 1. The Problem Statement

Penetration test reports serve multiple audiences: technical teams who need to understand and reproduce findings, management who need to understand business risk, and compliance frameworks that require evidence of security validation. A good report demands precision — exact timestamps, command sequences, target responses, and analyst reasoning for every significant action.

The problem is structural. During an active engagement, a tester's cognitive resources are entirely consumed by the work itself: reconnaissance, enumeration, exploitation, lateral movement. Stopping to document breaks flow and misses context. Attempting to reconstruct everything afterward from memory is unreliable — actions blur together, timestamps are estimated, and the reasoning behind decisions ("I targeted this service because...") is often lost entirely.

Existing tools do not solve this. Most AI-assisted pentest platforms are **agentic** — they drive the test themselves and report on what they did. These tools are useful for known vulnerability classes and automated scans, but they cannot replicate the judgment of an experienced human tester working through complex, logic-based attack chains. For human-led engagements, the documentation problem remains entirely unsolved.

---

## 2. The Proposed Solution

A lightweight browser extension with a **Start / Stop session button** overlaid on the Wazuh dashboard. When the tester clicks Start, the extension timestamps the beginning of the engagement session. From that point forward, Wazuh — already deployed on the pentester's machine and collecting telemetry continuously — serves as the single source of truth for what happened.

As the session progresses, the extension periodically pulls the accumulating logs from Wazuh for that time window and feeds them to an AI layer. The AI does not wait until the end to begin writing. It **drafts the report continuously**, building the narrative in real time as actions are detected. Process executions, network connections, file writes, tool signatures — all of it is translated into structured report language as it happens.

When the tester clicks Stop, the draft report is complete. The tester reviews it, fills in any flagged gaps where the AI lacked confidence, and approves. The time from Stop to deliverable-ready report is measured in minutes, not hours.

---

## 3. Design Philosophy: Assistive, Not Authoritative

This distinction is fundamental to the tool's value and its safe use.

The AI is a **live scribe**, not an analyst. Its job is to translate what the logs show into coherent, structured prose. It does not invent findings. It does not assess severity without tester confirmation. It does not declare vulnerabilities confirmed or exploits successful unless the telemetry clearly supports that conclusion.

Where the AI is confident — a recognizable Nmap scan, a Metasploit module execution, a Hydra brute-force sequence — it writes directly. Where the telemetry is ambiguous — a series of HTTP requests that could be manual browsing, directory enumeration, or Burp Suite fuzzing — it writes a placeholder and **flags the moment** for the tester to clarify with a single sentence.

The result is a collaborative workflow:

- The AI handles the writing burden for everything it can confidently interpret.
- The tester handles only the gaps the AI cannot fill without guessing.
- No finding is presented as confirmed without tester sign-off.

Every generated report section carries a consistent footer: *"This section is AI-drafted from Wazuh telemetry and requires tester review before submission."*

---

## 4. Why Wazuh on the Pentester's Machine is the Right Data Source

A common assumption is that useful pentest telemetry must come from the target. This proposal inverts that assumption deliberately.

Wazuh deployed on the **pentester's machine** captures a complete record of the tester's actions: process executions (Nmap, Metasploit, Hydra, Gobuster, etc.), network connections initiated, files written or modified, command-line activity, and tool output. This is precisely the data needed to reconstruct what the tester did — not what the target experienced.

The target's perspective — alert responses, service behavior, exploit confirmation — is captured through the tester's observations and tool outputs, which are themselves reflected in the telemetry. When an exploit succeeds, the tester's machine shows the resulting session. When a credential works, the authenticated connection appears in the network log. The attacker-side record is sufficient to reconstruct the engagement narrative.

This architecture has two additional advantages. First, it requires no access to or modification of the target's infrastructure, which is a hard constraint in most real engagements where the tester does not control the client's environment. Second, Wazuh is already deployed across many security teams as part of their standard tooling. There is no new infrastructure to provision — the data collection layer already exists.

---

## 5. Architectural Blueprint

### Step 1: Session Scoping (Browser Extension)

The extension provides a minimal UI overlay on the Wazuh dashboard. The tester clicks **Start** at the beginning of the engagement. The extension records the precise timestamp and associates it with a named session (engagement name, target scope, tester identity).

Optionally, the tester can mark **phase transitions** during the engagement — Reconnaissance, Enumeration, Exploitation, Post-Exploitation, Reporting — with a single tap. These phase markers are not required, but when present they give the AI structural anchors that significantly improve report organization and reduce the number of flagged gaps.

When the tester clicks **Stop**, the session window is closed and the final report generation pass begins.

### Step 2: Telemetry Retrieval (Backend Service)

The extension communicates with a lightweight internal backend service. This service authenticates with the Wazuh API and retrieves all events from the session time window for the tester's agent. It applies pre-processing to:

- Deduplicate repeated identical events (e.g., the same scan run multiple times)
- Group related events into logical clusters by tool signature and time proximity
- Strip noise events irrelevant to pentest activity (routine OS housekeeping, etc.)
- Preserve forensically significant fields: timestamps, process names, command-line arguments, source and destination IPs, ports, file paths, rule IDs

The pre-processed event stream is structured as an ordered timeline before being passed to the AI layer.

### Step 3: Live Report Drafting (AI Layer)

The AI receives the structured event timeline and drafts report sections continuously as new events arrive. Its behavior is governed by a confidence threshold:

**High confidence** — tool signature is unambiguous, action is clear:

> *14:03 — Tester initiated a TCP SYN port scan against 192.168.1.0/24 using Nmap, identifying open ports 22 (SSH), 80 (HTTP), and 443 (HTTPS) on host 192.168.1.42.*

**Low confidence** — ambiguous activity requiring tester clarification:

> *14:17 — Multiple HTTP requests detected toward 192.168.1.42:80. [FLAG: Unable to determine if this represents manual browsing, directory enumeration, or active fuzzing. Please add one line of context.]*

The tester sees the draft building in the extension's side panel as they work. Flags are highlighted. At session end, they address only the flagged items.

### Step 4: Report Assembly and Export

The completed, tester-reviewed draft is assembled into a structured report format covering:

- Engagement metadata (scope, dates, tester, methodology)
- Chronological activity log with phase organization
- Findings summary with tester-confirmed severity ratings
- Evidence appendix (tool outputs, timestamps, screenshots if attached)

Export formats: PDF, DOCX, and Markdown. Compliance-aligned templates (PTES, OWASP Testing Guide, NIST SP 800-115) are selectable at session start.

---

## 6. Tool Recognition Layer

The accuracy of the AI's confident-writing depends on how well it recognizes common pentest tools from Wazuh telemetry. The following table reflects the recognizability of the most common tooling:

| Tool | Wazuh Signal | AI Confidence |
| :--- | :--- | :--- |
| **Nmap** | Process execution with characteristic arguments (-sS, -sV, -p, -A, -O) | High |
| **Metasploit** | msfconsole / msfdb process; network connections to target on exploit ports | High |
| **Hydra / Medusa** | Process execution with wordlist arguments; rapid repeated connection attempts | High |
| **Gobuster / Dirb / Feroxbuster** | Process execution; high-volume sequential HTTP requests to target | High |
| **Burp Suite** | Java process; proxy port activity; HTTP traffic through local proxy | Medium |
| **SQLMap** | Process execution with URL arguments; characteristic SQL injection payloads in requests | High |
| **Nikto** | Process execution; characteristic HTTP request patterns | High |
| **Netcat / Socat** | Network connection establishment; common shell port activity | Medium |
| **Manual browser activity** | HTTP requests with no associated tool process | Low — flagged |
| **Custom scripts** | Unrecognized process; requires tester annotation | Low — flagged |

The recognition library is extensible. Teams can add signatures for internal tools or less common frameworks through a simple configuration file.

---

## 7. The Tester's Workflow

The total behavior change required from the tester is minimal by design:

1. **Click Start** when the engagement begins.
2. **Work normally.** Use whatever tools, techniques, and methodology the engagement requires. Nothing changes.
3. *(Optional)* **Tap phase markers** when transitioning between engagement phases. This takes under five seconds per phase and meaningfully improves report structure.
4. **Click Stop** when the engagement concludes.
5. **Review the draft** in the extension's side panel. Address flagged items with one-line annotations. Confirm findings.
6. **Export.** The report is done.

The annotation step at the end — addressing flagged items and confirming findings — is estimated to take 10 to 20 minutes for a typical engagement session, compared to 2 to 4 hours of retrospective writing from memory. The quality of output is also higher: timestamps are exact, tool sequences are complete, and no actions are forgotten.

---

## 8. Edge Cases and Failure Modes

| Scenario | Handling |
| :--- | :--- |
| **Wazuh agent offline or unreachable** | Extension displays a status indicator and degrades gracefully. Session timestamps are preserved; tester is notified that telemetry may be incomplete. |
| **AI produces an inaccurate summary** | Tester corrects it during review. The report is not submitted without explicit tester sign-off on each section. No downstream system depends on unreviewed AI output. |
| **Tool not recognized** | Activity is logged with timestamp and flagged for tester annotation. No finding is dropped; the tester fills in the context. |
| **High-volume engagement generates thousands of events** | Pre-processing deduplication and clustering reduce the event stream before AI processing. Report sections are generated incrementally to avoid context length constraints. |
| **Tester forgets to click Start** | Session can be manually backdated to a specific timestamp if the tester catches the omission early. Wazuh archives retain all events, so retrospective session scoping is possible within the archive retention window. |
| **Sensitive client data in logs** | All processing occurs on internal infrastructure. No log data is transmitted to external APIs. The same zero-exfiltration guarantee from the SOC tool proposal applies here. |

---

## 9. Compliance and Legal Considerations

Pentest reports in regulated environments carry compliance weight. Several design decisions address this directly.

All AI-generated sections are explicitly labeled as AI-drafted until tester-reviewed and approved. The audit trail shows which sections were modified by the tester versus accepted as-is, providing a clear record of human oversight.

The session log — a complete timestamped record of every action taken during the engagement — is preserved as an appendix and can serve as evidence of methodology in compliance audits or legal proceedings where the scope of the engagement is disputed.

Report templates aligned to PTES (Penetration Testing Execution Standard), OWASP Testing Guide, and NIST SP 800-115 ensure that output meets the structural requirements of common compliance frameworks without requiring the tester to manually format to spec.

---

## 10. Competitive Landscape

The AI pentest tool market is active and growing. Understanding where this proposal sits relative to existing solutions is necessary for evaluating the build decision.

### What Already Exists

**Agentic pentest platforms** (Strix, PentAGI, XBOW, NodeZero): These tools drive the test themselves. They are valuable for automated scanning of known vulnerability classes but cannot replicate human judgment in complex, logic-based attack chains. They generate reports on what they did — but they are not designed to document what a human tester did.

**AI report assistants** (PentestGPT, generic LLM wrappers): These take completed scan output or manually entered findings and generate report prose. They require the tester to have already collected and organized the information — they reduce writing time but do not solve the documentation-during-engagement problem.

**PTaaS platforms** (Rapid7, NetSPI, Secureworks): Managed service platforms with integrated reporting. Designed for organizations buying pentest as a service, not for internal red teams or independent consultants documenting their own work.

### Where This Proposal is Different

No existing solution uses **passive telemetry from the tester's own machine** as the documentation source. The key differentiators:

| Dimension | Existing Solutions | This Proposal |
| :--- | :--- | :--- |
| **Who drives the test** | AI agent (agentic tools) or human with no documentation assist | Human tester, fully in control |
| **When documentation happens** | After the fact, from memory or tool output | Continuously, in real time, during the engagement |
| **Data source** | Tool output exported manually, or AI agent's own action log | Wazuh telemetry from the tester's machine — passive, automatic |
| **Tester behavior change required** | Significant (adopt new toolchain or workflow) | Minimal (Start/Stop button; optional phase tags) |
| **Report quality dependency** | Tester's ability to reconstruct events afterward | Exact timestamps and tool sequences from live telemetry |
| **Target infrastructure access required** | Sometimes (agent-based tools) | Never |

### The Core Differentiator

The most significant gap this proposal fills is **temporal accuracy without behavioral burden**. Existing report-assist tools produce better prose, but they still depend on the tester to supply accurate, complete information after the fact. This proposal makes completeness automatic — the telemetry captures everything — and reduces the tester's contribution to confirmation and context-filling rather than reconstruction.

---

## 11. Market Context

The global penetration testing tools market was valued at approximately $2.24 billion in 2025, projected to reach $3.85 billion by 2034 at an 8.2% CAGR. AI integration and reporting automation are identified as primary growth drivers.

Despite this, a gap persists: the market is filling with tools that automate the *testing*, but the documentation burden on human testers conducting human-led engagements remains largely unaddressed. Human-led testing continues to uncover significantly more vulnerabilities than automated scans — particularly in API security, cloud configuration, and complex exploit chains — meaning the market for tools that support human testers, rather than replace them, is substantial and not yet well-served.

The natural buyers are internal red teams at mid-to-large enterprises, independent penetration testing consultancies, and Managed Security Service Providers (MSSPs) running high volumes of engagements where documentation overhead is a direct operational cost.

---

## 12. Implementation Strategy

The project is executed in sequenced phases, validating each layer before building on top of it.

**Phase 1 — Telemetry Validation:** Deploy Wazuh on a test pentest machine. Run a representative engagement (mix of Nmap, Metasploit, Hydra, Burp Suite). Export the session's Wazuh logs. Feed them to an LLM with a structured prompt and evaluate whether the output is coherent and useful. This experiment takes one day and answers the most fundamental question before any engineering begins. If the raw material is good, proceed. If the signal is insufficient, identify what's missing and address it first.

**Phase 2 — Backend Pipeline:** Build the backend service to authenticate with Wazuh, retrieve and pre-process session-scoped logs, and deliver a structured event timeline to the AI layer. Build and validate the tool recognition library against a representative set of engagement recordings.

**Phase 3 — AI Drafting Layer:** Implement the live report drafting logic. Define confidence thresholds for direct writing versus flagging. Build the flag resolution UI. Validate output quality against real engagement sessions with experienced testers reviewing for accuracy and completeness.

**Phase 4 — Browser Extension and UX:** Build the extension overlay. Implement Start/Stop, phase tagging, and the real-time draft side panel. Build report export with compliance-aligned templates. Conduct a pilot with 3 to 5 testers on real engagements and iterate on friction points.

**Phase 5 — Rollout and Iteration:** Deploy to a wider user base. Instrument the flag rate per session — if testers are resolving more than 15 to 20 flags per engagement, the recognition library needs improvement. Target a steady-state where the tester's post-session annotation takes under 15 minutes.

---

## 13. Conclusion

Penetration testing documentation is broken. The tools that could theoretically help either automate the test itself (removing the human) or assist with writing after the fact (still requiring the human to remember everything). Neither approach solves the real problem: that the information needed for a complete, accurate report exists in the moment the engagement is happening, and disappears from memory almost immediately after.

This proposal captures that information passively, in real time, from infrastructure the team already has. The tester does not change how they work. The AI writes as they go. The report review is a confirmation exercise, not a reconstruction exercise.

The result is a tool that is genuinely useful to the people who need it most — not because it replaces their expertise, but because it removes the documentation burden that has always been the worst part of using that expertise.