# Security Audit Report

## 1. Repository, Target Revision, and Freshness

| Field | Value |
|-------|-------|
| Repository | https://github.com/forked-oss/adk-js |
| Audited revision | `4169319f46e7bdac1e4aac0247f6ec7099c5097f` |
| Default branch | `main` |
| Target freshness | **fresh** |
| Audit run ID | `bc-b7fc1a99-2297-4544-a5e6-0ad4fb2962f6` |

## 2. Scope and Assumptions

**In scope:** `@google/adk` core library and `@google/adk-devtools` CLI/server (`dev/`), including `AdkApiServer`, A2A, skill tools, code executors, MCP client.

## 3. Repository Inventory

| Category | Details |
|----------|---------|
| Language | TypeScript (Node 18+) |
| Packages | `@google/adk` (core), `@google/adk-devtools` (dev) |
| HTTP server | Express (`dev/src/server/adk_api_server.ts`) |
| CLI | `adk web`, `adk api_server`, deploy commands |
| CI | `validation.yaml` (secretlint), `release-please.yml` |

## 4. Functional and Security Architecture

Express server exposes session management, agent execution (`/run`, `/run_sse`), artifacts, debug traces, and optional A2A routes. No authentication middleware. Skill script execution uses `UnsafeLocalCodeExecutor` when configured. OAuth discovery includes SSRF guards; MCP HTTP does not.

## 5. Review Policy

- No `SECURITY.md`
- `InMemoryPolicyEngine` allows all tool calls by default (prototyping)
- `UnsafeLocalCodeExecutor` explicitly warns of local code execution

## 6. Threat Model

**Attacker:** Unauthenticated HTTP client; LLM invoking skill scripts with malicious paths; MCP HTTP to internal URLs.

## 7. Historical Security Evidence

CHANGELOG documents CORS fixes on `/run_sse`, host binding restrictions, `express.urlencoded` removal.

## 8. Campaigns and Coverage

| Campaign | Status |
|----------|--------|
| HTTP auth | complete |
| Skill script injection | complete |
| A2A auth | complete |
| MCP SSRF | complete |
| SAST | not run |

## 9. Tools and Static Analysis

Manual review; secretlint in CI but not executed this run.

## 10. Confirmed Findings

| ID | Severity | Title |
|----|----------|-------|
| [FIND-JS-001](findings/FIND-JS-001.md) | Critical | Unauthenticated HTTP API executes agents and exposes sessions/artifacts |
| [FIND-JS-002](findings/FIND-JS-002.md) | High | Command injection in skill script wrapper via script_path |
| [FIND-JS-003](findings/FIND-JS-003.md) | High | A2A endpoints configured with noAuthentication |
| [FIND-JS-004](findings/FIND-JS-004.md) | High | MCP HTTP transport lacks SSRF protections |
| [FIND-JS-005](findings/FIND-JS-005.md) | Medium | Unauthenticated debug/trace endpoints leak telemetry |

**Counts:** Critical=1, High=3, Medium=1, Low=0, Informational=0

## 11. Unresolved Leads

| ID | Status |
|----|--------|
| LEAD-JS-001 | GCS artifact filename path segments — unresolved |
| LEAD-JS-002 | materializeFiles writes to process.cwd() — unresolved |

Unresolved: **2**

## 12. Rejected Leads

- Default `InMemoryPolicyEngine` allow-all — documented prototyping behavior
- `yaml.load` in skill loader — lower risk with Zod validation; unresolved not confirmed

## 13. Exploit-Chain Analysis

FIND-JS-001 + FIND-JS-002: Unauthenticated `/run` → agent invokes `run_skill_script` with injected `script_path` → shell wrapper executes attacker commands when `UnsafeLocalCodeExecutor` configured.

## 14. Overall Assessment

ADK-JS dev server is an unauthenticated agent control plane. Skill script wrapper concatenates unescaped paths into shell/PowerShell commands. Production deployments binding `0.0.0.0` (Docker default) are high risk without perimeter auth.

## 15. Coverage Limitations

- Minified browser bundles not fully reviewed
- No npm audit in this run
- No dynamic PoC

## 16. Runtime Metadata

Worktree: `/opt/cursor/artifacts/security-audit/bc-b7fc1a99-2297-4544-a5e6-0ad4fb2962f6/worktrees/adk-js`
