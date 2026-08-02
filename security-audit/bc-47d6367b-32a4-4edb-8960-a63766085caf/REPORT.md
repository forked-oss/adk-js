# Security Audit Report

## 1. Repository, Target Revision, and Freshness

| Field | Value |
|-------|-------|
| Repository | https://github.com/forked-oss/adk-js |
| Audited revision | `4169319f46e7bdac1e4aac0247f6ec7099c5097f` |
| Target freshness | fresh (origin/main fetched 2026-08-02) |
| Audit run | bc-47d6367b-32a4-4edb-8960-a63766085caf |
| Audit date | 2026-08-02 |

## 2. Scope and Assumptions

This audit covers the ADK JavaScript SDK at the pinned revision above. The ADK is a developer toolkit for building AI agents. HTTP servers (dev API, triggers, Agent Engine) are in scope when network-exposed. Assumes default launcher configurations and documented deployment patterns.

## 3. Repository Inventory

See `inventory.md` in audit artifacts. Primary languages and surfaces were enumerated via manifest analysis and entry-point tracing.

## 4. Functional and Security Architecture

ADK JavaScript provides agent orchestration (Runner), session/state management with `app:`/`user:` scoped keys, HTTP REST API for dev and deployment, artifact services, tool/MCP integrations, and optional triggers (Pub/Sub, Eventarc).

## 5. Review Policy

See `review-policy.md`. Findings require source-to-sink evidence, production reachability, and independent verification. Dev-only surfaces are reported with deployment prerequisites.

## 6. Threat Model

See `threat-model.md`. Primary trust boundary: network client → HTTP API → Runner → tools/MCP/filesystem. Protected assets: session data, app/user state, artifacts, LLM credentials, host filesystem.

## 7. Historical Security Evidence

Git history reviewed for auth, state validation, path confinement, and trigger verification patterns. Recurring fix themes projected onto current source. No historical issues reported without current-source verification.

## 8. Campaigns and Coverage

See `coverage-ledger.md` and campaigns summary. HTTP auth, state injection, triggers, path traversal, command execution, SSRF, and deserialization campaigns executed.

## 9. Tools and Static Analysis

CodeQL and Semgrep not available on audit host (resource/tooling limitation). Analysis based on manual source review, grep enumeration, and path algebra verification.

## 10. Confirmed Findings

| ID | Severity | Title |
|------|----------|-------|
| JS-001 | Critical | Missing Authentication on ADK API Server |
| JS-002 | High | Client-Controlled stateDelta Enables app:/user: State Escalation |
| JS-003 | High | Insecure Direct Object Reference on Sessions and Artifacts |
| JS-004 | Critical | Unauthenticated Agent Execution Enables Host RCE Chain |
| JS-005 | Medium | Unauthenticated Debug and Telemetry Exposure |
| JS-006 | High | Session Creation Accepts app:/user: State Injection |

Detailed reports in `findings/` directory.

## 11. Unresolved and Low-Priority Leads

See `unresolved-leads.md`. Count: 7 unresolved leads.

## 12. Rejected and Duplicate Leads

See `rejected-leads.md`.

## 13. Exploit-Chain Analysis

State injection chains with unauthenticated API access elevate impact across sessions. RCE chains require agent tool/code-executor configuration. No speculative multi-repo chains reported.

## 14. Overall Assessment

**2 Critical, 3 High, 1 Medium, 0 Low** confirmed findings under completed coverage. Dominant pattern: missing authentication on HTTP control plane combined with client-controlled `stateDelta` accepting `app:`/`user:` scoped keys.

## 15. Coverage Limitations

Semgrep/CodeQL not run. A2A auth and some MCP SSRF paths unresolved. Full transitive dependency CVE reachability not verified.

## 16. Runtime, Resource, and Cleanup Metadata

| Field | Value |
|-------|-------|
| Run ID | bc-47d6367b-32a4-4edb-8960-a63766085caf |
| Worktree | /tmp/audit-20260802T020146Z-8248ebb7/worktrees/adk-js |
| Output | /tmp/audit-20260802T020146Z-8248ebb7/adk-js |
| Host | cloud-agent VM |
