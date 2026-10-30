# Heritage National Bank — AI Banking Platform: Deployment Guide

## Overview

This document covers how to set up and run the Heritage National Bank AI Banking Platform. The platform is a Python application that connects to Anthropic Claude models via OpenRouter, exposes MCP servers for banking and customer data tools, and writes audit logs to a local `audit_log/` directory.

---

## Prerequisites

### Runtime Requirements

| Requirement  | Version / Notes                                                          |
|--------------|--------------------------------------------------------------------------|
| Python       | 3.14+ (pyc files in `.venv` confirm CPython 3.14)                       |
| Node.js/npx  | Required for the GitHub MCP server (`@modelcontextprotocol/server-github`) |
| pip          | For installing Python dependencies into the virtualenv                   |
| git          | Optional; needed only if using the GitHub MCP integration                |

### External Service Accounts

| Service    | Required | Purpose                                                    |
|------------|----------|------------------------------------------------------------|
| OpenRouter | Yes      | Proxies all Claude API calls; provides the API key         |
| GitHub     | Optional | Required only for the GitHub MCP server integration        |

### Network Access

The application makes outbound HTTPS calls to:

- `https://openrouter.ai/api` — all Claude model inference
- `https://api.github.com` — only if the GitHub MCP server is used

---

## Environment Variables

All secrets and configuration values must be supplied via environment variables. The application loads them from a `.env` file at startup using `python-dotenv`. **Never hard-code secrets in source files.**

Create or edit `.env` in the project root (`m4-capstone/`):

```
OPENROUTER_API_KEY="sk-or-v1-..."
ANTHROPIC_BASE_URL="https://openrouter.ai/api"
ANTHROPIC_AUTH_TOKEN="sk-or-v1-..."
ANTHROPIC_API_KEY=""
MODEL_HAIKU="anthropic/claude-3.5-haiku"
MODEL_SONNET="anthropic/claude-sonnet-4.5"
MODEL_OPUS="anthropic/claude-opus-4.1"
GITHUB_TOKEN="ghp_..."
```

### Variable Reference

| Variable              | Required | Description                                                       |
|-----------------------|----------|-------------------------------------------------------------------|
| `OPENROUTER_API_KEY`  | Yes      | OpenRouter API key; passed as `auth_token` to the Anthropic SDK  |
| `ANTHROPIC_BASE_URL`  | Yes      | API base URL; set to `https://openrouter.ai/api` for OpenRouter  |
| `ANTHROPIC_AUTH_TOKEN`| Yes      | Same as `OPENROUTER_API_KEY`; used by the SDK auth mechanism      |
| `ANTHROPIC_API_KEY`   | No       | Direct Anthropic API key; leave empty when using OpenRouter       |
| `MODEL_HAIKU`         | No       | Override the Haiku-tier model slug (default: `anthropic/claude-3.5-haiku`) |
| `MODEL_SONNET`        | No       | Override the Sonnet-tier model slug (default: `anthropic/claude-sonnet-4.5`) |
| `MODEL_OPUS`          | No       | Override the Opus-tier model slug (default: `anthropic/claude-opus-4.1`) |
| `GITHUB_TOKEN`        | No       | GitHub Personal Access Token for the GitHub MCP server            |

---

## Installation

### 1. Enter the project directory

```bash
cd /path/to/claude-bfsi/m4-capstone
```

### 2. Create and activate a Python virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate        # Linux / macOS
# .venv\Scripts\activate         # Windows
```

The project ships with a pre-populated `.venv/` directory. If using it directly, activate as shown above.

### 3. Install Python dependencies

```bash
pip install anthropic python-dotenv mcp bandit pytest
```

Key packages:

- `anthropic` — Anthropic Python SDK used for all model calls
- `python-dotenv` — loads `.env` into the process environment at startup
- `mcp` — provides `mcp.server.fastmcp.FastMCP` for the MCP servers
- `bandit` — static security scanning, run automatically by the Claude Code `PostToolUse` hook
- `pytest` — test runner

### 4. Configure environment variables

Copy the `.env` template above into `m4-capstone/.env` and fill in your OpenRouter API key. Do not commit `.env` to version control.

### 5. Verify the skills directory

The `skills/` directory must exist at `m4-capstone/skills/` and contain the six skill subdirectories. Each skill must have a valid `SKILL.md` with YAML front matter. The skills loader uses a relative `Path("skills")`, so the application must always be run from `m4-capstone/`.

```
skills/
  compliance-narrative/SKILL.md
  customer-support/SKILL.md
  fraud-triage/SKILL.md
  kyc-extraction/SKILL.md
  risk-scoring/SKILL.md
  sanctions-summary/SKILL.md
```

---

## Running the Platform

### End-to-End Workflow (main entry point)

Run from inside `m4-capstone/` with the virtualenv active:

```bash
cd m4-capstone
python -m bfsi.main
```

This executes:

1. Two production transactions (R1001 settles; R1002 is blocked by AML due to geo mismatch).
2. A batch fraud scoring run over four synthetic transactions at the Haiku tier.

Expected output when Claude API is reachable:

```
=== End-to-end banking workflow ===
  R1001: SETTLED
  R1002: REJECTED -> AML
=== Batch fraud pipeline ===
  B1: risk=LOW (claude)
  B2: risk=HIGH (claude)
  B3: risk=LOW (claude)
  B4: risk=HIGH (claude)
```

If the Claude API call fails (e.g., invalid key), the fraud engine falls back to deterministic rules and `engine` will show `fallback` rather than `claude`.

### Running MCP Servers Standalone

Each MCP server can be started as a standalone process for integration testing or IDE use:

```bash
python bfsi/mcp/customer_info_server.py
python bfsi/mcp/banking_api_server.py
```

### Running Tests

```bash
cd m4-capstone
pytest
```

Per the project definition of done, all tests must pass with zero failures and `bandit` must report no medium or higher severity findings.

### Running Security Scan Manually

```bash
bandit -r bfsi -f txt -o audit_log/bandit_scan.txt
```

---

## Claude Code IDE Integration

The project contains a `.claude/` directory that configures the Claude Code development environment. Start the Claude Code CLI from `m4-capstone/` so that `.claude/mcp.json` and `.claude/settings.json` are discovered automatically.

### MCP Server Registration (`.claude/mcp.json`)

```json
{
  "mcpServers": {
    "customer-info": {
      "command": "python3",
      "args": ["bfsi/mcp/customer_info_server.py"]
    },
    "banking-api": {
      "command": "python3",
      "args": ["bfsi/mcp/banking_api_server.py"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
```

The `customer-info` and `banking-api` servers are launched as subprocesses by Claude Code when needed. The `github` server requires `npx` in `PATH` and the `GITHUB_TOKEN` environment variable.

### Automated Hooks (`.claude/settings.json`)

Two hooks run automatically during Claude Code sessions:

| Hook         | Trigger             | Command                                                          | Purpose                                           |
|--------------|---------------------|------------------------------------------------------------------|---------------------------------------------------|
| `PreToolUse` | Any `mcp__.*` call  | Python one-liner appending JSON to `audit_log/mcp_tool_calls.jsonl` | Timestamps every MCP tool invocation            |
| `PostToolUse`| `Edit` or `Write`   | `bandit -q -r bfsi -f txt -o audit_log/bandit_scan.txt`          | Static security scan after every code change      |

### Custom Slash Commands (`.claude/commands/`)

| Command                        | Usage                               | Effect                                                                 |
|--------------------------------|-------------------------------------|------------------------------------------------------------------------|
| `/add-bfsi-skill`              | `/add-bfsi-skill <name> <model>`    | Scaffolds `skills/<name>/SKILL.md` with correct front matter           |
| `/generate-compliance-report`  | `/generate-compliance-report`       | Reads all audit logs; writes `reports/compliance_report.md`            |
| `/security-review`             | `/security-review <path-to-file>`   | PCI DSS review + `bandit` analysis of the target file                  |

---

## Directory Structure After Deployment

```
m4-capstone/
+-- .env                          # Secrets (do not commit to git)
+-- .claude/
|   +-- mcp.json                  # MCP server registrations for Claude Code
|   +-- settings.json             # Claude Code hooks (PreToolUse, PostToolUse)
|   +-- commands/                 # Custom slash commands
+-- bfsi/
|   +-- main.py                   # Entry point
|   +-- chatbot.py
|   +-- fraud_engine.py
|   +-- batch_fraud.py
|   +-- transactions.py
|   +-- model_router.py
|   +-- skills_loader.py
|   +-- kyc_vision.py
|   +-- compliance/
|   |   +-- pipeline.py
|   |   +-- sanctions_check.py
|   +-- mcp/
|       +-- registry.py
|       +-- banking_api_server.py
|       +-- customer_info_server.py
+-- skills/
|   +-- compliance-narrative/SKILL.md
|   +-- customer-support/SKILL.md
|   +-- fraud-triage/SKILL.md
|   +-- kyc-extraction/SKILL.md
|   +-- risk-scoring/SKILL.md
|   +-- sanctions-summary/SKILL.md
+-- audit_log/                    # Created automatically at runtime
|   +-- transactions.jsonl
|   +-- mcp_tool_calls.jsonl
|   +-- model_routing.jsonl
|   +-- batch_results.json
|   +-- bandit_scan.txt
+-- docs/                         # Documentation
+-- reports/                      # Generated compliance reports
+-- tests/                        # pytest test suite
+-- .venv/                        # Python virtual environment
```

---

## Audit Log Management

Audit logs are written to `audit_log/` relative to the working directory. The directory is created automatically on first run by the application. All `.jsonl` files are append-only; they are never truncated.

| Log File                  | Growth rate                                        | Retention recommendation                              |
|---------------------------|----------------------------------------------------|-------------------------------------------------------|
| `transactions.jsonl`      | One record per transaction                         | Retain per regulatory requirement (7+ years)          |
| `mcp_tool_calls.jsonl`    | Multiple records per transaction                   | Retain as supporting evidence for transactions        |
| `model_routing.jsonl`     | Two records per transaction + one per batch item   | Operational; retain for audit periods                 |
| `batch_results.json`      | Overwritten on each batch run                      | Archive before each batch run if history is needed    |
| `bandit_scan.txt`         | Overwritten after each file edit                   | Retain snapshots as part of SDLC compliance evidence  |

---

## Infrastructure Notes

### Current Architecture (Single-Process)

The platform runs as a single Python process. The MCP servers (`customer_info_server.py`, `banking_api_server.py`) are called in-process via `bfsi/mcp/registry.py` during normal Python execution. The data store for both servers is in-memory dicts initialized at import time; all state resets on process restart.

### Scaling Considerations

**Model API costs**: The three-tier routing strategy minimizes cost. Do not promote batch classification calls to Sonnet or Opus; the Haiku tier handles high-volume scoring at a fraction of the cost.

**Prompt caching**: Ensure `skills/` exists in the working directory so `build_cached_system()` can assemble and cache the skill library. Cache hits reduce both latency and cost for repeated chatbot and fraud-scoring calls within the Anthropic cache TTL window.

**MCP servers in production**: The MCP servers should be replaced with clients connecting to real banking core systems. The `registry.py` tool mapping makes this a drop-in replacement — implement the same function signatures in the server modules without changing any calling code.

**Data persistence**: The in-memory `_ACCOUNTS` and `_CUSTOMERS` dicts in the MCP servers must be replaced with a persistent database before production use.

**Sanctions list**: The demo watchlist in `sanctions_check.py` must be replaced with a live feed from OFAC, UN, or EU consolidated sanctions lists before any production use.

---

## PCI DSS Deployment Requirements

The following constraints from `CLAUDE.md` apply to all deployment environments:

1. **No full PANs in logs**: The MCP registry masks card-number fields to last 4 digits. Verify that no upstream logging (load balancer, reverse proxy, APM) captures unmasked card data.
2. **Input sanitization**: The chatbot's `sanitize()` function is the only user-input guard. Do not expose `fraud_engine.score_transaction()` or `compliance/pipeline.run_compliance()` directly to untrusted input without adding equivalent sanitization at the boundary.
3. **Rate limiting**: Per-session rate limiting on money-movement operations is required by `CLAUDE.md` but not yet implemented in the current codebase. This must be added before exposing the transfer endpoint to external callers.
4. **Secret management**: Rotate `OPENROUTER_API_KEY` and `GITHUB_TOKEN` on a schedule. Use a secrets manager (e.g., AWS Secrets Manager, HashiCorp Vault) in production rather than a plaintext `.env` file.
5. **Audit log integrity**: In production, ship the `audit_log/` JSONL files to an immutable log store (e.g., AWS CloudTrail, Splunk) immediately after write to prevent tampering.

---

## Troubleshooting

### Fraud engine falls back to rules / `engine: "fallback"` in logs

The `OPENROUTER_API_KEY` is missing or invalid, or OpenRouter returned a `NotFoundError`. Check that `.env` is present in the working directory (`m4-capstone/`) and that the key is correct. The fallback rule still produces correct risk assessments.

### `KeyError: 'OPENROUTER_API_KEY'`

The `.env` file was not loaded. Ensure you are running from `m4-capstone/` so that `load_dotenv()` in `bfsi/main.py` finds `.env`. Alternatively, export the variable directly before running:

```bash
export OPENROUTER_API_KEY=sk-or-v1-...
```

### `KeyError: unknown MCP tool <server>.<tool>`

The tool name or server name does not match an entry in `bfsi/mcp/registry.py`'s `_TOOLS` dict. Check that the server name is `customer-info` or `banking-api` and that the tool name matches exactly.

### Skills not loading / `KeyError` from `get_skill()`

The `skills/` directory must exist relative to the current working directory. `skills_loader.py` uses `Path("skills")` (relative). Always run the application from `m4-capstone/`.

### `bandit_scan.txt` is empty or missing

The `bandit` CLI is not installed, or `audit_log/` did not exist when the hook ran. Install bandit and run the scan manually:

```bash
pip install bandit
mkdir -p audit_log
bandit -r bfsi -f txt -o audit_log/bandit_scan.txt
```

### MCP servers not found by Claude Code

Ensure Claude Code is launched from the `m4-capstone/` directory so that `.claude/mcp.json` is discovered. The server commands use relative paths (`bfsi/mcp/customer_info_server.py`) that resolve from the project root.
