# Strategic Proposal: AI-Assisted, Privacy-First Wazuh Log Analyzer

---

## Executive Summary

The modern Security Operations Center (SOC) faces a critical data volume crisis. SIEM platforms like Wazuh successfully aggregate massive amounts of security telemetry, but the cognitive burden of reading, correlating, and interpreting millions of raw JSON logs falls entirely on human analysts. This manual data parsing inflates Mean Time to Resolve (MTTR) and leads to severe analyst fatigue.

This proposal outlines the architecture and business case for a self-hosted, AI-driven **assistive** tool. The operative word is assistive. This system is not designed to replace analyst judgment, render verdicts on incidents, or serve as an authoritative source of truth. It is designed to give analysts a fast, human-readable orientation to a dataset — a starting point, not a conclusion — so they know where to focus their investigation rather than where to end it.

By integrating a lightweight browser extension directly into the analyst's existing Wazuh workflow, we pipe filtered telemetry to a locally hosted Small Language Model (SLM). The result is instant, contextual summarization of complex security events, achieved with **zero workflow friction, zero recurring API token costs, and absolute data privacy.**

---

## 1. The Problem Statement

Currently, incident investigation requires analysts to manually scroll through thousands of lines of JSON logs in the Wazuh Discover tab. They must manually cross-reference timestamps, extract Indicators of Compromise (IoCs) like IPs and file hashes, and mentally reconstruct the attack chain.

While cloud-based Large Language Models (LLMs) like GPT-4 or Claude could theoretically summarize this data, they are fundamentally unsuited for a secure SOC environment for three reasons:

1. **Data Privacy & Compliance Risks:** Transmitting sensitive internal network logs, user credentials, and unpatched vulnerability telemetry to a third-party API violates core zero-trust principles and potentially breaches compliance frameworks (SOC 2, GDPR, HIPAA).
2. **Unpredictable API Costs:** Cloud LLMs charge per token. Feeding thousands of logs into an LLM for dozens of daily investigations results in highly unpredictable and potentially exorbitant monthly costs that scale with incident volume.
3. **Workflow Disruption:** Forcing an analyst to export logs to a CSV and upload them to a separate AI interface breaks their flow during critical, time-sensitive investigations.

---

## 2. The Proposed Solution

We propose building a proprietary, assistive AI pipeline tailored explicitly for Wazuh.

The core of the user experience is a **browser extension** that acts as an overlay on the Wazuh dashboard. Analysts use the native Wazuh/OpenSearch filters they already know. Once they isolate a suspicious timeframe and apply their filters, they click a "Capture" button in the extension.

Behind the scenes, the extension captures the query parameters and sends them to a secure, self-hosted backend server. This server dynamically queries the Wazuh API, downloads the relevant JSON logs, aggressively pre-processes them, and feeds them into a dedicated SLM hosted on internal company hardware. Within seconds, a correlated incident summary is streamed back to the analyst's browser as a **suggested narrative** — clearly framed as a starting orientation, not a definitive finding.

---

## 3. Design Philosophy: Assistive, Not Authoritative

This distinction is fundamental to how the tool is built, deployed, and used.

The AI summary is explicitly framed in the UI as a **"Suggested Starting Point"**. The summary will:

- Highlight the most statistically anomalous events in the filtered dataset
- Surface potential correlations between events for the analyst to verify
- Call out relevant IoCs (IPs, hashes, usernames) worth investigating first
- Clearly indicate its own confidence level and flag ambiguous patterns

The summary will never:

- Declare an incident confirmed or closed
- Recommend remediation actions autonomously
- Be logged or used as the sole basis for any security decision

Every summary ends with a consistent footer: *"This summary is AI-generated and requires analyst verification before any action is taken."*

**SOC SOP integration:** Onboarding documentation will explicitly train analysts — particularly Level 1 — to treat the AI output the way they would treat a brief from a junior colleague: useful orientation, not gospel. Verification against raw logs is always the required next step.

---

## 4. The Strategic Value & ROI

- **Absolute Data Privacy:** The SLM is hosted entirely on internal infrastructure. Data never leaves the company network, ensuring total compliance and zero risk of telemetry leakage.
- **Zero Learning Curve (High Adoption):** The tool eliminates prompt engineering. Analysts do what they do best — filter data natively in Wazuh — and the system translates those filters into the AI's context. It supercharges an existing workflow rather than forcing a new one.
- **Fixed Operational Cost:** By utilizing an SLM rather than commercial APIs, the computational cost is fixed to the hardware. Whether the SOC runs 10 investigations a day or 10,000, the per-investigation cost after initial hardware procurement is effectively zero.
- **Reduction in Analyst Orientation Time:** What currently requires 20 to 40 minutes of manual log triage to understand *where to begin* can be reduced to a 10 to 30-second AI-generated orientation, allowing analysts to spend their cognitive energy on the verification and remediation work that actually requires human judgment.

### Hardware Cost Transparency

This proposal acknowledges upfront that the fixed-cost model requires an initial capital investment. A single enterprise GPU capable of running a performant SLM — such as an NVIDIA L40S or equivalent — carries a procurement cost in the range of $10,000–$20,000 USD. This is a one-time infrastructure cost that replaces recurring per-token API spend and should be evaluated against projected investigation volume. A break-even analysis should be conducted during Phase 1 planning based on the SOC's actual daily investigation load.

---

## 5. Architectural Blueprint

To handle potentially millions of JSON logs without impacting the analyst's browser, the architecture uses a "thin client, heavy backend" approach.

### Step 1: The UI Trigger (Browser Extension)

- **Mechanism:** The extension does *not* read raw logs rendering on the DOM.
- **Action:** It silently captures the page state — specifically the OpenSearch query parameters, applied filters, and the selected time range.
- **Handoff:** It transmits these lightweight parameters to the internal backend server over a mutually authenticated internal HTTPS connection.

**Extension Security Posture:** Because this tool operates exclusively within the internal network, its external attack surface is minimal. However, the following hardening measures are applied:

- The extension communicates only with the pre-configured internal backend endpoint; no external domains are whitelisted
- Enterprise browser policy controls (Chrome/Edge managed deployment) are used to lock the extension to approved workstations, preventing personal browser installation
- The extension requests only the minimum browser permissions required: access to the Wazuh dashboard tab URL and the ability to make internal network requests
- Cross-browser compatibility is scoped to the SOC's standardized browser during initial rollout, with expansion evaluated post-stabilization

### Step 2: The Orchestrator & Data Pipeline (Backend Server)

- **Data Fetching:** The backend server (built in Go or Python for high concurrency) connects directly to the Wazuh Indexer API, executes the analyst's exact query, and downloads the raw JSON logs securely.
- **Credential Management:** Wazuh API credentials used by the backend service are stored in an internal secrets manager (e.g., HashiCorp Vault or equivalent), rotated on a defined schedule, and scoped to read-only access on the relevant index patterns. The service account has no write permissions.
- **Audit Logging:** Every query made through the backend service is logged — including which analyst triggered it, what filters were applied, and what time range was queried. This creates a full audit trail of AI-assisted investigations for compliance and internal review purposes.
- **Access Control:** The backend service is accessible only from analyst workstations on the SOC network segment. Access is enforced at the network layer via firewall rules and at the application layer via internal mTLS client certificates.

**Pre-Processing (Critical Step):** SLMs have limited context windows. The backend script parses the JSON logs, strips empty or irrelevant fields, and aggregates repetitive events. For example, 5,000 identical SSH failure logs become a single line: `[5,000 failed SSH logins from IP 192.168.1.50 between 14:00–14:05]`. This reduces data payload by up to 90% on noisy, high-repetition datasets.

**Important caveat:** Aggressive aggregation is tuned carefully. For investigation scopes involving low-and-slow adversary behavior — where each unique event is forensically significant — the aggregation thresholds are set conservatively, preserving granularity over compression. The aggregation logic is configurable per investigation type.

### Step 3: The Intelligence Engine (Self-Hosted SLM)

- **The Model:** A Small Language Model (7B–8B parameters, or larger if hardware supports) receives the dense, pre-processed context and generates a narrative summary.
- **Why an SLM?:** Security log correlation is a narrow, structured task focused on pattern recognition, timeline reconstruction, and anomaly flagging. An SLM appropriately sized for this task — summarizing pre-processed, structured data rather than performing open-ended reasoning — is well-suited to the assistive use case described in this proposal.
- **Inference Engine:** The model is served via vLLM or a comparable optimized inference engine on the internal GPU server, providing low-latency token generation.
- **Model Limitations — Acknowledged:** Current SLMs at the 7B–8B parameter range can produce plausible-sounding but inaccurate correlations. This is accepted and mitigated through the assistive framing described in Section 3. The tool's value is in reducing orientation time, not in being infallible.

### Step 4: The Delivery

The SLM processes the pre-processed timeline, identifies potential correlations (e.g., *a brute-force sequence followed by a successful login and a subsequent file integrity change in /etc/shadow*), and streams the human-readable summary back to the browser extension. The summary is clearly labeled as AI-generated and includes a direct link to the relevant raw log view in Wazuh for immediate analyst verification.

---

## 6. System Dynamics & Realistic Performance Estimates

The following estimates assume a standard enterprise internal network, a single high-end GPU for SLM inference, and the pre-processing pipeline operating at full efficiency. These are targets informed by benchmark data, not guarantees. Actual performance will be validated during Phase 2 load testing.

| Investigation Scope | Estimated Log Volume | Estimated End-to-End Time | Notes |
| :--- | :--- | :--- | :--- |
| **Targeted Event** | 10,000–50,000 logs | ~5–10 seconds | Feels nearly instantaneous. |
| **Complex Subnet Sweep** | 100,000–300,000 logs | ~15–25 seconds | Short wait, fluid workflow. |
| **Massive Incident** | 1,000,000+ logs | ~45–90 seconds | A loading state is shown; saves hours of manual triage. JSON parsing and I/O at this scale is the primary bottleneck, not model inference. |

Performance benchmarks will be measured formally during Phase 2 and Phase 3 and used to set accurate analyst expectations prior to SOC rollout.

---

## 7. Failure Modes & Resilience

A production security tool must define its behavior when things go wrong.

| Failure Scenario | System Behavior |
| :--- | :--- |
| **SLM server is offline** | Backend returns a clear error state to the extension. The analyst is notified immediately and continues investigation manually without degraded Wazuh functionality. |
| **Context window exceeded** | Backend detects oversized payload before submission and either further aggregates, truncates with a visible warning, or splits into sequential summaries. Analyst is informed of the truncation. |
| **Pre-processing drops unexpected fields** | Configurable field preservation list ensures forensically critical fields (timestamps, source IPs, rule IDs, user accounts) are never stripped regardless of aggregation settings. |
| **Backend service is unreachable** | Extension fails silently without impacting the Wazuh dashboard. A non-intrusive status indicator shows the AI assist is unavailable. |
| **Model produces a nonsensical summary** | The analyst dismisses it and proceeds manually. No downstream system depends on the AI output; it is advisory only. |

---

## 8. Model Maintenance & Drift

Threat actor TTPs evolve. The SLM, once deployed, is a static artifact unless actively maintained. The following operational commitments address this:

- **Quarterly Model Review:** Every quarter, the SOC team reviews whether the AI summaries remain directionally accurate for the current threat landscape. If significant drift is detected, model replacement or supplementary prompt engineering is evaluated.
- **Fine-Tuning (Optional, Phase 4):** If the SOC wishes to tune the model's output style to match internal reporting formats, a parameter-efficient fine-tune (LoRA) can be applied. This requires a curated dataset of several hundred to a few thousand high-quality analyst-written summaries and dedicated ML engineering time. This is an optional enhancement, not a launch requirement, and its scope should not be underestimated. It is listed as a Phase 4 activity with its own dedicated planning effort.
- **Model Replacement:** The inference server is model-agnostic. As better open-weights models become available, swapping the underlying model requires no changes to the pipeline architecture.

---

## 9. Compliance & Documentation

Even for a self-hosted, internal tool, AI-assisted security decisions require documentation in regulated environments.

- All AI-generated summaries that correspond to a formal incident investigation are logged alongside the raw investigation record, making it auditable which investigations used AI assistance.
- The audit log (see Section 5, Step 2) provides a complete record of what data was queried, by whom, and when.
- Compliance documentation explicitly classifies this tool as an **analyst decision-support aid**, not an automated decision system, which carries a lower compliance burden under most frameworks (SOC 2, ISO 27001, GDPR).

---

## 10. Implementation Strategy

The project is executed in a modular, sequenced pipeline that validates each layer before building on top of it.

1. **Phase 1 — Wazuh API & Data Pipeline:** Develop the backend service to authenticate with Wazuh, dynamically translate OpenSearch parameters from the browser, download logs in bulk, and execute the deduplication and aggregation logic. Validate pipeline performance and output quality independently of the AI layer. Conduct break-even hardware cost analysis.

2. **Phase 2 — SLM Server Provisioning:** Procure and configure the internal GPU server. Deploy an open-weights SLM using vLLM. Establish the internal API endpoint. Conduct load testing against the performance targets in Section 6. Validate that the model produces directionally useful summaries on representative SOC datasets.

3. **Phase 3 — Browser Extension Development:** Build the lightweight extension. Implement mTLS communication with the backend. Enforce enterprise browser policy deployment. Conduct security review of extension permissions. Roll out to a pilot group of analysts for feedback.

4. **Phase 4 — SOC Rollout, SOP Integration & Optional Tuning:** Deploy to the full analyst team with explicit SOP documentation establishing the assistive framing. Train analysts — particularly Level 1 — on appropriate use and the requirement to verify AI output against raw logs. If fine-tuning is pursued based on Phase 3 feedback, scope and resource it as a standalone project with proper ML engineering support.

---

## 11. Conclusion

This architecture represents a practical, privacy-respecting augmentation of how Level 1 and Level 2 SOC analysts orient themselves within large datasets. By being explicit about what the tool is — an AI-powered starting point, not an AI-powered verdict — we avoid the most common failure mode of security AI tooling: eroding analyst judgment rather than sharpening it.

The pipeline is self-contained, auditable, and operationally resilient. It respects data sovereignty, replaces unpredictable API spend with fixed infrastructure cost, and integrates into the workflow analysts already use. Most importantly, it is designed to make skilled analysts faster, not to substitute for them.