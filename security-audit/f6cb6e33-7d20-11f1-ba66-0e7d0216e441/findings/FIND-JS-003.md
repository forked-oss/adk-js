# Unauthenticated Debug Trace Disclosure

## Summary

`/debug/trace/:eventId` and `/debug/trace/session/:sessionId` return full trace dictionaries including LLM request/response attributes without authentication.

## Affected Repository and Revision

- Repository: forked-oss/adk-js @ `4169319f`

## Entry Point and Evidence Trace

1. `adk-js@4169319:dev/src/server/adk_api_server.ts:210` — event trace endpoint
2. `adk-js@4169319:dev/src/server/adk_api_server.ts:230` — session trace endpoint

## CWE and CVSS

- CWE-200 — **Medium (6.5)**

## Remediation

Gate debug routes behind dev mode flag or authentication.
