# Cross-session state injection via HTTP stateDelta

## Summary
HTTP clients inject `app:`/`user:` prefixed keys via `stateDelta` on `/run`/`/run_sse` and `state` on session create. Session services escalate prefixed keys to global stores.

## Affected Repository and Revision
- Repository: forked-oss/adk-js
- Revision: 4169319f46e7bdac1e4aac0247f6ec7099c5097f

## Technical Details
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:707-743` — stateDelta from body to runner
- `adk-js@4169319f:core/src/runner/runner.ts:316-318` — applied to event actions
- `adk-js@4169319f:core/src/sessions/in_memory_session_service.ts:275-288` — app:/user: routing
- `adk-js@4169319f:core/src/sessions/database_session_service.ts:148-164,449-467` — DB backend same behavior

## Impact
Cross-user/cross-session agent state manipulation.

## CWE and CVSS
- CWE-639
- CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:L/VI:H/VA:N/SC:N/SI:H/SA:N — 8.2 (High)

## Remediation
Strip or reject reserved prefixes from HTTP-supplied stateDelta/state.

## References
- Evidence: `adk-js@4169319f:dev/src/server/adk_api_server.ts:742`
