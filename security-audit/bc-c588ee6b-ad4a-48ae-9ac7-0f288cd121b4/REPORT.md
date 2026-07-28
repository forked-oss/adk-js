# Security Audit Report

## 1. Repository, Target Revision, and Freshness
- Repository: https://github.com/forked-oss/adk-js
- Audited revision: 4169319f46e7bdac1e4aac0247f6ec7099c5097f (4169319)
- Target freshness: fresh (HEAD matches `origin/main` at audit time)
- Run ID: bc-c588ee6b-ad4a-48ae-9ac7-0f288cd121b4

## 2. Scope and Assumptions
Headless security audit of production-reachable library and server code in the Agent Development Kit (adk-js). Assumes agents may be deployed with default dev/API servers network-reachable unless platform IAM compensates.

## 3. Repository Inventory
See campaign notes: languages, HTTP entry points (REST/WebSocket), CLI deploy tools, session services, skill loaders, code executors, MCP transports, CI workflows reviewed via source enumeration.

## 4. Functional and Security Architecture
ADK provides agent runtime (Runner), tool execution, session/artifact services, and optional HTTP dev/API servers. Cross-session state uses `app:` and `user:` key prefixes. HTTP APIs do not implement application-level authentication by default.

## 5. Review Policy
No SECURITY.md documenting accepted risk for missing API auth. Samples warn about unrestricted MCP/database modes. Java cross-session fix exists unmerged (30d81c5).

## 6. Threat Model
- Assets: agent behavior, session state, artifacts, host filesystem (skills), cloud credentials via tools
- Attackers: network clients on exposed dev/API servers; cross-site browsers
- Entry points: `/run`, `/run_sse`, `/run_live`, sessions, triggers, tools

## 7. Historical Security Evidence
Prior audit report commits on branches; path traversal fix #210 (JS artifacts); OAuth SSRF #354; prototype pollution #410; tool confirmation hardening (Python c03f3337); A2A auth propagation (Go #861).

## 8. Campaigns and Coverage
| Campaign | Status |
|----------|--------|
| HTTP/API authentication | complete |
| Cross-session state injection | complete |
| Skill path traversal | complete (Java filesystem gap found) |
| WebSocket/CORS | complete |
| Trigger authentication | complete |
| Code execution surfaces | partial (design-intentional executors noted) |
| Dependency/CVE scan | not run (no scanner binaries) |
| CodeQL/Semgrep | not run |

## 9. Tools and Static Analysis
No CodeQL/Semgrep/gitleaks installed. Manual source review and git history mining performed.

## 10. Confirmed Findings
- [High] FIND-JS-001: Unauthenticated AdkApiServer exposes agent execution API
- [High] FIND-JS-002: Cross-session state injection via HTTP stateDelta
- [Medium] FIND-JS-003: Permissive default CORS and WebSocket origin policy on dev server

Counts: Critical=0, High=2, Medium=1, Low=0, Informational=0

## 11. Unresolved and Low-Priority Leads
- MCP stdio command transport when YAML config is attacker-controlled
- OpenAPI/RestApiTool SSRF when specs are untrusted
- ContainerCodeExecutor Docker escape surface (operator configuration)
- Transitive dependency advisories (not scanned)

## 12. Rejected and Duplicate Leads
- Python/JS skill resource path traversal: in-memory maps prevent filesystem escape
- Go skill loader: `path.Clean` blocks traversal after prefix check

## 13. Exploit-Chain Analysis
Unauthenticated API (FIND-*-001) chains with cross-session state injection (FIND-*-002) for cross-user agent behavior manipulation. No full chain PoC executed.

## 14. Overall Assessment
Confirmed high-severity issues around missing HTTP authentication and cross-session state injection across all four SDKs, plus Java-specific filesystem path traversal in skill loading. Deployments must not expose dev/API servers without network-layer authentication.

## 15. Coverage Limitations
No dependency scanner, no fuzzing, no dynamic PoC execution, no CodeQL. Native code and browser bundle minified JS not deeply reviewed.

## 16. Runtime, Resource, and Cleanup Metadata
- Audit output: /tmp/cursor-audit-output/bc-c588ee6b-ad4a-48ae-9ac7-0f288cd121b4
- Worktrees: disposable detached snapshots at audited revisions
- CPU: 8 cores, 47Gi RAM available
