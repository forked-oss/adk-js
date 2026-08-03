# Security Audit Report

## 1. Repository, Target Revision, and Freshness
- **Repository:** https://github.com/forked-oss/adk-js
- **Audited revision:** `4169319f46e7bdac1e4aac0247f6ec7099c5097f` (4169319f)
- **Target freshness:** fresh (fetched `origin/main` at audit start)
- **Audit run:** `audit-20260803-020231-8cbd`
- **Audit date:** 2026-08-03

## 2. Scope and Assumptions
Static source-code security review of the isolated worktree at `origin/main`. Assumes ADK HTTP servers may be deployed network-exposed (default bind `0.0.0.0` in container deploy paths). Does not assume Cloud Run IAM or API gateway unless explicitly configured by operator.

## 3. Repository Inventory
# ADK-JS Security Inventory

**Repository:** forked-oss/adk-js  
**Revision:** `4169319f46e7bdac1e4aac0247f6ec7099c5097f` (4169319)  
**Audit date:** 2026-08-03

## Overview

Google Agent Development Kit (ADK) JavaScript/TypeScript monorepo. Code-first toolkit for building, evaluating, and deploying AI agents on Node.js.

## Packages

| Package | Path | Role |
|---------|------|------|
| `@google/adk` | `core/` | Core SDK: agents, runner, tools, sessions, memory, artifacts, A2A, auth, code executors |
| `@google/adk-devtools` | `dev/` | CLI (`adk`), `AdkApiServer`, debug web UI, deploy helpers, integration test runner |

Root workspace (`package.json`) orchestrates builds, lint, vitest, typedoc.

## Languages and Scale

- **Primary:** TypeScript (~195 source files in `core/src` + `dev/src`)
- **Debug UI:** Prebuilt Angular bundle in `dev/src/browser/` (minified JS)
- **Tests:** vitest unit, integration, e2e, cross-language suites in `tests/`

## HTTP Attack Surface

### AdkApiServer (`dev/src/server/adk_api_server.ts`)

Express application — **28 route handlers**, no authentication middleware.

| Route pattern | Methods | Purpose |
|---------------|---------|---------|
| `/list-apps` | GET | List loaded agents |
| `/debug/trace/:eventId` | GET | Internal trace dictionary |
| `/debug/trace/session/:sessionId` | GET | Session span telemetry |
| `/apps/:appName/users/:userId/sessions/*` | GET/POST/DELETE | Session CRUD |
| `/apps/:appName/users/:userId/sessions/:sessionId/artifacts/*` | GET/DELETE | Artifact access |
| `/apps/:appName/eval_sets/*` | various | Eval stubs (501) |
| `/run` | POST | Synchronous agent execution |
| `/run_sse` | POST | SSE streaming execution |
| `/dev-ui/*` | static | Debug Angular UI (when `serveDebugUI`) |

Optional A2A routes mounted at `/a2a/{appName}/*` when `--a2a` enabled (`core/src/a2a/agent_to_a2a.ts`).

### Default binding

- CLI default host: `localhost` (`dev/src/cli/cli.ts:105`)
- Cloud Run deploy CMD: `--host=0.0.0.0` (`dev/src/cli/deploy/deploy_utils.ts:63`)

## CLI Entry Points

| Command | Surface |
|---------|---------|
| `adk web` | Starts API server + debug UI on configured host/port |
| `adk api_server` | API server without UI |
| `adk run` | Local interactive/file-based agent run |
| `adk deploy cloud_run` / `agent_engine` | Containerized deployment |

## Code Execution Surfaces

| Component | Path | Notes |
|-----------|------|-------|
| `UnsafeLocalCodeExecutor` | `core/src/code_executors/unsafe_local_code_executor.ts` | Local spawn of node/python/shell — documented unsafe |
| `BuiltInCodeExecutor` | `core/src/code_executors/built_in_code_executor.ts` | Gemini built-in execution |
| `AgentEngineSandboxCodeExecutor` | `core/src/code_executors/agent_engine_sandbox_code_executor.ts` | Remote sandbox |
| `RunSkillScriptTool` | `core/src/tools/skill/run_skill_script_tool.ts` | Skill script wrapper |
| `RunSkillInlineScriptTool` | `core/src/tools/skill/run_skill_inline_script_tool.ts` | Inline script execution |
| MCP stdio transport | `core/src/tools/mcp/mcp_session_manager.ts` | Spawns MCP server subprocess |

## File I/O Surfaces

| Component | Path | Controls |
|-----------|------|----------|
| `FileArtifactService` | `core/src/artifacts/file_artifact_service.ts` | Path traversal checks, safe segment regex |
| `materializeFiles` | `core/src/utils/file_utils.ts` | Base-dir prefix validation |
| `loadSkillFromDir` | `core/src/skills/loader.ts` | Reads skill directory tree |
| `AgentLoader` | `dev/src/utils/agent_loader.ts` | esbuild compile + dynamic import |

## Auth and Policy

| Component | Path | Behavior |
|-----------|------|----------|
| OAuth2 discovery | `core/src/auth/oauth2/oauth2_discovery.ts` | HTTPS-only discovery, private IP blocklist |
| `SecurityPlugin` | `core/src/plugins/security_plugin.ts` | Default `InMemoryPolicyEngine` allows all tools |
| A2A handlers | `core/src/a2a/agent_to_a2a.ts` | `UserBuilder.noAuthentication` |

## Session State Model

State keys use prefixes: `app:`, `user:`, `temp:` (`core/src/sessions/state.ts`). Session services route `app:`/`user:` deltas to global stores (`core/src/sessions/in_memory_session_service.ts:275-288`).

## External Dependencies (notable)

Express, cors, `@a2a-js/sdk`, `@google/genai`, `@modelcontextprotocol/sdk`, esbuild, mikro-orm (DB sessions), google-auth-library, js-yaml (skill frontmatter).

## Prior Security Fixes (history mining)

| Commit | Fix |
|--------|-----|
| `8c0eaa1` | FileArtifactService path traversal (CWE-22) |
| `9008353` | Prototype pollution in streaming JSON path |
| `57b0af7` | OAuth SSRF via IPv4-mapped IPv6 |
| `04968b7` | Filter `temp:` keys on session creation |
| `1fe631f` | CORS/urlencoded parser hardening |
| `d2db989` | Remove wildcard ACAO from run_sse |
| `7be8c81` | Restrict server listen host |


## 4. Functional and Security Architecture
ADK provides agent orchestration (Runner), session management, tool execution, HTTP REST/WebSocket APIs for development and deployment, and optional trigger webhooks. Security boundary is at the HTTP layer — the framework provides outbound OAuth for tools but no inbound API authentication by default.

## 5. Review Policy
- HTTP servers are primarily intended for local development and testing.
- Production deployments documented to use Cloud Run IAM (`--no-allow-unauthenticated`) or perimeter auth.
- No explicit policy accepting cross-session state injection risk.
- Prior audit reports exist in repository history.

## 6. Threat Model
# ADK-JS Threat Model

**Revision:** `4169319f`  
**Primary asset owner:** Operators deploying ADK agents; end users interacting via web UI or API clients.

## System Context

```
                    ┌─────────────────────────────────────┐
                    │         AdkApiServer (Express)        │
                    │  /run, /run_sse, sessions, artifacts  │
                    │  /debug/trace/*, optional A2A routes  │
                    └──────────────┬──────────────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         ▼                         ▼                         ▼
   ┌───────────┐           ┌─────────────┐          ┌──────────────┐
   │  Runner   │           │ SessionSvc  │          │ ArtifactSvc  │
   │ + Agent   │           │ (app/user/  │          │ (in-mem/file │
   │ + Tools   │           │  session)   │          │  /GCS)       │
   └─────┬─────┘           └─────────────┘          └──────────────┘
         │
         ├── Code executors (local/sandbox/built-in)
         ├── MCP tools (stdio/HTTP subprocess)
         ├── OpenAPI tools (outbound HTTP via agent config)
         ├── Skill scripts (wrapper execution)
         └── Gemini API (LLM, URL context, search)
```

## Assets

| Asset | Sensitivity |
|-------|-------------|
| Agent runtime & tool execution | High — arbitrary tool/code paths |
| Session state (`app:`, `user:`, session keys) | High — drives agent behavior and instructions |
| Conversation events & artifacts | Medium–High — may contain PII/secrets |
| Host filesystem (executors, artifacts, agent loader) | High |
| Internal telemetry (`traceDict`, spans) | Medium |
| GCP credentials (ADC, OAuth tokens) | Critical |
| MCP server processes | High |

## Actors

| Actor | Capability |
|-------|------------|
| **Anonymous network client** | Any HTTP client when server bound to `0.0.0.0` or reachable on LAN |
| **Cross-origin browser** | When CORS `allowOrigins` permits attacker origin |
| **Malicious LLM output** | Indirect — steers tool calls, code execution, state deltas |
| **Agent developer** | Configures tools, skills, executors — trusted in dev model |
| **Co-tenant user** | Another `userId` on same server instance |

## Entry Points

1. **REST API** — `AdkApiServer` routes (no auth middleware)
2. **A2A JSON-RPC/REST** — `/a2a/{app}/jsonrpc`, `/a2a/{app}/rest` with `noAuthentication`
3. **CLI** — local operator; spawns server and loads agents from disk
4. **Agent tools** — LLM-invoked: MCP, skills, OpenAPI, code executors
5. **Debug UI** — Angular SPA calling same API; static assets from `/dev-ui`

## Trust Boundaries

| Boundary | Trust assumption | Violation impact |
|----------|------------------|------------------|
| Network → API server | **Should require auth in production** | Full agent/API access |
| HTTP client → `stateDelta`/`state` | **Should not control `app:`/`user:` keys** | Cross-session state pollution |
| LLM → tool arguments | Partially untrusted | Code exec, SSRF via configured tools |
| Skill `script_path` → wrapper code | Trusted skill registry keys | Injection if path not validated in wrapper |
| OAuth discovery URLs | Untrusted if misconfigured | SSRF (mitigated for discovery path) |

## Threat Scenarios

### T1 — Unauthenticated remote agent execution (CONFIRMED)

Attacker on network calls `POST /run` or `/run_sse` with valid session IDs. No credentials required. Cloud Run deploy binds `0.0.0.0`.

### T2 — Cross-session state injection (CONFIRMED)

Attacker supplies `stateDelta: {"app:config": "malicious"}` on `/run`. `InMemorySessionService.appendEvent` writes to global `appState`, affecting all users/sessions for that app.

### T3 — Session state spoofing via create (CONFIRMED subset)

Attacker creates session with `state: {"app:spoofed": "value"}`. `mergeStates` applies session keys last, spoofing instruction template variables for that session without global store write.

### T4 — A2A without authentication (CONFIRMED)

`UserBuilder.noAuthentication` on A2A REST and JSON-RPC handlers when `--a2a` enabled.

### T5 — Debug telemetry disclosure (CONFIRMED under T1)

`/debug/trace/*` returns internal span data without auth.

### T6 — Permissive CORS when configured (CONFIRMED partial)

`allowOrigins` enables `cors({origin: allowOrigins})`. Operator may set overly broad origin. Default without flag: no CORS middleware (browser reads blocked, but server still processes requests).

### T7 — Local code execution (design risk, unresolved)

`UnsafeLocalCodeExecutor` + skill tools execute host code when agent configured with them.

### T8 — Path traversal (REJECTED)

`FileArtifactService`, `materializeFiles`, `LoadSkillResourceTool` have explicit controls.

### T9 — OAuth discovery SSRF (REJECTED at discovery layer)

`validateDiscoveryUrl` enforces HTTPS and blocks private ranges with IPv4-mapped normalization.

### T10 — Prototype pollution via streaming (REJECTED)

`streaming_utils.ts` blocks `__proto__`, `constructor`, `prototype` in JSON paths.

## Security Controls Present

- Host default `localhost`; explicit host required for wide bind
- Artifact path traversal guards (`assertSafeSegment`, relative path checks)
- `trimTempState` / `trimTempDeltaState` for `temp:` keys
- OAuth discovery URL validation
- Streaming JSON path sanitization
- `UnsafeLocalCodeExecutor` runtime warning banner

## Out of Scope / Assumptions

- Perimeter auth (IAP, API gateway) may be applied by operators — not part of SDK
- Gemini API key protection is operator responsibility
- Dependency CVE scanning not performed in this audit
- Minified debug UI bundle not fully decompiled


## 7. Historical Security Evidence
Git history reviewed for security fix patterns (CORS hardening, path traversal fixes, template injection, OAuth SSRF). Recurring unfixed patterns: missing HTTP auth, state scope injection via `app:`/`user:` keys, permissive CORS/WebSocket origins.

## 8. Campaigns and Coverage
# ADK-JS Audit Coverage

**Revision:** `4169319f`  
**Audit run:** audit-20260803-020231-8cbd

## Campaign Matrix

| Campaign | Status | Notes |
|----------|--------|-------|
| **Inventory** | Complete | Packages, routes, executors, file I/O mapped |
| **Threat model** | Complete | Assets, actors, entry points, scenarios |
| **HTTP/API authentication** | Complete | No middleware found; all routes unauthenticated |
| **Unauthenticated endpoints** | Complete | 28 Express handlers + A2A routes |
| **State injection** | Complete | `stateDelta` and session `state` paths traced |
| **SSRF** | Complete | OAuth discovery validated; OpenAPI tool is parser-only at this revision; MCP HTTP URL is agent-config |
| **Injection (code/shell)** | Partial | Skill wrapper code path reviewed; LLM-driven exec surfaces documented as unresolved |
| **Path traversal** | Complete | FileArtifactService, materializeFiles, LoadSkillResourceTool reviewed |
| **CORS** | Complete | Optional cors middleware; prior wildcard fix verified absent |
| **XSS (web UI)** | Partial | Minified Angular bundle; no `innerHTML` grep hits in source; deep DOM review not performed |
| **A2A auth** | Complete | `noAuthentication` confirmed |
| **Debug endpoint exposure** | Complete | `/debug/trace/*` unauthenticated |
| **History mining** | Complete | 30+ security-related commits reviewed |
| **Prototype pollution** | Complete | Fix at `streaming_utils.ts:33` verified |
| **Dependency/CVE scan** | Not run | No lockfile advisory tooling executed |
| **Dynamic PoC** | Not run | Source-only confirmation |
| **CodeQL/Semgrep** | Not run | Manual review only |

## Files Reviewed (representative)

### Server / CLI
- `dev/src/server/adk_api_server.ts` (full)
- `dev/src/cli/cli.ts`, `cli_run.ts`, `deploy/deploy_utils.ts`
- `dev/src/utils/agent_loader.ts`, `file_utils.ts`
- `dev/test/server/adk_api_server_test.ts` (selective)

### Core runtime
- `core/src/runner/runner.ts`
- `core/src/sessions/in_memory_session_service.ts`
- `core/src/sessions/base_session_service.ts`
- `core/src/sessions/state.ts`
- `core/src/sessions/database_session_service.ts` (state delta paths)

### Security-sensitive tools
- `core/src/code_executors/unsafe_local_code_executor.ts`
- `core/src/tools/skill/run_skill_script_tool.ts`
- `core/src/tools/skill/run_skill_inline_script_tool.ts`
- `core/src/tools/skill/load_skill_resource_tool.ts`
- `core/src/tools/mcp/mcp_session_manager.ts`
- `core/src/tools/openapi_tool/openapi_spec_parser/operation_parser.ts`

### Artifacts / files
- `core/src/artifacts/file_artifact_service.ts`
- `core/src/utils/file_utils.ts`
- `core/src/skills/loader.ts`

### Auth / A2A
- `core/src/a2a/agent_to_a2a.ts`
- `core/src/auth/oauth2/oauth2_discovery.ts`
- `core/src/auth/oauth2/oauth2_utils.ts`
- `core/src/plugins/security_plugin.ts`

### Streaming / utils
- `core/src/utils/streaming_utils.ts`

## Confirmed Finding Count

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 2 |
| Medium | 1 |
| Low | 0 |

## Limitations

1. No runtime exploitation or network PoC
2. No npm audit / OSV dependency scan
3. Debug UI is prebuilt minified JS — XSS review incomplete
4. OpenAPI tool HTTP execution layer not present in reviewed parser-only files; full outbound request path not exhaustively traced
5. Database session backend state injection confirmed by parallel code path review, not integration test execution

## Prior Audit Cross-Check

Prior headless audit at same revision (`bc-3f1852db`, commit `e44dfc9`) reported identical three findings. This audit independently verified all three against current source at `4169319f`.


## 9. Tools and Static Analysis
- **Method:** Manual source review with grep/ripgrep enumeration
- **CodeQL/Semgrep:** not run (not available on audit host)
- **Dependency scanners:** not run
- **Dynamic PoC:** not executed

## 10. Confirmed Findings
**Counts:** Critical=0, High=2, Medium=1, Low=0, Informational=0

- **[HIGH]** [FIND-JS-001](findings/FIND-JS-001.md): Unauthenticated AdkApiServer exposes agent execution API
- **[HIGH]** [FIND-JS-002](findings/FIND-JS-002.md): Cross-session state injection via HTTP stateDelta and session state
- **[MEDIUM]** [FIND-JS-003](findings/FIND-JS-003.md): A2A endpoints use noAuthentication and optional permissive CORS

## 11. Unresolved and Low-Priority Leads
8 unresolved leads. See `evidence/adk-js/unresolved-leads.json`.

## 12. Rejected and Duplicate Leads
9 rejected leads investigated with decisive mitigations. See `evidence/adk-js/rejected-leads.json`.

## 13. Exploit-Chain Analysis
Primary chain: unauthenticated API access → arbitrary session/state manipulation → agent execution with attacker-controlled state affecting other users/sessions via `app:`/`user:` scope keys. Java-specific chain adds LocalSkillSource path traversal and ReplayPlugin arbitrary file read when plugin enabled.

## 14. Overall Assessment
**3 confirmed findings** require remediation before network exposure without perimeter authentication.

## 15. Coverage Limitations
- Static analysis only; no dynamic PoC execution
- No dependency/CVE scanning
- No SAST (CodeQL/Semgrep)
- Contrib/samples partially reviewed
- Agent Engine GCP IAM boundary not tested live

## 16. Runtime, Resource, and Cleanup Metadata
- **Run ID:** audit-20260803-020231-8cbd
- **Worktree:** `/tmp/audit-20260803-020231-8cbd/worktrees/adk-js`
- **Evidence:** `/tmp/audit-20260803-020231-8cbd/evidence/adk-js/`
- **Delivery:** report-only PR branch `security-audit/audit-20260803-020231-8cbd`
