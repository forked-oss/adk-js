# Unauthenticated Debug/Trace Endpoints Leak Telemetry

## Summary

`GET /debug/trace/:eventId` and `GET /debug/trace/session/:sessionId` return OpenTelemetry span data without authentication.

## Evidence

`adk-js@4169319:dev/src/server/adk_api_server.ts:210-259`

## Impact

Disclosure of LLM prompts, tool inputs/outputs, and session execution metadata.

## CWE and CVSS

- **CWE:** CWE-200
- **CVSS 4.0:** **5.3 (Medium)**

## Remediation

Disable debug routes in `api_server` mode; require auth.

## Structured Evidence Record

```json
{"id": "FIND-JS-005", "truth": "confirmed", "evidence": ["adk-js@4169319:dev/src/server/adk_api_server.ts:210"]}
```
