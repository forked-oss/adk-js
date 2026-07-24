# Unauthenticated Agent Execution API

## Summary
`AdkApiServer` exposes `/run`, `/run_sse`, session/artifact CRUD, and A2A routes with no authentication. `userId` is client-controlled.

## Affected Repository and Revision
forked-oss/adk-js @ `4169319f46e7bdac1e4aac0247f6ec7099c5097f`

## Entry Point and Evidence Trace
- `POST /run` — `dev/src/server/adk_api_server.ts:707-756`
- A2A `UserBuilder.noAuthentication` — `core/src/a2a/agent_to_a2a.ts:114,121`
- Production deploy binds `0.0.0.0` — `dev/src/cli/deploy/deploy_utils.ts:63`

## Impact
Remote agent execution, session hijack, LLM abuse when exposed.

## CWE and CVSS
CWE-306; CVSS 4.0 High 8.7

## Remediation
Add API key/OAuth auth; restrict production bind address.
