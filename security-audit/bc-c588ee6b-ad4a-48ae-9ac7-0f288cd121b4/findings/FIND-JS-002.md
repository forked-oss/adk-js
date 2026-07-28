# Cross-session state injection via HTTP stateDelta

## Summary
`AdkApiServer` forwards `stateDelta` from HTTP bodies to the runner. `InMemorySessionService` applies `app:` and `user:` prefixed keys to shared state maps.

## Affected Repository and Revision
- Repository: forked-oss/adk-js
- Revision: 4169319f46e7bdac1e4aac0247f6ec7099c5097f

## Technical Details
- `adk_api_server.ts:708,742` — stateDelta from request body
- `in_memory_session_service.ts:275-288` — app/user prefix handling

Evidence: `adk-js@4169319:dev/src/server/adk_api_server.ts:708`, `adk-js@4169319:core/src/sessions/in_memory_session_service.ts:277`

## Impact
Cross-session / cross-user state tampering.

## CWE and CVSS
- CWE-284
- CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:L/VI:H/VA:N/SC:N/SI:H/SA:N (8.2 High)

## Remediation
Reject privileged state key prefixes from HTTP clients.
