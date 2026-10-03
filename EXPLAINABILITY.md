# EXPLAINABILITY.md

This document explains the internal mechanisms, data lineage, operational boundaries, and governance framework of **AgentOps** (`agentops`) in accordance with the **OpenGAP v0.1.0** specification for the **HiDevs GitAgent Passport** clearance pipeline.

> **Agent Name:** AgentOps (`agentops`)  
> **Specification:** OpenGAP v0.1.0  
> **Category / Domain:** Agent Observability, Tracing & Monitoring  
> **Compliance Standard:** OpenGAP Checkpoint 2 (Explainability & Decision Governance), OWASP LLM Top 10, MITRE ATLAS  

---

## How the Agent Decides

AgentOps evaluates runtime telemetry, calculates anomaly likelihoods, and triggers automated alerting through a deterministic 5-stage decision pipeline.

### 1. Decision Architecture

The runtime intake, state classification, evaluation, and execution tracking operate across a deterministic, five-stage pipeline:

```
+-----------------------------------------------------------------------------------+
|                           5-STAGE DECISION PIPELINE                               |
+-----------------------------------------------------------------------------------+
|  [Stage 1: Telemetry Event Ingestion & PII Redaction]                             |
|  - Ingest spans, strip credentials/PII, validate OpenTelemetry schema             |
|                                     |                                             |
|                                     v                                             |
|  [Stage 2: Session Trace Assembly & Graph Construction]                          |
|  - Link parent-child spans, calculate turn durations and step sequences           |
|                                     |                                             |
|                                     v                                             |
|  [Stage 3: Metric Aggregation & Anomaly Scoring]                                  |
|  - Compute S_anomaly based on token spikes, loop repetition, and tool error rates |
|                                     |                                             |
|                                     v                                             |
|  [Stage 4: Threshold Evaluation & Circuit Breaking]                               |
|  - Evaluate tau >= 0.65; trigger alerts or halt recursive agent executions        |
|                                     |                                             |
|                                     v                                             |
|  [Stage 5: Persisted Storage, Dashboard Emission & Incident Escalation]           |
|  - Commit replay timeline, stream metrics to dashboard, notify ops team           |
+-----------------------------------------------------------------------------------+
```

### 2. Decision Logic & Routing Formulations

For an active agent session $s_i$ observed over time window $T$ comprising steps $\{e_1, e_2, \dots, e_M\}$, the anomaly score $S_{\text{anomaly}}(s_i, T)$ is formulated as:

$$S_{\text{anomaly}}(s_i, T) = w_{\text{loop}} R(s_i) + w_{\text{err}} E(s_i) + w_{\text{cost}} C(s_i) + w_{\text{lat}} L(s_i)$$

Where:
- $R(s_i) = \frac{\text{RepeatedActions}(s_i)}{M}$ measures loop repetition (repeated identical prompt/tool calls).
- $E(s_i) = \frac{\sum_{k=1}^M \mathbb{I}(\text{status}(e_k) = \text{FAILED})}{M}$ is the empirical tool and inference error rate.
- $C(s_i) = \min\left(1, \frac{\text{TokensUsed}(s_i)}{\text{TokenQuota}(s_i)}\right)$ represents normalized token consumption relative to the configured ceiling.
- $L(s_i) = \min\left(1, \frac{\text{Duration}(s_i)}{\text{DurationBudget}(s_i)}\right)$ captures session run duration elongation.
- Standard default weights: $w_{\text{loop}} = 0.35$, $w_{\text{err}} = 0.30$, $w_{\text{cost}} = 0.20$, $w_{\text{lat}} = 0.15$ with $\sum w = 1.0$.

Circuit breaking and alert triggers execute when:

$$S_{\text{anomaly}}(s_i, T) \ge \tau \quad (\tau = 0.65)$$

### 3. Thresholding & Refusal Decision Criteria

AgentOps enforces strict operational boundaries and deterministic refusal thresholds:
- **Refusal on ERR_INVALID_SESSION_ID**: Session ID is null, malformed, or unregistered halts execution with code `ERR_INVALID_SESSION_ID`.
- **Refusal on ERR_TRACE_CORRUPTED**: Missing parent span or timestamp backward drift halts execution with code `ERR_TRACE_CORRUPTED`.
- **Refusal on ERR_ANOMALY_CRITICAL**: $S_{\text{anomaly}} \ge 0.65$ or recursive loop detected halts execution with code `ERR_ANOMALY_CRITICAL`.
- **Refusal on ERR_RATE_LIMIT_EXCEEDED**: Ingestion throughput exceeds 10,000 events/sec halts execution with code `ERR_RATE_LIMIT_EXCEEDED`.
- **Refusal on ERR_STORAGE_UNREACHABLE**: Telemetry database cluster offline halts execution with code `ERR_STORAGE_UNREACHABLE`.

### 4. Fallback Decision Mechanism

Continuous operational stability is maintained through layered fault recovery:
- **Tier 1 (Local Disk Spooling):** If cloud ingestion endpoints experience temporary latency or outages, buffer telemetry events locally in compressed SQLite files.
- **Tier 2 (Batch Exponential Backoff):** Reattempt ingestion flush with exponential backoff and jitter once cloud health probes indicate endpoint recovery.
- **Model Fallback Cascade**: High-level reasoning and synthesis default to `gemini-2.0-flash` with automatic failover to `gpt-4o` and `claude-3-5-sonnet`.

### 5. Human-in-the-Loop Governance

Human operators retain sovereign authority over the multi-agent execution lifecycle:
- **Tier 3 (Human Incident Escalation):** If critical agent loops or cost runaways exceed hard safety caps, notify the designated oncall engineer via PagerDuty/Slack webhooks with full session dump.
- **Session Telemetry Auditing**: Operators inspect execution logs, routing traces, and token usage to maintain oversight.

---

## The Data It Uses

AgentOps operates under strict principles of data minimization, environment isolation, and privacy protection.

### 1. Ingested Input Data

The framework processes only operational data necessary to perform its functions:
- **Execution Spans**: Start/end timestamps, step names, tool arguments, and return status codes.
- **LLM Call Metadata**: Prompt tokens, completion tokens, model names, temperature, and latency.
- **Session Attributes**: Environment tags (`prod`, `staging`), user IDs, and framework identifiers (LangChain, CrewAI, AutoGen).

### 2. Configuration & Reference Data

- **Model Pricing Tables**: Comprehensive price-per-thousand-tokens tables for all major LLM foundation providers.
- **Known Failure Signatures**: Regex patterns for known provider errors (HTTP 429 rate limits, context window overflow, content filter triggers).
- **Historical Session Baselines**: Statistical distributions of expected latency and cost for similar agent workloads.

### 3. Base Model & Inference Lineage

- **Heuristic & Statistical Engines**: Decision logic relies on deterministic rule sets, moving window averages, and cosine similarity embeddings.
- **No External LLM Dependencies**: Anomaly detection logic executes locally without requiring third-party LLM evaluation calls, eliminating recursive telemetry loops.

### 4. Data Privacy, Storage, and Retention

- **OWASP LLM & MITRE ATLAS Hardened**: Defended against indirect prompt injection, credential leakage, and unauthorized external API dispatch.
- **Local Environment Isolation**: Agent execution workspaces, intermediate scratchpads, and vector stores reside strictly within designated local project directories.
- **Automated Secret Scrubbing**: API keys, database credentials, and personal credentials are automatically redacted prior to embedding or logging.
- **Zero Commercial Monetization**: Prompts, intermediate reasoning trajectories, and task deliverables are never commercialized or shared with third parties.

---

## Limitations

Understanding the operational boundaries and technical constraints of AgentOps is essential for effective deployment.

### 1. Ingestion overhead can impact high-throughput agents
- **Limitation**: Ingestion overhead can impact high-throughput agents if telemetry calls are executed synchronously.
- **Mitigation**: The AgentOps SDK utilizes non-blocking background daemon threads with bounded memory ring buffers.

### 2. False-positive loop detection can trigger if
- **Limitation**: False-positive loop detection can trigger if an agent legitimately performs iterative refinement on complex tasks.
- **Mitigation**: The loop detector requires identical argument hashing across successive turns before flagging recursive oscillation.

### 3. Network partitions between agent runtimes and
- **Limitation**: Network partitions between agent runtimes and the telemetry collector can cause delayed trace visual updates.
- **Mitigation**: Local persistent SQLite write-ahead logging guarantees zero telemetry data loss during connectivity outages.

### 4. Token cost tracking can diverge slightly
- **Limitation**: Token cost tracking can diverge slightly if foundation providers update pricing tiers without manifest refresh.
- **Mitigation**: AgentOps pulls updated pricing tables automatically on startup and allows user-defined custom pricing overrides.

### 5. Extremely large prompt inputs (e
- **Limitation**: Extremely large prompt inputs (e.g., 100k+ token documents) can exhaust local telemetry serialization buffers.
- **Mitigation**: Automatic payload truncation trims middle tokens while preserving prompt prefixes, suffixes, and total token count tallies.

---

## Summary & Compliance Checklist

| Checkpoint 2 Requirement | Corresponding Section | Status |
| :--- | :--- | :---: |
| **How the agent decides** | [How the Agent Decides](#how-the-agent-decides) | **Covered** |
| - Decision architecture & 5-stage pipeline | Section 1 | Verified |
| - Decision logic & routing formulations | Section 2 | Verified |
| - Thresholding & refusal decision criteria | Section 3 | Verified |
| - Fallback decision mechanism | Section 4 | Verified |
| - Human-in-the-loop governance & oversight | Section 5 | Verified |
| **The data it uses** | [The Data It Uses](#the-data-it-uses) | **Covered** |
| - Ingested input data & query streams | Section 1 | Verified |
| - Configuration & reference schemas | Section 2 | Verified |
| - Base model lineage & deterministic engines | Section 3 | Verified |
| - Data privacy, retention lifecycle & MITRE/OWASP | Section 4 | Verified |
| **Its limitations** | [Limitations](#limitations) | **Covered** |
| - Ingestion overhead can impact high-throughput agents | Section 1 | Verified |
| - False-positive loop detection can trigger if | Section 2 | Verified |
| - Network partitions between agent runtimes and | Section 3 | Verified |
| - Token cost tracking can diverge slightly | Section 4 | Verified |
| - Extremely large prompt inputs (e | Section 5 | Verified |
