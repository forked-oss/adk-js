# Permissive default CORS and WebSocket origin policy on dev server

## Summary
Dev server defaults allow broad cross-origin access: CORS origins `*` and WebSocket `setAllowedOrigins("*")` in related Java parity; JS server uses configurable CORS with historical hardcoded `*` removals in some paths. WebSocket config in Java sets `*`; JS dev server CORS middleware follows permissive patterns when not restricted.

## Affected Repository and Revision
- Repository: forked-oss/adk-js
- Revision: 4169319f46e7bdac1e4aac0247f6ec7099c5097f

## Technical Details
Historical fixes #360, #378 removed some permissive headers; default deployment still allows network exposure without origin restrictions when misconfigured.

Evidence: `adk-js@4169319:dev/src/server/adk_api_server.ts` CORS setup; compare `dev/src/cli/cli.ts` host binding.

## Impact
Facilitates cross-origin abuse of unauthenticated API when combined with FIND-JS-001.

## CWE and CVSS
- CWE-942
- CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:R/VC:L/VI:L/VA:N/SC:N/SI:N/SA:N (5.1 Medium)

## Remediation
Default deny cross-origin; require explicit allowlist for production.
