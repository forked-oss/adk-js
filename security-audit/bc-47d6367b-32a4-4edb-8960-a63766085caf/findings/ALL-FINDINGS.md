# Security Findings — adk-js @ 4169319f

## JS-001 — Missing Authentication on ADK API Server

| Field | Value |
|-------|-------|
| **Severity** | Critical |
| **CWE** | CWE-306: Missing Authentication for Critical Function |
| **CVSS 3.1** | 9.8 (`AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`) when bound beyond localhost |
| **Component** | `dev/src/server/adk_api_server.ts`, `dev/src/cli/cli.ts` |

### Description

`AdkApiServer` exposes a full agent control plane over HTTP with no authentication, authorization, or rate limiting. All session, artifact, run, and debug endpoints are reachable by any client that can reach the listening socket.

### Evidence

```191:194:dev/src/server/adk_api_server.ts
    app.use((req: Request, res: Response, next: express.NextFunction) => {
      this.logger.info(`${req.method} ${req.originalUrl}`);
      next();
    });
```

No auth middleware is registered anywhere in `init()`. Endpoints include `/run`, `/run_sse`, session CRUD, artifact CRUD, `/list-apps`, and `/debug/trace/*`.

Default host is `localhost` (`dev/src/cli/cli.ts:105`), but `--host` accepts any binding address (e.g. `0.0.0.0`), making network exposure a one-flag configuration change.

### Trace

1. Attacker sends `POST /run` (or any session/artifact endpoint) with no credentials.
2. Express logging middleware runs, then handler executes.
3. Agent runs, session state mutates, artifacts read/written — all unauthenticated.

### Impact

Full read/write of sessions and artifacts, arbitrary agent invocation, potential host compromise when combined with code-execution tools (see JS-004).

---

## JS-002 — Client-Controlled `stateDelta` Enables `app:` / `user:` State Escalation

| Field | Value |
|-------|-------|
| **Severity** | High |
| **CWE** | CWE-639: Authorization Bypass Through User-Controlled Key |
| **CVSS 3.1** | 8.1 (`AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N`) |
| **Component** | `dev/src/server/adk_api_server.ts`, `core/src/runner/runner.ts`, `core/src/sessions/in_memory_session_service.ts`, `core/src/sessions/database_session_service.ts` |

### Description

The `/run` and `/run_sse` endpoints accept a client-supplied `stateDelta` object and pass it directly into the runner. Keys prefixed with `app:` or `user:` are promoted to application-wide or user-wide persistent state — a privilege level intended for trusted agent/tool code, not HTTP clients.

### Evidence

API accepts unvalidated `stateDelta`:

```707:742:dev/src/server/adk_api_server.ts
    app.post('/run', async (req: Request, res: Response) => {
      const {appName, userId, sessionId, newMessage, stateDelta} = req.body;
      // ...
        for await (const e of runner.runAsync({
          userId,
          sessionId,
          newMessage,
          stateDelta,
```

Runner attaches `stateDelta` to the user event without validation:

```310:318:core/src/runner/runner.ts
            await this.sessionService.appendEvent({
              session,
              event: createEvent({
                invocationId: invocationContext.invocationId,
                author: 'user',
                actions: stateDelta
                  ? createEventActions({stateDelta})
                  : undefined,
```

`InMemorySessionService` promotes prefixed keys to global scopes:

```275:289:core/src/sessions/in_memory_session_service.ts
    if (event.actions && event.actions.stateDelta) {
      for (const key of Object.keys(event.actions.stateDelta)) {
        if (key.startsWith(State.APP_PREFIX)) {
          this.appState[appName] = this.appState[appName] || {};
          this.appState[appName][key.replace(State.APP_PREFIX, '')] =
            event.actions.stateDelta[key];
        }
        if (key.startsWith(State.USER_PREFIX)) {
          // ... updates userState[appName][userId]
```

`DatabaseSessionService` applies the same logic on `appendEvent` (lines 444–467).

### Trace

1. `POST /run` with body `{ "appName":"x", "userId":"victim", "sessionId":"...", "stateDelta": {"app:admin_flag": true, "user:role": "admin"}, "newMessage": {...} }`
2. `runner.runAsync` → `appendEvent` with `stateDelta`
3. Session service writes to `appState` / `userState` tables or in-memory maps
4. All future sessions for the app/user inherit poisoned state

### Impact

Cross-session state poisoning, credential or policy bypass if agents read `app:`/`user:` keys for authorization decisions, persistence across sessions.

---

## JS-003 — Insecure Direct Object Reference on Sessions and Artifacts

| Field | Value |
|-------|-------|
| **Severity** | High |
| **CWE** | CWE-639 / CWE-284: Improper Access Control |
| **CVSS 3.1** | 7.5 (`AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:L/A:N`) |
| **Component** | `dev/src/server/adk_api_server.ts` |

### Description

`userId`, `sessionId`, and `artifactName` are taken from URL paths or request bodies without verifying that the caller owns or is authorized for those identifiers. Combined with JS-001, any network client can enumerate or guess IDs and access other users' data.

### Evidence

Session read uses only path parameters — no ownership check:

```340:359:dev/src/server/adk_api_server.ts
    app.get(
      '/apps/:appName/users/:userId/sessions/:sessionId',
      async (req: Request, res: Response) => {
        const appName = req.params['appName'];
        const userId = req.params['userId'];
        const sessionId = req.params['sessionId'];
        const session = await this.sessionService.getSession({
          appName, userId, sessionId,
        });
        res.json(session);
```

`/run` trusts `userId` and `sessionId` from JSON body with no binding to caller identity.

### Trace

1. Attacker learns or guesses `userId`/`sessionId` (predictable UUIDs, logs, error messages).
2. `GET /apps/myapp/users/victim/sessions/{id}` returns full session including events and merged state.
3. `POST /run` with victim identifiers executes agent in victim's session context.

### Impact

Unauthorized disclosure of conversation history, state, and artifacts; ability to inject messages and state into arbitrary sessions.

---

## JS-004 — Unauthenticated Agent Execution Enables Host RCE Chain

| Field | Value |
|-------|-------|
| **Severity** | Critical |
| **CWE** | CWE-78: Improper Neutralization of Special Elements used in an OS Command ('OS Command Injection') |
| **CVSS 3.1** | 9.8 (`AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`) when API is network-exposed and agent uses local code execution |
| **Component** | `core/src/code_executors/unsafe_local_code_executor.ts`, `core/src/tools/skill/run_skill_inline_script_tool.ts`, `core/src/tools/skill/run_skill_script_tool.ts`, chained via JS-001 |

### Description

`UnsafeLocalCodeExecutor` deliberately executes LLM-supplied code via `child_process.spawn` without sandboxing. `RunSkillInlineScriptTool` passes arbitrary `script_content` to the configured executor. When the API server is reachable (JS-001), an attacker can invoke agents that trigger these tools and obtain arbitrary code execution on the host.

### Evidence

```202:206:core/src/code_executors/unsafe_local_code_executor.ts
        const child = spawn(command, args, {
          timeout: this.timeoutSeconds * 1000,
          killSignal: 'SIGKILL',
          cwd: tempDir,
        });
```

```96:105:core/src/tools/skill/run_skill_inline_script_tool.ts
      const result = await codeExecutor.executeCode({
        invocationContext: toolContext.invocationContext,
        codeExecutionInput: {
          code: inlineScriptContent,
          inputFiles: [],
          language: language as CodeExecutionLanguage,
          args: scriptArgs,
        },
      });
```

Class documentation explicitly warns of no sandboxing (lines 106–107, 145–150 of `unsafe_local_code_executor.ts`).

### Trace

1. Attacker calls unauthenticated `POST /run` (JS-001).
2. Prompt causes agent to call `run_skill_inline_script` with malicious `script_content` (shell/JS/Python).
3. `UnsafeLocalCodeExecutor.spawn` runs attacker code in server process context.
4. Host compromise.

### Impact

Full server/host takeover when agents are configured with `UnsafeLocalCodeExecutor` or equivalent and the API is exposed.

---

## JS-005 — Unauthenticated Debug and Telemetry Exposure

| Field | Value |
|-------|-------|
| **Severity** | Medium |
| **CWE** | CWE-200: Exposure of Sensitive Information to an Unauthorized Actor |
| **CVSS 3.1** | 5.3 (`AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N`) |
| **Component** | `dev/src/server/adk_api_server.ts` |

### Description

Debug endpoints expose OpenTelemetry span data and per-event trace dictionaries without authentication.

### Evidence

```210:228:dev/src/server/adk_api_server.ts
    app.get('/debug/trace/:eventId', (req: Request, res: Response) => {
      const eventId = req.params['eventId'];
      const eventDict = this.traceDict[eventId];
      // ... returns eventDict
    });

    app.get('/debug/trace/session/:sessionId', ...);
```

### Trace

1. Attacker requests `GET /debug/trace/session/{sessionId}`.
2. Server returns span names, attributes, trace IDs for that session.

### Impact

Information disclosure of agent internals, tool arguments, model metadata, and operational telemetry.

---

## JS-006 — Session Creation Accepts `app:` / `user:` State Injection

| Field | Value |
|-------|-------|
| **Severity** | High |
| **CWE** | CWE-639: Authorization Bypass Through User-Controlled Key |
| **CVSS 3.1** | 7.5 (`AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:N`) |
| **Component** | `dev/src/server/adk_api_server.ts`, `core/src/sessions/database_session_service.ts` |

### Description

`POST /apps/:appName/users/:userId/sessions` accepts a `state` object from the client. `DatabaseSessionService.createSession` splits `app:` and `user:` prefixed keys into global app/user state stores — the same escalation primitive as JS-002, reachable at session creation time.

### Evidence

API passes client state to session service:

```430:442:dev/src/server/adk_api_server.ts
    app.post(
      '/apps/:appName/users/:userId/sessions',
      async (req: Request, res: Response) => {
        const state = req.body['state'] || {};
        const createdSession = await this.sessionService.createSession({
          appName, userId, state,
        });
```

Database backend promotes prefixed keys:

```147:164:core/src/sessions/database_session_service.ts
    if (state) {
      for (const [key, value] of Object.entries(state)) {
        if (key.startsWith(State.APP_PREFIX)) {
          appStateDelta[key.replace(State.APP_PREFIX, '')] = value;
        } else if (key.startsWith(State.USER_PREFIX)) {
          userStateDelta[key.replace(State.USER_PREFIX, '')] = value;
```

### Trace

1. `POST /apps/foo/users/bar/sessions` with `{"state": {"app:feature_flags": {"admin": true}}}`
2. `DatabaseSessionService` writes to `StorageAppState`
3. All sessions under app `foo` see poisoned `app:feature_flags`

### Impact

Persistent cross-session state manipulation without running the agent.

---

## Summary Table

| ID | Title | Severity | CVSS |
|----|-------|----------|------|
| JS-001 | Missing Authentication on ADK API Server | Critical | 9.8 |
| JS-002 | Client-Controlled stateDelta Enables app:/user: State Escalation | High | 8.1 |
| JS-003 | IDOR on Sessions and Artifacts | High | 7.5 |
| JS-004 | Unauthenticated Agent Execution Enables Host RCE Chain | Critical | 9.8 |
| JS-005 | Unauthenticated Debug and Telemetry Exposure | Medium | 5.3 |
| JS-006 | Session Creation Accepts app:/user: State Injection | High | 7.5 |
