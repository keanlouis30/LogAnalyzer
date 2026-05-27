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

We propose building a proprietary, assistive AI pipeline tailored explicitly for Wazuh, with initial support for the Wazuh dashboard. Support for Kibana and OpenSearch dashboards will be added in subsequent development phases as the URL parameter parsing layer is extended.

The core of the user experience is a **browser extension** that acts as an overlay on the Wazuh dashboard. Analysts use the native Wazuh filters they already know. Once they isolate a suspicious timeframe and apply their filters, they click a **Capture** button in the extension.

Behind the scenes, the extension captures the active URL — including all filter parameters and the selected time range — and sends it to a secure, self-hosted backend server. This server parses the URL parameters, queries the Wazuh API directly using those parameters, downloads the relevant JSON logs, aggressively pre-processes them, and feeds them into a dedicated SLM hosted on internal company hardware. The SLM returns a structured report written in Markdown, displayed directly in the extension and expandable to a full browser tab.

---

## 3. Design Philosophy: Assistive, Not Authoritative

This distinction is fundamental to how the tool is built, deployed, and used.

The AI report is explicitly framed as a **"Suggested Starting Point"**. The report will:

- Provide a hypothesis of what likely occurred based on the log evidence
- Surface the actual findings derived from the log data
- Propose generic remediation steps as a reference, not a prescription
- Display a per-section confidence score derived from token-level log probabilities — not from the model's self-assessment

The report will never:

- Declare an incident confirmed or closed
- Recommend remediation actions autonomously or with authority
- Be used as the sole basis for any security decision

Every report ends with a consistent footer: *"This report is AI-generated from filtered Wazuh telemetry and requires analyst verification before any action is taken."*

**SOC SOP integration:** Onboarding documentation will explicitly train analysts — particularly Level 1 — to treat the AI output the way they would treat a brief from a junior colleague: useful orientation, not gospel. Verification against raw logs is always the required next step.

---

## 4. Report Structure

The SLM produces a structured Markdown report with three defined sections. Using Markdown as the output format minimizes token consumption, constrains the model's output shape to reduce hallucination risk, and renders identically to a formatted document in the extension's built-in viewer.

Each section carries a **confidence score** badge derived from the mean log probability of the tokens generated in that section. This score reflects the mathematical certainty of the model's output — not the model's self-reported confidence, which is itself subject to hallucination. A low confidence score on any section is a direct signal to the analyst to treat that section with additional skepticism and verify carefully against raw logs.

### Section 1: Hypothesis
*Confidence score displayed as a badge on the section header.*

The SLM's best reconstruction of what likely happened based on the log evidence. Written as "here is what we think occurred" — descriptive, not prescriptive. The hypothesis does not tell the analyst what to investigate next; it tells them what the data appears to suggest so they can evaluate it critically. The language is deliberately hedged: "the logs are consistent with," "this pattern may indicate," "a possible explanation is."

### Section 2: Actual Findings
*Confidence score displayed as a badge on the section header.*

A structured presentation of what the logs concretely show: specific events, timestamps, source and destination IPs, affected accounts, rule IDs triggered, and observed sequences. This section stays close to the data — less inference, more extraction. It is the evidentiary layer that either supports or complicates the hypothesis.

### Section 3: Proposed Remediation
*Confidence score displayed as a badge on the section header.*

Generic best-practice remediation steps relevant to the finding type identified. These are reference suggestions — block the IP, rotate the credentials, patch the service — not environment-specific prescriptions. The section is explicitly labeled as proposed and generic. Environment-specific remediation remains the analyst's responsibility.

---

## 5. Confidence Scoring via Token Log Probabilities

The confidence score is a first-class feature of both this tool and the Pentest Report Generator. It is documented here in full as the canonical reference.

### Why Logprobs, Not Self-Assessment

Asking the SLM to rate its own confidence produces a number that is itself generated by the same model that may be hallucinating. It is unreliable by design. Token log probabilities are a mathematical property of the model's output — the probability distribution the model assigned to each token it generated. High mean logprob across a section means the model generated those tokens with high certainty. Low mean logprob means the model was uncertain, hedging between multiple possible continuations. This is an honest signal that does not depend on the model's self-awareness.

### Computation Method

1. The backend requests logprobs alongside the generated text from the inference engine.
2. For each generated section (Hypothesis, Findings, Remediation), the mean log probability of all tokens in that section is computed.
3. The mean logprob is converted to a 0–100 confidence score using a calibrated scaling function.
4. The score is passed back to the extension alongside the report text and displayed as a color-coded badge on each section header.

**Score interpretation:**

| Score Range | Badge Color | Analyst Guidance |
| :--- | :--- | :--- |
| 80–100 | Green | Model generated this section with high token certainty. Standard verification applies. |
| 50–79 | Yellow | Model showed meaningful uncertainty. Treat as a starting hypothesis only. |
| 0–49 | Red | Model was significantly uncertain. This section requires close manual verification before any reliance. |

### Inference Engine Requirement

Logprob support is a **hard requirement** for SLM inference engine selection. The exact model is undecided pending evaluation, but any candidate inference engine must expose per-token log probabilities in its API response. vLLM is the leading candidate as it exposes logprobs cleanly, supports most open model architectures, and is production-grade for self-hosted deployments. This requirement must be validated against any alternative inference engine before it is adopted.

### Purpose of the Confidence Score

The confidence score serves three audiences:

- **The SOC analyst:** Immediate signal on which sections to scrutinize most carefully before acting.
- **The tool developer:** Empirical data on where the model performs well and where it degrades, informing fine-tuning priorities.
- **Compliance documentation:** Evidence that AI-assisted findings were evaluated with awareness of model uncertainty, supporting the assistive-not-authoritative classification.

---

## 6. The Strategic Value & ROI

- **Absolute Data Privacy:** The SLM is hosted entirely on internal infrastructure. Data never leaves the company network, ensuring total compliance and zero risk of telemetry leakage.
- **Zero Learning Curve (High Adoption):** The tool eliminates prompt engineering. Analysts do what they do best — filter data natively in Wazuh — and the system translates those filters into the AI's context. It supercharges an existing workflow rather than forcing a new one.
- **Fixed Operational Cost:** By utilizing an SLM rather than commercial APIs, the computational cost is fixed to the hardware. Whether the SOC runs 10 investigations a day or 10,000, the per-investigation cost after initial hardware procurement is effectively zero.
- **Reduction in Analyst Orientation Time:** What currently requires 20 to 40 minutes of manual log triage to understand where to begin can be reduced to a 10 to 30-second AI-generated orientation, allowing analysts to spend their cognitive energy on the verification and remediation work that actually requires human judgment.

### Hardware Cost Transparency

This proposal acknowledges upfront that the fixed-cost model requires an initial capital investment. A single enterprise GPU capable of running a performant SLM — such as an NVIDIA L40S or equivalent — carries a procurement cost in the range of $10,000–$20,000 USD. This is a one-time infrastructure cost that replaces recurring per-token API spend and should be evaluated against projected investigation volume. A break-even analysis should be conducted during Phase 1 planning based on the SOC's actual daily investigation load.

---

## 7. Architectural Blueprint

To handle potentially millions of JSON logs without impacting the analyst's browser, the architecture uses a thin client, heavy backend approach.

### Step 1: The UI Trigger (Browser Extension)

The analyst applies their filters natively in the Wazuh dashboard — time range, agent, rule ID, severity, or any combination. Once satisfied with the filter state, they click **Capture** in the extension overlay.

The extension does not read raw logs from the DOM. It captures the active page URL, which encodes all applied filter parameters and the selected time range as query parameters. These lightweight parameters are transmitted to the internal backend server over a mutually authenticated internal HTTPS connection.

**Platform support:** The prototype supports the Wazuh dashboard URL parameter structure. Kibana and OpenSearch dashboard support will be added in a subsequent development phase as the URL parsing layer is extended to handle their respective parameter schemas.

**Extension Security Posture:** The extension operates exclusively within the internal network. Hardening measures include read-only DOM permissions, no external network calls, all communication routed through the internal backend only, and enterprise browser policy deployment to prevent unauthorized installation.

### Step 2: Backend Query and Pre-Processing

The backend service receives the URL parameters, parses them into a structured Wazuh API query, and retrieves the relevant log events for the specified time window and filter criteria.

Pre-processing is applied before any data reaches the SLM:

- Remove duplicate events — identical rule triggers within short time windows are collapsed into a single representative event with a count annotation
- Strip fields irrelevant to security analysis (rendering metadata, UI state fields, internal Wazuh housekeeping)
- Preserve forensically critical fields unconditionally: timestamps, source and destination IPs, rule IDs, severity levels, agent names, usernames, file paths, process names
- Structure the cleaned event stream as an ordered timeline

**Note on pre-processing logic:** The pre-processing pipeline is shared between the Log Analyzer and the Pentest Report Generator in the initial implementation. As both tools mature and their use cases diverge, the pre-processing layer will be forked and tuned independently — incident investigation and penetration testing produce different noise patterns that benefit from different deduplication and retention strategies.

### Step 3: SLM Report Generation

The pre-processed event timeline is passed to the SLM with a structured prompt and a rigid Markdown template skeleton. The skeleton defines the three report sections and their expected content, constraining the model's output shape and preventing it from generating unsolicited sections or reordering findings.

The SLM generates the report in Markdown. Logprobs are requested alongside the generated tokens. The backend computes per-section confidence scores from the logprob data and appends them to the report as metadata.

### Step 4: Report Delivery and Display

The completed Markdown report and confidence score metadata are returned to the extension. The extension renders the report inline using its built-in Markdown viewer, with confidence score badges displayed on each section header.

The analyst can expand the report to a full browser tab for comfortable reading and review. From the full tab view, the report can be downloaded in three formats:

- **Markdown (.md)** — the native output format, suitable for version control and further editing
- **PDF** — for formal documentation and sharing
- **DOCX** — for integration into existing Word-based reporting workflows

---

## 8. Edge Cases and Failure Modes

| Scenario | Handling |
| :--- | :--- |
| **URL parameters cannot be parsed** | Extension displays an error state. Analyst is prompted to verify their filter state and retry. No partial query is sent. |
| **Wazuh API returns zero events for the filter window** | Backend returns a no-data response. Extension displays a clear message: "No log events found for the selected filter. Adjust the time range or filters and retry." |
| **Pre-processing drops unexpected fields** | Configurable field preservation list ensures forensically critical fields are never stripped regardless of aggregation settings. |
| **Backend service is unreachable** | Extension fails silently without impacting the Wazuh dashboard. A non-intrusive status indicator shows the AI assist is unavailable. |
| **SLM produces a low-confidence report** | Red confidence badges are displayed on affected sections. Analyst is advised to treat those sections with heightened skepticism and verify against raw logs. The report is still returned — the analyst decides whether it is useful. |
| **Log volume exceeds SLM context window** | Pre-processing deduplication reduces volume first. If the cleaned event stream still exceeds context limits, the backend truncates to the highest-severity and most temporally significant events and notes the truncation in the report metadata. |

---

## 9. Model Maintenance & Drift

Threat actor TTPs evolve. The SLM, once deployed, is a static artifact unless actively maintained.

- **Quarterly Model Review:** Every quarter, the SOC team reviews whether the AI reports remain directionally accurate for the current threat landscape. If significant drift is detected, model replacement or prompt engineering adjustments are evaluated.
- **Confidence Score Monitoring:** Sustained low confidence scores across reports are an early signal of model drift or data distribution shift. The development team monitors aggregate confidence score trends as a leading indicator.
- **Fine-Tuning (Phase 4):** If the SOC wishes to tune the model's output style to match internal reporting formats, a parameter-efficient fine-tune (LoRA) can be applied. This requires a curated dataset of several hundred to a few thousand high-quality analyst-written reports and dedicated ML engineering time. This is an optional enhancement, not a launch requirement.
- **Model Replacement:** The inference server is model-agnostic. As better open-weights models become available, swapping the underlying model requires no changes to the pipeline architecture, provided logprob support is confirmed for the replacement model.

---

## 10. Compliance & Documentation

Even for a self-hosted, internal tool, AI-assisted security decisions require documentation in regulated environments.

- All AI-generated reports that correspond to a formal incident investigation are logged alongside the raw investigation record, making it auditable which investigations used AI assistance.
- Per-section confidence scores are included in the audit log, providing a record of the model's uncertainty at the time of generation.
- Compliance documentation explicitly classifies this tool as an **analyst decision-support aid**, not an automated decision system, which carries a lower compliance burden under most frameworks (SOC 2, ISO 27001, GDPR).

---

## 11. Implementation Strategy

**Phase 1 — Wazuh API & Data Pipeline:** Develop the backend service to authenticate with Wazuh, parse URL parameters from the Wazuh dashboard, query the API, and execute the pre-processing logic. Validate pipeline performance and output quality independently of the AI layer. Conduct break-even hardware cost analysis.

**Phase 2 — SLM Server Provisioning:** Procure and configure the internal GPU server. Deploy an open-weights SLM using vLLM (or equivalent inference engine confirmed to support logprob output). Establish the internal API endpoint. Validate logprob extraction and confidence score computation. Conduct load testing. Validate that the model produces directionally useful reports on representative SOC datasets.

**Phase 3 — Browser Extension Development:** Build the lightweight extension with the Capture button overlay, the inline Markdown viewer with confidence score badges, and the full-tab expansion view. Implement download options (MD, PDF, DOCX). Roll out to a pilot group of analysts for feedback.

**Phase 4 — SOC Rollout, SOP Integration & Platform Expansion:** Deploy to the full analyst team with explicit SOP documentation. Train analysts on appropriate use and the requirement to verify AI output against raw logs. Extend URL parameter parsing to support Kibana and OpenSearch dashboards. If fine-tuning is pursued based on pilot feedback, scope it as a standalone project.

---

## 12. Competitive Landscape & Differentiation

### What Already Exists

**Wazuh's Native LLM Integration:** Wazuh has begun introducing LLM integration using models like LLaMA 3 running on Ollama combined with LangChain, allowing analysts to query logs using natural language through a chatbot embedded in the OpenSearch UI.

**Enterprise SIEM AI Assistants:** Vendors including Fortinet, Rapid7, Elastic, and Microsoft Sentinel have embedded AI assistants directly into their platforms. These are cloud-connected, subscription-priced, and built for their own proprietary platforms.

**Open Source Tools:** Projects like LogSentinelAI offer LLM-powered log analysis using declarative extraction schemas, with support for Elasticsearch/Kibana integration and local inference via Ollama and vLLM. These are pipeline tools, not workflow-integrated analyst aids.

### Where This Proposal Is Different

| Dimension | Existing Solutions | This Proposal |
| :--- | :--- | :--- |
| **Workflow Integration** | Chatbot panels requiring the analyst to formulate natural language queries | Captures the analyst's existing filter state with one click — zero behavior change |
| **Privacy Architecture** | Cloud-connected AI or opt-in local setup | Self-hosted by design, hard architectural constraint from day one |
| **Interaction Model** | Query-based: analyst must know what to ask | Filter-based: analyst works natively in Wazuh, tool translates filter state automatically |
| **Output Format** | Free-form text or chat response | Structured MD report with defined sections and per-section confidence scores |
| **Confidence Signal** | None, or self-reported by model | Token logprob-derived per-section score — mathematically honest |
| **Deployment Model** | Native platform feature or standalone pipeline | Lightweight browser extension on any existing Wazuh deployment |
| **Cost Model** | Per-seat SaaS or significant cloud spend | Fixed hardware cost, zero per-investigation cost at scale |

### The Core Differentiator

The most significant gap this proposal fills is the **interaction model combined with honest uncertainty quantification**. Every existing solution requires the analyst to formulate a question and provides no reliable signal about how much to trust the output. This proposal inverts the interaction model entirely — the analyst filters, the AI derives — and adds logprob-based confidence scores that give the analyst a mathematically grounded basis for calibrating their reliance on each section of the report.

---

## 13. Conclusion

This architecture represents a practical, privacy-respecting augmentation of how Level 1 and Level 2 SOC analysts orient themselves within large datasets. By being explicit about what the tool is — an AI-powered starting point, not an AI-powered verdict — we avoid the most common failure mode of security AI tooling: eroding analyst judgment rather than sharpening it.

The structured report format, the logprob-based confidence scoring, and the assistive framing work together to produce a tool that analysts can trust precisely because it is honest about its own uncertainty. A red confidence badge is not a failure — it is the tool doing its job correctly.

The pipeline is self-contained, auditable, and operationally resilient. It respects data sovereignty, replaces unpredictable API spend with fixed infrastructure cost, and integrates into the workflow analysts already use. Most importantly, it is designed to make skilled analysts faster — not to substitute for them.