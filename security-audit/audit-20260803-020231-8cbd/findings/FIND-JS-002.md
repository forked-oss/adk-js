# Cross-session state injection via HTTP stateDelta and session state

## Summary
Session POST handlers accept req.body.state without filtering app:/user: prefixes (adk_api_server.ts:398,436). /run extracts stateDelta from body (708) and passes to runner.runAsync (742). Runner attaches stateDelta to user event actions (runner.ts:316). InMemorySessionService.appendEvent routes app: keys to this.appState and user: keys to this.userState (in_memory_session_service.ts:275-288). DatabaseSessionService applies identical logic (database_session_service.ts:449-453). trimTempState only removes temp: keys, not app:/user: (base_session_service.ts:235-244). Session create with app: keys also spoofs merged instruction state via mergeStates ordering (base_session_service.ts:255-268).

**Severity:** HIGH | **CWE:** CWE-639 | **CVSS 4.0:** CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:L/VI:H/VA:N/SC:N/SI:H/SA:N

## Affected Repository and Revision
- Repository: https://github.com/forked-oss/adk-js
- Revision: `4169319f46e7bdac1e4aac0247f6ec7099c5097f`

## Affected Feature
['dev/src/server/adk_api_server.ts', 'core/src/runner/runner.ts', 'core/src/sessions/in_memory_session_service.ts', 'core/src/sessions/database_session_service.ts', 'core/src/sessions/base_session_service.ts']

## Threat Scenario
An unauthenticated network attacker (or malicious web page via CORS/WebSocket) reaches the ADK development or production HTTP server and exploits missing authentication and/or state-scope controls.

## Root Cause
Server endpoints accept client-supplied identity (`user_id`, `session_id`) and state mutations without binding to an authenticated principal or validating state key scopes.

## Technical Details
Session POST handlers accept req.body.state without filtering app:/user: prefixes (adk_api_server.ts:398,436). /run extracts stateDelta from body (708) and passes to runner.runAsync (742). Runner attaches stateDelta to user event actions (runner.ts:316). InMemorySessionService.appendEvent routes app: keys to this.appState and user: keys to this.userState (in_memory_session_service.ts:275-288). DatabaseSessionService applies identical logic (database_session_service.ts:449-453). trimTempState only removes temp: keys, not app:/user: (base_session_service.ts:235-244). Session create with app: keys also spoofs merged instruction state via mergeStates ordering (base_session_service.ts:255-268).



## Entry Point and Evidence Trace
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:398`
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:436`
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:708`
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:742`
- `adk-js@4169319f:core/src/runner/runner.ts:316`
- `adk-js@4169319f:core/src/sessions/in_memory_session_service.ts:275`
- `adk-js@4169319f:core/src/sessions/database_session_service.ts:449`

## Failed or Missing Protection
- No inbound authentication middleware on HTTP/WebSocket routes
- No rejection of `app:`/`user:` prefixed keys from HTTP-supplied `stateDelta`/`state`
- No server-side binding of `userId` to authenticated identity

## Proof of Concept
Static source verification only. Dynamic PoC not executed in this audit run.


## Impact
Attacker can poison application-wide agent configuration state affecting all users, manipulate per-user state across sessions, or spoof instruction template variables to alter agent behavior.

## CWE and CVSS
- **CWE:** CWE-639
- **CVSS 4.0:** CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:L/VI:H/VA:N/SC:N/SI:H/SA:N
- **Attacker prerequisites:** Network-reachable ADK HTTP server without perimeter authentication

## Audit Priority
High — affects default network-exposed deployment patterns (bind 0.0.0.0, Cloud Run without IAM).

## Disposition and Maintainer Context
New finding. ADK HTTP servers are documented primarily for local development; production deployments are expected to use perimeter authentication (Cloud Run IAM, API gateway). The framework does not enforce this at the application layer.

## Remediation
Reject or strip app: and user: prefixed keys from HTTP-supplied state and stateDelta at API boundary; restrict global state mutation to trusted agent/tool code paths only; require authentication before accepting state mutations.

## References
- Evidence files under `/tmp/audit-20260803-020231-8cbd/evidence/adk-js/`

## Limitations
Static analysis only; no dynamic exploitation performed.

## Structured Evidence Record
```json
{
  "id": "FIND-JS-002",
  "title": "Cross-session state injection via HTTP stateDelta and session state",
  "severity": "high",
  "cwe": "CWE-639",
  "evidence": [
    "adk-js@4169319f:dev/src/server/adk_api_server.ts:398",
    "adk-js@4169319f:dev/src/server/adk_api_server.ts:436",
    "adk-js@4169319f:dev/src/server/adk_api_server.ts:708",
    "adk-js@4169319f:dev/src/server/adk_api_server.ts:742",
    "adk-js@4169319f:core/src/runner/runner.ts:316",
    "adk-js@4169319f:core/src/sessions/in_memory_session_service.ts:275",
    "adk-js@4169319f:core/src/sessions/database_session_service.ts:449"
  ]
}
```
