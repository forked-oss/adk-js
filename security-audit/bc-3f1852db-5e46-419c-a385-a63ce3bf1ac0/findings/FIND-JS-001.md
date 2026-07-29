# Unauthenticated AdkApiServer exposes agent execution API

## Summary
`AdkApiServer` Express application registers agent execution, session, artifact, and debug routes without authentication middleware.

## Affected Repository and Revision
- Repository: forked-oss/adk-js
- Revision: 4169319f46e7bdac1e4aac0247f6ec7099c5097f

## Affected Feature
`AdkApiServer` — `/run`, `/run_sse`, sessions, artifacts, `/debug/trace/*`, `/list-apps`

## Technical Details
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:191-194` — request logging only, no auth
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:707-743` — `/run` executes agent without credentials
- `adk-js@4169319f:dev/src/server/adk_api_server.ts:210-249` — debug traces unauthenticated
- `adk-js@4169319f:dev/src/cli/deploy/deploy_utils.ts:63` — Cloud Run CMD `--host=0.0.0.0`

## Impact
Unauthorized agent execution, session/artifact IDOR, telemetry disclosure.

## CWE and CVSS
- CWE-306
- CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:L/SC:N/SI:N/SA:N — 8.7 (High)

## Remediation
Add auth middleware; use IAP/reverse proxy on Cloud Run; disable debug routes in production.

## References
- Evidence: `adk-js@4169319f:dev/src/server/adk_api_server.ts:191`
