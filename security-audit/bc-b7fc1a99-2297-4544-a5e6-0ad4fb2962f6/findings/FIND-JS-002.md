# Command Injection in Skill Script Wrapper via script_path

## Summary

`buildWrapperCode` interpolates raw `scriptPath` into shell, PowerShell, and CMD wrapper strings without escaping, enabling command injection when the LLM or caller supplies a malicious path.

## Affected Repository and Revision

- Repository: forked-oss/adk-js
- Revision: `4169319f46e7bdac1e4aac0247f6ec7099c5097f`

## Evidence

`adk-js@4169319:core/src/tools/skill/run_skill_script_tool.ts:145-164`

Example: `scriptPath` of `x; id` in SHELL case becomes `source ./x; id "$@"`.

Lookup uses `relScriptPath` but wrapper uses raw `scriptPath` at line 125.

## Impact

Arbitrary command execution when `UnsafeLocalCodeExecutor` or shell-based executor is configured.

## CWE and CVSS

- **CWE:** CWE-78
- **CVSS 4.0:** `CVSS:4.0/AV:N/AC:L/AT:P/PR:L/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N` — **7.6 (High)**

## Remediation

Use basename-only paths; escape shell metacharacters; avoid shell wrappers.

## Structured Evidence Record

```json
{"id": "FIND-JS-002", "truth": "confirmed", "controlled_value": "script_path parameter", "evidence": ["adk-js@4169319:core/src/tools/skill/run_skill_script_tool.ts:157"]}
```
