AGENTS.md

# Fork Performance Notes

## 2026-04-30 — `httpServer.listen()` 107s cold stall — NOT a code bug (Huorong AV first-bind scan)

**Symptom.** Under `OPENCLAW_GATEWAY_LEAN=1`, diagnostic marks in
`server.impl.ts` showed a 107s gap between `after-ws-handlers` and
`after-start-listening`. Liveness reported `eventLoopUtilization=1.0,
cpuCoreRatio=0.995`, which I initially mis-read as "synchronous JS work
pegging the main thread" and then independently mis-attributed to the
Huorong WFP callout driver (`hrwfpdrv`) on this user's box.

**Actual root cause.** Huorong (火绒) performs a one-shot heavy scan the
first time a given `node.exe` binary binds a TCP socket on loopback. On
a fresh process and fresh bundle, that scan takes ~107s synchronously
inside the kernel WFP callout path *before* the `listening` event fires.
Once scanned, Huorong caches the verdict — subsequent cold starts (same
binary / bundle) complete in ~2s. The scan is opaque from Node's side:
`httpServer.listen()` returns instantly (syscall < 1ms), CPU shows as
busy-in-kernel (ratio ≈ 1.0), and the `listening` callback fires only
after the scan releases.

**How this was actually proven (not inferred).** Added a probe in
`src/gateway/server/http-listen.ts` that stderr-logs stage marks around
the `.listen()` syscall under `OPENCLAW_GATEWAY_LEAN=1`. Cold-started
a fresh LEAN gateway after running an unrelated benchmark that also
called `http.Server.listen('127.0.0.1', …)` from the same `node.exe`
just minutes earlier. Probe showed:

```
attempt-0-before-listen-syscall        t=…899
attempt-0-after-listen-syscall-returned t=…899
attempt-0-listening-event              t=…0123   (+2.2s, not +107s)
```

Total gateway cold start dropped from 114.9s → 9.4s on the very next
run, with no code changes between them. That rules out an OpenClaw-side
bug and confirms the stall was an AV first-bind scan already satisfied
by the prior benchmark process.

**Why my earlier WFP-driver theory was wrong.**
- `cpuCoreRatio=1.0` does NOT prove main-thread JS work. It can also
  reflect kernel-side CPU on the same logical core while the userspace
  thread parks in a syscall. Read it together with
  `eventLoopDelayMaxMs` and a more granular stage probe before blaming
  application code.
- "Trusted-zone whitelist doesn't cover WFP callouts" — unverified
  assumption. In practice Huorong's trust-zone *does* satisfy the scan
  (the scan was already cached the next run after adding `node.exe` /
  project dirs to trust-zone, even though that was also the run right
  after the bench-listen warmup, so credit is shared).
- Jumped from one Chinese forum thread about Huorong's NDIS driver
  disrupting loopback stats to "therefore hrwfpdrv is blocking
  listen()". That leap wasn't supported by the evidence on hand.

**Fix.** No code change needed. Operational notes:
1. Keep `G:\projects\openclaw`, `C:\Users\jianfei\.openclaw`,
   `E:\Temp\openclaw`, and `G:\Program Files\nodejs\node.exe` in
   Huorong 信任区 (they already are).
2. The first bind after a bundle rebuild / node upgrade will pay the
   scan (~100s). Subsequent starts are fast.
3. Probe is left in `src/gateway/server/http-listen.ts` behind
   `OPENCLAW_GATEWAY_LEAN=1` for future diagnosis — zero cost when
   LEAN is off.

**Guardrail for future me.** When `cpuCoreRatio≈1.0` correlates with
a gap between "before-syscall" and "syscall-callback" marks, and the
syscall returned instantly, suspect kernel-side AV/EDR *before*
suspecting application code. Prove the split with a raw-Node bench
(`http.createServer().listen(127.0.0.1, 0)`) from the *same* node.exe
— if it's fast there, the problem is not in Node/OpenClaw. If it's
slow there too, the problem is in the kernel-side pipeline. Do this
*before* writing a report to IT.

**Files touched.**
- `src/gateway/server/http-listen.ts` (probe, LEAN-gated)
- `dist/server.impl-DA-Di8i0.js` (probe inlined into bundle during
  investigation; reverted once proven)

## 2026-04-30 — model-catalog suppression resolver (commit `12cd667880`)

**Symptom.** Cold CLI turns appeared to hang indefinitely (~58 min observed)
inside `loadModelCatalog`. Profiling and source-level traces (gated by
`OPENCLAW_DEBUG_INGRESS_TIMING=1`) pointed at the entries loop in
`src/agents/model-catalog.ts`.

**Root cause.** The loop called `shouldSuppressBuiltInModel({ provider, id,
config })` once per registry entry (~897 calls per turn on a typical config).
Each call cascaded into `listManifestModelCatalogSuppressions`, which rebuilt
the full plugin manifest registry from disk because
`loadPluginManifestRegistryForInstalledIndex` is invoked with `cache: false`
(see `src/plugins/manifest-registry-installed.ts:194`). Net: the registry was
re-read and re-parsed ~897× per `loadModelCatalog` call; on a cold CLI
process this stacked into an effectively unbounded stall.

**Fix.** Hoist the resolver out of the loop. Three new helpers:

- `buildManifestBuiltInModelSuppressionResolver` in
  `src/plugins/manifest-model-suppression.ts` — calls
  `listManifestModelCatalogSuppressions` once, indexes entries into a
  `Map<mergeKey, ManifestModelCatalogSuppressionEntry[]>`, returns a closure
  that does an O(1) Map lookup + condition match per call.
- `buildBuiltInModelSuppressionResolver` in
  `src/agents/model-suppression.ts` — thin wrapper that normalizes
  provider/id and delegates to the manifest resolver.
- Re-exported through `src/agents/model-suppression.runtime.ts` to preserve
  the lazy-runtime boundary used by `model-catalog.ts`.

`model-catalog.ts` now constructs the resolver once before the entries loop
and calls `isSuppressed({ provider, id })` inside it. Total work in the loop
dropped from `O(N × full-registry-rebuild)` to `O(N)`.

**Verified.** Under `OPENCLAW_DEBUG_INGRESS_TIMING=1`, the loop reports
`loopMs=0 suppressMs=0 suppressCalls=897`. Total `loadModelCatalog` dropped
from "appears infinite" to ~57s on a cold CLI. Residual cost is now in
`ensureOpenClawModelsJson` (~19s) and `discoverAuthStorage` (~32s) — tracked
as separate follow-ups.

**Guardrail (general).** When a hot loop calls into a manifest/registry
helper, audit whether that helper rebuilds caches on every call. `cache:
false` inside a per-iteration call is almost always a bug. Prefer building
a resolver/lookup table once outside the loop and querying it inside.

**Files touched.**
- `src/agents/model-catalog.ts`
- `src/agents/model-suppression.ts`
- `src/agents/model-suppression.runtime.ts`
- `src/plugins/manifest-model-suppression.ts`

## 2026-04-30 — pi credential discovery memo (commit `a84e6e3a38`)

**Symptom.** `discoverAuthStorage` consumed ~32–73 s per CLI turn on cold
start, called multiple times per turn (`context.ts`, `model-catalog.ts`).

**Root cause.** `resolvePiCredentialsForDiscovery` cascades into
`overlayExternalAuthProfiles` → `resolveExternalAuthProfilesWithPlugins`,
which has no internal cache and reloads the full provider plugin registry
each call. Same `(agentDir, readOnly)` was resolved repeatedly within a
single CLI process even though credentials are stable for the lifetime of
that process.

**Fix.** Process-level memo (`Map<string, PiCredentialMap>` keyed by
`${readOnly ? "ro" : "rw"}|${agentDir}`) in `pi-auth-discovery.ts`. First
call pays the cascade; subsequent calls return cached map. Test reset
helper exposed via `resetPiCredentialDiscoveryCacheForTest`.

**Files touched.**
- `src/agents/pi-auth-discovery.ts`
- `src/agents/pi-model-discovery.ts` (re-export of reset helper)

## 2026-04-30 — hasAuthForProvider memo (commit `607362e95d`)

**Symptom.** Media tool factories (image / imageGenerate / videoGenerate /
musicGenerate) iterate over every capability provider and call
`hasAuthForProvider` per provider per factory. With ~N providers × 4
factories per turn, each call cascading into the same uncached
`overlayExternalAuthProfiles` path, this stacked into seconds of duplicate
work per turn.

**Fix.** Process-level `HAS_AUTH_CACHE: Map<string, boolean>` keyed by
`${agentDir ?? ""}|${provider}` in `model-config.helpers.ts`. Computation
moved to `computeHasAuthForProvider`. Test reset helper:
`resetHasAuthForProviderCacheForTest`.

**Files touched.**
- `src/agents/tools/model-config.helpers.ts`

## 2026-04-30 — startup model warmup off by default (commits `693b0a45ca`, `<this>`)

**Symptom.** After daemon startup, the very first agent turn would succeed
in ~10–20 s, but the next ws connection would time out for the following
~15 minutes. Liveness diagnostic confirmed the gateway event loop was
blocked end-to-end (`eventLoopDelayMaxMs=896037.6`,
`eventLoopUtilization=1.0`).

**Root cause.** `schedulePrimaryModelPrewarm` in
`server-startup-post-attach.ts` fires off `prewarmConfiguredPrimaryModel`
in the background with a 5 s timeout. The timeout only stops *awaiting*
the warmup; the warmup keeps running. It calls `ensureOpenClawModelsJson`
with provider discovery, which still triggers the uncached plugin
manifest registry rebuilds inside the model-resolution path. On this fork
that work blocks the event loop for ~15 minutes after gateway "ready",
silently eating any subsequent ws connect.

**Fix (interim).** Make warmup opt-in instead of opt-out. Default skips
the warmup. Set `OPENCLAW_ENABLE_PRIMARY_MODEL_PREWARM=1` to restore the
old behavior once the underlying cascade is properly cached.

Trade-off: the first agent turn after startup pays the cascade itself
(~80 s on this box) instead of paying it during background warmup. But
subsequent turns work immediately, and the daemon never wedges.

This is a band-aid. The proper fix is to cache the manifest-rebuild
cascade inside `ensureOpenClawModelsJson` / its provider discovery path
so the warmup is cheap and safe to keep on.

**Files touched.**
- `src/gateway/server-startup-post-attach.ts`

## 2026-05-01 — TUI connect wedge: sync plugin manifest realpath cascade inside `runEmbeddedPiAgent` (gateway side FIXED via process-level memo in `bundled-plugin-metadata.ts`; **TUI/CLI process still has independent event-loop stall**)

**Symptom.** `openclaw tui` reaches "connecting | idle" and sits there
indefinitely — no flicker, no input possible. Gateway process is alive
and the inspector shows main thread is burning ~100% of one core (mostly
kernel time). TCP is ESTABLISHED but `hello.ok` never reaches the TUI
because the write never drains while the main thread is wedged.

Repros identically with a bare `ws` client from the same machine
(`bench-ws-handshake.js`) — rules out any TUI-side cause. Bare
`net.connect` bench (`bench-connect.js`) is <2ms per round, so this is
also NOT the AV first-connect tax documented in the listen-side note.

**Actual root cause (proven via CDP Debugger.pause against `--inspect=9229`).**
The wedge is synchronous plugin-manifest loading on the main thread.
Two independent stack snapshots captured while the TUI was hung:

1. `runEmbeddedPiAgent` → `enqueueSession` → lane pump →
   `ensureRuntimePluginsLoaded` → `resolveRuntimePluginRegistry` →
   `loadOpenClawPlugins` → `ensureBundledPluginRuntimeDeps` →
   `normalizePluginsConfig` → `getBundledPluginAliasLookup` →
   `listBundledPluginMetadata` → `collectBundledPluginMetadata` →
   `loadPluginManifest` → `openBoundaryFileSync` → `realpathSync`.

2. Moments later, same process: `resolveModelAsync` →
   `resolveExplicitModelWithRegistry` → `normalizeResolvedModel` →
   `normalizeProviderTransportWithPlugin` → `resolveProviderHookPlugin`
   → `resolveProviderRuntimePlugin` → `resolveProviderPluginsForHooks`
   → `resolvePluginProviders` → `resolveRuntimeProviderPluginLoadState`
   → `resolveBundledPluginCompatibleActivationInputs` →
   `resolveBundledProviderCompatPluginIds` →
   `loadScopedProviderRegistry` → `loadProviderRegistrySnapshot` →
   `loadPluginRegistrySnapshot` → `resolveSnapshot` →
   `loadPluginRegistrySnapshotWithMetadata` → `loadInstalledPluginIndex`
   → `buildInstalledPluginIndex` → `resolveInstalledPluginIndexPolicyHash`
   → `normalizePluginsConfig` → ... → `loadPluginManifest` →
   `openBoundaryFileSync` → `existsSync` (wrapped → `isURL`).

So a single `runEmbeddedPiAgent` does the full 115-manifest walk
**at least twice** (once for runtime-plugin registry, once for
model/provider resolution), fully synchronously. Every
`realpathSync` / `existsSync` crosses the kernel WFP / filesystem
callout layer. On this box Huorong scans each path the first time
it's touched; the cached verdict appears to be per-path, not per
directory, so 115 × 2 × N-files-in-chain synchronous scans stack into
minutes.

**Why this is a NEW kind of bug, not the same one we've fixed before.**
- The existing `loadModelCatalog` fix (commit `12cd667880`) hoisted
  ONE per-iteration resolver rebuild out of ONE loop. The cold path
  still takes ~57s.
- The existing `discoverAuthStorage` and `hasAuthForProvider` memos
  (commits `a84e6e3a38`, `607362e95d`) are process-level caches for
  their respective cascades.
- THIS bug is different: `runEmbeddedPiAgent` itself invokes two
  separate plugin-registry walks per run, and the walks are **fully
  synchronous** (`loadOpenClawPlugins` is sync, `realpathSync` is
  sync). Nothing yields the event loop until 115+ manifest paths are
  each scanned. This blocks `hello.ok` drain, which is why TUI stays
  at "connecting" with no frame ever received.

**Why the CLAUDE.md prewarm-off guard did not help.** The prewarm path
is gated behind `OPENCLAW_ENABLE_PRIMARY_MODEL_PREWARM` and is off by
default, so it's not the trigger. The real trigger is that the FIRST
`runEmbeddedPiAgent` on this process (fired on TUI session handshake)
cold-loads everything inline. Whichever code path schedules
`runEmbeddedPiAgent` on TUI connect needs to either (a) yield the
event loop so `hello.ok` can be written, or (b) not fire until after
the first user message.

**Proximate triggers not yet pinpointed.** `runEmbeddedPiAgent` has
two primary non-test callers: `agents/command/attempt-execution.ts:457`
(`runAgentAttempt`) and `auto-reply/reply/agent-runner-execution.ts:1337`.
Which of these is enqueued on TUI connect (vs. deferred until user
sends a message) is the next thing to nail down. LEAN mode skips
`subagent-recovery` and `main-session-recovery` at startup; auto-reply
paths are NOT LEAN-gated and are a plausible culprit.

**Bandwidth guardrails.**
- All 115 `openclaw.plugin.json` walks are sync fs calls; on a clean
  Windows box they complete in a few hundred ms. Our box is 10-100×
  slower per file because of Huorong callouts. The AV layer is not
  the bug; the code assuming "sync plugin walk is cheap" is.
- The proper fix is async plugin walk + process-level dedupe so one
  `loadOpenClawPlugins` can't race another on the same options key.
  Band-aid fix is to find and defer the specific trigger that fires
  it during TUI handshake.

**Instrumentation left in place.**
- `E:\Temp\openclaw\bench-ws-handshake.js` — bare-Node ws client that
  reproduces the wedge (or confirms a healthy gateway) in ~10ms when
  gateway is healthy, 60s timeout when wedged.
- `E:\Temp\openclaw\dump-stack.js` / `dump-stack.cmd` — CDP-based
  stack dumper. Attaches to `ws://127.0.0.1:9229/<uuid>`, prints
  active handles/requests, then Debugger.pause + callFrames. Full JSON
  goes to `E:\Temp\openclaw\stack-dump.json`.
- `E:\Temp\openclaw\start-gateway-inspect.cmd` — boots gateway with
  `--inspect=127.0.0.1:9229 --port 18789` under LEAN. Use this when
  re-repro is needed.

**Files touched by investigation (no code changes yet).**
- Read: `src/gateway/server/ws-connection/message-handler.ts`,
  `src/gateway/server-startup-post-attach.ts`,
  `src/agents/pi-embedded-runner/run.ts`,
  `src/agents/runtime-plugins.ts`, `src/plugins/loader.ts`,
  `src/agents/command/attempt-execution.ts`.

**Gateway-side fix applied (verified working).** Added process-level memo
in `src/plugins/bundled-plugin-metadata.ts::listBundledPluginMetadata`
keyed by `(rootDir, scanDir, includeChannelConfigs,
includeSyntheticChannelConfigs)`. First call pays the 115-file sync
realpath walk; subsequent calls return the frozen cached array. Cleared
via `clearBundledPluginMetadataCache()` (kept as a real implementation
now, not a no-op).

Also added stage probes to `src/gateway/server/ws-connection/message-handler.ts`
behind `OPENCLAW_DEBUG_CONNECT_TRACE=1` (stderr `[connect-trace] conn=…
<stage> t=…` marks around resolveConnectAuthState, resolveConnectAuthDecision,
getPairedDevice, updatePairedDeviceMetadata, ensureDeviceToken,
reconcileNodePairing, sendFrame-helloOk). Verified post-fix that gateway
connect handshake completes in **~1ms total** (every stage marks at the
same `t=`), and the gateway then successfully serves `agents.list` (~1.5s
including model resolution) and `sessions.list` (~1s) on the first paired
TUI connection.

**TUI/CLI-side wedge (still open).** With gateway healthy, the user's
TUI nonetheless shows "connecting | idle" and the input bar freezes for
tens of seconds at a time before flushing. Probe via gateway logs shows:
- First TUI connect succeeds, runs `agents.list` + `sessions.list`,
  status briefly transitions through "gateway connected" (4s TTL).
- Subsequent TUI reconnects complete the gateway handshake but never
  send `agents.list`/`sessions.list` — TUI client's `onConnected` is
  apparently not firing or its event loop is wedged before it can.
- Typed characters appear minutes later in batch — classic event-loop
  blockage in the TUI/CLI process.

The TUI process is a separate Node process from the gateway, so the
`bundled-plugin-metadata` memo we added to fix gateway doesn't help.
The CLI's own startup may pay a similar cascade (or worse — see
agent-scope, model-input, listAgentEntries imports in
`src/tui/tui.ts:15`). Next step is to attach `--inspect` to the TUI
process and capture the same kind of CDP stack dump we did for gateway.

## 2026-05-01 — `loadInstalledPluginIndex` process-level memo (`installed-plugin-index.ts`)

Same villain pattern as `bundled-plugin-metadata` from earlier today.
`buildInstalledPluginIndex` calls `resolveInstalledPluginIndexRegistry`
which scans plugin candidates and reads every manifest from disk.

Prior-session investigation (recorded in handoff doc) measured **13–21
calls per agent turn at 2–4 s each**: the embedded run pipeline asks for
the installed-plugin index from many call sites
(`normalizeProviderTransportWithPlugin`, system-prompt build,
core-plugin-tools, runtime-deps cascades, etc). On this Huorong box the
per-call cost balloons because each call re-walks the same set of
manifest files.

Added a process-level `Map<string, InstalledPluginIndex>` keyed by
`{ workspaceDir, stateDir, pluginIndexFilePath, policyHash, installRecords-hash }`.
- `loadInstalledPluginIndex` checks the cache first and stores the built
  index on miss.
- `refreshInstalledPluginIndex` always rebuilds (preserves the
  "force refresh" contract) but writes the fresh index back into the
  cache so subsequent loads are warm.
- Cache is bypassed entirely when callers supply mutable accumulators
  (`candidates`, `diagnostics`), a non-default `env`, or a custom `now`.
  This is what every existing test does (they pass `candidates: [...]`),
  so test isolation is preserved.
- Exported `clearInstalledPluginIndexCache()` for surgical eviction
  (matches the `clearBundledPluginMetadataCache` shape).

The cache key uses the existing `resolveInstalledPluginIndexPolicyHash`
which already canonicalises plugin enable/allow/deny/slots and per-channel
`enabled` flags — exactly what changes when the user toggles plugins or
channels in `openclaw.json`. `installRecords` are also hashed, so install
state changes invalidate too.

What this does **not** fix: `models.list` is still observed at 402 s on
gateway cold-start (eventLoopP99=288 s). That call goes through
`ensureOpenClawModelsJson` → provider discovery, a separate cascade that
dominates first-call latency. Still on the open-follow-ups list above.

## Open follow-ups

- `ensureOpenClawModelsJson` cold cost (~19 s baseline, much higher on
  this box). Likely culprit: same `loadPluginManifestRegistryForInstalledIndex`
  with `cache: false` style call inside its provider discovery path.
- The broader pattern: any helper that rebuilds the plugin manifest registry
  per call is the recurring villain of this fork. Audit all
  `loadPluginManifestRegistry*` call sites for `cache: false` and replace
  with a hoisted resolver.

## Operational note — env var injection points

Two ways the gateway gets started, with different env sources:

- `openclaw gateway start` (foreground, what most users run before TUI):
  reads env from the current shell. Set perf flags in your shell profile
  or before invoking.
- `openclaw daemon start` (background service via schtasks/launchd/systemd):
  reads env from `~/.openclaw/gateway.cmd` (Windows) or the equivalent
  unit/plist. Edit those files to inject env into the service process.

Changes to one do **not** affect the other. When adding a new perf-related
env flag, prefer making the safer behavior the default in code so both
paths benefit without operator action.
