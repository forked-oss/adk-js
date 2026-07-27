# Unauthenticated HTTP API Executes Agents and Exposes Sessions/Artifacts

## Summary

`AdkApiServer` registers all production routes (`/run`, `/run_sse`, sessions, artifacts) without authentication middleware.

## Affected Repository and Revision

- Repository: forked-oss/adk-js
- Revision: `4169319f46e7bdac1e4aac0247f6ec7099c5097f`

## Evidence

`adk-js@4169319:dev/src/server/adk_api_server.ts:196-831` — no auth middleware; tests confirm unauthenticated access (`adk_api_server_test.ts:488-833`).

## Impact

Full agent execution and session/artifact access for network clients.

## CWE and CVSS

- **CWE:** CWE-306
- **CVSS 4.0:** **8.7 (High)** — Critical audit priority.

## Remediation

Add authentication middleware; default localhost; gate debug routes.

## Structured Evidence Record

```json
{"id": "FIND-JS-001", "truth": "confirmed", "severity": "critical", "evidence": ["adk-js@4169319:dev/src/server/adk_api_server.ts:707-831"]}
```
