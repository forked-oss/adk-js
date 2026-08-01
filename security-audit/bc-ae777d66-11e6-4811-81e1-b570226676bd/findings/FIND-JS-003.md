# Permissive default CORS and WebSocket origin policy on dev server

## Summary
`AdkApiServer` optional CORS and A2A WebSocket configuration default to permissive origins. A2A explicitly uses `UserBuilder.noAuthentication`.

## Affected Repository and Revision
- Repository: forked-oss/adk-js
- Revision: 4169319f46e7bdac1e4aac0247f6ec7099c5097f

## Technical Details
- `adk-js@4169319f:core/src/a2a/agent_to_a2a.ts:114,121` — `UserBuilder.noAuthentication`
- CORS controlled by `allowOrigins` CLI option; when unset, cross-origin behavior depends on Express defaults
- A2A JSON-RPC mounted without ADK-level auth when `--a2a` enabled

## Impact
Cross-origin agent invocation when server is reachable and CORS/origin checks are permissive.

## CWE and CVSS
- CWE-346
- CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:R/VC:L/VI:L/VA:N/SC:N/SI:N/SA:N — 5.1 (Medium)

## Remediation
Restrict CORS origins; add A2A authentication; do not enable `--a2a` on public interfaces without perimeter auth.

## References
- Evidence: `adk-js@4169319f:core/src/a2a/agent_to_a2a.ts:114`
