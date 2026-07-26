# Security Audit Report

## 1. Repository, Target Revision, and Freshness

| Field | Value |
|-------|-------|
| Repository | https://github.com/forked-oss/adk-js |
| Audited revision | `4169319f46e7bdac1e4aac0247f6ec7099c5097f` |
| Target freshness | **fresh** |
| Audit run | `f6cb6e33-7d20-11f1-ba66-0e7d0216e441` |
| Audit date | 2026-07-26 |

## 2. Scope and Assumptions

**In scope:** `@google/adk` core library, `@google/adk-devtools` CLI and `AdkApiServer`, A2A integration, OAuth2, MCP, code executors, skill tools.

**Assumptions:** Production Docker deploy binds `0.0.0.0`; findings apply when API server is network-exposed.

## 3. Repository Inventory

| Category | Details |
|----------|---------|
| Language | TypeScript (~437 TS files), bundled browser assets |
| Package manager | npm workspaces (`core`, `dev`) |
| Build | TypeScript, esbuild, vitest |
| CLI | `adk` → `dev/src/cli_entrypoint.ts` |
| HTTP server | Express — `dev/src/server/adk_api_server.ts` |
| Security tooling | ESLint, secretlint (CI + pre-commit) |

## 4. Functional and Security Architecture

`AdkApiServer` registers all API routes without authentication middleware. Debug trace routes are always mounted regardless of `serveDebugUI`. A2A explicitly uses `UserBuilder.noAuthentication`. Default `SecurityPlugin` policy engine allows all tool calls.

## 5. Review Policy

| Item | Status |
|------|--------|
| `SECURITY.md` | Not present |
| `UnsafeLocalCodeExecutor` | Documented as intentionally unsafe |
| OAuth discovery SSRF | Fixed in #354 |
| Prototype pollution (SSE) | Fixed in #410 |

## 6. Threat Model

**Attacker:** Unauthenticated network client; model-influenced tool arguments when agent is running.

**Assets:** Session state, artifacts, LLM prompts/tool I/O, host execution (via code executor chain).

## 7. Historical Security Evidence

| Commit | Fix |
|--------|-----|
| `9008353` (#410) | Prototype pollution in SSE `jsonPath` |
| `57b0af7` (#354) | OAuth discovery SSRF blocklist |
| `1fe631f` (#378) | Removed `express.urlencoded` from API server |

## 8. Campaigns and Coverage

| Campaign | Status |
|----------|--------|
| HTTP auth enumeration | complete |
| Debug telemetry | complete |
| Path traversal (skills/artifacts) | complete — mitigated |
| Code execution primitives | complete |
| OAuth/MCP SSRF | complete — partial gaps |
| Prototype pollution | complete — fixed |
| CodeQL/Semgrep | not-run |

## 9. Tools and Static Analysis

Manual review and git history only.

## 10. Confirmed Findings

| ID | Severity | Title |
|----|----------|-------|
| FIND-JS-001 | High | Unauthenticated agent execution API |
| FIND-JS-002 | High | Unauthenticated debug telemetry endpoints |
| FIND-JS-003 | High | UnsafeLocalCodeExecutor reachable via unauthenticated API chain |
| FIND-JS-004 | Medium | MCP HTTP transport SSRF (config-driven) |
| FIND-JS-005 | Medium | OAuth2 token endpoint fetch without URL validation |
| FIND-JS-006 | Low | Permissive default SecurityPlugin policy |
| FIND-JS-007 | Low | 50MB JSON body limit (DoS) |

## 11. Unresolved Leads

None material.

## 12. Rejected Leads

| Lead | Verdict |
|------|---------|
| Skill path traversal | Mitigated |
| Artifact path traversal | Mitigated |
| OAuth discovery SSRF | Fixed |
| `eval`/`vm` in production | Not found |

## 13. Exploit Chain Analysis

**Confirmed:** FIND-JS-001 + agent with `UnsafeLocalCodeExecutor` + `run_skill_inline_script` → unauthenticated RCE.

## 14. Overall Assessment

**7 confirmed findings (3 High, 2 Medium, 2 Low).** Active security awareness in OAuth SSRF and prototype pollution fixes. Primary architectural risk is unauthenticated HTTP API deployed to `0.0.0.0` in production containers, chainable with intentional code execution primitives.

## 15. Coverage Limitations

- Semgrep/CodeQL not run
- Browser UI assets reviewed at surface level
- npm dependency CVE reachability not mechanically verified

## 16. Runtime, Resource, and Cleanup Metadata

Worktree preserved. User checkout unmodified.
