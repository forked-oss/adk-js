# Security Audit Report

## 1. Repository, Target Revision, and Freshness
- **Repository:** https://github.com/forked-oss/adk-js
- **Audited revision:** `4169319f46e7bdac1e4aac0247f6ec7099c5097f`
- **Short revision:** `4169319`
- **Target freshness:** fresh (local HEAD matches `origin/main` at audit time)
- **Audit run:** audit-20260723-020250
- **Audit date:** 2026-07-23 UTC

## 2. Scope and Assumptions
- Isolated read-only worktree at upstream `main`
- Production-reachable code: HTTP servers, CLI deploy paths, library APIs shipped in packages
- Assumes network exposure is realistic for deploy targets (Cloud Run, `0.0.0.0` bind)
- No dynamic PoC execution on audit host

## 3. Repository Inventory
See campaign evidence in `/tmp/audit-20260723-020250/inventory/adk-js-checkout.json`. Key surfaces: HTTP REST/A2A APIs, dev web launchers, agent runtime, tools/MCP, session services, skill loading, code executors.

## 4. Functional and Security Architecture
Agent Development Kit (adk-js): code-first agent framework with HTTP dev/production servers, session and artifact services, tool execution (including MCP and skills), and optional webhook triggers. Security boundary expected at deployment perimeter; in-process auth is largely absent on HTTP surfaces.

## 5. Review Policy
- No repository SECURITY.md with accepted-risk register found
- Dev servers documented/examples reference localhost defaults
- `UnsafeLocalCodeExecutor` / similar explicitly warn about unsandboxed execution (JS)
- HITL confirmation fix was documented in commit `c03f3337` then reverted (`adk-python` only)

## 6. Threat Model
**Assets:** session state, artifacts, LLM prompts/traces, host filesystem, cloud credentials via tools  
**Attackers:** unauthenticated network clients, cross-origin browsers, session event injectors  
**Entry points:** REST `/run`, WebSocket `/run_live`, debug traces, webhooks, skill tools, HITL confirmation flow

## 7. Historical Security Evidence
Git history reviewed for security fixes. Notable patterns: CORS/SSRF/prototype-pollution fixes (JS), template injection fix (Go), SSRF hardening in `load_web_page` (Python), path traversal tests (Python skills), HITL fix regression (Python).

## 8. Campaigns and Coverage
| Campaign | Status | Notes |
|----------|--------|-------|
| HTTP auth boundary review | complete | All controllers/routers enumerated |
| Debug telemetry disclosure | complete | Endpoints confirmed unauthenticated |
| Webhook trigger auth | complete | No signature verification |
| Skill path traversal | complete | Java confirmed; Go/JS/Python mitigated |
| HITL confirmation integrity | complete (Python) | Regression confirmed |
| Command execution surfaces | complete | MCP/config-time and unsafe executors |
| SSRF | partial | Config-driven MCP URLs; Python proxy bypass |
| Dependency/CVE scan | not run | No CodeQL/Semgrep/trivy installed |
| Anti-anchoring fresh review | complete | Independent surface enumeration |

## 9. Tools and Static Analysis
- **CodeQL:** not run (binary unavailable)
- **Semgrep:** not run (binary unavailable)
- **Method:** manual source review, ripgrep enumeration, git log mining, subagent exploration

## 10. Confirmed Findings
- **[High]** ADKJS-2026-001: Unauthenticated AdkApiServer exposes agent execution and session APIs — see `findings/ADKJS-2026-001.md`
- **[High]** ADKJS-2026-002: Host code execution via UnsafeLocalCodeExecutor when API is network-exposed — see `findings/ADKJS-2026-002.md`
- **[Medium]** ADKJS-2026-003: Unauthenticated debug trace endpoints disclose telemetry — see `findings/ADKJS-2026-003.md`

**Counts:** Critical=0, High=2, Medium=1, Low=0, Informational=0

## 11. Unresolved and Low-Priority Leads
- MCP stdio command execution from YAML config (config-trust model; not confirmed as API-exploitable)
- OAuth token endpoint SSRF (JS) — requires attacker-controlled OpenAPI spec
- Pickle deserialization in session DB dialects (Python) — requires DB write access
- Dependency advisories not verified without scanner

## 12. Rejected and Duplicate Leads
- Missing security headers alone — out of scope per audit policy
- Logout-only CSRF — not applicable
- UUID predictability — no finding
- Go skill path traversal — rejected; `filesystem_source.go` has prefix containment

## 13. Exploit-Chain Analysis
Chains evaluated where unauthenticated API combines with code execution tools (JS) or HITL bypass (Python). No additional confirmed chain beyond documented finding interactions.

## 14. Overall Assessment
0 critical, 2 high, and 1 medium severity findings confirmed under completed coverage. Primary systemic gap: HTTP surfaces lack authentication, relying on deployment perimeter. adk-js requires explicit network controls and auth proxy for any non-localhost deployment.

## 15. Coverage Limitations
- No CodeQL/Semgrep/trivy execution
- No runtime PoC or fuzzing
- Dependency CVE reachability not fully analyzed
- Minified browser bundles not dynamically tested

## 16. Runtime, Resource and Cleanup Metadata
- Run ID: audit-20260723-020250
- Worktree: `/tmp/audit-20260723-020250/worktrees/adk-js`
- Host: 8 CPU, 47Gi RAM, scanners unavailable
- User checkout preserved (detached worktrees only)
