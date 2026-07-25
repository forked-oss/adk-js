# Unauthenticated Agent Execution and Session API

## Summary

`AdkApiServer` exposes `/run`, `/run_sse`, session CRUD, and artifact endpoints without authentication middleware.

## Affected Repository and Revision

- Repository: forked-oss/adk-js @ `4169319f`

## Entry Point and Evidence Trace

1. `adk-js@4169319:dev/src/server/adk_api_server.ts:120` — Express app, no auth middleware
2. `adk-js@4169319:dev/src/server/adk_api_server.ts:707` — `POST /run`
3. `adk-js@4169319:dev/src/server/adk_api_server.ts:340` — session CRUD with client-supplied `userId`

## Impact

Unauthorized agent execution, session/artifact IDOR.

## CWE and CVSS

- CWE-306, CWE-639 — **High (8.7)**

## Remediation

Add API key or OIDC middleware; document auth requirements for Cloud Run deploy.
