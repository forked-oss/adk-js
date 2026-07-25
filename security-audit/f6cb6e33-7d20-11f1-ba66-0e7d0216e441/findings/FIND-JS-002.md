# Unauthenticated A2A REST/JSON-RPC Endpoints

## Summary

A2A handlers explicitly use `UserBuilder.noAuthentication` for both REST and JSON-RPC transports.

## Affected Repository and Revision

- Repository: forked-oss/adk-js @ `4169319f`

## Entry Point and Evidence Trace

1. `adk-js@4169319:core/src/a2a/agent_to_a2a.ts:110` — REST handler with `UserBuilder.noAuthentication`
2. `adk-js@4169319:core/src/a2a/agent_to_a2a.ts:117` — JSON-RPC handler, same

## Impact

Unauthenticated A2A agent invocation when `--a2a` enabled on exposed host.

## CWE and CVSS

- CWE-306 — **High (8.7)**

## Remediation

Provide authenticated `UserBuilder` implementation; require auth for production A2A.
