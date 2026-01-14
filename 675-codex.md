Goal

  - Programmatically retrieve the same underlying “status” data that /status displays: effective config + session/thread metadata + token usage + account rate limits/credits.

  Non-goals

  - Reproducing Codex TUI rendering/layout.
  - Calling a /status HTTP endpoint (none exists).

  Primary Interface: codex app-server (recommended)

  - Transport: spawn codex app-server; communicate via JSON-RPC 2.0–shaped messages over newline-delimited JSON on stdin/stdout (the "jsonrpc":"2.0" field is omitted).
  - Files to read for protocol: codex-rs/app-server/README.md:18, codex-rs/app-server-protocol/src/protocol/common.rs:488.

  Handshake

  1. Client → Server request:
      - {"id": <int>, "method": "initialize", "params": {"clientInfo": {"name": "...","title":"...","version":"..."}}}
  2. Server → Client response:
      - {"id": <same>, "result": {"userAgent": "<string>"}} or {"id":..., "error": {...}}
  3. Client → Server notification:
      - {"method":"initialized"}

  Data Model (“Status” fields)

  - config: effective config values and origin metadata.
  - session/thread: thread id, cwd, model/provider, approval policy, sandbox policy, cli version, path to rollout.
  - token usage: last + total token usage plus model context window.
  - account rate limits: primary/secondary windows + credits + plan type.

  Retrieval API (request/response)

  1. Effective config
      - Request: {"id":1,"method":"config/read","params":{"includeLayers":false}}
      - Response: {"id":1,"result":{"config":{...},"origins":{...},"layers":null|[...]}}
      - Protocol types: ConfigReadParams/ConfigReadResponse in codex-rs/app-server-protocol/src/protocol/v2.rs:441.
  2. Account rate limits (requires ChatGPT auth)
      - Request: {"id":2,"method":"account/rateLimits/read"}
      - Response: {"id":2,"result":{"rateLimits":{...}}} or error if not authenticated / wrong auth mode.
      - Protocol: GetAccountRateLimits in codex-rs/app-server-protocol/src/protocol/common.rs:184.
  3. Thread/session metadata
      - Request: {"id":3,"method":"thread/start","params":{}}
          - Optional params allow overriding model/cwd/sandbox/approval.
      - Response includes the thread plus the resolved session settings:
          - {"id":3,"result":{"thread":{...},"model":"...","modelProvider":"...","cwd":"...","approvalPolicy":"...","sandbox":{...},"reasoningEffort":...}}
      - Protocol types: ThreadStartParams/ThreadStartResponse in codex-rs/app-server-protocol/src/protocol/v2.rs:1015.

  Streaming Updates (notifications)
  To mirror “live” status (token usage + limits), listen to stdout for notifications after you’ve started a thread and/or turn:

  - Token usage updates:
      - Method: thread/tokenUsage/updated
      - Payload: { "threadId": "...", "turnId": "...", "tokenUsage": { total, last, modelContextWindow } }
      - Defined in codex-rs/app-server-protocol/src/protocol/common.rs:544 and codex-rs/app-server-protocol/src/protocol/v2.rs:1345.
  - Account rate limit updates:
      - Method: account/rateLimits/updated
      - Payload: { "rateLimits": { primary, secondary, credits, planType } }
      - Defined in codex-rs/app-server-protocol/src/protocol/common.rs:544 and codex-rs/app-server-protocol/src/protocol/v2.rs:2024.

  How to force token usage to exist

  - Token usage is emitted as part of running a turn (it’s not just a static read in the protocol).
  - Start a minimal turn:
      - Request: {"id":4,"method":"turn/start","params":{"threadId":"<id>","input":[{"type":"text","text":"..."}]}}
      - Then wait for thread/tokenUsage/updated and/or turn/completed.
      - Protocol: TurnStartParams in codex-rs/app-server-protocol/src/protocol/v2.rs:1451.

  Error handling requirements

  - Any request may return {"error":{code,message,data}}; your client must treat this as failure for that field.
  - account/rateLimits/read can fail if:
      - No cached auth available
      - Auth mode isn’t ChatGPT
      - Backend fetch fails (network/auth)
      - Server implementation: codex-rs/app-server/src/codex_message_processor.rs:1101.

  Minimal “Status” algorithm

  1. Spawn codex app-server.
  2. Perform initialize + initialized handshake.
  3. Call config/read.
  4. Call account/rateLimits/read (optional; only if authenticated for ChatGPT).
  5. Call thread/start to get thread metadata + resolved runtime settings.
  6. (Optional but recommended) Call turn/start with a tiny prompt; wait for:
      - thread/tokenUsage/updated (token usage/context window)
      - account/rateLimits/updated (limits snapshot)
      - turn/completed (turn lifecycle confirmation)
  7. Aggregate into one “status object” and return/print.

  Related: auth refresh automation

  - The Codex auth refresher (`backlog/675-codex-refresher.md`) uses the same `codex app-server` transport and handshake.
  - Refresh is triggered by calling `account/read` with `{"refreshToken": true}`, then reading the updated `$CODEX_HOME/auth.json` from disk.

  Related: historical monitoring (rate limits/credits)

  - The Codex monitor (`backlog/675-codex-monitor.md`) builds on this same retrieval API to export rate-limit windows as Prometheus metrics for historical storage + alerting.

  Secondary (lower-level) option

  - If you only need rate limits and have ChatGPT auth, Codex internally fetches them from a backend usage endpoint via BackendClient::get_rate_limits() (codex-rs/backend-
    client/src/client.rs:161), but app-server is the supported programmatic surface.
