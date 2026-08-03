# Unauthenticated AdkApiServer exposes agent execution API

## Summary
Express init() applies request logging middleware only (line 191) with no auth gate before route handlers. POST /run (707) and POST /run_sse (758) execute agents using body-supplied appName/userId/sessionId. GET /debug/trace/:eventId (210) and GET /debug/trace/session/:sessionId (230) expose internal telemetry. Cloud Run container CMD hardcodes --host=0.0.0.0 (deploy_utils.ts:63), binding all interfaces without complementary server-side auth.

**Severity:** HIGH | **CWE:** CWE-306 | **CVSS 4.0:** CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:L/SC:N/SI:N/SA:N

## Affected Repository and Revision
- Repository: https://github.com/forked-oss/adk-js
- Revision: `4169319f46e7bdac1e4aac0247f6ec7099c5097f`

## Affected Feature
['dev/src/server/adk_api_server.ts', 'dev/src/cli/deploy/deploy_utils.ts']

## Threat Scenario
An unauthenticated network attacker (or malicious web page via CORS/WebSocket) reaches the ADK development or production HTTP server and exploits missing authentication and/or state-scope controls.

## Root Cause
Server endpoints accept client-supplied identity (`user_id`, `session_id`) and state mutations without binding to an authenticated principal or validating state key scopes.

## Technical Details
Express init() applies request logging middleware only (line 191) with no auth gate before route handlers. POST /run (707) and POST /run_sse (758) execute agents using body-supplied appName/userId/sessionId. GET /debug/trace/:eventId (210) and GET /debug/trace/session/:sessionId (230) expose internal telemetry. Cloud Run container CMD hardcodes --host=0.0.0.0 (deploy_utils.ts:63), binding all interfaces without complementary server-side auth.



## Entry Point and Evidence Trace
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:191`
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:707`
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:758`
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:210`
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:230`
- `adk-js@4169319f:dev/src/cli/deploy/deploy_utils.ts:63`

## Failed or Missing Protection
- No inbound authentication middleware on HTTP/WebSocket routes
- No rejection of `app:`/`user:` prefixed keys from HTTP-supplied `stateDelta`/`state`
- No server-side binding of `userId` to authenticated identity

## Proof of Concept
Static source verification only. Dynamic PoC not executed in this audit run.


## Impact
Unauthorized remote agent execution, IDOR on sessions/artifacts by guessing or enumerating IDs, disclosure of internal trace/span telemetry, potential cost abuse via LLM/tool invocations.

## CWE and CVSS
- **CWE:** CWE-306
- **CVSS 4.0:** CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:L/SC:N/SI:N/SA:N
- **Attacker prerequisites:** Network-reachable ADK HTTP server without perimeter authentication

## Audit Priority
High — affects default network-exposed deployment patterns (bind 0.0.0.0, Cloud Run without IAM).

## Disposition and Maintainer Context
New finding. ADK HTTP servers are documented primarily for local development; production deployments are expected to use perimeter authentication (Cloud Run IAM, API gateway). The framework does not enforce this at the application layer.

## Remediation
Add authentication middleware (API keys, OAuth2, mTLS) before route registration; disable or gate /debug/* in production; document that perimeter auth (IAP, API gateway) is mandatory for network exposure; consider binding warnings when host is not localhost.

## References
- Evidence files under `/tmp/audit-20260803-020231-8cbd/evidence/adk-js/`

## Limitations
Static analysis only; no dynamic exploitation performed.

## Structured Evidence Record
```json
{
  "id": "FIND-JS-001",
  "title": "Unauthenticated AdkApiServer exposes agent execution API",
  "severity": "high",
  "cwe": "CWE-306",
  "evidence": [
    "adk-js@4169319f:dev/src/server/adk_api_server.ts:191",
    "adk-js@4169319f:dev/src/server/adk_api_server.ts:707",
    "adk-js@4169319f:dev/src/server/adk_api_server.ts:758",
    "adk-js@4169319f:dev/src/server/adk_api_server.ts:210",
    "adk-js@4169319f:dev/src/server/adk_api_server.ts:230",
    "adk-js@4169319f:dev/src/cli/deploy/deploy_utils.ts:63"
  ]
}
```
