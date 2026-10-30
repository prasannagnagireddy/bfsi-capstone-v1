# Heritage National Bank — AI Banking Platform: Architecture

## Overview

The Heritage National Bank AI Banking Platform is a unified, production-ready system that integrates AI-driven skills, MCP (Model Context Protocol) servers, compliance automation, fraud detection, transaction processing, and a customer service chatbot. It is designed to meet PCI DSS requirements and provide a fully auditable trail for every banking operation.

The codebase lives under `bfsi/` (Banking, Financial Services and Insurance) within the project root at `m4-capstone/`.

---

## Layered Architecture

The platform is organized into five logical layers that interact from top to bottom:

```
+--------------------------------------------------------------+
|  1. Entry Points / Orchestration Layer                        |
|     bfsi/main.py  |  bfsi/chatbot.py                         |
+--------------------------------------------------------------+
|  2. Business Logic Layer                                      |
|     bfsi/transactions.py  |  bfsi/batch_fraud.py             |
|     bfsi/fraud_engine.py  |  bfsi/kyc_vision.py              |
|     bfsi/compliance/pipeline.py                              |
|     bfsi/compliance/sanctions_check.py                       |
+--------------------------------------------------------------+
|  3. AI / Skills Layer                                         |
|     bfsi/skills_loader.py  +  skills/<name>/SKILL.md         |
|     bfsi/model_router.py                                     |
+--------------------------------------------------------------+
|  4. MCP (Tool-Use) Layer                                      |
|     bfsi/mcp/registry.py                                     |
|     bfsi/mcp/customer_info_server.py                         |
|     bfsi/mcp/banking_api_server.py                           |
+--------------------------------------------------------------+
|  5. Audit / Observability Layer                               |
|     audit_log/transactions.jsonl                             |
|     audit_log/mcp_tool_calls.jsonl                           |
|     audit_log/model_routing.jsonl                            |
|     audit_log/batch_results.json                             |
|     audit_log/bandit_scan.txt                                |
+--------------------------------------------------------------+
```

---

## Layer 1: Entry Points / Orchestration

### `bfsi/main.py`

The primary CLI entry point. It calls `dotenv.load_dotenv()` to load environment variables from `.env`, then runs two demonstration flows:

1. **Production transaction workflow** — calls `bfsi.transactions.process()` for two transactions (one expected to settle, one expected to be blocked by AML due to a geo mismatch).
2. **Batch fraud scoring pipeline** — calls `bfsi.batch_fraud.run_batch()` to score four synthetic transactions at the Haiku tier.

### `bfsi/chatbot.py`

A compliance-guardrailed customer service interface. Responsibilities:

- **Input sanitization** (`sanitize()`): strips non-printable characters, removes prompt-injection patterns (`ignore previous`, `system prompt`, `full card number`), and truncates to 2,000 characters. This is a PCI DSS requirement.
- **Fraud signal enrichment**: calls `customer-info` MCP tools (`check_customer_history`, `verify_ip_location`) to attach live fraud signals to every conversation turn.
- **Streamed AI response**: uses the `customer-support` skill via `build_cached_system()`, routed to the Sonnet model. Supports both streaming and non-streaming modes.

---

## Layer 2: Business Logic

### Transaction Processing (`bfsi/transactions.py`)

The `process()` function is the authoritative money-movement entry point. Its flow:

```
process(request_id, customer_id, name, from_account, to_account, amount, ip_country)
  |
  +-- route("reason", request_id)          # log: Sonnet tier selected
  +-- run_compliance(...)                  # KYC -> AML -> Sanctions -> Approval
  |
  +-- if BLOCK:  result = {REJECTED, reason=failed_stage}
  |
  +-- if APPROVE:
       +-- route("decide", request_id)    # log: Opus tier selected (money movement)
       +-- registry.call("banking-api", "transfer_funds", ...)
       +-- result = {SETTLED|FAILED, transfer, compliance}

  +-- append result to audit_log/transactions.jsonl
```

Every transaction outcome — both rejected and settled — is written to the append-only JSONL audit log.

### Compliance Pipeline (`bfsi/compliance/pipeline.py`)

Implements a linear gate-based compliance pipeline. A transaction advances only if the preceding stage passes; the first failure returns immediately with `decision: BLOCK` and the name of the failed stage.

| Stage     | Mechanism                                                    | Blocking condition                          |
|-----------|--------------------------------------------------------------|---------------------------------------------|
| KYC       | `customer-info` MCP -> `get_customer`; check `kyc_verified` | `kyc_verified` is not `True`                |
| AML       | `fraud_engine.score_transaction()`; check risk level         | `risk == "HIGH"`                            |
| Sanctions | `compliance.sanctions_check(name)`; check match strength     | `hit == True` AND `strength != "PARTIAL"`   |
| Approval  | Always passes if above stages clear                          | n/a                                         |

### Fraud Engine (`bfsi/fraud_engine.py`)

Provides the `score_transaction()` function used by both the compliance pipeline (AML stage) and the batch fraud runner. It:

1. Fetches customer transaction history via `customer-info` -> `check_customer_history` to compute baseline average amount.
2. Fetches geolocation mismatch via `customer-info` -> `verify_ip_location`.
3. Constructs a signals dict: `{amount, avg_amount, geo_mismatch}`.
4. Passes signals as JSON to the `fraud-triage` skill (Haiku model, prompt-cached) and parses the JSON response `{risk, reason}`.
5. Falls back to a deterministic rule if the Claude call fails: `HIGH` if amount > max(avg * 3, 5000) or geo mismatch, otherwise `LOW`.

Returns: `{"risk": "LOW|MEDIUM|HIGH", "reason": "...", "engine": "claude|fallback", "signals": {...}}`

### Batch Fraud Scoring (`bfsi/batch_fraud.py`)

Emulates a batch API pattern. Iterates over a list of synthetic transactions, calls `route("classify", id)` for each (routing to Haiku tier and logging the rationale), then calls `fraud_engine.score_transaction()`. Results are written to `audit_log/batch_results.json`.

### Sanctions Check (`bfsi/compliance/sanctions_check.py`)

Screens a customer name against a demo watchlist (`{"ivan petrov", "acme shell holdings", "redline logistics"}`). Returns:

- `strength: "STRONG"` — exact normalized name match -> blocks the transaction
- `strength: "PARTIAL"` — substring match -> allowed through (per current policy)
- `strength: "NONE"` — no match -> clear

In production this module's interface would be identical, screening against OFAC, UN, and EU consolidated lists.

### KYC Vision (`bfsi/kyc_vision.py`)

Extracts structured identity fields from a document description using the `kyc-extraction` skill (Sonnet model). Returns `{full_name, doc_type, doc_number, expiry}` as parsed JSON. The skill instruction masks `doc_number` to the last 4 characters. Falls back to `UNKNOWN` fields if the Claude call fails.

---

## Layer 3: AI / Skills Layer

### Skills Architecture (`skills/<name>/SKILL.md`)

Each skill is a directory under `skills/` containing a single `SKILL.md` file. The file uses YAML front matter followed by the system prompt body:

```
---
name: <kebab-case>
description: <one-line description used for routing and recall>
model: haiku | sonnet | opus
cache: true
---
<system prompt / reasoning instructions>
```

The six defined skills and their assigned model tiers:

| Skill name             | Model  | Purpose                                                      |
|------------------------|--------|--------------------------------------------------------------|
| `fraud-triage`         | Haiku  | Score a transaction LOW/MEDIUM/HIGH from signals JSON        |
| `risk-scoring`         | Haiku  | Assign customer risk tier from signals                       |
| `customer-support`     | Sonnet | Answer retail banking questions with compliance guardrails   |
| `kyc-extraction`       | Sonnet | Extract identity fields from document text; mask doc number  |
| `compliance-narrative` | Sonnet | Draft audit-ready compliance narrative from stage results    |
| `sanctions-summary`    | Opus   | Summarize a sanctions hit; recommend CLEAR or ESCALATE       |

### Skills Loader (`bfsi/skills_loader.py`)

Reads all `SKILL.md` files from the `skills/` directory, parses front matter, and assembles an Anthropic-format system prompt list with `cache_control: {type: ephemeral}`. This enables prompt caching: the entire skill library is a large, stable context billed once and reused across calls within the cache TTL.

`build_cached_system(active: list[str] | None)` accepts an optional filter to include only specific skills (e.g., `["fraud-triage"]` for the fraud engine).

### Model Router (`bfsi/model_router.py`)

Implements cost-optimized model routing across three tiers:

| Task class | Env var        | Default model                   | Use case                                               |
|------------|----------------|---------------------------------|--------------------------------------------------------|
| `classify` | `MODEL_HAIKU`  | `anthropic/claude-3.5-haiku`    | High-volume, low-stakes: batch fraud scoring, tagging  |
| `reason`   | `MODEL_SONNET` | `anthropic/claude-sonnet-4.5`   | Default: chatbot, compliance narrative, KYC extraction |
| `decide`   | `MODEL_OPUS`   | `anthropic/claude-opus-4.1`     | Highest-stakes: final approval, judge, ambiguous fraud |

Every routing decision is written to `audit_log/model_routing.jsonl` with the request ID, task class, selected model, and rationale string. Unknown task classes fall back to the `reason` tier.

---

## Layer 4: MCP (Tool-Use) Layer

The Model Context Protocol layer exposes banking tools as callable functions. The platform uses an in-process registry pattern rather than out-of-process MCP servers during normal Python execution; the `.claude/mcp.json` configures the servers for the Claude Code IDE integration.

### MCP Registry (`bfsi/mcp/registry.py`)

Central dispatcher. Maintains a static `_TOOLS` dict mapping `(server, tool)` tuples to Python functions. Every call is intercepted by `_audit()` which writes a record to `audit_log/mcp_tool_calls.jsonl`. PCI DSS masking is applied before logging: any argument key containing `"card"` with a value longer than 4 characters is masked to `****{last4}`.

### `customer-info` MCP Server (`bfsi/mcp/customer_info_server.py`)

Built with `FastMCP`. Exposes:

| Tool                     | Arguments                           | Returns                                                         |
|--------------------------|-------------------------------------|-----------------------------------------------------------------|
| `get_customer`           | `customer_id: str`                  | Customer master record: name, kyc_verified, tier, home_country, risk |
| `check_customer_history` | `customer_id: str`                  | `{count, avg_amount}` — baseline for fraud scoring             |
| `verify_ip_location`     | `customer_id: str, ip_country: str` | `{home_country, ip_country, geo_mismatch: bool}`               |

Demo data includes two customers: `C5001` (Asha Rao, India, retail) and `C5002` (Liam Chen, Singapore, premier).

### `banking-api` MCP Server (`bfsi/mcp/banking_api_server.py`)

Built with `FastMCP`. Exposes:

| Tool             | Arguments                                           | Returns                                                      |
|------------------|-----------------------------------------------------|--------------------------------------------------------------|
| `get_balance`    | `account_id: str`                                   | `{account_id, balance}`                                      |
| `transfer_funds` | `from_account, to_account: str, amount: float`      | `{ok, from, to, amount, from_balance}` or error              |
| `apply_loan`     | `customer_id: str, amount: float, term_months: int` | `{decision: APPROVED|REFERRED}` (approved if amount <= 25,000) |

`transfer_funds` enforces non-negative balances: returns `insufficient_funds` error if the source account cannot cover the amount.

### External `github` MCP Server

Configured in `.claude/mcp.json` to run `@modelcontextprotocol/server-github` via `npx`. Provides `create_branch`, `create_commit`, and `create_pull_request` tools authenticated with `GITHUB_TOKEN`.

---

## Layer 5: Audit / Observability

All audit files are append-only JSONL (or JSON for batch results), written to `audit_log/`:

| File                        | Written by                                                   | Content                                                       |
|-----------------------------|--------------------------------------------------------------|---------------------------------------------------------------|
| `transactions.jsonl`        | `bfsi/transactions.py`                                       | Full transaction record: compliance stages, transfer result   |
| `mcp_tool_calls.jsonl`      | `bfsi/mcp/registry.py` + Claude Code `PreToolUse` hook       | Every MCP tool invocation with args (masked) and outcome      |
| `model_routing.jsonl`       | `bfsi/model_router.py`                                       | Request ID, task class, model selected, rationale             |
| `batch_results.json`        | `bfsi/batch_fraud.py`                                        | Batch fraud scoring results: `{id, risk, engine}`             |
| `bandit_scan.txt`           | Claude Code `PostToolUse` hook (bandit)                      | Static security analysis output after file edits              |

The `.claude/settings.json` configures two Claude Code hooks:

- **PreToolUse** on `mcp__.*`: appends a timestamp record to `mcp_tool_calls.jsonl` for every MCP tool call made through the Claude Code IDE.
- **PostToolUse** on `Edit|Write`: runs `bandit -q -r bfsi` to perform automated static security analysis after any code edit.

---

## Data Flows

### Single Transaction Flow

```
Client calls process(request_id, customer_id, name, from_acct, to_acct, amount, ip_country)
  |
  +-- model_router.route("reason")         -> log to model_routing.jsonl
  |
  +-- compliance/pipeline.run_compliance()
  |     +-- MCP: customer-info.get_customer        -> KYC verified?
  |     |        audit: mcp_tool_calls.jsonl
  |     +-- fraud_engine.score_transaction()
  |     |     +-- MCP: customer-info.check_customer_history
  |     |     +-- MCP: customer-info.verify_ip_location
  |     |     +-- Claude (Haiku, fraud-triage skill, cached) -> {risk, reason}
  |     +-- compliance/sanctions_check.sanctions_check(name) -> {hit, strength}
  |
  +-- if BLOCK -> result = {REJECTED, failed_stage}
  |
  +-- if APPROVE:
        +-- model_router.route("decide")    -> log to model_routing.jsonl
        +-- MCP: banking-api.transfer_funds -> {ok, balances}
                 audit: mcp_tool_calls.jsonl
  |
  +-- append result to audit_log/transactions.jsonl
```

### Chatbot Interaction Flow

```
chatbot.reply(customer_id, message, ip_country)
  |
  +-- sanitize(message)          # strip control chars, remove injection patterns
  +-- MCP: customer-info.check_customer_history    -> avg_amount
  +-- MCP: customer-info.verify_ip_location        -> geo_mismatch
  +-- Claude (Sonnet, customer-support skill, cached, streaming)
       system: [cached skill library block]
       user:   "Customer {id} asks: {clean_msg}\nFraud signals: {signals}"
```

### Batch Fraud Flow

```
batch_fraud.run_batch()
  |
  for each synthetic transaction:
  +-- model_router.route("classify")    -> log Haiku tier to model_routing.jsonl
  +-- fraud_engine.score_transaction()  -> {risk, engine}
  |
  +-- write all results to audit_log/batch_results.json
```

---

## Tech Stack

| Component              | Technology                                          |
|------------------------|-----------------------------------------------------|
| Language               | Python 3.14                                         |
| AI API client          | `anthropic` SDK                                     |
| AI model provider      | OpenRouter (proxied Anthropic Claude models)        |
| MCP server framework   | `mcp.server.fastmcp.FastMCP`                        |
| Environment loading    | `python-dotenv`                                     |
| Security scanning      | `bandit`                                            |
| Testing                | `pytest`                                            |
| GitHub integration     | `@modelcontextprotocol/server-github` via `npx`     |
| Virtual environment    | `.venv/` (Python venv)                              |

Claude models in use (via OpenRouter):

- `anthropic/claude-3.5-haiku` — classification and batch scoring
- `anthropic/claude-sonnet-4.5` — reasoning, chatbot, KYC, compliance narrative
- `anthropic/claude-opus-4.1` — highest-stakes decisions, sanctions summary, judge

---

## Design Decisions

### PCI DSS Compliance by Default

Security constraints are structural, not optional:

- Full PANs/card numbers are never logged; the MCP registry masks them to last 4 digits at the point of audit write.
- Chatbot input is sanitized before reaching any model or tool.
- All secrets (API keys, tokens) are loaded exclusively from environment variables via `.env` / `dotenv`.
- Every money-movement call is audited end-to-end across three log files.

### Prompt Caching for Performance

The `skills_loader.build_cached_system()` function wraps the assembled skill library in an Anthropic `cache_control: {type: ephemeral}` block. Because skill definitions are large and stable, they are billed once per cache period and reused across requests, reducing latency and cost at high call volumes.

### Gate-Based Compliance Pipeline

The KYC -> AML -> Sanctions -> Approval sequence is implemented as a strict linear gate. Any stage failure immediately returns with `decision: BLOCK` and the `failed_stage` identifier. This design ensures:

- The pipeline never performs unnecessary downstream work after a block.
- The audit record precisely identifies which regulatory requirement caused rejection.
- Each stage can be replaced independently without affecting others.

### Tiered Model Routing

The three-tier model routing strategy (Haiku / Sonnet / Opus) balances cost against decision stakes. Batch classification at Haiku keeps per-transaction AI cost low; final money-movement approvals at Opus ensure the highest-capability model makes irreversible decisions. All routing decisions are logged with rationale for audit traceability.

### In-Process MCP Registry

For Python-to-Python calls (the normal execution path), the platform uses `bfsi/mcp/registry.py` as an in-process dispatcher rather than spawning MCP subprocesses. This eliminates serialization overhead while preserving the full MCP interface contract. The `.claude/mcp.json` configuration enables the same servers as true out-of-process MCP endpoints when running under the Claude Code IDE.

### Fallback Resilience

Both the fraud engine and the KYC extractor have deterministic rule-based fallbacks that activate if the Claude API call fails. This ensures the compliance pipeline continues to function and produce conservative (safe) outputs during API outages, without silently passing risky transactions.

### Claude Code Custom Commands

Three slash commands are registered under `.claude/commands/`:

- `/add-bfsi-skill` — scaffolds a new `skills/<name>/SKILL.md` with valid front matter
- `/generate-compliance-report` — reads all audit logs and produces `reports/compliance_report.md`
- `/security-review` — runs PCI DSS review and `bandit` analysis on a target file
