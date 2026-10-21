# Bossbot — Build Specification (v2)

Bossbot adds multi-session orchestration to Claude Code. One orchestrator (the **lead**) delegates work to named **worker** sessions, reviews and integrates what they produce, and curates a project wiki that acts as long-term memory. It ships as one Claude Code plugin (with a mod) and builds on native Agent Teams.

How this spec is organized:
- **§4–§5** define an architecture where new enhancements are additions, not rewrites.
- **§7** specifies every feature once, in a fixed format.
- **§8** schedules those features into phases.

---

## 0. How to use this file (instructions for Claude Code)

**Each session:**
- Build **one phase per session**.
- At the start of every session, read:
  - §0–§5
  - the phase section in §8
  - every feature in §7 that the phase lists
  - `docs/PROGRESS.md` and `docs/API-MAP.md`, if present

**Source-of-truth order:**
1. Generated types in `.claude-plugin/types/` for the pinned build
2. Official docs
3. This spec

Record every deviation in `docs/API-MAP.md`. If a deviation changes architecture (§4), also write an ADR in `docs/adr/` using Appendix H.

**End of phase:** update `docs/PROGRESS.md` (Appendix G) and `CHANGELOG.md`.

**Proceed without checkpoints.** Stop only when:
- a required API is missing and §7 gives no fallback, or
- an action would be destructive outside the repo.

**Kickoff prompt per session:**
```
Read BOSSBOT_SPEC.md §0–§5, the Phase <N> section in §8, and every feature it references in §7.
Read docs/PROGRESS.md and docs/API-MAP.md if they exist. Execute Phase <N> only. Follow §10 definition of done.
```

---

## 1. Product summary

| Capability | Summary |
|---|---|
| Orchestration | The lead spawns up to N workers. Each worker has a role, a purpose, and an isolated workspace. |
| Messaging | Code acknowledges a `TASK` the moment it arrives, without a model turn. Dependent tasks are held back and released automatically when their dependencies finish. |
| Safety | Workspace guard, file leases, journal, log gate, test gate, git block, declarative policies, deterministic pause. |
| Memory | Workers write logs of 30 lines or fewer. Only the lead writes the wiki. Wiki pages are refreshed only when they go stale. |
| Modes | Git mode (worktrees, merges) and no-git mode (shared folder, leases, journal). Switching works live in both directions. |
| Visibility | A band in the lead session, a dashboard session with tabs, and an event log. |
| Reliability | Heartbeats, health states, a bounded restart policy, re-anchoring after compaction, checkpoint handoffs. |
| Workflow | Typed phases. Stages are derived from events, not reported by the model. Repair counter. Per-worker token budgets. |
| Onboarding | Deep exploration of an existing project into a baseline wiki that is verified and confirmed by the user. |
| Setup | Two install commands, then `/bossbot init` and `/bossbot doctor`. |

---

## 2. Target

- **Claude Code 2.1.304, pinned.** Agent Teams is enabled with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.
- **Docs:** agent-teams, plugins, and plugins/mods (interface, api, events, reference, test, troubleshoot) at https://code.claude.com/docs/en/
- **Reference code:** the built-in mods in `anthropics/claude-code/mods` (diff, agents-md, sec-default, telemetry).

---

## 3. Constraints

### 3.1 Cost
- **Worker cap:** at most `maxWorkers` workers (default 4). The mod checks the cap before any model call, and `/spawn` checks again as a backstop.
- **Coordination runs as code, never as model turns.** That covers acks, heartbeats, gates, dispatch, stage tracking, dashboard refresh, scans, reverts and setup.
- **Prompt cache:** never change system-prompt text per request. Per-turn context is injected only through the supported context mechanism.
- **Lead context:** the lead never reads worker transcripts. Information flows only through logs and wiki pages.
- **Hub-and-spoke:** workers message only the lead, unless the lead grants peer contact for a specific task (marker `[peer: <name>]`).
- **No roster broadcasts.** Each session reads shared state when it needs it.

### 3.2 Safety
- **Guards fail closed.** Any error or a 5s deadline (`core/guard.ts`) denies the call. Observers, which run after a call, fail open and only log.
- **No LLM calls inside hooks.**
- **No auto-restart loops.** Auto-restarts are limited per worker per hour.
- **Destructive actions** (teardown, revert, force-release) always ask first via `$.ui.ask`.
- **Workers never write to** `.bossbot/`, `.claude/`, or the wiki, except their own log folder.

### 3.3 Mod runtime rules
- No `setTimeout`, `fetch`, or Node APIs in mod code. Use `$.clock`, `$.http`, `$.fs`, `$.process`.
- Never alias or destructure `$`. Event names must be string literals. No dynamic `import()`.
- **Only hubs (§4.3) call `on(...)`.** Exactly one unmatched `session.start` hook exists, and it lives in `register.ts`.
- No busy loops. Keep hook time well under the host limit (targets in §3.4).
- Open panes only in response to a user action. Use toasts for anything unsolicited.
- **`$.state`** resets on `/clear`, `/resume` and `/branch`. Rebuild it from the store on `classic.SessionStart` with source `clear`, `resume`, `fork` or `compact`.
- **`$.store`** is shared by every session on the machine, across all projects:
  - namespace every key (§4.7)
  - use one key per item
  - never read-modify-write a shared blob
- Give every UI control a `key`. Check `e.surface` before using `Raster` (terminal only) or `Svg` (desktop only).

### 3.4 Performance budgets (verified in P2 and P4)

| Path | Budget |
|---|---|
| Pre-call gate pipeline | p95 < 30 ms when no file copy is needed; < 300 ms with a journal copy |
| Message router | p95 < 20 ms |
| Scheduler tick | < 50 ms. Long jobs (merge preview) run one at a time and are skipped if the previous run hasn't finished. |
| Store writes per session | ≤ 2/s sustained. Heartbeats are throttled. |
| Dashboard redraw | ≤ 2 s after a store change |

---

## 4. Architecture

### 4.1 Layers

```
Claude Code mod API ($, events)
        │
  core/host.ts        the only module that knows raw event payload shapes; maps them to internal types
        │
  hubs/*              the only modules that call on('…'); one hub per mod event family
        │
  features/*          contribute units: gate rules, message handlers, jobs, commands, tabs,
        │             band segments, doctor checks, setup steps, phase types, observers
        │
  logic (pure)        decisions + Effect[]; reads go through Ports, writes go through Effects
        │
  core/effects.ts     the only module that performs writes (store, fs, send, toast, prompt, events)
        │
  data                config file · store · event log · wiki files
```

**The principle: reads go through Ports, writes go through Effects, and only hubs touch events.** This keeps each feature small and testable, and isolates Claude Code API changes to `host.ts` and the hubs.

### 4.2 Feature contract (`core/types.ts`)

```ts
type Role = 'lead' | 'worker' | 'dashboard';
type Mode = 'git' | 'nogit';

interface Ports {                       // read-only; fakes provided in test/fixtures
  config: Config;
  store: { get<T>(key: Key): Promise<T | undefined>; list(prefix: KeyPrefix): Promise<Key[]> };
  fs: { exists(p): Promise<boolean>; read(p): Promise<string>; list(dir): Promise<string[]>; stat(p) };
  run(cmd: string, opts?): Promise<{ code: number; stdout: string; stderr: string }>; // jobs/commands only
  now(): number;
  self: { role: Role; worker?: string; sessionId: string; projectRoot: string; workspace: string };
  mode(): Promise<Mode>;
}

type Effect =
  | { t: 'send'; to: Address; text: string }
  | { t: 'toast'; text: string }
  | { t: 'store.set'; key: Key; value: unknown } | { t: 'store.del'; key: Key }
  | { t: 'event'; e: DomainEvent }
  | { t: 'fs.write'; path; content } | { t: 'fs.writeIfChanged'; path; content }
  | { t: 'fs.mergeJson'; path; patch }  | { t: 'fs.replaceBlock'; path; marker; content }
  | { t: 'fs.copy'; from; to } | { t: 'fs.delete'; path }
  | { t: 'prompt.submit'; text: string }            // lead only; costs a lead turn
  | { t: 'state.set'; key: StateKey; value: unknown }
  | { t: 'invalidate' };

type Decision = { kind: 'allow' } | { kind: 'deny'; message: string } | { kind: 'warn'; message: string };

interface GateRule {          // pre-call; pure apart from Ports reads
  id: string; order: number; tools: ToolName[]; roles: Role[]; modes?: Mode[];
  check(call: ToolCall, p: Ports): Promise<{ decision: Decision; effects?: Effect[] }>;
}
interface Rewrite {           // pre-call argument transform, applied after all rules allow
  id: string; order: number; tools: ToolName[]; roles: Role[];
  apply(call: ToolCall, p: Ports): Promise<ToolCall>;
}
interface Observer {          // post-call
  id: string; tools: ToolName[] | '*'; roles: Role[];
  observe(call: ToolCall, result: ToolResult, p: Ports): Promise<Effect[]>;
}
interface MessageHandler { verb: Verb; roles: Role[]; handle(msg: Parsed, from: Address, p: Ports): Promise<Effect[]> }
interface OutboundHandler { verb: Verb; roles: Role[]; handle(msg: Parsed, to: Address, p: Ports): Promise<{ send: boolean; text?: string; effects?: Effect[] }> }
interface Job     { id: string; roles: Role[]; everyMs: number; exclusive?: boolean; run(p: Ports): Promise<Effect[]> }
interface Command { name: string; sub?: string; roles: Role[]; immediate?: boolean; confirm?: string;
                    args?: ArgSpec; run(args, p: Ports): Promise<Effect[]> }
interface Tab     { id: string; title: string; hotkey: string; order: number; roles: Role[];
                    render(vm: ViewModel, el: Elements, surface: Surface): Tree }
interface BandSegment { id: string; priority: number; order: number; render(vm: ViewModel): string | null }
interface DoctorCheck { id: string; title: string; run(p: Ports): Promise<{ status: 'ok' | 'warn' | 'fail'; detail: string; fix?: string }> }
interface SetupStep  { id: string; order: number; applies(p: Ports): Promise<boolean>;
                       ask?: AskSpec; apply(answer: unknown, p: Ports): Promise<Effect[]> }
interface PhaseType  { id: string; stages: StageRule[] }   // StageRule: { name; active(ev): boolean; done(ev): boolean }
interface LifecycleHook { on: 'start' | 'reset' | 'compact'; roles: Role[]; run(p: Ports): Promise<Effect[]> }

interface Feature {
  id: string;                    // e.g. 'acks'
  title: string;
  defaultEnabled: boolean;       // overridden by config.features[id]
  requires?: string[];           // other feature ids; checked at load, reported by doctor
  gates?: GateRule[]; rewrites?: Rewrite[]; observers?: Observer[];
  handlers?: MessageHandler[]; outbound?: OutboundHandler[];
  jobs?: Job[]; commands?: Command[]; tabs?: Tab[]; band?: BandSegment[];
  checks?: DoctorCheck[]; setup?: SetupStep[]; phaseTypes?: PhaseType[]; lifecycle?: LifecycleHook[];
}
```

`features/index.ts` exports `FEATURES: Feature[]`, a static list. **Adding a feature means adding a folder and one line in that list.**

### 4.3 Hubs (event ownership)

| Hub | Mod events owned | Dispatches to | Semantics |
|---|---|---|---|
| `hubs/tool.ts` | `tool.call` (+ `classic.PostToolUse` if post-call results aren't available from `next`) | gates → rewrites → `next` → observers | Rules run in `order`. The first `deny` wins. `warn` results are collected into toasts and events. Rule `effects` run **before** `next` (for example, journal copies). The pipeline is wrapped in fail-closed. Observers fail open. |
| `hubs/message.ts` | `session.receive`, `session.send` | handlers / outbound by verb | Unknown verbs pass through untouched. Parsing happens once in `core/protocol.ts`. |
| `hubs/command.ts` | `command.run` (registration happens in `session.start`) | commands by name and subcommand | `/bossbot <sub>` router plus top-level aliases (`/team`, `/status`, `/dashboard`). `confirm` goes through `$.ui.ask`. |
| `hubs/lifecycle.ts` | `session.start` (single), `classic.SessionStart`, `turn.step`, `turn.complete` | lifecycle hooks, heartbeat, usage | `start` runs migrations (lead only), loads config, and registers commands. `reset` and `compact` rebuild `$.state`. |
| `hubs/scheduler.ts` | one `$.clock.every(1000)` | jobs by `everyMs` | Jobs whose roles match are due on a modulo schedule. `exclusive` jobs never overlap with themselves. |
| `hubs/ui.ts` | `ui.render` (Pane, AbovePrompt) | tabs and band segments | Builds the ViewModel once per render from the cached snapshot. Resolves elements and passes them to `Tab.render`. |
| `hubs/prompt.ts` | `prompt.submit` | dashboard drop, context injection | The dashboard role drops every prompt. Context injection serves re-anchoring. |

New mod events are only ever added by **extending a hub**, and that requires an ADR.

### 4.4 Effects execution (`core/effects.ts`)
- Effects run sequentially, in the order they are returned.
- `fs.writeIfChanged`, `fs.mergeJson` and `fs.replaceBlock` are idempotent.
- `fs.mergeJson` deep-merges objects and unions arrays. It never deletes user keys. If the existing JSON is invalid, it aborts with a fix message.
- `fs.delete` is allowed only inside `.bossbot/` and the worktree root, or after a confirmed command.
- Every executed effect may also emit a debug event when `config.debug` is on.

### 4.5 Roles
Detection order, decided in P0:
1. env `BOSSBOT_ROLE`
2. Agent Teams config (lead marker)
3. a `.bossbot-role` file in the working directory
4. the `/dashboard` command

The result is stored in `$.state.role`. Hubs filter contributions by role.

### 4.6 Session identity
- `self.worker` comes from `.bossbot-role`, which contains `{ name, task?, workspace }`.
- `self.projectRoot` comes from config discovery: walk up to find `.bossbot/config.json`. For worktrees, use the path recorded in `.bossbot-role`.

### 4.7 Data model

**Config file:** `.bossbot/config.json`
- Has `schemaVersion` and is validated against `schemas/config.schema.json`.
- Defaults are in Appendix A.
- Unknown keys produce a warning, never an error.

**Store keys** (`core/store.ts` builds them; no raw strings anywhere else):
```
c:<ph>:schema                 store schema version (integer)
c:<ph>:mode                   { mode, since }
c:<ph>:worker:<name>          { role, purpose, model, workspace, branch?, status, task?, health, restarts:[ts], paused? }
c:<ph>:task:<id>              { worker, text, dependsOn:[ids], status, createdAt, sentAt?, ackedAt?, repairs, phase, noTests? }
c:<ph>:taskseq                last task number (written by the lead only)
c:<ph>:queue:<id>             TASK held back for dependencies { to, text }
c:<ph>:hb:<name>              heartbeat ts
c:<ph>:lease:<relpath>        { worker, task, since }
c:<ph>:touch:<relpath>        { worker, task, at }
c:<ph>:test:<name>            { cmd, code, at }
c:<ph>:edit:<name>            last edit ts
c:<ph>:usage:<name>           { tokens, contextPct?, turns }
c:<ph>:recent:<name>          ring of last 5 { tool, argsHash, errHash }
c:<ph>:phase:current          { n, type, goal, startedAt }
c:<ph>:phase:<n>              finalized record
c:<ph>:preview:<name>         { status: 'clean'|'conflicts', files?, at }
c:<ph>:stale:<module>         true
c:<ph>:dashboard              { sessionId, since }        (authorizes CMD messages)
c:<ph>:counter:gitBlocked     integer
```
`<ph>` is a 10-character hash of the project root's absolute path.

**Event log:** `.bossbot/events/<YYYY-MM-DD>/<sessionId>.jsonl`
- **One file per session** means no write contention between sessions.
- Buffered and flushed every 5 s, or immediately for phase and mode events.
- Each line: `{ ts, v: 1, type, role, session, worker?, task?, phase?, data }`. Types are listed in Appendix E.
- Readers merge files by `ts`.
- Retention: `eventRetentionDays`.
- If `$.fs` has no append, write per-flush chunk files instead (`<sessionId>-<seq>.jsonl`).

**Files on disk:**

| Path | Writer | Notes |
|---|---|---|
| `.bossbot/config.json` | init, user | committed |
| `.bossbot/events/`, `.bossbot/journal/`, `.bossbot/cache/` | mod | gitignored |
| `<wikiDir>/**` | lead (and onboarding subagents for `modules/`) | committed |
| `<wikiDir>/logs/<name>/` | worker `<name>` only | absolute main-tree path, even from a worktree |
| `CLAUDE.md` block, `rules/core.md`, `.claude/settings.json` | init | managed block and merges only |
| `.worktrees/<name>/` or the path from `worktreeDir` | `/spawn` | gitignored |
| `.bossbot-role` | `/spawn` | inside each worktree (or a per-worker file in no-git mode, §7 F07) |

**Wiki in worktrees.** A worker's worktree contains a stale copy of `wiki/`. Workers must read the wiki and write logs **through the main tree's absolute path**. Writes to `<worktree>/<wikiDir>/**` are denied (F05).

### 4.8 Protocol (`core/protocol.ts`)

```
TASK <id>: <text> [after: T-1, T-2] [no-tests] [peer: <name>]     lead → worker
ACK <id>                                                          worker → lead (auto)
CMD <action> <json>                                               dashboard → lead
PAUSE <reason> | RESUME | CHECKPOINT                              lead → worker
```
- Parsing is tolerant: whitespace is flexible and verbs are case-insensitive. Unknown markers are kept in `text`.
- The format is unversioned in v1. If it ever needs a breaking change, add a `v2:` prefix and keep accepting v1 for one minor release.
- `TASK` without an id: the lead's outbound handler assigns the next `T-n` and rewrites the text.

### 4.9 Workspace modes
See F05–F09. Summary:
- **Git mode:** a worktree per worker; review, merge and drop are done with git.
- **No-git mode:** a shared folder, leases, a journal, git blocked, and merging replaced by journal review and revert.

### 4.10 Repository layout

```
bossbot/
├─ .claude-plugin/marketplace.json
├─ plugins/bossbot/
│  ├─ .claude-plugin/plugin.json
│  ├─ hooks/hooks.json                 # { "modules": ["./src/register.ts"] } or ./dist/register.js if bundled (P0 decides)
│  ├─ src/
│  │  ├─ register.ts                   # role detect → load config → wire hubs with enabled FEATURES
│  │  ├─ core/   host.ts types.ts ports.ts effects.ts store.ts config.ts protocol.ts events.ts
│  │  │          guard.ts paths.ts messages.ts migrations.ts viewmodel.ts ids.ts
│  │  ├─ hubs/   tool.ts message.ts command.ts lifecycle.ts scheduler.ts ui.ts prompt.ts
│  │  ├─ features/
│  │  │  ├─ index.ts                   # FEATURES list
│  │  │  └─ <feature-id>/ index.ts (Feature object) logic.ts (pure) README.md (5–15 lines)
│  │  └─ ui/     components.ts theme.ts
│  ├─ agents/     implementer.md tester.md reviewer.md researcher.md explorer.md
│  ├─ commands/   spawn.md phase-start.md phase-close.md resume-team.md onboard.md
│  ├─ templates/  claude-md-block.md rules-core.md log.md module-page.md wiki/ config.json
│  ├─ schemas/    config.schema.json
│  ├─ types/index.d.ts
│  └─ test/
│     ├─ unit/<feature>/*.spec.ts       # vitest, pure logic
│     ├─ plugin/<hub|feature>.test.ts   # claude plugin test
│     ├─ contract/2.1.304/*.json        # recorded raw event payloads
│     └─ fixtures/ ports.ts store.ts fs.ts clock.ts
├─ scripts/ bossbot-up.sh bossbot-up.ps1 new-feature.sh
├─ docs/ ARCHITECTURE.md EXTENDING.md API-MAP.md CONFIG.md EVENTS.md TROUBLESHOOTING.md SECURITY.md DEMO.md PROGRESS.md adr/
├─ examples/sample-project/
├─ .github/workflows/ci.yml
└─ README.md CHANGELOG.md CONTRIBUTING.md LICENSE
```

### 4.11 Code conventions
- TypeScript `strict`, ESLint and Prettier. No `any` without a justification comment.
- **Features never import other features.** Shared needs go into `core/`, and only when a second feature needs them.
- `logic.ts` has no `$` and no I/O apart from Ports. `index.ts` only assembles the `Feature` object.
- Files ≲ 200 lines. Functions do one thing and are named after their behavior.
- No magic numbers: every threshold is a config key with a schema entry and a default.
- Every user-facing string lives in `core/messages.ts`, in sentence case and active voice, and says what to do next.
- No dead code, no commented-out code, no TODO without an issue link.
- Conventional Commits. Semver.

### 4.12 Testing strategy

| Layer | Tool | What it covers |
|---|---|---|
| Unit | vitest | All `logic.ts`, protocol parsing, store key builders, stage derivation, health classification, path checks. Table-driven tests where possible. |
| Plugin | `claude plugin test` | Each hub with stub features. Each feature's real contributions through its hub, with stubs for every `$` call. |
| Contract | vitest | `host.ts` maps the recorded raw payloads in `test/contract/<version>/` to internal types. |
| UI | vitest snapshots | `Tab.render` and `BandSegment.render` with fixture ViewModels, for both surfaces. |
| Smoke | manual script per phase | Run on `examples/sample-project` and recorded in PROGRESS. |

Fixtures in `test/fixtures/` provide fake Ports (in-memory store and fs, controllable clock, scripted `run`).

### 4.13 Observability
- The event log (§4.7), plus `/bossbot events [n] [type]`, which prints the last n events with no model turn.
- With `config.debug = true`, executed effects and gate decisions are logged as `Debug*` events.
- The Status tab shows the last 20 events, the store schema version, the mode, and feature flags.

### 4.14 Versioning and migrations
- **Plugin:** semver. Any change to the store schema, config schema or protocol bumps the minor version, with a migration.
- **Store:** `migrations.ts` holds an ordered list `{ from, to, run(p): Effect[] }`.
  - The lead runs pending migrations on `start`, guarded by a `c:<ph>:migrating` key with a ts that expires after 60 s.
  - Workers and the dashboard check `schema === CURRENT`. If not, they toast: "Restart the lead to finish upgrading Bossbot."
- **Config:** `schemaVersion`, plus `config.migrate(raw)` for old shapes. The file is rewritten only after `/bossbot init` confirms.
- **Claude Code upgrade procedure** (documented in `docs/EXTENDING.md`):
  1. bump the pin
  2. regenerate types
  3. record new contract payloads
  4. run all tests
  5. fix only `host.ts` and the hubs
  6. update API-MAP

---

## 5. Extensibility

### 5.1 Rules
1. New behavior is a **feature contribution**, never a direct `on(...)` call. A new hub requires an ADR.
2. Every feature has a **flag**: `config.features.<id>`, defaulting to `defaultEnabled`. Experimental features default to `false`.
3. Features communicate **only** through store keys, domain events and protocol verbs, all of which are defined in `core/`.
4. New store keys, events, verbs and config keys are added in `core/` together with docs: `docs/EVENTS.md`, `docs/CONFIG.md`, and the protocol section of `ARCHITECTURE.md`.
5. `scripts/new-feature.sh <id>` scaffolds `features/<id>/{index.ts,logic.ts,README.md}`, a unit test and a plugin test, and adds the line to `FEATURES`.

### 5.2 Recipes (also in `docs/EXTENDING.md`, with a code sample each)

| To add… | Do this |
|---|---|
| A safety rule | `GateRule` in a feature. Pick an `order` (guards 100–199, leases/journal 200–299, gates 300–399, policies 400–499). |
| An argument rewrite | `Rewrite`. Make it idempotent (check the flag isn't already present). |
| Something triggered after a tool runs | `Observer`. Must be fail-open and fast. |
| A message verb | Add the verb to the `Verb` union and the parser in `core/protocol.ts`, then a `MessageHandler` or `OutboundHandler`. |
| A periodic task | `Job` with `everyMs`. Mark it `exclusive` if it shells out. |
| A slash command | `Command`. Use `immediate: true` if it should work while Claude is busy, and `confirm` if it's destructive. |
| A dashboard tab | `Tab` with a free hotkey (digit). Render only from the ViewModel, and extend `viewmodel.ts` if it needs new data. |
| A band item | `BandSegment` with a priority (lower priority is dropped first on narrow terminals). |
| A health or setup check | `DoctorCheck`. |
| A wizard question | `SetupStep`. |
| A new phase type | `PhaseType` with `StageRule`s built from domain events, plus a default template in config. |
| A worker role | `agents/<role>.md` (Appendix F rules). It shows up automatically in the spawn form's role list. |
| A rule with no code | `config.customRules` (§5.3). |

### 5.3 Declarative policies (`config.customRules`)

```json
{ "id": "no-prod-config", "tools": ["Edit","Write"], "roles": ["worker"], "match": { "path": "config/prod*.yaml" },
  "action": "deny", "message": "Production config is read-only for workers. Ask the lead." }
```
- **Match options:** `path` (glob, relative to the project root) and/or `command` (regex on the Bash command).
- **Actions:** `deny` or `warn`.
- Invalid patterns are skipped, reported by doctor, and announced with a toast once per session.
- **Defaults shipped:** workers can't read `.env*`, `*.pem`, `*.key` or `id_rsa*` (via Read or `cat`/`less`/`head` in Bash).

### 5.4 Future enhancements and their extension points

| Enhancement | Extension point |
|---|---|
| Shadow-git mode | New `Mode` value in `core/mode.ts` (ADR) + setup step + git port switch |
| Plan approval for risky tasks | `[plan]` TASK marker + GateRule that denies Edit/Write until a `PLAN-OK <id>` verb arrives |
| Decision-conflict check | Observer on wiki/log writes + a job comparing against `decisions.md` |
| Selective wiki context | Prompt hub context injection + a feature that picks pages by task keywords |
| Permanent reviewer worker | Role template + phase-close script variant |
| Dependency graph view | Tab using `Svg` (desktop) |
| Activity heatmap | Tab using `Raster` (terminal) |
| Desktop/OS notifications | Observer on `AckTimedOut`/`HealthChanged` → `$.process` notifier, behind a flag |
| Metrics export | Job that rolls events into `.bossbot/metrics.json` |
| Multi-project dashboard | Dashboard reading several `<ph>` namespaces (config `dashboard.projects`) |

---

## 6. End-user setup

```bash
claude plugin marketplace add <owner>/bossbot
claude plugin install bossbot@bossbot
cd my-project && claude
/bossbot init          # wizard (F02)
/bossbot doctor        # checks (F01)
/bossbot install-launcher && bossbot-up    # optional tmux layout (F18)
```

Side questions to the lead go through native `/btw`. It doesn't add to the conversation history, but it's single-turn and CLI-only. This is documented in the README.

---

## 7. Feature catalog

Each entry uses the same fields:
- **Phase**
- **Roles**
- **Contributes**
- **Behavior**
- **Data/Events**
- **Config**
- **UI**
- **Edge cases**
- **Tests**

Feature ids match the folder names.

### F01 `doctor`
- **Phase:** P0 · **Roles:** all · **Contributes:** Command `doctor` (immediate) and the base DoctorChecks. Other features add their own checks.
- **Behavior:** runs every check from enabled features and prints one line each: `✓` / `!` / `✗` + title + detail, plus `fix:` when failing. Ends with a summary count. No model turn.
- **Base checks:**
  - Claude Code version ≥ pin
  - Agent Teams env flag
  - plugin enabled at project scope
  - config present and valid
  - store schema current
  - role detection result
  - project root writable
  - wiki skeleton present
  - git state: none / no commits / dirty / ok
  - `git merge-tree --write-tree` support (git ≥ 2.38, needed for F25)
  - tmux (warn only; on Windows, explain in-process mode)
  - companion plugins installed
  - custom rules valid
  - feature `requires` satisfied
- **Edge cases:** no config yet → fail with `fix: run /bossbot init`. A check that throws → `✗ <title>: check crashed (<message>)`; the remaining checks keep running.
- **Tests:** each base check against fake Ports (ok/warn/fail); a crashing check is isolated; summary counts.

### F02 `setup` (init wizard + second brain)
- **Phase:** P1 · **Roles:** lead · **Contributes:** Command `init`; SetupSteps; DoctorChecks for managed files.
- **Steps, in order:**
  1. **Git state:** detect none / no commits / dirty / ok.
     - none → ask: *Use no-git mode* or *Initialize git* (runs `git init` + initial commit)
     - no commits → offer an initial commit
     - dirty → offer a snapshot commit `chore: bossbot snapshot`
  2. **Test command:** auto-detect (`pyproject`/`pytest.ini` → `pytest`; `package.json` scripts.test → `npm test`; `go.mod` → `go test ./...`; `Cargo.toml` → `cargo test`). Ask to confirm or edit; `none` is allowed.
  3. **Worker cap**, default 4.
  4. **Budgets:** per-worker tokens and onboarding tokens.
  5. **Onboarding depth:** quick / standard / deep.
  6. **Companion plugins:** list installed plugins and ask which to enable for all sessions (e.g. ponytail, caveman).
  7. **Write managed files** (idempotent):
     - `.bossbot/config.json`
     - `.bossbot/.gitignore` (`events/`, `journal/`, `cache/`)
     - the wiki skeleton from `templates/wiki/`
     - the `CLAUDE.md` block between `<!-- bossbot:start -->` and `<!-- bossbot:end -->` (lead rules, task format, "read `<wikiDir>/INDEX.md` first", compaction instructions, `@rules/core.md`)
     - `rules/core.md` (≤ `coreRulesMaxLines`)
     - `.claude/settings.json` merge: `env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS = "1"` and `enabledPlugins` (bossbot + chosen companions)
     - `.gitignore` additions: `.worktrees/`, `.bossbot/events/`, `.bossbot/journal/`, `.bossbot/cache/`
- **Rerun:** current values are the defaults. Unchanged answers produce zero writes. `init --step <id>` reruns one step.
- **Second brain layering** (documented in `rules/core.md` and the README):
  - always-on: `CLAUDE.md` block + `rules/core.md`, kept small
  - on demand: skills
  - enforced: gates and custom rules
- **Edge cases:**
  - invalid `settings.json` → abort that step with a fix message, never overwrite
  - `CLAUDE.md` missing → create it
  - read-only filesystem → fail with a clear message
  - user cancels mid-wizard → nothing is written (effects are collected first, then executed)
- **Tests:** idempotency (second run → zero effects); merge keeps unrelated keys and unions arrays; block replacement doesn't duplicate; test-command detection table; cancel writes nothing.

### F03 `templates` (agents + model-run commands)
- **Phase:** P1 · **Roles:** lead and workers (as content).
- **Agents** (`agents/*.md`) each contain:
  - role scope
  - the workspace rule (worktree path, or shared folder + leases)
  - "read the wiki and write logs at the absolute main-tree path"
  - task discipline: claim, then `in_progress`, then `done` or `blocked: <reason>`
  - the log rule and template (Appendix B)
  - the quiet-output habit
  - on `PAUSE`: stop and wait; on `CHECKPOINT`: write a wip log (and a WIP commit in git mode), then stop
  - never run git outside its own worktree
- **Roles:** implementer, tester, reviewer, researcher. Explorer is read-only and writes only `wiki/modules/<slice>.md`.
- **Commands:** spawn, phase-start, phase-close, resume-team and onboard. Exact steps are in Appendix F.
- **Edge cases:** plugin namespacing (e.g. `/bossbot:spawn`). Use whatever P0 finds, consistently, everywhere (README, messages, CMD mapping).
- **Tests:** a lint script checks each agent file has the required sections. Manual smoke covers the commands.

### F04 `acks`
- **Phase:** P2 · **Roles:** worker (auto-ack), lead (tracking).
- **Contributes:**
  - worker `MessageHandler(TASK)`
  - lead `OutboundHandler(TASK)` (assign id, record, hold back if dependencies are unmet — the hold-back is F28's)
  - lead `MessageHandler(ACK)`
  - lead `Job ackTimeouts` (every 5 s)
- **Behavior:**
  - **Worker:** on `TASK`, emit `send ACK <id>` to the sender, then pass the message through (the model reads it). Also write `.bossbot-role.task = id`.
  - **Lead:**
    - outbound TASK → `task:<id>` (`sentAt`, worker, text, markers), `TaskSent` event
    - ACK → `ackedAt`, `TaskAcked` event (`latencyMs`)
    - the job flags tasks with no ACK after `ackTimeoutSec` → toast, worker row status `warn`, `AckTimedOut` event (once per task)
- **Data/Events:** `task:<id>`, TaskSent, TaskAcked, AckTimedOut.
- **Config:** `ackTimeoutSec`.
- **UI:** row status `sent` / `acked` / `warn`; band "unacked n".
- **Edge cases:**
  - duplicate TASK → re-ack, idempotent
  - ACK for an unknown id → `Debug` event, ignored
  - the lead restarts while acks are pending → pending state is rebuilt from the store
  - a peer-to-peer TASK (with a `[peer]` marker) acks to the actual sender
- **Tests:** parse variants; auto-ack effect ordering (ACK before the message passes through); timeout fires once; unknown ACK ignored; id assignment when missing.

### F05 `workspace-guard` (git mode)
- **Phase:** P2 · **Roles:** worker · **Contributes:** GateRule `workspace` (order 100), GateRule `git-scope` (order 110).
- **Behavior:**
  - **Write/Edit/MultiEdit/NotebookEdit:** every target path is normalized (relative to cwd, `..` resolved, realpath when available) and must be inside the workspace **or** inside `<projectRoot>/<wikiDir>/logs/<self>/`.
  - **Denied:** `<workspace>/<wikiDir>/**` (stale copy), `.bossbot/**`, `.claude/**`.
  - **Bash (best effort):** split on `&&`, `||`, `;` and `|`. For each segment:
    - deny `cd` to a path outside the workspace
    - deny absolute paths outside the allowed roots when used as a write target (`>`, `>>`, `tee`, `cp`/`mv` destination, `rm`)
  - **`git-scope`:**
    - allow `git status|diff|log|add|commit|stash` inside the workspace
    - deny `checkout|switch|merge|rebase|reset --hard|push|pull|worktree|branch -D` and any `-C <path>` outside the workspace
- **Messages:** `denyOutsideWorkspace`, `denyWikiCopy`, `denyGitScope`.
- **Edge cases:** symlinks that escape the workspace → denied (if realpath is available; otherwise documented as a limit). MultiEdit → all paths must pass. Paths with spaces or quotes are handled by the tokenizer.
- **Tests:** a path table (inside, outside, `..`, absolute, symlink, wiki copy, log allowlist); a Bash table (redirects, chained commands, `cd`, git verbs); timeout → deny.

### F06 `git-block` (no-git mode)
- **Phase:** P2 · **Roles:** lead and worker · **Contributes:** GateRule (order 120); BandSegment `gitBlocked`.
- **Behavior:** in no-git mode, any Bash segment whose first word is `git` is denied with `denyGitNoRepo`, and `counter:gitBlocked` increments. The mod's own git calls skip the gate (they don't use the Bash tool).
- **Edge cases:** `GIT_DIR=… git` and `command git` count as git. A word that merely contains "git" (e.g. `digit`) does not.
- **Tests:** a segment table; the counter increments; git mode doesn't block.

### F07 `leases` (no-git mode)
- **Phase:** P2 · **Roles:** worker; lead (force-release) · **Contributes:** GateRule `lease` (order 200); Command `lease release <path>` (confirm); Job `leaseJanitor` (every 30 s).
- **Behavior:**
  - **Write/Edit** to a path:
    - no lease → set `{ worker, task, since }`, re-read, and if the value isn't ours, deny (race loss)
    - our lease → allow
    - another worker's lease → deny with `denyLeaseHeld`
  - **Release:** on TaskStatus `done` or `rejected` (observer), on worker `gone` (janitor), or by the lead's command.
  - **Identity in shared mode:** there is no worktree, so `.bossbot-role` can't live in the cwd. The worker's name comes from env `BOSSBOT_WORKER`, set at spawn. If that isn't possible, use `.bossbot/roles/<sessionId>.json`, resolved via `self.sessionId`. P0 decides.
- **Edge cases:** a new file creation takes a lease. Rename/move via Bash isn't covered (documented). The lease key uses the normalized relative path.
- **Tests:** acquire/hold/deny; race (simulated interleaved set) → exactly one wins; release paths; janitor releases leases of `gone` workers.

### F08 `journal` (no-git mode)
- **Phase:** P2 · **Roles:** worker (capture); lead (revert) · **Contributes:** GateRule `journal` (order 210, returns copy effects); Command `revert <task>` (confirm).
- **Behavior:**
  - **Capture:** on the first Write/Edit to a path within a task, before `next`:
    - if the file exists, `fs.copy` it to `.bossbot/journal/<task>/<relpath>`
    - otherwise append the path to `.bossbot/journal/<task>/_created.txt`
    - files larger than `journal.maxFileMB` → not copied, warn, `JournalSkipped` event
  - **Revert:** copy originals back, delete created files, release leases, emit a `RevertDone` event.
  - **Archive:** phase-close moves `journal/<task>` to `journal/_archive/<phase>/`.
- **Edge cases:**
  - revert when a file has since been changed by another task → impossible while leases hold; if a lease was force-released, ask before overwriting
  - Bash writes aren't journaled (documented in SECURITY and in the agent rules)
- **Tests:** first-touch copies once; created-file tracking; revert restores the exact bytes and deletes created files; size skip.

### F09 `modes`
- **Phase:** P2 · **Roles:** lead (watcher, switch); all (read) · **Contributes:** Job `modeWatch` (every 30 s); Command `git-init` (confirm); BandSegment `mode`; LifecycleHook `start`.
- **Behavior:**
  - **Detection:** `.git` exists **and** `git rev-parse --is-inside-work-tree` returns true, on two consecutive checks (debounce).
  - **nogit → git:** toast, then `$.ui.ask` "Switch to git mode?". On yes:
    - offer an initial commit if there are none
    - set `mode`
    - shared workers get `nextWorkspace: worktree` and finish their current task first
    - when no shared tasks remain: archive journals, clear leases, `ModeChanged` event
  - **git → nogit:** set `mode`, block git, pause the merge preview, toast. Worktrees are left on disk.
  - **`git-init`:** `git init`, initial commit (with excludes), then the switch flow.
- **UI:** band `⎇ git · <branch>` or `⚠ no git · leases <n>`. The pane header shows the mode plus a [Connect git] button.
- **Edge cases:** `.git` as a file (worktree or submodule) → resolved via rev-parse. A repo created inside a subfolder → ignored (the root must match).
- **Tests:** debounce; both switch directions with in-flight shared tasks; `git-init` effects.

### F10 `log-gate`
- **Phase:** P2 · **Roles:** worker · **Contributes:** GateRule (order 300) on the task-update tool (name from P0).
- **Behavior:** setting status `done` (or `wip` via CHECKPOINT) requires a file `<wikiDir>/logs/<self>/*-<taskId>-*.md` that has every Appendix B heading and at most `logMaxLines` lines. Otherwise deny with the specific gap ("missing ## Decisions", "42 lines, max 30").
- **Tests:** missing file; missing heading; too long; valid.

### F11 `test-gate`
- **Phase:** P2 · **Roles:** worker · **Contributes:** Observer `testRuns` (Bash), Observer `edits` (Write/Edit), GateRule (order 310).
- **Behavior:**
  - the observers record `test:<self>` `{ cmd, code, at }` when a Bash command matches `testCommand`, and `edit:<self>` on writes
  - **gate:** `done` requires `test.code === 0 && test.at > edit`
  - in no-git mode the gate warns instead of denying
  - skipped when the task has `[no-tests]`, or when `testCommand` is null (doctor warns)
- **Edge cases:** the exit code must survive quiet rewrites (F12 guarantees `pipefail`). A test command run from a subfolder still matches by prefix.
- **Tests:** a pass/fail/stale-after-edit table; `no-tests` marker; warn-only mode.

### F12 `quiet-output`
- **Phase:** P2 · **Roles:** worker · **Contributes:** Rewrite (order 10).
- **Behavior:**
  - for commands matching a `quietCommands` key, append its flags if they aren't already present
  - for `tailCommands`, wrap as `set -o pipefail; <cmd> 2>&1 | tail -n <tailLines>` so the exit code is preserved
  - never rewrite a command that already contains a pipe or redirect
- **Tests:** idempotent; pipefail present; commands with pipes are left alone.

### F13 `policies`
- **Phase:** P2 · **Roles:** per rule · **Contributes:** GateRules generated from `customRules` (order 400+); DoctorCheck. Default secret rules (§5.3).
- **Tests:** glob and regex matching; invalid rule skipped and reported; role filter.

### F14 `band`
- **Phase:** P2 (base), with segments added in later phases · **Roles:** lead · **Contributes:** the `AbovePrompt` render through the UI hub.
- **Behavior:**
  - composes segments by `order`
  - when the line is too wide, drops the lowest-priority segments first
  - values come from `$.state`, written by jobs and handlers, so redraws happen automatically
- **Base segments:** mode (F09), counts (workers / busy / attention / unacked), gitBlocked (F06).
- **Later segments:** pipeline (F26), budget (F29).
- **Tests:** composition snapshots at 60, 100 and 160 columns.

### F15 `team-pane` (compact, in the lead)
- **Phase:** P2 · **Roles:** lead · **Contributes:** Command `team` (alias `/team`, immediate); a Tab rendered in a lead pane.
- **Behavior:**
  - **Pinned lead row**, then worker rows: name, role, task, status, and branch or "shared · holds n files".
  - **Row buttons:** send (Input → outbound TASK), stop (confirm → `prompt.submit /spawn`-family stop instruction), log (Markdown view).
  - **`+` form:** Input name (validated `[a-z][a-z0-9-]{1,15}` and unique), Select role (from `agents/`), Input purpose.
  - **Submit:** cap check (F30), then `prompt.submit('/spawn …')`.
- **Edge cases:** an empty roster shows "No workers yet. Press + to add one." A narrow terminal puts the pane above the prompt (handled natively).
- **Tests:** render snapshots; form validation; cap refusal produces no submit effect.

### F16 `dashboard`
- **Phase:** P3 (shell + Team/Status/Logs/Setup), P5 (Phase tab), P6 (Wiki tab) · **Roles:** dashboard.
- **Contributes:** Command `dashboard`; Job `snapshot` (every `dashboardPollMs`; builds the ViewModel, stores it in a module cache, then `invalidate`); Tabs; LifecycleHook (registers `c:<ph>:dashboard`).
- **Tabs:**

  | Tab | Hotkey | Contents |
  |---|---|---|
  | Team | 1 | roster with health, tokens, branch/leases, actions (send, stop, restart, release leases, log) |
  | Phase | 2 | per-worker lanes, stage times, tokens lead/workers, repairs |
  | Wiki | 3 | tree, stale markers, preview, onboarding confirm |
  | Logs | 4 | logs by worker, Markdown view |
  | Status | 5 | git log or lease table, last 20 events, schema version, mode, feature flags |
  | Setup | 6 | config summary, rerun-step buttons, run doctor |

- **Behavior:**
  - the prompt hub drops every prompt with a toast
  - actions: send → `send` directly to the worker; everything else → `CMD` to the lead (F17)
- **Surfaces:** Phase lanes are an `Svg` on desktop and text in the terminal.
- **Edge cases:**
  - no lead running → banner "Lead not running. Start it to act on the team."; read-only views still work
  - two dashboards → the second registers itself and the lead accepts CMD from the latest one
- **Tests:** a no-model-call assertion (no `prompt.submit` or `model.*` effects or calls); tab snapshots for both surfaces; actions map to the right effects.

### F17 `cmd`
- **Phase:** P3 · **Roles:** lead · **Contributes:** `MessageHandler(CMD)`.
- **Behavior:**
  - accept only from the address in `c:<ph>:dashboard`
  - the action must be on the whitelist: `spawn`, `stop`, `restart`, `phase-start`, `phase-close`, `onboard`, `release-lease`, `revert`, `resume`
  - args are validated against a per-action schema
  - **deterministic actions** run as effects: release-lease, resume, and revert (after confirm)
  - **model actions** become `prompt.submit` of the matching slash command
  - every CMD emits a `CmdReceived` event
- **Tests:** unauthorized sender rejected; unknown action rejected; arg validation; mapping table.

### F18 `launcher`
- **Phase:** P3 · **Contributes:** `scripts/bossbot-up.sh`, `scripts/bossbot-up.ps1`, Command `install-launcher`.
- **Behavior (sh):**
  - check tmux; reattach if session `bossbot-<dir>` exists
  - otherwise create it: lead pane (`BOSSBOT_ROLE=lead claude`), and a dashboard pane on the right at 45% (`BOSSBOT_ROLE=dashboard claude --model <dashboard model> '/dashboard'`)
  - Agent Teams adds the worker panes itself (tmux display mode)
- **Behavior (ps1):** Windows Terminal `wt` split-pane with the same env; workers run in-process (documented).
- **`install-launcher`:** copies the script to `~/.local/bin/bossbot-up` (or `%USERPROFILE%\bin`) and prints a PATH hint if needed.
- **Tests:** shellcheck in CI; a dry-run flag that prints the tmux commands.

### F19 `heartbeat`
- **Phase:** P4 · **Roles:** worker · **Contributes:** writes from the lifecycle hub on `turn.step` and the tool hub on every call.
- **Behavior:** write `hb:<self>` at most every `heartbeat.writeThrottleSec`. Also record `usage:<self>` when the build exposes usage (P0).
- **Tests:** the throttle table.

### F20 `health`
- **Phase:** P4 · **Roles:** lead · **Contributes:** Job `health` (every 10 s); Command `restart <name>`; BandSegment `attention`.
- **Behavior:**
  - **Classifier** (pure): `gone` (session ended or the process isn't listed by Agent Teams) > `unresponsive` (no heartbeat for more than `unresponsiveMin` while a task is assigned) > `busy-long` (one tool running longer than `busyLongMin`) > `healthy`
  - a change emits a `HealthChanged` event
  - **auto-restart** on `unresponsive` when fewer than `restart.maxAutoPerHour` restarts happened in the last hour → `prompt.submit('/bossbot:resume-team <name>')`
  - otherwise `blocked: unhealthy` + toast
- **Tests:** classifier table; restart cap; Ctrl+Z simulation (stale heartbeat) → unresponsive.

### F21 `reanchor`
- **Phase:** P4 · **Roles:** worker · **Contributes:** LifecycleHook `compact`.
- **Behavior:** after compaction, inject context with: current task id and text, latest log path, workspace path, mode, and "continue from the log's Next step". Exact mechanism: the context injection of `classic.SessionStart`, or the prompt hub on the next prompt (P0 decides).
- **Tests:** context content snapshot; no injection when there's no task.

### F22 `handoff`
- **Phase:** P4 · **Roles:** lead (trigger), worker (gate) · **Contributes:** Job `handoffWatch`; worker handlers for `CHECKPOINT`, `PAUSE` and `RESUME`; GateRule `paused` (order 50).
- **Behavior:**
  - **Trigger:** `contextPct ≥ handoff.contextPct`, or `turns ≥ handoff.turnsFallback` if context % isn't exposed → send `CHECKPOINT`
  - **On CHECKPOINT,** the worker's model:
    - writes a wip log (the log gate accepts `wip`)
    - makes a WIP commit (git mode)
    - marks the task `wip`
  - **Then the lead:** stops the worker and respawns it fresh via resume-team (the wip log is preferred)
  - **PAUSE (deterministic):** the worker gate denies every tool call except writes to its own log folder, with "Paused by lead: <reason>. Write your log if needed, then wait for RESUME." `RESUME` clears it.
- **Tests:** trigger thresholds; paused gate allow-list; resume.

### F23 `stuck`
- **Phase:** P4 · **Roles:** worker (observe), lead (react) · **Contributes:** Observer `recent` (all tools); Job `stuckScan`.
- **Behavior:**
  - a ring of the last 5 `{ tool, argsHash, errHash }`
  - `repeatThreshold` identical failing entries in a row → worker status `blocked: loop`, a `StuckDetected` event, a lead toast, and `PAUSE loop` to the worker
- **Tests:** ring logic; threshold; successful repeats don't trigger.

### F24 `overlap` (git mode)
- **Phase:** P4 · **Roles:** worker (record), lead (react) · **Contributes:** Observer `touch` (Write/Edit); Job `overlapScan`.
- **Behavior:**
  - record `touch:<relpath>` (relative to the workspace root, mapped to a project-relative path)
  - a second worker touching the same path → `OverlapDetected` event, toast, both rows flagged `overlap`
  - cleared on merge or drop of either branch
- **Tests:** detection; clearing; same worker repeated doesn't trigger.

### F25 `merge-preview` (git mode)
- **Phase:** P4 · **Roles:** lead · **Contributes:** Job (`mergePreviewSec`, exclusive).
- **Behavior:**
  - for each worker branch: `git merge-tree --write-tree main w/<name>`; exit code 0 → clean, 1 → conflicts (files parsed from the output)
  - write `preview:<name>`
  - skipped in no-git mode or when git is older than 2.38 (doctor warns)
- **Tests:** parsing of clean and conflict outputs (fixtures); exclusive overlap prevention.

### F26 `pipeline`
- **Phase:** P5 · **Roles:** lead (derive), dashboard (view) · **Contributes:** PhaseTypes (onboard, feature, bugfix, refactor); Job `stageDerive` (every 5 s); BandSegment `pipeline`; Phase Tab (F16).
- **Behavior:**
  - `/phase-start <type> "<goal>"` (model command) writes `phase:current` and emits `PhaseStarted`
  - stages come only from events, via the `StageRule`s:

    | Stage | Active / done when |
    |---|---|
    | Plan | from PhaseStarted until the first TaskSent |
    | Build / Fix / Change | while any TaskStatus is `in_progress` |
    | Verify | test-gate passes counted per task; done when every task in the phase has passed |
    | Review | from PhaseCloseStarted |
    | Repair | while a TaskResent exists without a later done |
    | Integrate | MergeDone / RevertDone events |
    | Record | WikiUpdated until PhaseClosed |
    | Reproduce (bugfix) | the first failing TestRun recorded before any edit |
    | Baseline tests (refactor) | a passing TestRun before the first edit |

  - stage transitions emit `PhaseStageChanged` with timings
  - `/phase-close` writes stage times, tokens (lead/workers) and repairs to `<wikiDir>/phases/<n>.md`
- **UI:** band `Phase n · type  Plan ✓ › Build 2/3 › Verify › … ↺n`; dashboard lanes.
- **Tests:** a table-driven derivation over scripted event sequences for each type; a custom phase type from config loads.

### F27 `repairs`
- **Phase:** P5 · **Roles:** lead · **Contributes:** OutboundHandler hook-in (a TASK re-sent for a rejected task → `repairs++`, `TaskResent` event).
- **Behavior:** at `repairs ≥ 2` → toast `toastRepairs`, and the row shows ↺n.
- **Tests:** counting; threshold toast fires once.

### F28 `deps`
- **Phase:** P5 · **Roles:** lead · **Contributes:** OutboundHandler (TASK with `[after: …]`); Observer/Job `release` (on TaskStatus done).
- **Behavior:**
  - outbound TASK whose dependencies aren't done → don't send; store `queue:<id>`; `TaskQueued` event; toast "T-x queued until T-y is done."
  - when a task completes, every queue entry whose dependencies are now all done is sent (no lead turn) and `TaskReleased` is emitted
  - cycle or unknown id → reject at queue time with a message to the lead
- **Tests:** release on the last dependency; multi-dependency; cycle rejection; queue survives a lead restart.

### F29 `meter`
- **Phase:** P5 · **Roles:** lead · **Contributes:** Job `budget`; BandSegment `budget`; Team tab column.
- **Behavior:**
  - totals from `usage:<name>`
  - at ≥ 80% of `budgets.workerTokens` → a warn toast, once
  - at ≥ 100% → `PAUSE budget` + `BudgetExceeded` event
  - `/bossbot budget <name> +<tokens>` raises the limit (confirm)
  - if usage isn't exposed (P0), the feature disables itself and doctor reports it
- **Tests:** thresholds; raise; disabled path.

### F30 `cap`
- **Phase:** P2 · **Roles:** lead, dashboard · **Contributes:** a pure `canSpawn(p)` used by F15 and F17, plus a doctor check.
- **Behavior:** counts workers whose status isn't `stopped` or `gone`. At or over the cap → `toastTeamFull`, no submit.
- **Tests:** counting rules.

### F31 `onboard-scan`
- **Phase:** P6 · **Roles:** lead · **Contributes:** Command `scan` (also the first step of onboarding).
- **Behavior:** writes `<wikiDir>/_scan.json` with no model:
  - tree (respecting `excludes`, depth ≤ 6)
  - languages by line count (extension map)
  - manifests found
  - entry-point candidates (main files, `bin`, scripts, `__main__`, `cmd/`)
  - test folders
  - top 20 largest files
  - git hotspots (`git log --since=6.months --name-only` counts) in git mode
- **Edge cases:** huge repos → cap the file walk at `onboard.maxFiles` and mark the scan `partial`.
- **Tests:** fixture repo → expected JSON.

### F32 `onboard-run` (model)
- **Phase:** P6 · **Roles:** lead · **Contributes:** `commands/onboard.md` (Appendix F); budget tracking under the `onboard` key.
- **Depth:**
  - **quick:** scan → `/init` → overview + INDEX by the lead
  - **standard:** adds slicing (3–6 slices) → explorer subagents per slice → synthesis
  - **deep:** adds flows (Mermaid) and verification (10 concrete claims)
- **Explorers** are read-only subagents (built-in Explore if present, else `agents/explorer.md`). Each writes only its module page and returns ≤ 5 lines.
- **Tests:** manual smoke (§8 P6). The command file passes the structure lint.

### F33 `onboard-confirm`
- **Phase:** P6 · **Roles:** dashboard (Wiki tab), lead · **Contributes:** Wiki tab actions [Accept] / [Correct: Input].
- **Behavior:**
  - Correct → CMD to the lead → the lead applies the correction
  - Accept → `phase:0` recorded, `OnboardConfirmed` event
- **Tests:** action mapping.

### F34 `staleness`
- **Phase:** P6 · **Roles:** lead · **Contributes:** Job `staleScan`; Wiki tab markers.
- **Behavior:**
  - module pages declare `source:` (a commit, or an mtime snapshot hash) and `paths:` (globs)
  - any `touch:` matching a module's globs → `stale:<module>`
  - `/phase-close` refreshes only stale pages, then clears the flags
- **Tests:** glob mapping; clear on refresh.

### F35 `status`
- **Phase:** P3 · **Roles:** lead · **Contributes:** Command `status` (alias `/status`, immediate).
- **Behavior:** prints the roster, pending acks, the phase line, preview results, and the last 10 commits (git) or the lease table (no-git), as command output. No model turn.

### F36 `teardown`
- **Phase:** P7 · **Roles:** lead · **Contributes:** Command `teardown` (confirm, lists what will be removed first).
- **Behavior:**
  - after confirmation: stop workers, remove worktrees and branches `w/*` (git), journals, events, and every `c:<ph>:*` store key
  - keep the wiki, config and managed blocks (`--purge` also removes the managed blocks and config, after a second confirm)
- **Tests:** effect list per mode; second confirmation for purge.

### F37 `events`
- **Phase:** P0 (core writer), P3 (command + Status tab) · **Contributes:** `core/events.ts` writer; Command `events [n] [type]`.
- **Tests:** flush rules; chunk fallback; merge-read ordering.

---

## 8. Phases

Each phase lists:
- **Goal**
- **Depends on**
- **Features**
- **Steps**
- **Deliverables**
- **Smoke script**
- **Acceptance**
- **Risks**

### P0 — Verify, scaffold, core
- **Goal:** answer every API unknown, then build the skeleton every feature plugs into.
- **Depends on:** nothing.
- **Features:** core (§4.1–§4.7), all hubs (empty dispatch), F01 `doctor`, F37 event writer.
- **Steps:**
  1. Read the docs and generated types. Write `docs/API-MAP.md` covering every item below as ✅ or ❌, with evidence (type name or doc link) and a fallback:
     - static multi-file imports (if not supported: esbuild bundle to `dist/register.js`, and `validate` must pass on the bundle)
     - passing `$` into core helpers (`makePorts($)`) without validator complaints
     - `command.run` arguments
     - env access
     - `$.ui.ask`
     - whether `$.session.send` reaches teammates and independent sessions
     - whether `session.send` can consume or rewrite outbound messages
     - task-update tool name and args
     - post-call result access in `tool.call` (vs `classic.PostToolUse`)
     - token usage and context % exposure
     - teammate spawn API
     - plugin command and agent namespacing
     - whether plugin agents and commands are visible inside teammate sessions
     - built-in Explore subagent
     - context-injection mechanism
     - `classic.SessionStart` sources
     - `$.fs` append support
     - `$.store` write latency (measure 100 writes)
  2. Scaffold §4.10. Configure tsconfig (strict), ESLint, Prettier, vitest, and CI (validate, plugin test, unit, lint, typecheck, shellcheck).
  3. Implement `core/` (types, host, ports, effects, store keys, config load + schema, protocol parser, messages, guard, paths, ids, migrations framework at schema 1, event writer) and every hub with dispatch over an empty `FEATURES`.
  4. Implement F01 and `version`.
  5. `scripts/new-feature.sh`, plus `docs/ARCHITECTURE.md` and `docs/EXTENDING.md` (recipes from §5.2, each with a code sample).
- **Deliverables:** an installable plugin that does nothing harmful, `/bossbot doctor` working, docs.
- **Smoke:**
  1. `claude --plugin-dir plugins/bossbot` in a git repo, a repo with no commits, and a non-git folder → run `/bossbot doctor`
  2. `claude plugin validate`
- **Acceptance:**
  - doctor output is correct in all three folders
  - validate is clean
  - CI is green
  - API-MAP is complete
  - a sample feature created with `new-feature.sh` runs through its hub in a plugin test, then is deleted
- **Risks:** the validator rejects the Ports pattern → fall back to hubs passing `$` to a small set of named host functions. Record in an ADR.

### P1 — Init and templates
- **Goal:** turn any folder into a Bossbot project. Make manual orchestration work before any automation exists.
- **Depends on:** P0.
- **Features:** F02 `setup`, F03 `templates`.
- **Steps:**
  1. SetupSteps 1–7 with effects collected first, then executed.
  2. Templates: agents, commands (Appendix F), CLAUDE.md block, `rules/core.md`, wiki skeleton.
  3. A structure lint for agents and commands, run in CI.
- **Smoke:**
  1. Run `init` in a fresh repo, a dirty repo, an empty repo and a non-git folder.
  2. Run `init` again and verify zero writes.
  3. In the git repo: `/spawn api implementer "…"`, then `/spawn tests tester "…"`. The lead sends a TASK to each, the workers complete their tasks and write logs, and `/phase-close` merges one and rejects the other with a follow-up TASK.
- **Acceptance:** idempotency; settings merge keeps existing keys; the manual orchestration flow works end to end.
- **Risks:** command namespacing differs from expectations → update messages and README in one pass.

### P2 — Coordination core
- **Goal:** deterministic coordination and safety in both modes.
- **Depends on:** P1.
- **Features:** F04, F05, F06, F07, F08, F09, F10, F11, F12, F13, F14 (base), F15, F30.
- **Steps (each step includes its unit and plugin tests):**
  1. acks (F04)
  2. workspace guard and git scope (F05)
  3. modes (F09)
  4. git block (F06)
  5. leases and journal (F07, F08)
  6. gates (F10, F11) and quiet output (F12)
  7. policies (F13)
  8. band (F14) and team pane (F15) with cap (F30)
  9. measure the §3.4 budgets, recorded in PROGRESS
- **Smoke:**
  1. Git mode: 2 workers. One tries to edit outside its worktree (denied), one tries `done` without a log (denied), then both complete.
  2. Copy the sample without `.git`: 2 workers. A git command is blocked. A lease conflict is denied. A rejected task is reverted.
  3. Run `/bossbot git-init` mid-run → mode switches after the in-flight shared task completes.
  4. Simulate an ack timeout by sending to a stopped worker → toast.
- **Acceptance:** all plugin tests pass; performance budgets are met; smoke passes; the band reflects every state above.
- **Risks:** Bash parsing gaps → keep them documented as best effort, and rely on F13 policies plus the agent rules.

### P3 — Dashboard
- **Goal:** a full monitoring and control surface outside the lead session.
- **Depends on:** P2.
- **Features:** F16 (shell + Team/Status/Logs/Setup), F17, F18, F35, F37 (command + tab).
- **Steps:**
  1. Dashboard role and prompt drop
  2. Snapshot job and ViewModel
  3. Tabs
  4. CMD handler with authorization
  5. Launcher scripts and `install-launcher`
  6. `/status` and `/bossbot events`
- **Smoke:**
  1. `bossbot-up` → layout appears. Spawn from the dashboard. Send a task from the dashboard. Stop a worker. Release a lease (no-git).
  2. Type a prompt in the dashboard → dropped with a toast.
  3. Kill the lead → the dashboard shows its banner.
- **Acceptance:** the no-model-call test passes; actions reach their targets; redraw ≤ 2 s; Windows script works with in-process workers.
- **Risks:** pane height control differs by terminal → fall back to `rows` defaults and document it.

### P4 — Reliability
- **Goal:** survive crashes, suspends, compaction, loops and conflicts.
- **Depends on:** P3.
- **Features:** F19, F20, F21, F22, F23, F24, F25.
- **Steps:**
  1. heartbeat (F19)
  2. health and restart (F20)
  3. reanchor (F21)
  4. handoff, pause and resume (F22)
  5. stuck detection (F23)
  6. overlap detection (F24)
  7. merge preview (F25)
  8. update resume-team for wip logs
- **Smoke (chaos script in `docs/DEMO.md`):**
  1. kill a worker process
  2. Ctrl+Z a worker for 6 minutes
  3. force `/compact` in a worker mid-task
  4. script a failing command loop
  5. two workers edit the same file
  6. create a conflicting branch
- **Acceptance:** every smoke case produces the specified state, toast and recovery; at most one auto-restart per hour.
- **Risks:** context % not exposed → the turn-count fallback is active; documented.

### P5 — Workflow and cost
- **Goal:** visible phases, automatic sequencing, budget control.
- **Depends on:** P4.
- **Features:** F26, F27, F28, F29, plus the F14 pipeline and budget segments and the F16 Phase tab.
- **Steps:**
  1. Phase types and stage derivation
  2. `/phase-start`, plus the phase-close record
  3. repairs (F27)
  4. dependency queue and release (F28)
  5. meter and budgets (F29)
  6. Phase tab (Svg on desktop, text in the terminal)
- **Smoke:** a feature phase with 3 tasks (T-3 after T-1), one rejection and a repair. The band and Phase tab show the correct stages and ↺1. T-3 auto-sends when T-1 is done. A low budget triggers PAUSE.
- **Acceptance:** the derivation table tests pass; the smoke matches expected stage times in the phase record.
- **Risks:** event ordering across sessions → the merge-read sorts by `ts`, and stage rules tolerate up to 2 s of skew.

### P6 — Onboarding
- **Goal:** a trustworthy baseline wiki for an existing project.
- **Depends on:** P5.
- **Features:** F31, F32, F33, F34, F16 Wiki tab.
- **Steps:**
  1. scan
  2. `onboard.md` for all three depths
  3. explorer agent
  4. synthesis templates
  5. verification
  6. confirm flow
  7. staleness (`paths:` globs in module pages)
  8. auto-offer on the first `/team` when INDEX is empty
- **Smoke:**
  1. Run deep onboarding on a 20–50k LOC open-source repo.
  2. Inject one wrong claim into a module page → verification flags it.
  3. Accept.
  4. Edit one module, then `/phase-close` → only that page refreshes.
- **Acceptance:** all pages are produced and match their templates; budget respected; verification report present; staleness is correct.
- **Risks:** explorer output quality varies → the template lint plus verification plus the user confirm gate together are the mitigation; never skip confirm.

### P7 — Release
- **Goal:** anyone can install and use it from the README alone.
- **Depends on:** P6.
- **Features:** F36, docs, packaging.
- **Steps:**
  1. `README.md`: what it is, a GIF, a 30-second quickstart (§6), commands, a modes table, `/btw` tip, uninstall.
  2. `CONFIG.md` generated from the schema; `EVENTS.md` generated from the event types.
  3. `TROUBLESHOOTING.md` generated from the doctor checks.
  4. `SECURITY.md`: mods are unsandboxed; exact reads, writes and processes; `--safe-mode`; the default policies.
  5. `examples/sample-project` and `DEMO.md`.
  6. teardown (F36)
  7. `marketplace.json`, `1.0.0` tag, CHANGELOG summary, pinned-version note.
- **Smoke:** a clean machine (or container) follows the README only, once with a git project and once with a non-git project, then runs teardown.
- **Acceptance:** zero undocumented steps; teardown leaves only the wiki and config.

---

## 9. Out of scope (v1)
- custom message bus
- Agent SDK rewrite
- nested teams
- live worker terminals embedded in panes
- peer broadcast chat
- workers editing the wiki outside their own logs
- a vector database
- the lead auto-spawning workers without user action
- LLM calls in hooks
- shadow-git mode (§5.4)

## 10. Definition of done (every phase)
1. `claude plugin validate`, `claude plugin test`, unit, contract, lint, typecheck and shellcheck all pass locally and in CI.
2. New logic is pure and unit-tested. Every contribution is plugin-tested through its hub.
3. New store keys, events, verbs and config keys are documented and have schema entries and defaults.
4. No user-facing string outside `messages.ts`.
5. Performance budgets checked where the phase touches a hot path.
6. API-MAP, PROGRESS (Appendix G), CHANGELOG and README sections updated. ADRs for architecture changes.
7. The phase smoke script has been run and its results recorded.

---

## Appendix A — Config defaults (`.bossbot/config.json`)

```json
{
  "schemaVersion": 1,
  "maxWorkers": 4,
  "models": { "lead": "opus", "worker": "sonnet", "explorer": "sonnet", "dashboard": "haiku" },
  "testCommand": null,
  "wikiDir": "wiki",
  "worktreeDir": ".worktrees",
  "companionPlugins": [],
  "features": {},
  "budgets": { "workerTokens": 400000, "onboardTokens": 600000, "warnPct": 80 },
  "ackTimeoutSec": 60,
  "heartbeat": { "writeThrottleSec": 15, "busyLongMin": 5, "unresponsiveMin": 5 },
  "restart": { "maxAutoPerHour": 1 },
  "stuck": { "repeatThreshold": 3, "ringSize": 5 },
  "handoff": { "contextPct": 70, "turnsFallback": 60 },
  "mergePreviewSec": 120,
  "dashboardPollMs": 1500,
  "wiki": { "maxPageLines": 200, "logMaxLines": 30, "coreRulesMaxLines": 80 },
  "journal": { "maxFileMB": 20 },
  "quietCommands": { "pytest": "-q", "npm test": "--silent" },
  "tailCommands": [],
  "tailLines": 80,
  "excludes": ["node_modules", ".venv", "dist", "build", "__pycache__", ".bossbot", ".worktrees"],
  "onboard": { "depth": "standard", "maxFiles": 20000, "slices": [3, 6], "verifyClaims": 10 },
  "eventRetentionDays": 14,
  "debug": false,
  "customRules": [],
  "phaseTemplates": {
    "onboard":  ["Scan", "Explore", "Synthesize", "Verify", "Confirm"],
    "feature":  ["Plan", "Build", "Verify", "Review", "Repair", "Integrate", "Record"],
    "bugfix":   ["Reproduce", "Fix", "Verify", "Review", "Integrate", "Record"],
    "refactor": ["Baseline tests", "Change", "Verify", "Review", "Integrate", "Record"]
  }
}
```

## Appendix B — Worker log (`<wikiDir>/logs/<name>/<YYYYMMDD-HHMM>-<taskId>-<slug>.md`, ≤ 30 lines)

```
status: done | wip
## Task
## Changed (file: one-line why)
## Decisions (choice — reason)
## Open issues / risks
## Next suggested step
```

## Appendix C — Module page (`<wikiDir>/modules/<slice>.md`)

```
source: <commit | mtime-hash>   verified: yes | no | partial   paths: ["src/billing/**"]
## Purpose
## Entry points
## Key files
## Dependencies (internal / external)
## Data in / out
## Invariants
## How to test
## Known issues
## Open questions
```

## Appendix D — Messages (seed for `core/messages.ts`)

| Id | Text |
|---|---|
| denyOutsideWorkspace | This path is outside your workspace. Edit `<workspace>/<path>` instead. |
| denyWikiCopy | This is a stale copy of the wiki. Read and write `<projectRoot>/<wikiDir>/…` instead; you may only write your own log folder. |
| denyGitScope | Only the lead runs `<verb>`. Commit inside your worktree and report back. |
| denyGitNoRepo | This folder isn't a git repo, so git is disabled. Continue without git. The lead will switch modes if a repo is connected. |
| denyLeaseHeld | File held by `<worker>` until `<task>`. Ask the lead or pick other work. |
| denyMissingLog | Write your log at `<path>` first (`<gap>`), then mark the task done. |
| denyTestsNotPassing | Run `<testCommand>` in your workspace and get it passing after your last edit, then mark `<task>` done. |
| denyPaused | Paused by lead: `<reason>`. Write your log if needed, then wait for RESUME. |
| denyGuardTimeout | Guard check timed out. Retry the call. |
| toastAckTimeout | No ACK from `<worker>` for `<task>` after `<n>` s. Check the session or reassign the task. |
| toastTeamFull | Team is full (`<n>` of `<max>`). Stop a worker to add `<name>`. |
| toastRepairs | `<task>` has bounced twice. Rewrite the task or check its assumptions. |
| toastQueued | `<task>` queued until `<deps>` are done. |
| toastGitDetected | Git repo detected. Switch to git mode? |
| toastOverlap | `<a>` and `<b>` both edited `<path>`. Expect a conflict at merge. |
| toastStuck | `<worker>` repeated a failing call `<n>` times and was paused. |
| toastBudget | `<worker>` used `<pct>`% of its token budget. |
| toastDashboardNoPrompt | Dashboard session: use the lead pane to talk to Claude. |
| toastUpgrade | Restart the lead to finish upgrading Bossbot. |

## Appendix E — Domain events (`core/events.ts`, documented in `docs/EVENTS.md`)

| Area | Events |
|---|---|
| Workers | WorkerSpawned, WorkerStopped, HealthChanged, Restarted |
| Tasks | TaskCreated, TaskSent, TaskAcked, AckTimedOut, TaskStatusChanged, TaskRejected, TaskResent, TaskQueued, TaskReleased |
| Gates and files | GateDenied, GateWarned, LeaseTaken, LeaseReleased, FileTouched, JournalSkipped, OverlapDetected, StuckDetected |
| Runs and integration | TestRun, LogWritten, MergeDone, RevertDone, PreviewUpdated |
| Wiki | WikiUpdated, PageStale |
| Phases | PhaseStarted, PhaseStageChanged, PhaseCloseStarted, PhaseClosed |
| Mode and cost | ModeChanged, BudgetWarned, BudgetExceeded, CheckpointRequested, Paused, Resumed |
| Onboarding | OnboardStep, OnboardConfirmed |
| Control | CmdReceived |
| Debug | DebugEffect, DebugDecision |

## Appendix F — Model-run command scripts

**Agent template rules:** each `agents/*.md` must contain these headings: Role, Scope, Workspace, Wiki and logs, Task discipline, Logging, Quiet output, Control messages (PAUSE/RESUME/CHECKPOINT), Never.

**`spawn.md`** — args: `<name> <role> "<purpose>"`
1. Read `.bossbot/config.json` and run the cap check; refuse if at the cap.
2. **Git mode:** `git worktree add <worktreeDir>/<name> -b w/<name>` from `main`, then write `<worktree>/.bossbot-role`.
   **No-git mode:** register the worker identity per the F07 decision.
3. Spawn the teammate from `agents/<role>.md` on `models.worker`, with the purpose, workspace path, absolute wiki path and mode injected.
4. Upsert the row in `<wikiDir>/roster.md`.
5. Report one line.

**`phase-start.md`** — args: `<type> "<goal>"`
1. Confirm no phase is open.
2. Write `phase:current` via `/bossbot phase open` (mod command, deterministic).
3. Break the goal into tasks with ids, workers and `[after:]` markers.
4. Send the TASKs.

**`phase-close.md`**
1. Run `/bossbot phase closing` (emits PhaseCloseStarted).
2. Read only unarchived logs.
3. For each worker, review: `git diff main...w/<name>` (git mode) or `/bossbot journal diff <task>` (no-git mode).
4. Decide merge or reject per task, with reasons.
5. **Git mode:** `git merge --no-ff`. **No-git mode:** `/bossbot revert <task>` for each rejection.
6. Send a follow-up TASK for each rejection.
7. Update INDEX, decisions (append-only), architecture, and stale module pages only.
8. Write `phases/<n>.md`.
9. Archive logs and journals.
10. Run `/bossbot phase close` (record + PhaseClosed).
11. Keep every page ≤ `maxPageLines`.

**`resume-team.md`** — optional `<name>`
1. Read the roster, the last phase summary, and per worker the latest log (prefer wip).
2. Respawn each listed (or named) worker that isn't running, with "continue from the log's Next step".
3. Re-send open TASKs with the same ids.

**`onboard.md`** — depth from config
1. `/bossbot scan`
2. `/init`
3. **quick:** overview and INDEX. Stop there.
4. **standard:** slice per `onboard.slices` → one explorer subagent per slice (returns ≤ 5 lines) → synthesis (overview, architecture, glossary, risks, INDEX).
5. **deep:** also `flows/*.md` (Mermaid) → verification subagent checks `verifyClaims` claims and marks pages `verified:`.
6. Ask the user to review in the dashboard Wiki tab.
7. Stay within `budgets.onboardTokens`. If it would be exceeded, stop at the current step and report.

## Appendix G — `docs/PROGRESS.md` template

```
## Phase <N> — <name>   status: done | partial
Built: <features>
Deviations from spec: <item → reason → ADR/API-MAP link>
Perf: <measured vs §3.4>
Smoke: <steps → result>
Open issues: <links>
Next: Phase <N+1> notes
```

## Appendix H — ADR template (`docs/adr/NNNN-<slug>.md`)

```
# NNNN <title>
Status: proposed | accepted | superseded by NNNN
Context:
Decision:
Consequences:
Alternatives considered:
```
