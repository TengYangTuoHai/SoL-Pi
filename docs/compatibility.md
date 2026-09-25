# Pi Compatibility

SoL-Pi is developed and tested against `@earendil-works/pi-coding-agent` 0.87.1. The Online Context Compact feasibility check mirrors the projected compaction preparation introduced in Pi 0.87 (see below); it was differentially verified against Pi 0.87.1's `prepareCompaction` over recorded sessions and randomized branches. The current 19 test files (160 tests), type checking, package inspection, public API checks, and offline extension startup all pass on that release. Older Pi releases remain loadable through the `*` peer dependency, but only the tested release is a compatibility guarantee: pre-0.87 runtimes use the conservative branch of the compaction feasibility check, and mechanisms that need newer APIs (for example dynamic tool activation) fall back to their eager behavior. Previous checks covered the public API surface of Pi 0.81.1, the base used by the original Pi fork; they are not a current full-suite compatibility guarantee.

SoL-Pi imports only public package exports:

- `createEditToolDefinition`
- `createWriteToolDefinition`
- `createBashToolDefinition`
- extension types and `ExtensionAPI.registerTool`
- `context`, `before_provider_request`, `tool_result`, `turn_end`, `agent_settled`, and `session_before_tree` extension events
- native compaction events, `ExtensionContext.getContextUsage()`, and `ExtensionContext.compact()`
- `ExtensionContext.model` and `ExtensionContext.modelRegistry`
- the public session-manager methods exposed through `ExtensionContext`

## Action Fusion

The built-in edit/write definitions capture their working directory, so SoL-Pi caches one definition per `ctx.cwd`. Its own per-file queue surrounds the built-in mutation and follow-up command. It does not nest Pi's built-in mutation queue.

Action Fusion decodes `file://` targets with Node's `fileURLToPath()` before resolving the queue and hash-check path. This keeps file URLs, including percent-encoded filenames and Pi's optional `@` prefix, aligned with the file handled by the built-in mutation tool.

On Windows it applies the same drive-path conversion Pi's own resolver applies, so Git Bash, MSYS, Cygwin, and WSL targets such as `/c/src/app.ts` and home-relative `~\` paths resolve to the file the built-in mutation tool wrote. On other platforms those inputs keep their POSIX meaning.

The queue covers only fused operations registered by this SoL-Pi instance. External processes, direct built-in-tool calls outside the replacement, and unrelated extensions are not globally locked. SoL-Pi hashes the target immediately before launching `then_run` and skips the command if it observes an intervening content change.

## ObservationPack

ObservationPack changes only the messages projected through the public `context` event. Stored session history remains intact. Original bytes and the JSONL ledger live under the session-derived SoL-Pi directory.

On Pi 0.86.1 and newer, `obs_recall` uses dynamic tool activation: it stays out of the prompt on sessions with nothing to recall, is activated one provider request before the first placeholder would appear, and stays active when a resumed session's ledger already recorded placeholders. Activation failures fail open to the eagerly available tool, and older runtimes without `getAllTools`/`getActiveTools`/`setActiveTools` keep the eager behavior.

## Evidence-Preserving Reducer

The reducer calls the configured model through the public `ExtensionContext.modelRegistry`: `find()` resolves the reducer model and `complete()` performs the call with Pi-managed authentication. On runtimes whose registry does not expose `complete()`, the reducer reports the model as unavailable and preserves the original result. The reducer preserves the original result whenever the configured reducer model is unavailable or eligibility, model-call, schema, source-hash, exact-quote, size, or likely-secret checks fail.

All persistent paths use `SessionManager.getSessionDir()` and `getSessionId()`, which are public on the tested release. SoL-Pi creates no configurable storage-path surface.

The unpublished shared artifact layout is not read or migrated. Each session starts from its own `<sessionDir>/sol-pi/<sessionId>/` directory.

## Online Context Compact

Online Context Compact uses ordinary public `context` and `before_provider_request` handlers instead of fork-only post-transform observer methods. Public handlers run in extension load order, so the SoL-Pi entrypoint registers Online Context Compact after its other context transformers. A third-party transformer loaded later is outside the context-growth observation used by its estimate.

Before aborting a turn for a boundary compaction, the extension predicts whether Pi's native `AgentSession.compact()` will accept the session. Pi 0.87 changed that preparation from raw-entry cut-point counting to a projected window that filters system prompt state, drops interior compaction entries, and applies context-edit omissions. The check mirrors the projected semantics through the public `buildContextEntries`, `sessionEntryToContextMessages`, and `estimateTokens` exports, so it stays a prediction on both the pre-0.87 and 0.87+ algorithms and never aborts a turn for a compaction Pi would refuse with "Nothing to compact (session too small)". If a refusal still occurs (for example after an unrecognized future Pi change), the extension treats it as a benign skip, resumes the parent task once, and stops resuming on a repeated skip instead of surfacing an extension error.

Pi does not expose the effective retained-tail compaction budget through the public extension context. The standalone extension therefore uses Pi's default of 20,000 tokens for its economic estimate. Its programmatic factory accepts an explicit matching value for a non-default Pi setting. Pi 0.86 added per-model compaction budgets (`compaction.keepRecentTokens` and `compaction.modelOverrides`); a non-default budget can make the predicted cut point differ from the executed one, in which case the extension treats the refused compaction as a benign skip instead of breaking the turn.

`ExtensionContext.compact()` aborts the active agent before it summarizes, and `agent_settled` fires only once a whole run has drained every turn, retry, auto-compaction, and queued continuation. A plan boundary that selects compaction therefore saves its plan and progress state, calls `ExtensionContext.abort()` to stop the run, and runs compaction from the `agent_settled` that stop produces. The handler awaits the compaction's own `onComplete`/`onError` callbacks. On success, the extension sends a hidden reminder through public `ExtensionAPI.sendMessage()` with `triggerTurn: true`, so Pi starts a new turn against the compacted context and rebuilds the plan even when the native summary omits that instruction.

A settlement barrier keeps the original `agent_settled` dispatch open until the triggered continuation settles. Print- and JSON-mode processes therefore complete the compact-and-continue sequence within the same Pi invocation; an outer driver does not need to resume the session or send `Continue working`. This continuation is armed only by a successful boundary compaction. Cancelling or exiting does not schedule one. A Pi build that never emits `agent_settled` starts no boundary compaction.

`ExtensionAPI.sendMessage()` returns `void`. The barrier is therefore verified for standalone SoL-Pi and depends on Pi starting the requested turn synchronously. A later-loaded third-party extension that performs long asynchronous work in its own `agent_settled` handler is outside this guarantee and needs an integration test with that extension set.

Pi reports the session as idle while an extension-requested manual compaction is running. SoL-Pi cancels `session_before_tree` during that interval to prevent tree navigation from moving the active leaf underneath the compaction. Navigation works normally after the compaction callback settles.

Online Context Compact reads `ExtensionContext.getContextUsage()` for both the context window and the provider-counted context size. When Pi reports no size — as it does between a compaction and the next answered request — the boundary falls back to its own estimate.

The standalone entry passes `cacheWriteReadRatio` from `sol-pi.json` directly into Online Context Compact's economic check. It does not inspect model price metadata. Changing models during a session does not change the ratio; users who want a different decision policy update the configuration and start a new session.

## Interactive TUI

The lightning savings treatment uses Pi's public `renderCall`,
`renderResult`, `ctx.ui.notify()`, and keyed `ctx.ui.setStatus()` APIs. It checks
`ctx.mode === "tui"` rather than `ctx.hasUI`, because RPC mode also reports UI
support. The renderer therefore changes only the interactive terminal display;
it does not change session messages, provider requests, tool results, JSON
events, print output, or RPC UI requests.

## Test doubles

The test suite drives every extension through the same public `ExtensionAPI` and `ExtensionContext` surface Pi provides, over a real public `SessionManager`, without calling a remote model provider. That keeps the suite zero-spend and independent of the deleted Pi monorepo test harness. Suites that need a genuine session tree — branch order, compaction entries, custom entries, resume — use `SessionManager.inMemory()` or `SessionManager.create()` rather than reimplementing them.

`tests/pi-package-integration.test.ts` loads the actual TypeScript entrypoint through Pi's `DefaultResourceLoader`, reads a trusted all-enabled project configuration, and executes a fused write/command and a plan update in a real `AgentSession`. `tests/online-context-compact-agent-session.test.ts` verifies one and two consecutive native compactions and waits for automatic continuation before the original prompt returns. These integration tests use Pi's deterministic faux provider; they verify runtime compatibility, not live provider authentication or token savings.

The 0.84.2 backward-compatibility run used an isolated copy of the current source and tests, separate dependencies, and an empty Pi agent directory. Only the copy's four Pi development dependency versions, lockfile, and installation-guide version mentions changed. No source or test changes were needed. The run included all four mechanisms and the native compaction/continuation integration tests; it did not repeat live-provider benchmarks on 0.84.2.

The 0.87.1 adoption used the same isolated-copy procedure first: 159 of 160 tests passed on the untouched source, the only failure being the installation-guide version-consistency guard, and the projected compaction feasibility divergence was fixed and differentially verified before the tested release was moved. The follow-up enhancements (registry-only reducer calls, on-demand `obs_recall` activation) were then developed and validated directly against the new baseline.
