# Security Audit Report

## 1. Repository, Target Revision, and Freshness
- **Repository:** https://github.com/forked-oss/adk-js
- **Audited revision:** `4169319f46e7bdac1e4aac0247f6ec7099c5097f`
- **Branch:** `origin/main`
- **Freshness:** fresh (fetched 2026-07-24T02:01Z)

## 2. Scope and Assumptions
- **Scope:** Core library, dev server (`AdkApiServer`), A2A, skills, code executors, deploy utils.
- **Assumptions:** Production Docker deploy binds `0.0.0.0` without auth layer.

## 3. Repository Inventory
| Category | Details |
|----------|---------|
| Languages | TypeScript |
| Package | `package.json`, npm |
| HTTP | Express (`dev/src/server/adk_api_server.ts`) |
| Streaming | SSE (no WebSocket) |
| Tools | Skills, MCP, UnsafeLocalCodeExecutor |
| Deploy | Cloud Run via `deploy_utils.ts` |

## 4. Functional and Security Architecture
Express-based API server with agent execution, session/artifact management, debug telemetry, and optional A2A routes. Skill resource loading has path normalization. OAuth2 discovery has SSRF protections; MCP HTTP does not.

## 5. Review Policy
- `UnsafeLocalCodeExecutor` explicitly warns about unsandboxed execution.
- CORS disabled by default; debug UI gated by `serveDebugUI` but debug trace routes always mounted.
- Default bind `localhost`; production deploy uses `0.0.0.0`.

## 6. Threat Model
| Element | Description |
|---------|-------------|
| Assets | LLM credentials, session data, host via code executor |
| Attackers | Unauthenticated HTTP clients |
| Entry points | `/run`, `/run_sse`, A2A, debug traces |
| Dangerous ops | UnsafeLocalCodeExecutor, skill scripts, MCP HTTP |

## 7. Historical Security Evidence
OAuth2 discovery SSRF blocklist added (`oauth2_discovery.ts:173-239`). Skill path traversal tests present.

## 8. Campaigns and Coverage
| Campaign | Status |
|----------|--------|
| HTTP API auth | complete |
| Debug telemetry | complete — always mounted |
| Skill path traversal | complete — protected |
| Code execution | complete — intentional via UnsafeLocalCodeExecutor |
| SSRF (OAuth/MCP) | complete — MCP unprotected |
| CORS/postMessage | complete |
| Anti-anchoring review | complete |

## 9. Tools and Static Analysis
Semgrep/CodeQL not installed. Manual review complete.

## 10. Confirmed Findings
| ID | Severity | Title |
|----|----------|-------|
| FIND-JS-001 | High | Unauthenticated agent execution API |
| FIND-JS-002 | High | Unauthenticated debug telemetry endpoints |

## 11. Unresolved and Low-Priority Leads
- MCP HTTP SSRF (config-driven): medium, unresolved
- OAuth token endpoint SSRF: medium, unresolved
- 50MB JSON body DoS: low

## 12. Rejected and Duplicate Leads
- Skill path traversal: rejected — `load_skill_resource_tool.ts:70-99`
- WebSocket handlers: not applicable — uses SSE only

## 13. Exploit-Chain Analysis
FIND-JS-001 + UnsafeLocalCodeExecutor: unauthenticated `/run` with agent using skill tools and unsafe executor enables RCE. Requires specific agent configuration.

## 14. Overall Assessment
**2 High confirmed.** Unauthenticated API and always-on debug routes are primary risks. Skill loading and OAuth discovery show security awareness.

## 15. Coverage Limitations
- No Semgrep/CodeQL
- Minified browser bundle not fully decompiled
- Dependency CVE reachability not verified

## 16. Runtime, Resource, and Cleanup Metadata
- Run ID: `headless-audit-20260724-020135`
- Worktree: `/tmp/audit-output/headless-audit-20260724-020135/worktrees/adk-js`
