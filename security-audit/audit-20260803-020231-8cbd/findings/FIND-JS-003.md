# A2A endpoints use noAuthentication and optional permissive CORS

## Summary
toA2a() mounts restHandler and jsonRpcHandler with userBuilder: UserBuilder.noAuthentication (agent_to_a2a.ts:114,121). AdkApiServer enables CORS only when allowOrigins CLI option is set, passing the string directly as cors origin (adk_api_server.ts:177-181). Prior wildcard ACAO on run_sse was removed (commit d2db989). Without allowOrigins, no CORS middleware is applied; server still processes cross-origin requests from non-browser clients.

**Severity:** MEDIUM | **CWE:** CWE-346 | **CVSS 4.0:** CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:R/VC:L/VI:L/VA:N/SC:N/SI:N/SA:N

## Affected Repository and Revision
- Repository: https://github.com/forked-oss/adk-js
- Revision: `4169319f46e7bdac1e4aac0247f6ec7099c5097f`

## Affected Feature
['core/src/a2a/agent_to_a2a.ts', 'dev/src/server/adk_api_server.ts']

## Threat Scenario
An unauthenticated network attacker (or malicious web page via CORS/WebSocket) reaches the ADK development or production HTTP server and exploits missing authentication and/or state-scope controls.

## Root Cause
Server endpoints accept client-supplied identity (`user_id`, `session_id`) and state mutations without binding to an authenticated principal or validating state key scopes.

## Technical Details
toA2a() mounts restHandler and jsonRpcHandler with userBuilder: UserBuilder.noAuthentication (agent_to_a2a.ts:114,121). AdkApiServer enables CORS only when allowOrigins CLI option is set, passing the string directly as cors origin (adk_api_server.ts:177-181). Prior wildcard ACAO on run_sse was removed (commit d2db989). Without allowOrigins, no CORS middleware is applied; server still processes cross-origin requests from non-browser clients.



## Entry Point and Evidence Trace
- `adk-js@4169319f:core/src/a2a/agent_to_a2a.ts:114`
- `adk-js@4169319f:core/src/a2a/agent_to_a2a.ts:121`
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:177`
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:179`

## Failed or Missing Protection
- No inbound authentication middleware on HTTP/WebSocket routes
- No rejection of `app:`/`user:` prefixed keys from HTTP-supplied `stateDelta`/`state`
- No server-side binding of `userId` to authenticated identity

## Proof of Concept
Static source verification only. Dynamic PoC not executed in this audit run.


## Impact
Unauthenticated A2A agent invocation when --a2a enabled on reachable interfaces. Browser-based cross-origin invocation possible when CORS origin is misconfigured.

## CWE and CVSS
- **CWE:** CWE-346
- **CVSS 4.0:** CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:R/VC:L/VI:L/VA:N/SC:N/SI:N/SA:N
- **Attacker prerequisites:** Network-reachable ADK HTTP server without perimeter authentication

## Audit Priority
High — affects default network-exposed deployment patterns (bind 0.0.0.0, Cloud Run without IAM).

## Disposition and Maintainer Context
New finding. ADK HTTP servers are documented primarily for local development; production deployments are expected to use perimeter authentication (Cloud Run IAM, API gateway). The framework does not enforce this at the application layer.

## Remediation
Implement A2A authentication (UserBuilder with credential validation); restrict --a2a to authenticated deployments; validate allowOrigins against allowlist; document secure defaults for production.

## References
- Evidence files under `/tmp/audit-20260803-020231-8cbd/evidence/adk-js/`

## Limitations
Static analysis only; no dynamic exploitation performed.

## Structured Evidence Record
```json
{
  "id": "FIND-JS-003",
  "title": "A2A endpoints use noAuthentication and optional permissive CORS",
  "severity": "medium",
  "cwe": "CWE-346",
  "evidence": [
    "adk-js@4169319f:core/src/a2a/agent_to_a2a.ts:114",
    "adk-js@4169319f:core/src/a2a/agent_to_a2a.ts:121",
    "adk-js@4169319f:dev/src/server/adk_api_server.ts:177",
    "adk-js@4169319f:dev/src/server/adk_api_server.ts:179"
  ]
}
```
