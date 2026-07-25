# Security Audit Report

## 1. Repository, Target Revision, and Freshness

| Field | Value |
|-------|-------|
| Repository | https://github.com/forked-oss/adk-js |
| Audited revision | `4169319f46e7bdac1e4aac0247f6ec7099c5097f` |
| Target freshness | **fresh** |
| Audit run ID | `f6cb6e33-7d20-11f1-ba66-0e7d0216e441` |

## 2. Scope and Assumptions

Audited `@google/adk` core SDK and `@google/adk-devtools` server. Scope includes `AdkApiServer`, A2A handlers, code executors, and deploy tooling.

## 3. Repository Inventory

| Category | Details |
|----------|---------|
| Language | TypeScript (Node.js) |
| Packages | `@google/adk`, `@google/adk-devtools` |
| HTTP | Express-based `AdkApiServer`, A2A REST/JSON-RPC |
| Auth | OAuth for tools only; **no API server auth** |
| CI | secretlint, validation workflow |

## 4. Functional and Security Architecture

ADK JS provides agent SDK and dev server. File artifact service has path traversal protections. Skill loading and OAuth discovery have SSRF mitigations. API server binds `localhost:8000` by default; Cloud Run deploy uses `0.0.0.0`.

## 5. Review Policy

No `SECURITY.md`. `UnsafeLocalCodeExecutor` explicitly documented as dangerous. secretlint in CI.

## 6. Threat Model

Network-exposed `AdkApiServer` without reverse-proxy auth enables full API access.

## 7. Historical Security Evidence

| Commit | Fix |
|--------|-----|
| `57b0af7` | OAuth discovery SSRF blocklist |
| `9008353` | Prototype pollution in streaming |
| `1fe631f` | CORS/urlencoded fix |
| `8d5cc0a` | Skill materializeFiles traversal checks |

## 8. Campaigns and Coverage

| Campaign | Status |
|----------|--------|
| API auth | complete — missing |
| A2A auth | complete — `UserBuilder.noAuthentication` |
| Debug endpoints | complete — exposed |
| Skill path traversal | complete — mitigated |
| MCP SSRF | partial — HTTP MCP lacks blocklist |
| CodeQL/Semgrep | not-run |

## 9. Tools and Static Analysis

Manual review only.

## 10. Confirmed Findings

| ID | Severity | Title |
|----|----------|-------|
| FIND-JS-001 | High | Unauthenticated agent execution and session API |
| FIND-JS-002 | High | Unauthenticated A2A REST/JSON-RPC endpoints |
| FIND-JS-003 | Medium | Unauthenticated debug trace disclosure |

## 11. Unresolved Leads

| ID | Claim |
|----|-------|
| LEAD-JS-001 | MCP HTTP SSRF via user-configured URL |
| LEAD-JS-002 | OAuth token endpoint lacks discovery URL validation |
| LEAD-JS-003 | 50MB JSON body DoS |

## 12. Rejected Leads

| Lead | Reason |
|------|--------|
| File artifact traversal | `FileArtifactService` segment regex + relative check |
| Skill path traversal | `materializeFiles` containment |

## 13. Exploit-Chain Analysis

FIND-JS-001 + `UnsafeLocalCodeExecutor` → arbitrary code execution when agent configured with unsafe executor. Chain requires agent configuration, not standalone API flaw.

## 14. Overall Assessment

**2 High, 1 Medium confirmed.** Skill and artifact protections are present. Primary risk is missing API authentication on network-exposed servers.

## 15. Coverage Limitations

- No npm audit executed
- No dynamic testing
- CodeQL/Semgrep unavailable

## 16. Runtime Metadata

Worktree: `/tmp/audit-output/f6cb6e33-7d20-11f1-ba66-0e7d0216e441/repos/adk-js/worktree`
