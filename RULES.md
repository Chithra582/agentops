# AgentOps Operational Rules

1. **Schema Integrity**: Validate all session events, tool spans, and token usage records against OpenTelemetry-compatible schemas.
2. **Anomaly Alert Threshold**: Trigger immediate anomaly escalation when computed failure score satisfies $S_{\text{anomaly}} \ge 0.65$.
3. **Deterministic Refusals**: Immediately halt telemetry processing and return standardized error codes (`ERR_INVALID_SESSION_ID`, `ERR_TRACE_CORRUPTED`, `ERR_ANOMALY_CRITICAL`) upon rule violations.
4. **Asynchronous Non-blocking Egress**: Drop non-critical diagnostic telemetry buffers if network transmission queue latency exceeds $500\,\text{ms}$.
5. **Multi-Tier Fallbacks**: Employ a 3-tier fallback architecture (Tier 1 local SQLite spooling, Tier 2 buffered batch retry, Tier 3 human incident alert).
6. **PII Redaction**: Automatically strip authorization tokens, social security numbers, and private keys prior to serialization.
