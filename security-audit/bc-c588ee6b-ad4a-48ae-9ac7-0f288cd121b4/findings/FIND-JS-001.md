# Unauthenticated AdkApiServer exposes agent execution API

## Summary
`AdkApiServer` (Express) exposes `/run`, `/run_sse`, sessions, artifacts, and eval routes without authentication.

## Affected Repository and Revision
- Repository: forked-oss/adk-js
- Revision: 4169319f46e7bdac1e4aac0247f6ec7099c5097f

## Technical Details
- `dev/src/server/adk_api_server.ts` — Express routes for agent execution
- `dev/src/cli/cli.ts` — `web` and `api_server` commands start server

Evidence: `adk-js@4169319:dev/src/server/adk_api_server.ts:157`

## Impact
Unauthorized agent execution when server binds beyond localhost.

## CWE and CVSS
- CWE-306
- CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:N/SC:N/SI:N/SA:N (8.7 High)

## Remediation
Add authentication middleware; default secure bind configuration.
