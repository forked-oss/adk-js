# MCP HTTP Transport Lacks SSRF Protections

## Summary

MCP streamable HTTP and SSE transports connect to user-configured URLs via `new URL()` without blocking private IPs, localhost, or link-local addresses, unlike OAuth discovery which implements `validateDiscoveryUrl`.

## Evidence

- `adk-js@4169319:core/src/tools/mcp/mcp_session_manager.ts:90-107` (no SSRF filter)
- `adk-js@4169319:core/src/auth/oauth2/oauth2_discovery.ts:194-246` (SSRF guards present for comparison)

## Impact

Server-side requests to internal services when agent uses MCP HTTP with attacker-influenced URL.

## CWE and CVSS

- **CWE:** CWE-918 (SSRF)
- **CVSS 4.0:** `CVSS:4.0/AV:N/AC:L/AT:P/PR:L/UI:N/VC:L/VI:L/VA:N/SC:L/SI:N/SA:N` — **5.9 (Medium)** — High audit priority.

## Remediation

Apply same SSRF validation as OAuth discovery to MCP HTTP URLs.

## Structured Evidence Record

```json
{"id": "FIND-JS-004", "truth": "confirmed", "evidence": ["adk-js@4169319:core/src/tools/mcp/mcp_session_manager.ts:102"]}
```
