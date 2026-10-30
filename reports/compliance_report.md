# Heritage National Bank — AI Banking Platform
# Compliance Audit Report

**Report Date:** 2026-10-30
**Scope:** Audit logs produced by the m4-capstone AI banking platform
**Audit Log Sources:**
- `audit_log/model_routing.jsonl`
- `audit_log/mcp_tool_calls.jsonl`
- `audit_log/transactions.jsonl`
- `audit_log/bandit_scan.txt` (absent — see Gaps section)

---

## 1. Audit Event Summary

### 1.1 Model Routing (`model_routing.jsonl`) — 7 events

| Request ID | Task       | Model                          | Rationale                                      |
|------------|------------|--------------------------------|------------------------------------------------|
| R1001      | reason     | claude-sonnet-4.5              | Default reasoning: chatbot, KYC, compliance    |
| R1001      | decide     | claude-opus-4.1                | Highest-stakes decision: final approval        |
| R1002      | reason     | claude-sonnet-4.5              | Default reasoning: chatbot, KYC, compliance    |
| B1         | classify   | claude-3.5-haiku               | High-volume, low-stakes classification         |
| B2         | classify   | claude-3.5-haiku               | High-volume, low-stakes classification         |
| B3         | classify   | claude-3.5-haiku               | High-volume, low-stakes classification         |
| B4         | classify   | claude-3.5-haiku               | High-volume, low-stakes classification         |

**Counts by model tier:**
- Haiku (batch/classification): 4 calls
- Sonnet (reasoning/narrative): 2 calls
- Opus (high-stakes decisions): 1 call

**Notable finding:** Request R1002 logged a `reason` routing entry (Sonnet) but no corresponding `decide` entry. For a transaction that was ultimately BLOCKED at AML stage, it is unclear whether an Opus-level decision review was triggered or whether the pipeline halted early and bypassed the decide step. The missing routing record creates an audit gap.

---

### 1.2 MCP Tool Calls (`mcp_tool_calls.jsonl`) — 15 events

**Tool call distribution:**

| Tool                    | Server        | Count | All OK? |
|-------------------------|---------------|-------|---------|
| get_customer            | customer-info | 3     | Yes     |
| check_customer_history  | customer-info | 6     | Yes     |
| verify_ip_location      | customer-info | 6     | Yes     |
| get_balance             | banking-api   | 1     | Yes     |
| transfer_funds          | banking-api   | 2     | Yes     |

All 15 tool calls returned `ok: true`. No tool call failures were recorded.

**Notable findings:**

1. **Country diversity in `verify_ip_location` calls for a single customer:** Customer C5001 appears with IP country codes `IN`, `IN`, `RU`, `RU` across four separate `verify_ip_location` calls. Within the same session/batch, the same customer's IP location oscillates between India and Russia. This is a strong geo-anomaly signal and warrants investigation — it may indicate session hijacking, VPN switching, or a data pipeline replay issue.

2. **Customer C5002 verified against three distinct countries** (RU, SG, NG) across four `verify_ip_location` calls. Rapid geo-rotation across high-risk jurisdictions (Russia, Nigeria) is a recognised AML red flag.

3. **Sequence numbering resets to seq=1 mid-file:** The `mcp_tool_calls.jsonl` file contains two separate runs starting at `seq=1`. This indicates the log is an append-only file across multiple sessions, but the sequence counter resets per session rather than incrementing globally. While this is not inherently wrong, it makes cross-session tracing harder and increases the risk of log correlation errors. Globally unique `request_id` or `session_id` fields are absent from MCP call records.

4. **No timestamps on MCP call records:** Every MCP event lacks a timestamp field. The CLAUDE.md requirement states "All money-movement and compliance decisions must be audited (who/what/when)." The absence of `when` (timestamp) on tool calls — including the two `transfer_funds` calls — represents a direct PCI DSS audit trail gap.

---

### 1.3 Transactions (`transactions.jsonl`) — 2 events

| Request ID | Status   | Decision | Amount | From   | To     | AML Risk | Fraud Engine | Geo Mismatch |
|------------|----------|----------|--------|--------|--------|----------|--------------|--------------|
| R1001      | SETTLED  | APPROVE  | $750   | AC5001 | AC5002 | LOW      | fallback     | false        |
| R1002      | REJECTED | BLOCK    | $5,000 | —      | —      | HIGH     | fallback     | true         |

**Transaction R1001 — SETTLED:**
- Passed all four compliance stages: KYC, AML (LOW risk), Sanctions (no hit, "NONE" strength), Approval.
- Sanctions check on "Asha Rao" returned no match.
- Fraud engine fell back to rule-based logic due to a `NotFoundError` (ML model unavailable). The fallback correctly assessed LOW risk given amount ($750) near average ($575) and no geo mismatch, but the reliance on a fallback engine in production is a reliability and audit risk.

**Transaction R1002 — REJECTED:**
- Blocked at AML stage (HIGH risk). KYC passed; pipeline halted before Sanctions and Approval stages as required.
- Fraud signals show a $5,000 transaction against a $12,000 average (not inherently suspicious by amount), but geo_mismatch=true aligns with the geographic anomalies observed in MCP logs for C5002.
- Fraud engine also fell back to rule-based logic due to `NotFoundError`, identical to R1001. This confirms the ML fraud model is systematically unavailable across all transactions in scope.

---

## 2. Compliance Posture Assessment

Assessment is made against the CLAUDE.md-defined controls for the Heritage National Bank AI Banking Platform.

### 2.1 PCI DSS Constraints

| Control                                                               | Status      | Evidence / Notes                                                                                                    |
|-----------------------------------------------------------------------|-------------|---------------------------------------------------------------------------------------------------------------------|
| Never log full PAN/card numbers; mask to last 4 digits               | PASS        | No PAN data observed in any log file. Account IDs (AC5001, AC5002) are non-card identifiers.                       |
| Sanitize all chatbot/user input before reaching tools or the model   | UNVERIFIABLE| No chatbot input logs present. Cannot confirm sanitization from audit data alone.                                   |
| Enforce per-session rate limiting on money-movement operations       | UNVERIFIABLE| Logs do not include session identifiers or rate-limit enforcement records.                                          |
| All money-movement and compliance decisions must be audited (who/what/when) | PARTIAL FAIL | Transactions and compliance decisions are logged with `what` and outcome, but MCP tool call records lack timestamps (`when`) and user/session attribution (`who`). |
| Secrets from environment variables only; never hard-code keys        | UNVERIFIABLE| Source code not reviewed in this audit. No secrets observed in log files.                                          |

### 2.2 Model Routing Policy

| Requirement                                                          | Status | Notes                                                                                           |
|----------------------------------------------------------------------|--------|-------------------------------------------------------------------------------------------------|
| Haiku for high-volume classification                                 | PASS   | 4 batch classification calls correctly routed to Haiku.                                        |
| Sonnet for default reasoning (chatbot, compliance narrative, KYC)    | PASS   | Reasoning steps for R1001 and R1002 use Sonnet.                                                |
| Opus for highest-stakes decisions (final approval, ambiguous fraud)  | PARTIAL| Opus used for R1001 decide step. No Opus entry for R1002 (BLOCK case). Gap in rationale log.  |
| Log routing rationale for every request                              | PARTIAL| Rationale logged for all 7 recorded events. R1002 missing a `decide`-tier entry.              |

### 2.3 Audit Log Coverage

| Required Log                          | Present | Complete |
|---------------------------------------|---------|----------|
| audit_log/mcp_tool_calls.jsonl        | Yes     | Partial — no timestamps, no session IDs                |
| audit_log/model_routing.jsonl         | Yes     | Partial — R1002 missing decide-step entry              |
| audit_log/transactions.jsonl          | Yes     | Yes — both transactions fully recorded                 |
| audit_log/bandit_scan.txt             | No      | Missing — security scan output not present             |

### 2.4 Compliance Pipeline

| Requirement                                                          | Status | Notes                                                              |
|----------------------------------------------------------------------|--------|--------------------------------------------------------------------|
| Pipeline order: KYC -> AML -> Sanctions -> Approval                  | PASS   | R1001 follows full sequence. R1002 correctly halts at AML (BLOCK).|
| BLOCK stops pipeline and is audited                                  | PASS   | R1002 records `failed_stage: AML` and `decision: BLOCK`.          |

### 2.5 Definition of Done Criteria

| Criterion                                                             | Status       | Notes                                                                |
|-----------------------------------------------------------------------|--------------|----------------------------------------------------------------------|
| pytest passes with zero failures                                      | UNVERIFIABLE | Test results not in audit scope.                                    |
| bandit has no medium+ findings                                        | UNVERIFIABLE | bandit_scan.txt is absent; cannot confirm.                          |
| Every MCP hop is audited                                              | PARTIAL FAIL | MCP hops logged but missing timestamps and session attribution.     |
| Claude-as-Judge scores platform >= 4 on compliance rubric            | UNVERIFIABLE | Judge evaluation results not present in audit logs.                 |

---

## 3. Risks and Gaps

### Critical

1. **Fraud ML model systematically unavailable (fallback on all transactions):** Both transactions used the fallback rule-based engine due to `NotFoundError`. If the ML model is the primary control for sophisticated fraud detection, running 100% on fallback in production means the platform is operating below its intended risk-management capability. This is a production reliability failure.

2. **MCP tool call records lack timestamps:** `mcp_tool_calls.jsonl` entries contain no `timestamp` field. PCI DSS and the project's own definition of done require auditable `who/what/when` for all money-movement operations. Two `transfer_funds` calls cannot be precisely placed in time using the audit log alone.

3. **`bandit_scan.txt` absent:** The CLAUDE.md Definition of Done requires `bandit` to produce no medium+ findings and the output to be stored in `audit_log/bandit_scan.txt`. The file is missing. There is no evidence that a security scan was run, leaving potential code-level vulnerabilities unverified.

### High

4. **Geo-anomaly: customer C5001 IP oscillates between India and Russia:** Four `verify_ip_location` calls for the same customer show alternating `IN` and `RU` origins. This pattern — if from live data — is a strong indicator of account takeover, VPN abuse, or a testing/replay data issue that could mask real fraud signals.

5. **Customer C5002 verified across three high-risk jurisdictions (RU, SG, NG):** Rapid multi-country rotation including Russia and Nigeria across a short session window is an AML red flag. The transaction for C5002 was correctly blocked on AML grounds, but the `verify_ip_location` calls still returned `ok: true` without any recorded alert or escalation event.

6. **R1002 missing Opus-level decide routing entry:** The BLOCK decision for the high-value ($5,000), high-risk (AML HIGH, geo_mismatch) transaction has no evidence of Opus-tier review in the model routing log. Policy requires Opus for "ambiguous fraud" and "final approval." A BLOCK under these conditions should arguably trigger an Opus review. The missing log entry may indicate either a pipeline gap or an incomplete audit trail.

7. **No session or user attribution in any log:** No `user_id`, `session_id`, `operator_id`, or equivalent appears in MCP call or model routing records. PCI DSS requires knowing `who` initiated money-movement operations. Without this, post-incident attribution is impossible.

### Medium

8. **Sequence counter resets across sessions in MCP log:** Makes cross-session event correlation dependent on file line ordering rather than a reliable unique identifier. This increases the risk of log tampering going undetected and complicates forensic analysis.

9. **Only two transactions in scope:** The transaction log contains just two records. It is unclear whether this represents the full production volume or a partial export. A partial log would undermine the completeness of this audit.

10. **No `github` MCP server calls logged:** The MCP registry declares a `github` server (branch/commit/PR operations). No calls to this server appear in the audit log. Either no code changes occurred during the audited period (acceptable) or the PreToolUse hook is not capturing all server activity (a gap).

---

## 4. Recommendations

### Immediate Actions

1. **Restore and run the fraud ML model.** Investigate the `NotFoundError` causing systematic fallback. Until resolved, flag all fallback-routed decisions for human review and document the fallback as an approved compensating control with an expiry date.

2. **Add timestamps to all MCP tool call log entries.** Update the PreToolUse hook to inject an ISO-8601 timestamp into every `mcp_tool_calls.jsonl` record. This is required for PCI DSS compliance and the project's own "who/what/when" audit mandate.

3. **Run and retain bandit security scan.** Execute `bandit -r bfsi/` and write output to `audit_log/bandit_scan.txt`. If medium+ findings exist, remediate before next production deployment.

4. **Investigate the C5001 IP geo-oscillation.** Determine whether the alternating India/Russia IP pattern in `verify_ip_location` calls is caused by test data replay, a real account anomaly, or a pipeline bug. Raise a SAR (Suspicious Activity Report) if the data reflects live activity.

### Short-Term (within 2 weeks)

5. **Add session and user attribution to all log schemas.** Every log record (model routing, MCP calls, transactions) should carry a `session_id` and an `operator_id` or `user_id`. Without these, post-incident forensics and PCI DSS requirement 10 (track user access) cannot be fully satisfied.

6. **Define and enforce Opus routing for BLOCK decisions on high-risk transactions.** Update `model_router.py` to log an Opus-tier `decide` step for any transaction with AML risk HIGH, geo_mismatch=true, or amount above a defined threshold, even if the outcome is a BLOCK. This preserves the audit trail and allows Opus to override an automated block where appropriate.

7. **Implement global sequence IDs or UUIDs in MCP call logs.** Replace the per-session resetting `seq` counter with a UUID or monotonic global counter so events from multiple sessions can be unambiguously ordered and cross-referenced.

### Ongoing

8. **Expand transaction log coverage.** Confirm whether the two-transaction log represents all production activity or a subset. If partial, fix the log export process and retain complete records per your data retention policy.

9. **Implement alert events for high-risk geo patterns.** `verify_ip_location` calls returning high-risk country codes (OFAC-sanctioned or FATF high-risk jurisdictions) should produce a dedicated alert log entry, not merely an `ok: true` response, so that risk events are surfaced proactively.

10. **Schedule periodic Claude-as-Judge compliance rubric evaluations** and retain the score and rationale in `audit_log/` for trend analysis and regulatory evidence.

---

## 5. Summary

| Area                        | Overall Status |
|-----------------------------|----------------|
| PCI DSS Controls            | Partial Fail   |
| Model Routing Policy        | Partial Pass   |
| Compliance Pipeline Logic   | Pass           |
| Audit Log Completeness      | Partial Fail   |
| Security Scan (bandit)      | Not Verified   |
| Fraud Engine Integrity      | Fail (fallback)|

The platform's compliance pipeline logic (KYC → AML → Sanctions → Approval) is functioning correctly and correctly blocked a high-risk transaction. However, the audit trail has material gaps — missing timestamps on MCP calls, missing user/session attribution, an absent bandit scan, and a non-operational ML fraud engine — that must be remediated before the platform can be considered production-ready under PCI DSS or the project's own Definition of Done.

---

*Generated by the compliance documentation agent on 2026-10-30. All findings are based solely on the audit log files present at time of report generation.*
