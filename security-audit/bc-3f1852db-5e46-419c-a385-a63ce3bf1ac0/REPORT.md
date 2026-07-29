# Security Audit Report

## 1. Repository, Target Revision, and Freshness
- Repository: https://github.com/forked-oss/adk-js
- Audited revision: 4169319f46e7bdac1e4aac0247f6ec7099c5097f (4169319)
- Target freshness: fresh (HEAD matches `origin/main`)
- Run ID: bc-3f1852db-5e46-419c-a385-a63ce3bf1ac0

## 2. Scope and Assumptions
Headless audit of ADK JavaScript/TypeScript SDK. `AdkApiServer` is primary HTTP surface. Default bind is localhost but Cloud Run deploy uses `0.0.0.0`.

## 3. Repository Inventory
- **Languages:** TypeScript (core, dev), Angular debug UI
- **Packages:** `@google/adk` (core), dev CLI/server
- **HTTP:** Express `AdkApiServer` — REST, optional A2A
- **CLI:** `adk web`, `adk api_server`, `adk run`, deploy commands
- **Executors:** `UnsafeLocalCodeExecutor`, `BuiltInCodeExecutor`
- **Build:** npm/pnpm, esbuild for agent loading

## 4. Functional and Security Architecture
Express server with no auth middleware. `Runner.runAsync` accepts `stateDelta`. Session services route `app:`/`user:` keys to global stores. A2A uses `UserBuilder.noAuthentication`. Default `InMemoryPolicyEngine` allows all tool calls.

## 5. Review Policy
`UnsafeLocalCodeExecutor` documents danger explicitly. No SECURITY.md for API auth. Cloud Run deploy binds `0.0.0.0` without server-side auth.

## 6. Threat Model
- **Assets:** agent runtime, sessions, artifacts, host via code executors/MCP
- **Attackers:** network clients; cross-origin browsers
- **Entry points:** `/run`, `/run_sse`, sessions, artifacts, `/debug/trace/*`, A2A routes

## 7. Historical Security Evidence
- Artifact path traversal fix (#210 referenced in prior audits)
- Prototype pollution fix (#410)
- OAuth SSRF fix (#354)

## 8. Campaigns and Coverage
| Campaign | Status |
|----------|--------|
| HTTP/API authentication | complete |
| State injection | complete |
| Skill path traversal | complete (mitigated) |
| CORS/WebSocket | complete |
| Code execution surfaces | partial |
| A2A auth | complete |
| CodeQL/Semgrep | not run |
| Anti-anchoring | complete |

## 9. Tools and Static Analysis
CodeQL/Semgrep not installed. Manual review performed.

## 10. Confirmed Findings
- [High] FIND-JS-001: Unauthenticated AdkApiServer exposes agent execution API
- [High] FIND-JS-002: Cross-session state injection via HTTP stateDelta
- [Medium] FIND-JS-003: Permissive default CORS and WebSocket origin policy on dev server

Counts: Critical=0, High=2, Medium=1, Low=0, Informational=0

## 11. Unresolved and Low-Priority Leads
- `UnsafeLocalCodeExecutor` when enabled on network-exposed agents
- MCP stdio subprocess spawn
- `yaml.load` on skill frontmatter
- 50MB JSON body DoS limit
- Dependency advisories (not scanned)

## 12. Rejected and Duplicate Leads
- Skill resource path traversal: prefix normalization blocks escape
- File artifact service: explicit `..` rejection

## 13. Exploit-Chain Analysis
Missing auth + state injection chain confirmed by source. No PoC executed.

## 14. Overall Assessment
Confirmed high-severity missing auth and state injection. Skill/artifact path controls adequate. Do not expose API without perimeter auth.

## 15. Coverage Limitations
No dependency scan, no dynamic PoC, no minified UI bundle deep review.

## 16. Runtime, Resource, and Cleanup Metadata
- Output: `/tmp/audit-20260729T020126Z-6de27f8a/`
- Worktree: detached at 4169319
- User checkout: unchanged
