English | [日本語](README.ja.md) | [简体中文](README.zh-CN.md)

# Live Compact

As a conversation grows, read logs, command runs, and tool outputs pile up. Live Compact checks at the end of each turn and locally compacts long outputs that can be shortened safely—without calling a model. Only the selected span is replaced, leaving everything before the cut point byte-for-byte intact. Original content is backed up before compaction; if the backup fails, the original span is kept untouched.

Current stable release: [v1.1.14](https://github.com/Tranquillitatis-Lab/dsh-live-compact/releases/tag/v1.1.14). See [CHANGELOG.md](CHANGELOG.md) for version changes.

## Benchmarks

Full data, raw methodology, and limitations are in [docs/benchmarks.md](docs/benchmarks.md), and structured figures are in [docs/benchmark-results.json](docs/benchmark-results.json).

- **1.0.5 recall comparison**: On the same real DeepSeek session, both stock and Live Compact scored 15/15 on 15 recall questions; on mid-log detail questions, Live Compact scored 3/3 twice, whereas stock scored 1/3 and 3/3.
- **Historical DSH long chains**: Earlier 150 short tasks and 20 continuous WebUniverse tasks retain stock vs. Live Compact comparison tables; these are historical runs, not 1.1.12 acceptance benchmarks.
- **OpenCode port reference**: The independent project [opencode-live-compact](https://github.com/Haoming9527/opencode-live-compact) ported this plugin to OpenCode and compared it against OpenCode's built-in autocompact: running Aider 150 and Web-Bench React 20 on OpenCode 2.2.6 + GPT-6 Luna achieved 150/150 and 20/20 task completion (with 59/59 final browser tests) while reducing estimated cost by 14.3% and 42.1%, respectively. These are OpenCode project results cited to demonstrate a reproducible post-port benchmark methodology, not DSH runtime results.

![Earlier DSH two-run benchmarks](docs/slide-results.jpg)

![OpenCode Aider / Web-Bench comparison](docs/benchmarks-opencode-2026-11-09.png)

![1.1.12 GLM-5.3 Flash: three-way comparison of stock DSH, DSH Billion, and Live Compact](docs/benchmark-dsh-three-way-glm53-1.1.12-2026-11-17.png)

## Cloud and Local Usage

- **Cloud APIs (DeepSeek, GLM, etc.)**: Install with the commands below. Check answer correctness first, then evaluate cache hits, cache misses, and actual cost together. Cache hit rate alone is not cost; when total input shrinks, compare token counts alongside percentages.
- **Local or intranet models**: Install with the same commands below. Prioritize answer correctness, whether current context size dropped, and how many tokens were actually saved. Seeing "20k+ context" in the UI shows what remains retained, not that compaction failed. Local KV cache reuse is also improved where possible, without enforcing fixed cloud-billing hit-rate thresholds; you can compare reuse and prefill latency across the same task.

By default, the plugin begins checking for compactable spans once context reaches 20,000 tokens. Whether a cut actually happens also depends on what was read in the turn and how long the next turn is projected to be. When context nears the window limit, DSH can still invoke the model for a full official summary.

When a task that has executed tools hits the single-step output cap, 1.1.13 automatically continues in the same session, preserving the original task, existing edits, and the original output cap. If the first continuation hits the cap again during a reasoning-only phase, the plugin issues one final continuation prompting the model to begin with a small tool call. Auto-continuation defers to pending user input, cancelled tasks, and responses containing unexecuted tool calls.

Default settings are sufficient for everyday use. `SPINE_THUNDER` is an experimental model-summarization mode that is off by default; only set it to `1` when testing it, and keep it off for local or intranet models.

Headless:

```sh
dsh plugin --profile headless add dsh-live-compact@1.1.14
```

Web:

```sh
dsh plugin --profile web add dsh-live-compact@1.1.14
```

Install locally first to inspect the version:

```sh
npm install -g dsh-live-compact
dsh-live-compact --version
```

## How Cloud and Local Cuts Differ

Cloud vs. local mode is determined by the configured endpoint address for the active provider. Request headers usually carry only the provider name. `baseURL` from the current configuration takes priority, followed by official DeepSeek defaults and the built-in catalog. Public addresses use the cloud strategy; loopback, private-network, or unreadable addresses use the local strategy. Set `SPINE_ENGINE=local` or `cloud` to override. This does not change the official gate's headroom.

Cloud and local cuts are planned separately. Official DeepSeek uses the public cloud strategy. Providers whose name contains `strata`, or whose address is localhost, private, or unreadable, use the local strategy and never cross the latest user message.

Account mode without an explicit URL, restoring older official cards, and exact progress notices compose cleanly; genuine user messages and unknown notices still remain boundaries.

The latest user message is never included in a cut span. Earlier user messages inside a cut are preserved verbatim as single-line JSON inside the card.

## Cards and Cut Points

Cards do not use the official `<compacted-summary>` tag or `compact-checkpoint` kind. Official compaction treats those nodes as prior summaries and re-summarizes them, shrinking extracted cards repeatedly. New cards use `spine-checkpoint`. Previously written legacy cards still act as stop boundaries and are not re-cut.

In Issue #19, the cut that sent next-turn input back to 210k–240k and invalidated the cached prefix happened because the cut walked to the left of an existing card. The newest card acts as a floor: walking left stops at any card, and if a newer card exists to the right of a candidate span, that cut is skipped. Already-cached prefixes are never rewritten by later cuts by design. If the newest span is under 1,000 tokens, it is skipped rather than reaching further back to assemble a cut.

If a turn contains a `read` of at least 1,000 tokens, that cut targets only that round without pulling in the 3 recent short rounds meant to stay verbatim. If a single succeeded or unknown-status tool result in that round reaches 32,000 tokens, only a sequence pointer is kept; recognized failures preserve full diagnostics or keep the original span. Older rounds are considered only if this round cannot be cut.

A single succeeded or unknown-status tool result must exceed 32,000 tokens before switching to pointer-only mode (replacing both the call and its result). Recognized failure results never take the pointer-only branch. To customize this threshold, set `SPINE_POINTER_TOKENS` before startup (e.g., `8000`). It is an absolute token count, not a ratio.

Small cuts stop at 650,000 tokens. Even on larger context windows, small cuts stop above 650,000 and hand off to official compaction. A single drain pass may commit multiple cuts, recording rewrite cost once per drain: the post-first-cut prefix tail minus savings from subsequent cuts. `dsh-live-compact stats` uses this figure for `rewrite`. Per-cut details are written to `spine-cuts.jsonl` in the shadow directory with only sequence numbers, token counts, and cut points—no message text. There is no "savings must exceed rewrite" gate, as adding one would suppress later cuts in the same drain.

Checkpoints, user boundaries, and tool pairing are re-validated right before committing a cut, including any new messages that arrived while waiting.

An idle turn snapshots the session log only once. Arriving tool results reuse recorded call IDs rather than copying the full log per lookup. The drain loop reuses that single snapshot's job and call tables. Token measurement runs once at the start and once after each committed cut; turns with no cut measure only once.

## What Stays in a Card

Short tool results remain in the summary labeled as tool output (never "confirmed"). Outputs up to 12 lines and 512 characters are kept in full. Longer succeeded or unknown-status results keep a bounded tail excerpt and note that the body was omitted; failure results follow the full-diagnostics gate below. Omitted raw outputs are stored locally under `~/.dsh/compaction-spine/shadow` with the cut's short ID in the filename—read only when needed, not by default. Reasoning blocks are never included in summaries.

- Reasoning is excluded from summaries and stored separately in local backups labeled as reasoning.
- User messages and assistant lines follow attribution rules and label who spoke.
- Every kept line carries its original role; a line containing "user said" does not become a new instruction.
- No "confirmed" markers. The card preamble explicitly states that this is an excerpt of earlier context, not a new instruction and not an authorization.
- If a candidate cut does not produce a valid card, the original span is left untouched without calling a model. Cards can be very short.

Errors take priority in cards. The 3 most recent finished tool rounds remain verbatim in their original tool calls and results without truncation; only older rounds are compacted into cards.

Card pointers record session log event sequence numbers, never backup file paths.

Failed tool results keep their exit code and full diagnostics. If diagnostics and call metadata do not fit the card budget, or multiple failures cannot all be preserved together, the cut leaves the original span untouched. Extra fields in structured results cannot be silently dropped. Truncated display previews do not qualify a cut for commit, and failed cards are never labeled as completed normally. `npm run test:gate` verifies this path.

Command outputs are marked as verbatim errors only on structural signals: `isError`, non-zero exit code, `[stderr]`, `[exit code: N]`, `[Command finished with exit code N]`, `exit_code=N`, or `"exitCode": N` in ptc results. Without sufficient evidence, status is marked unknown and the tail lines are kept.

Assistant prose claiming "committed / released" never becomes a hard line. Cards record only tool events with real tool call IDs, never keyword heuristics.

Background jobs do not block cuts. Still-running jobs are recorded on the card with a note that files on disk are authoritative for later changes. A failed kill call is never recorded as finished.

Raw tool outputs are persisted by `toolCallId` before official pruning runs. If writing fails, the card records "No complete backup" without leaving dangling paths. Backups are organized into per-session directories and are never auto-loaded into the next prompt.

Configuration errors are no longer masked as "service already registered"; startup failures surface their original cause.

Official summarization still delegates to the host's `super.summarize`. When used alongside another plugin that takes over `compaction-basic`, if the host returns an empty or unusable summary, Live Compact logs an explicit warning and leaves the span untouched; it does not guarantee bypassing the other plugin to generate a summary independently.

## Restart and Capture

After a process restart, catch-up capture covers only the 3 most recent rounds. If a saved cursor is still valid, capture resumes from that position; if the saved cursor exceeds the current log length, it falls back to the 3 most recent rounds instead of rescanning from 0. On restart, existing backups and cursors are verified before continuing from the latest working position.

Cursors and retryable capture results are cached per session to avoid rescanning long histories; tool results and raw/meta byte counts and SHA-256 digests continue to be verified after restart recovery.

Tool-result shadow/ACL capture is queued onto the next event-loop tick and deduplicated by `toolCallId`. `finishTurn` does not wait for a full historical catch-up; a background queue recovers and retries in batches, and cuts verify only the raw files required by the selected span. Unready targets, verification failures, or permission denials preserve the original span. Raw capture and cold-directory ACL initialization still perform synchronous filesystem operations; queuing itself does not make all I/O asynchronous.

Large catch-ups during idle maintenance yield to the event loop between batches. Transient Windows file-busy errors automatically retry up to 8 times with backoff; permanent failures remain fail-closed without blocking turn completion.

## Presets

The four Web and Desktop presets swap stock compaction in their group for this package at load time. If loading fails, they fall back to stock so presets still start. TUI is untouched.

- Presets no longer hard-wait on the gate. If the gate fails to load, web presets do not stall; the package is probed before swapping and retains the stock name on failure.
- Swapped presets default to `0.8`, `65536`, and `8192`, while explicit preset configuration takes precedence.
- Both outer and inner preset IDs are matched: `standard`, `ptc`, `cordis`, and `minimal` are swapped on `0.2.1-rc.5`. TUI remains untouched.
- When stock compaction is already active, Live Compact takes over only after initialization succeeds. On activation failure, stock remains in control and a takeover-skipped warning is logged. If an older profile user patch separately disabled `compaction-basic`, re-enable it in the host's plugin manager. If neither engine can load, the shell reports the failure reason.
- Headless host bundles disable stock compaction themselves, and the plugin never re-enables that entry. Web presets do not inject a separate gate; if a swap is missed for a session, stock compaction continues running.

## Windows Backup Permissions

On Windows, backup access permissions are re-applied strictly; on failure, backup writes stop and the original tool output is kept intact. Permission failures no longer crash the host, never fall back to `chmod`, and warn only once; any prune or cut depending on that backup is skipped. The main thread and managed workers share a process-wide permission fuse so worker recycling or recreation never clears a tripped fuse.

Dual-user testing (Actions run `42367713007`, Windows 11 Enterprise ARM / Server 2022 x64) confirmed that `/inheritance:r /grant:r` leaves explicit ACEs from other principals, allowing a second standard user to read backups. Windows backups therefore replace the entire DACL. Slow storage, very large histories, and shared-directory contention remain areas for ongoing observation.

ACL writes use the fixed `%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe` (Windows PowerShell 5.1 / .NET Framework) with `-NoProfile -NonInteractive -Command`. The script is embedded verbatim in the bundled `shadow.js`; target path, SID, and entry kind are passed via child-process environment variables rather than interpolated into the script. The current user's SID is queried and cached per execution context via fixed System32 `whoami.exe` (re-queried if a worker is rebuilt) and cross-checked inside PowerShell against its own Windows identity. A new protected DACL is built granting FullControl only to the current user, `SYSTEM`, and `Administrators`, with `OI`/`CI` flags on directory ACEs and no inheritance flags on directly fixed files. Only the DACL is modified—no privilege elevation, no owner/group/SACL changes, and no temporary permissive ACL. After writing, the actual DACL is read back and verified for protection flags, exact ACE count, SIDs, Allow type, rights, and inheritance flags before caching. Tool failure, timeout, invalid SID, or read-back mismatch trips the process fuse for subsequent backups without falling back to `chmod`.

Directory caching verifies canonical path, device/file IDs, and creation time; replacing a directory or redirecting a known alias rejects subsequent backups. Empty files created with `wx` inside an already-verified directory verify that the open handle matches the target path and rely on the inherited allowlist without spawning a subprocess per file, completing the check before writing content. Pre-existing files are never trusted solely from parent directory cache: cached trust applies only when this process has already verified the same file identity; otherwise the file's DACL is replaced directly. Symlinks, non-regular files, and multi-hardlink files are rejected. Historical backups are not bulk-scanned or bulk-repaired. Restart recovery performs read-only verification of existing raw/meta pairs without tightening each file's ACL up front; cursor reads still perform single-file ACL checks. Before a raw file is actually used for pruning or cutting, the target file is re-authorized and checked against the initially confirmed byte length and SHA-256 digest—re-authorization never blesses modified content as an original backup. Untouched legacy files (including files read externally or via `shadow-search`) are not guaranteed to be repaired and previously exposed data cannot be retroactively retracted. Do not point shadow storage at a directory shared with unrelated files.

Caching assumes the same user and administrators do not mutate ACLs in place while the process is running; metadata checks cannot detect in-place ACL edits on the same file or eliminate all races against concurrent path changes by the same user or an administrator. `SystemRoot`/`windir` must point to a trusted Windows installation, and the filesystem must provide stable file identity and Windows DACL support; missing tools or execution-policy blocks refuse backups. Unix continues to use `0700` for directories and `0600` for files. Cut points, token/cap policies, #27 root coordination, and restart rules are unchanged.

Run the offline gate with `npm run test:gate` (Windows permission tests use injected filesystems and runners to verify failure branches; real permissions are additionally verified with two standard Windows accounts checking writes, readable control files, and backup read access). Run offline scorer tests with `npm run test:live-gate`.

## Environment Variables

Common compaction parameters can be configured in the DSH plugin settings UI while retaining full environment-variable compatibility. To change the small-cut starting threshold, set `minTokens` in the settings UI (`SPINE_MIN_TOKENS` remains supported).

| Variable | Default | Purpose |
|---|---:|---|
| `SPINE_THRESHOLD_RATIO` | 0.8 | Official full-compaction trigger ratio, read when constructing the engine or loading presets. |
| `SPINE_HEADROOM_TOKENS` | 65536 | Official full-compaction headroom (tokens), read when constructing the engine or loading presets. |
| `SPINE_RETAIN_TOKENS` | 8192 | Official full-compaction retention budget (tokens), read when constructing the engine or loading presets; explicit `retainRatio` takes precedence. |
| `SPINE_POINTER_TOKENS` | `SPINE_CHUNK_MAX` (default 32000) | Token threshold above which a single tool result keeps only a sequence pointer. |
| `SPINE_MIN_TOKENS` | 20000 | Minimum context token count before post-turn small cuts run. |
| `SPINE_KEEP_ROUNDS` | 3 | Number of most recent finished tool rounds kept verbatim (at least 1; invalid legacy values fall back). |
| `SPINE_KEEP_ROUND_TOKENS` | 32000 | Cloud drain stop line; the 3 most recent finished tool rounds always stay verbatim. |
| `SPINE_CHUNK_MIN` | 1000 | Minimum tokens required in a single cut. |
| `SPINE_CHUNK_MAX` | 32000 (`RAIL_TOKENS`) | Maximum tokens in a single cut, and the default pointer threshold. |
| `SPINE_COOLDOWN_MS` | 0 | Extra post-official-compaction cooldown window in ms; only positive numbers enable it (`0`, negative, blank, or non-numeric disables it). |
| `SPINE_SHADOW_DIR` | `~/.dsh/compaction-spine/shadow` | Backup directory; if `DSH_HOME` is set, it replaces `~/.dsh`. |
| `SPINE_SHADOW_SINGLE_PROCESS` | off | Only `1` enables the exclusive-directory contract; requires a dedicated `SPINE_SHADOW_DIR` (see below). |
| `SPINE_SHADOW_MAX_BYTES` | 268435456 | Positive byte cap for backups; takes priority over `SPINE_SHADOW_MAX_MB`. |
| `SPINE_SHADOW_MAX_MB` | 256 | Backup cap in MB when `SPINE_SHADOW_MAX_BYTES` is not set to a positive integer. |
| `SPINE_ENGINE` | auto | Set to `cloud` or `local` to force the cut strategy; other values are ignored. |
| `SPINE_THUNDER` | off | Set to `1` to enable Thunder mode. Keep off for local/edge models. |
| `SPINE_THUNDER_MODEL` | `flash` | Set to `27b` to send less on official compactions when prefix exceeds 80,000 tokens. Do not set for local/edge models. |
| `SPINE_ENGINE_FAIL` | none | **Test only**: inject engine failure at `import`, `construct`, or `after-super`. |
| `SPINE_STOCK_FAIL` | none | **Test only**: set to `1` to disallow fallback to the stock engine. |

Clearing or resetting a numeric field in the settings UI removes its explicit override. Precedence is: **explicit config > valid environment variable > default**. Empty strings, whitespace-only strings, and non-numeric environment variables are treated as unset; explicit `0` remains valid for fields that permit zero. Unconfigured fields may appear blank with defaults shown in their hints.

Small-cut settings take a consistent per-engine snapshot at the start of each post-turn maintenance pass; saved changes take effect on the next pass without overwriting other engines. Official full-compaction parameters follow the host's construction-time snapshot and require recreating the engine or session after saving. Host-supported options (`retainRatio`, `maxTokens`, summary model routing, `modelPolicies`, overflow retries, etc.) continue to be validated and preserved by the host; `retainRatio` and `retainTokens` are mutually exclusive.

The takeover toggle affects only subsequently loaded presets and isolated `compaction` groups; directly registered presets and Include-loaded groups are swapped in place as well. New presets resolve **explicit preset engine config > root plugin config > valid env var > default**; existing presets retain the values captured when they loaded, so later root changes require reloading the preset. Native client slots, resource declarations, and installation follow existing host interfaces.

Each new `compact-checkpoint` causes post-turn small cuts to skip once; `spine-checkpoint` does not trigger this skip.

`gate.js` defaults official compaction presets to `thresholdRatio: 0.8`, `headroomTokens: 65536`, and `retainTokens: 8192`, overridable via environment variables before startup. These values govern only official compaction and do not alter small-cut points.

Cloud defaults remain `0.8`, `65536`, and `8192` regardless of endpoint address. TUI `/settings` does not expose this entry. `cordis.patch.yml` does not interpolate environment variables; omitted fields are filled by the engine from environment variables and defaults. Explicit values in the config block take precedence, for example:

```yaml
- id: compaction-spine
  name: dsh-live-compact
  config:
    auto: true
    thresholdRatio: 0.8
    headroomTokens: 16384
    retainTokens: 8192
    compactionRetries: 0
```

## Installation

Install from npm. `headless` and `web` are separate profiles and each must be installed separately. Desktop is a distinct profile and plain `dsh plugin --profile desktop` is rejected.

```sh
dsh plugin --profile headless add dsh-live-compact@1.1.14
dsh plugin --profile web add dsh-live-compact@1.1.14
```

Install locally first to check the version:

```sh
npm install -g dsh-live-compact
dsh-live-compact --version
```

You can also install from the mirror:

```sh
dsh plugin --profile headless add https://download.merkyorlynn.com/downloads/plugins/dsh-live-compact-1.1.14.tgz
dsh plugin --profile web add https://download.merkyorlynn.com/downloads/plugins/dsh-live-compact-1.1.14.tgz
npm install -g --force "https://download.merkyorlynn.com/downloads/plugins/dsh-live-compact-1.1.14.tgz"
dsh-live-compact --version
```

Requires DSH `>=0.2.0-rc.2 <0.3.0`. After installing, run a long task; seeing `spine cut …ms` in the logs confirms that a cut took effect. Project page: https://download.merkyorlynn.com/live-compact.html

Do not install on DSH-TUI yet. Although TUI v0.20.0 supports rendering `spine-checkpoint` partial cards, this package currently takes over compaction only in Web and Desktop presets while TUI presets still use stock compaction; TUI support will open once takeover and joint validation are complete.

Cut originals are written to `~/.dsh/compaction-spine/shadow`. Unix permissions are `0700` for directories and `0600` for files; on Windows, newly created or fixed backups allow the current user, `SYSTEM`, and `Administrators` (see the ACL section above for legacy files and caching caveats). The default cap is 256MB, rotating out the oldest backups when approaching the limit. Sequence numbers on cards are session log event numbers. To search within the current session's backups, use `import { searchShadow } from "dsh-live-compact/shadow-search"`; the plugin never searches backups on its own.

`spine-maintenance.jsonl` records end-of-turn maintenance status inside an existing shadow directory. Each `operation` corresponds to one attempt: `deferred` means catch-up capture has not finished yet, and `cuts` under `completed` records how many small cuts were committed. When the host rejects a model pressure budget, the configuration reason is written to `spine-decisions.jsonl`.

In `dsh-live-compact stats`, `saved` is the immediate token reduction from cuts, while `rewrite` is the actual tail rewritten for a drain pass (counted once per drain, not multiplied by cut count). Because the two metrics measure different things, their ratio is not a net profit/loss figure. Desktop has no terminal, so `rg "spine cut"` applies only to terminal logs. Each shadow root's `spine-cuts.jsonl` retains up to 1 MiB of recent complete per-cut records, while cumulative counters live in `spine-stats.json`.

The filesystem backing POSIX shadow directories must actually enforce `0700`/`0600` permissions. The plugin reads back directory and file modes and refuses backups whose permissions do not match before writing content.

Change the cap:

```sh
export SPINE_SHADOW_MAX_BYTES=67108864
```

Clean up manually:

```sh
rm -f ~/.dsh/compaction-spine/shadow/*/*.txt ~/.dsh/compaction-spine/shadow/*.txt
```

If `DSH_HOME` or `SPINE_SHADOW_DIR` is set, files live there instead. If old backups cannot be deleted to free space, the cut is skipped and warned once per session.

Default directories (including paths under `DSH_HOME`) and directories without an exclusive declaration always run a full rotation scan so they remain safe to share with legacy writers. Setting `SPINE_SHADOW_DIR` alone does not enable fast-path estimation. To reduce scans, point `SPINE_SHADOW_DIR` at a **dedicated local directory** and also set `SPINE_SHADOW_SINGLE_PROCESS=1` (explicitly naming the default directory will still not enable it). This contract requires that the entire directory is used exclusively by this single plugin instance in this process—never shared with legacy writers, non-cooperating writers, or external tools that modify files or trees inside it. Custom code invoking lower-level backup helpers must pass the same `root`, never a session subdirectory. Keep the default behavior if this contract cannot be met.

After the first full scan of a dedicated root, payload directory traversal is skipped only when directory identity, capacity cap, and ownership remain unchanged, estimated usage is strictly below 80%, and less than 60 seconds have elapsed since the last full calibration. Once 60 seconds expire, the next rotation performs a full rescan; there is no background timer cleanup. Fast-path returns and writes do not extend the calibration window, and clock anomalies or backward jumps force a full scan. Incomplete scans or invalid measurements reject any backup requiring capacity admission and keep original content intact rather than admitting based on undercounted totals.

Under the normal lock path, all standard raw/cut/official writes acquire the root atomic lock first and check coordination markers; cut/official capacity checks and payload writes occur inside the same critical section. Accounting conservatively uses exact UTF-8/Buffer payload lengths without subtracting on overwrites or failed-write cleanups, correcting on the next full scan. Raw writes follow their original capture rules and do not themselves trigger rotation; consecutive raw writes may temporarily exceed the cap until the next rotation. Capacity tracks payload sizes only (excluding companion meta, owner, stats, or lock files) and is not a hard physical-disk quota. Oversize backup exceptions, deletion order, and reference protection remain unchanged; other session directories are cleaned only when their owning process is confirmed dead, and conservatively preserved when permissions are insufficient or owner status is unknown. Standard cut/official scans, deletions, and publish transactions execute in a managed worker serialized per root, with idle workers automatically recycled; shared directories always perform full scans without estimation skips. Lower-level sync APIs and raw capture retain their synchronous semantics.

`.spine-rotation.owner` stores the PID and a random lifetime token. When no `shared` fence exists, the record is well-formed, and the previous PID is confirmed dead, a new lifetime may safely reclaim ownership and run a fresh full scan; ownership is never preempted when death cannot be confirmed. A second cooperating process, worker, or plugin instance (including path aliases) creates `.spine-rotation.shared` before writing payloads, permanently downgrading that root to full scans even after the other process exits. This marker detects cooperating writers only, not legacy or external writers. Locks are published by fully writing, flushing, and closing a temporary file in the same directory and atomically hard-linking it to `.spine-rotation.lock` with holder PID and per-transaction unique ID. Cleanup failure on the temporary staging name after publication does not compromise lock ownership or block subsequent backups, though an unrotated staging name may remain.

Only when hard-linking itself fails with `ENOTSUP`, `EOPNOTSUPP`, or `ENOSYS` does the plugin flush a `shared` fence, verify no prior active or potentially exclusive transaction exists, and allow fallback to legacy full rotation and raw/cut/official/cursor writes (disabling exclusive estimation for that root). Fence and lock state are still checked before writing payloads. Ambiguous errors (`EPERM`, `EACCES`, `EIO`, `EXDEV`, `EINVAL`, etc.) or unsupported errors at any other step reject the backup. This fallback inherits legacy shared-concurrency semantics and provides no exclusive capacity guarantee.

Active locks, reused PIDs, legacy empty-directory locks, corrupted records, or unconfirmed deaths cause the backup to be rejected with original content preserved; locks are never stolen by age, and `EPERM` or unknown errors never count as dead. Only when `process.kill(pid, 0)` explicitly returns `ESRCH` does the plugin persist the `shared` marker, clear estimates, and atomically record that lock's full identity in `.spine-rotation.recovered` (or an empty `.spine-rotation.recovered-<pid>-<txid>` marker file flushed to disk when hard links are unsupported, avoiding half-written JSON). The original lock remains in place as a fence and is never deleted or replaced; all cooperating instances then follow legacy full rotation and payload writes for that confirmed-dead identity, unaffected if the OS later reuses the PID. Post-recovery shared operations follow legacy concurrency semantics without exclusive capacity critical sections. Marker files are flushed to disk (and directories fsynced on POSIX; Node on Windows cannot fsync directory handles, so directory-entry durability across sudden power loss is not guaranteed).

On a clean restart with no leftover lock, only a well-formed owner record whose PID is confirmed dead may be rebuilt; legacy PID-less owners, existing `shared` fences, and unreadable records conservatively fall back to full scans. Because distinct main-thread and worker lifetimes can also trigger `shared`, this toggle does not guarantee every production write takes the estimation path. To re-enable exclusive estimation manually, choose a new dedicated directory, or—only after all users have stopped—remove `.spine-rotation.lock`, `.spine-rotation.owner`, `.spine-rotation.shared`, `.spine-rotation.recovered`, and any per-transaction recovered marker files in that root. Ambiguous records such as legacy empty locks must likewise be handled manually only after confirming all users have stopped. Never declare the dedicated-directory contract when sharing with legacy writers that do not follow the protocol.

MIT License — see `LICENSE`.
