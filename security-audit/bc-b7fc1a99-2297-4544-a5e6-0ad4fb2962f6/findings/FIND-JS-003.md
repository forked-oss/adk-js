# A2A Endpoints Configured with noAuthentication

## Summary

A2A REST and JSON-RPC handlers explicitly use `UserBuilder.noAuthentication`, exposing agent RPC without credentials.

## Evidence

`adk-js@4169319:core/src/a2a/agent_to_a2a.ts:114,121`

## Impact

Unauthenticated remote agent invocation when A2A is enabled.

## CWE and CVSS

- **CWE:** CWE-306
- **CVSS 4.0:** **7.5 (High)**

## Remediation

Require authentication for A2A endpoints in production configurations.

## Structured Evidence Record

```json
{"id": "FIND-JS-003", "truth": "confirmed", "evidence": ["adk-js@4169319:core/src/a2a/agent_to_a2a.ts:114"]}
```
