# Unauthenticated Debug Telemetry Endpoints

## Summary
`GET /debug/trace/:eventId` and `/debug/trace/session/:sessionId` are always registered regardless of `serveDebugUI`, exposing span attributes without auth.

## Affected Repository and Revision
forked-oss/adk-js @ `4169319f46e7bdac1e4aac0247f6ec7099c5097f`

## Entry Point and Evidence Trace
- `adk_api_server.ts:210-259`
- `telemetry_utils.ts:57-59` stores LLM/tool span data in `traceDict`

## Impact
Prompt and tool I/O disclosure.

## CWE and CVSS
CWE-200; CVSS 4.0 High 7.5 (when deployed to 0.0.0.0)

## Remediation
Gate debug routes behind dev flag; never enable in production.
