# Skill

'/init' -- a native icarus skill.

# Note

if you see `<inference>` `<inference\>` anywhere, it is meant for you to
inference against what is inside the inference block.

# Ownership Legend

<!-- [+] added 2026-08-20 — the ownership boundary (the caveat): icarus agent vs the /init skill binary -->

Every section below is tagged with WHO owns the behaviour. A bullet with no tag inherits its
section's tag. This is the line between the icarus agent (runtime) and the skill (binary) that
stamps the scaffolding into a green/brownfield project.

| tag | owner | lives in | ships as |
| --- | --- | --- | --- |
| `[harness]` | icarus runtime: agent loop, hooks bus, tools, spawn, compaction, Ra client | `~/Programs/ai/icarus` (`internal/*`, `pkg/*`, `cmd/icarus`) | `/usr/local/bin/icarus` |
| `[init]` | the `/init` scaffolder — deterministic, idempotent, **no model call** | `icarus init` subcommand (`cmd/icarus/main.go` dispatch) + REPL/web `/init` | the same Go binary |
| `[project]` | artifacts `/init` stamps into the repo and agents maintain at runtime | `<repo>/agents.md`, `<repo>/memory/**`, `<repo>/evals/**`, `<repo>/loops/**`, `<repo>/skills/**`, `<repo>/.icarus/**` | git-tracked files |
| `[ob1]` | OpenBrain — the long-term, cross-project, cross-agent store | `~/openbrain`, ob-mcp `http://10.0.0.4:8200` (WG-only) | service |
| `[claude]` | the Claude Code harness — reused or bridged, never duplicated | `~/.claude/{hooks/go,agents,commands,projects/*/memory}` | `claude-hooks` |
| `[monty]` | always-on loop + inference host | `monty` systemd user units, `sglang-monty` :30000 | systemd timers |

Rules of the boundary:

1. `[init]` never calls a model. It copies embedded templates, touches files, detects brownfield
   state, prints what it did, exits 0/1. Anything that needs inference is `[harness]` at runtime
   or a `[project]` skill.
2. `[harness]` writes into the project dir ONLY through the append seams this spec declares
   (`memory/*.md`, `memory/loops.json`, `memory/tasks.md` via `internal/projectmem`). Today it
   writes nothing there and reads only `.icarus/tools.toml`, `.icarus/skills/*/skill.md`,
   `.claude/commands/*.md` — never `AGENTS.md`/`CLAUDE.md`/`agentic_instructions.md` (verified
   2026-08-20). Every project read/write below is therefore a NEW seam.
3. `[project]` files are human-readable, append-only unless gated (RULES), and are the auditable
   projection of `[ob1]`. `[ob1]` is the source of truth for RETRIEVAL; `[project]` is the source
   of truth for the repo's own rules/lessons/shas (reviewable in a PR, readable without ob1).
4. `[claude]` pieces are reused by shelling out (`POST /cmp` already runs
   `claude-hooks compact llm --dry-run`) or by porting the Go package into icarus — never by
   re-implementing a second schema for the same thing.
5. Markers: every block added on 2026-08-20 starts with an HTML comment `<!-- [+] … -->`
   (`rg '\[\+\]'` lists them). Lines that need YOUR call are prefixed **DECIDE:** and collected
   under `# Open decisions` at the end.

Verified-absent today (each is net-new work, not a refactor): `/init`; project-dir writes; an
agent-definition file format; a child-executing `spawn` (`deferredSpawnRunner` returns "not yet
wired"); tokens/model/tool-calls/SHAs in the session transcript; a webhook/callback server; a
cron/timer loop on monty that runs autoresearch; server-side Ra in ob-mcp.

# Workflow

1. user opens icarus

- either in working directory or '/cd' into the working directory

2. user runs '/init'
3. script 'icarus_init.sh' runs

(consider the following script, only modifications to make:

1. functionality if there is incorrect syntax.
2. where commented for inference

```bash
#!/usr/bin/env bash
# icarus_init.sh — reference behaviour for the [init] binary.
# Syntax-fixed 2026-08-20 (permitted modification #1). Semantics unchanged except:
#   (a) headers are written only when a file is empty (re-running /init on a brownfield
#       repo must never truncate a memory file — append-only rule, RULES);
#   (b) the four files that were echoed but missing from ASSETS (evals, ra, tasks, tools);
#   (c) agents.md lives in the project root ("./agents.md"), not "/agents.md".
set -euo pipefail

MEMORY_DIR="./memory"
AGENT_FILE="./agents.md"

ASSETS=(
  "$MEMORY_DIR/context.md"
  "$MEMORY_DIR/evals.md"
  "$MEMORY_DIR/events.md"
  "$MEMORY_DIR/graph.md"
  "$MEMORY_DIR/icarus.md"
  "$MEMORY_DIR/lessons.md"
  "$MEMORY_DIR/logs.md"
  "$MEMORY_DIR/loops.md"
  "$MEMORY_DIR/ra.md"
  "$MEMORY_DIR/rules.md"
  "$MEMORY_DIR/shas.md"
  "$MEMORY_DIR/skills.md"
  "$MEMORY_DIR/tasks.md"
  "$MEMORY_DIR/tools.md"
)

mkdir -p "$MEMORY_DIR"
touch "$AGENT_FILE"
for i in "${ASSETS[@]}"; do
    touch "$i"
done

# header FILE TITLE — write the H2 only when FILE is empty (idempotent, brownfield-safe).
header() { [ -s "$1" ] || printf '## %s\n' "$2" > "$1"; }

# AGENTS
header "$AGENT_FILE" "AGENTS"
# CONTEXT
header "$MEMORY_DIR/context.md" "Context"
# EVALS
header "$MEMORY_DIR/evals.md" "EVALS"
# EVENTS
header "$MEMORY_DIR/events.md" "EVENTS"
# GRAPH
header "$MEMORY_DIR/graph.md" "GRAPH"
# ICARUS [harness]
header "$MEMORY_DIR/icarus.md" "ICARUS"
# LESSONS
header "$MEMORY_DIR/lessons.md" "LESSONS"
# LOGS
header "$MEMORY_DIR/logs.md" "LOGS"
# LOOPS
header "$MEMORY_DIR/loops.md" "LOOPS"
# RA
header "$MEMORY_DIR/ra.md" "RA"
# RULES
header "$MEMORY_DIR/rules.md" "RULES"
# SHAS
header "$MEMORY_DIR/shas.md" "SHAS"
# SKILLS
header "$MEMORY_DIR/skills.md" "SKILLS"
# TASKS
header "$MEMORY_DIR/tasks.md" "TASKS"
# TOOLS
header "$MEMORY_DIR/tools.md" "TOOLS"
```

##### INIT — executable contract `[init]`

<!-- [+] added 2026-08-20 — what happens at each step of /init; why it must be code, not a prompt skill -->

The script above is the behavioural reference. Per the fleet rule (Go is the default language),
the shipped artifact is a Go subcommand: `icarus init` (new arm in `cmd/icarus/main.go`
`dispatchSubcommand`, next to `sessions|evolve|attach`), exposed to the REPL as
`Command{Name:"/init"}` (`internal/cli/repl/commands.go`) and to the web as `POST /init` plus a
`CommandInfo` row in the `GET /command` catalog — the exact path `/cd` took
(`internal/web/cd.go`: pure filesystem op, ack envelope `{success, command, path, error}`).

A native `skill.md` cannot be `/init`: skills are prompt payloads (`internal/skills` expands the
body into the outbound turn), so a skill-based `/init` would ask the MODEL to run the scaffold
with `write`/`zsh` — non-deterministic by construction. `/init` must be code.

Step-by-step (deterministic, no model call, exit 0 on success / 1 on refusal):

1. **Resolve root.** `git rev-parse --show-toplevel` from the serve cwd (the one `/cd` set);
   refuse outside a git repo unless `--no-git`.
2. **Classify.** `greenfield` when `git rev-list --count HEAD` == 0 AND none of
   `{CLAUDE.md, AGENTS.md, agentic_instructions.md, .claude/, SPEC/, memory/}` exist; otherwise
   `brownfield`. Print the verdict and every pre-existing file it will LINK, not touch.
3. **Stamp.** `agents.md` + `memory/` + `evals/{Makefile,general/,custom/}` + `loops/` +
   `skills/` + `.icarus/{tools.toml,skills/}` from `go:embed` templates; header-only when the
   file is empty (the `header()` rule), never overwriting a non-empty file.
4. **Brownfield link.** In `agents.md` write a `## Existing instructions` table pointing at each
   pre-existing instruction file (path, size, last commit). `--dir-instructions` additionally
   runs `/dir_instructions` (`[claude]`, generates `agentic_instructions.md` per directory) —
   opt-in because it calls a model.
5. **Register.** Append one `memory/logs.md` JSONL row (`event_type:"init"`) and one ob1
   `project` item (`tags: [project:<name>, init, host:<h>]`) so other sessions learn the project
   was scaffolded. ob1 unreachable ⇒ warn, still exit 0 (the local files are the contract).
6. **Verify.** A re-run is a no-op; `--check` prints drift and exits 1 if a required file is
   missing or a header is wrong. This is eval `general/init_idempotent`.

Flags: `--check` · `--dry-run` · `--no-git` · `--dir-instructions` · `--force` (may rewrite ONLY
the TEMPLATE region of `agents.md` between `<!-- init:begin -->` / `<!-- init:end -->`; never
touches `memory/`).

Templates embedded in the binary — one per `[project]` file; the `agents.md` skeleton is under
CONTEXT_OUTCOMES, each file's record format is in its FILES section below.

### FILES

#### AGENTS

##### PROJECT-AGENTS

###### ORCHESTRATOR

- `orchestrator`: Manage and coordinate complex workflows across
  multiple agents, tools, and systems. This tool enables high-level task planning,
  resource allocation, and execution tracking, ensuring efficient and reliable
  completion of multi-step, interdependent tasks in dynamic environments. It supports
  dynamic scaling, error recovery, and real-time monitoring of system state.

- `sub-agents`: Manage and coordinate specialized AI sub-agents for
  distributed problem-solving, enabling modular, scalable, and fault-tolerant workflows
  across complex tasks such as system orchestration, multi-step planning, and
  cross-domain reasoning. This tool allows dynamic instantiation and communication
  between sub-agents, facilitating advanced autonomous execution in large-scale
  environments.

<!-- [+] added 2026-08-20 — exact tooling + skills for ORCHESTRATOR -->

[tools] (groups — TOOLS §CATALOG): `agent.spawn` (+ `task_id`, `tools[]`), `agent.await`,
`agent.message`, `task.*`, `memory.recall`, `eval.run`, `event.emit`, `notify`, `fs.read`,
`git.log|blame` (read-only) — **no** `fs.write/edit`, **no** `exec.shell`: the orchestrator never
authors or builds, it delegates. Today `spawn` admits (depth 3, fan-out 8, 512 MiB ledger) and
then returns "sub-agent execution not yet wired" (`internal/cli/spawn_guard.go`) — the Runner is
GRAPH deliverable #1.

[skill]:

1. DECOMPOSE: `# Outcome` section (the intent gate already parses it) → task DAG
   (`pkg/orchestration.Task{ID, Description, AgentSpec{Role}, Inputs, DependsOn, Deadline}`),
   `task_id = T<n>` scoped to this run; writes `memory/tasks.md` rows (TASKS) after the dedupe
   gate.
2. DISPATCH: topological order (`Supervisor.Spawn` honours `DependsOn`, `max_workers` 8), each
   worker prompt = task + "read `agents.md` and the local `agentic_instructions.md`", nothing
   else; terse protocol back (`Done.` / `Done. Output: <path>` / `Blocked: …` / `Error: …`,
   `ParseTerseResult`).
3. RECONCILE: on `task.done|blocked|error` update `tasks.md`, append `logs.md`, call `eval.run`
   for the task's suite, hand shas to GIT-AGENT, append the `graph.md` edge; a rejected review
   becomes a `lesson` candidate (LESSONS).

###### SYS-ADMIN

[tools]:

- `bash`: Execute shell commands directly for system-level operations, such as
  managing files, starting services, or checking system status. This tool enables
  direct interaction with the underlying OS, useful for deployment, debugging, and
  automation tasks.
- `python` : Execute Python scripts and code snippets for data processing, model
  training, or automation tasks. This tool supports complex logic, mathematical
  operations, and integration with external libraries, making it ideal for AI-driven
  workflows and system management.

  [skill]:

1. FILE-MANAGER:

- [role]:
  Manage files and directories, including creating, reading,
  updating, and deleting files. This tool supports advanced file operations such as
  copying, moving, and archiving, and is useful for organizing project structures,
  managing assets, and maintaining clean codebases.

<!-- [+] added 2026-08-20 — exact tooling + skills for SYS-ADMIN; python contention -->

[tools] (groups): `exec.shell` (zsh/bash — `toolrules` pack applies), `exec.go` (`go run` /
`go test -race` / `go vet`; bitfield/script for pipelines), `fs.*`, `sys.service`
(`systemctl --user`), `sys.host` (the procfs/sysfs cards already in `internal/metrics/host.go`),
`notify`, `event.emit`.

**Decided 2026-08-21:** `python` STAYS in SYS-ADMIN (`exec.python` group) — ops scripting on the
hosts is where it already lives. The fleet rule "Go is the default language" applies to NEW
artifacts (hooks, skills, loops, binaries), which SYS-ADMIN writes in Go; Python is for driving
the existing Python surfaces (the ML stack, the `~/openbrain` Python API, vendor tooling).
`exec.python` is also granted to the `[ai]` skills.

[skill]:

1. FILE-MANAGER (above): the `fs.*` tool group; its "organize project structures" intent is
   skill 4 below (decided 2026-08-21 — FILE-MANAGER is no longer a separate agent).
2. HOST-INVENTORY (D0): refresh the AI FLEET HOSTS JSON after a reconfig (the `** update model
   and runtimes on each host **` rule) — `ssh -o BatchMode=yes <host> 'lscpu; free -h;
   nvidia-smi || rocm-smi; df -h /'` → JSON → `memory/context.md` + ob1 `reference`. Never
   `ssh -t` / `zsh -i` from an agent.
3. SERVICE-CHECK (D0): `systemctl --user is-active` over a unit list (icarus-serve, ob-mcp,
   coms-hub, sglang on monty); failures → `events.md` + `notify`.
4. PROJECT-STRUCTURE (D0 + D1): every project follows a similar, current-standards layout.
   `/init` detects the project type and stamps `structure.toml` (Go: `cmd/ internal/ pkg/
   evals/ loops/ skills/`; web: MVC or feature-sliced; Python: `src/` layout; mixed: one
   section per language) from embedded templates; `icarus init --check` diffs the tree against
   it (eval `general/project_structure`, D0); drift is fixed as rename-preserving commits
   proposed to GIT-AGENT (D1 only for the proposal text). Rules it enforces: no spec-shaped
   files at the repo root (`SPEC/<name>/` layout), exactly one `agents.md`, `memory/` never
   reorganised, generated artifacts gitignored.

###### RESEARCHER

[tools]:

- `websearch` : Perform a web search to gather up-to-date information, research, or
  insights on a given topic. This tool is ideal for verifying facts, finding
  documentation, or exploring new technologies and best practices. Use it when external
  data is required to inform decisions or complete tasks.

<!-- [+] added 2026-08-20 — exact tooling + skills for RESEARCHER -->

[tools] (groups): `web.search` (icarus `websearch`: `browser` default, `searxng`/`brave`/`tavily`
backends — Recipe 42), `web.fetch` (`webfetch`: SSRF allowlist `[webfetch].allow`, bot-wall
detector), `browser.*` (7 CDP tools, session-bound), `memory.recall`, `memory.remember`
(`kind:fact`, `identity:agent`), `fs.read`, `fs.write` limited to `research/**` — **no**
`exec.shell`, **no** `git.*`.

[skill]:

1. DIGEST (D1): URL / video / paper → dense factual summary with source + date →
   `memory.remember`. For video the deterministic part is `yt-dlp --skip-download
   --write-auto-sub --sub-lang en` (installed on lewis) → VTT → de-duplicated text; the model
   only summarises. (This spec's memory section was built exactly that way.)
2. COMPARE (D1): N sources → claim table with per-claim citations; contradictions are recorded
   as `lesson` candidates, never as facts.
3. WATCH (D0 + D1): a LOOP (`loops/watch-<topic>.toml`) that re-runs DIGEST on a schedule and
   remembers only deltas (hash dedupe in the reconcile step).

###### GIT-AGENT

[tools]:

-`git`: Interact with GitHub repositories to manage code, track issues, create
pull requests, and handle repository settings. This tool enables seamless
integration with version control, allowing for collaborative development, code
reviews, and automated workflows directly from the system.

<!-- [+] added 2026-08-20 — exact tooling + skills for GIT-AGENT; trailers are the pointer scheme -->

[tools] (groups): `git.status|diff|log|blame|commit|branch|worktree|push`, `fs.read`, `fs.glob`,
`fs.append:memory/shas.md` — **no** `fs.write/edit` (it commits work, it does not author it),
**no** general `exec.shell`. Reuse: `internal/gitwork` (native `/gwp`: single-concern commit
decomposition, messages drafted through the `pkg/spec.Generator` seam, mechanical fallback,
never force-pushes).

[skill]:

1. LOGICAL-COMMIT (D1): decompose the worktree into single-concern commits (`gitwork`) and sign
   every commit with trailers `Task-Id: <run_id>/T<n>` · `Agent: <agent_id>` ·
   `Session: <session_id>` (+ `Co-Authored-By` for cloud models). Trailers are what let
   `git blame -L` / `git log --grep='Task-Id: .*T7'` answer "which task, agent and session
   touched this line" — the pass-by-reference pointer the seed file needs (Data Flows).
2. SHA-LOG (D0): after every commit append `memory/shas.md` (SHAS format) and a `logs.md`
   `commit` row; eval `general/shas_match_log` checks both directions.
3. BLAME-ATTRIBUTE (D0): bug report citing a file/line → `git blame` → trailer → `task_id` →
   the session JSONL + compact-index that produced it (the TASKS dedupe gate and the
   bug-fix/feature-request pointer chain).

###### FILE-MANAGER

[tools]:

- file_manager: Manage files and directories, including creating, reading,
  updating, and deleting files. This tool supports advanced file operations such as
  copying, moving, and archiving, and is useful for organizing project structures,
  managing assets, and maintaining clean codebases.

<!-- [+] added 2026-08-20 — is this agent necessary? · decided 2026-08-21 -->

**Decided 2026-08-21:** FILE-MANAGER is not an agent; it is baked into SYS-ADMIN as the
PROJECT-STRUCTURE skill (SYS-ADMIN `[skill]` 4). The intent it carries forward: every project
follows a similar, current-standards structure (Go: `cmd/ internal/ pkg/`; web: MVC or
feature-sliced; Python: `src/` layout), the structure is declared once (`structure.toml`,
stamped by `/init` from the detected project type), drift is detected deterministically
(`icarus init --check` → eval `general/project_structure`), and corrections land as
rename-preserving commits through GIT-AGENT. File CRUD itself is the `fs.*` tool group (TOOLS);
the `[tools]` paragraph above stays as that group's description.

###### MEMORY

[tools]:

- recall : Retrieve previously stored information from memory or context logs to
  inform current decisions or actions. This tool enables efficient access to
  historical data, ensuring continuity and consistency across sessions and tasks. Use
  it to reference past interactions, completed work, or system states when needed.

<!-- [+] added 2026-08-20 — exact tooling + skills for MEMORY; then the roster assessment (missing agents) -->

[tools] (groups): `memory.recall` (the explicit retrieval surface — MEMORY §M4),
`memory.remember` (explicit write through the same extract + reconcile path as the automatic
per-turn ingest — §M3), `memory.forget` (GATED: marks `superseded`, never deletes — RULES),
`memory.consolidate` (loop-only — §M5), `ob1.query|read|write|assist` (raw, audits only),
`fs.read` + `fs.append` on `memory/**` only — **no** `exec.shell`, **no** `git.*`.

[skill]:

1. RECALL (D1): optional query rewrite → hybrid retrieval → top-k with provenance (§M4).
2. REMEMBER (D1): "remember that …" / explicit agent write → extraction → ADD / UPDATE /
   SUPERSEDE / NOOP → ob1 + `memory/<kind>s.md` projection (§M3).
3. LESSON-LOG (D1): a failed eval, a user correction, a loop break, a rejected review → one
   `lesson` record (LESSONS format) → ob1 + `lessons.md`.
4. CONSOLIDATE (D1, loop on monty): expire, decay, merge near-duplicates, promote repeated
   lessons to rules, re-verify stale playbooks (§M5).

##### AGENT ROSTER — assessment `[harness]`

<!-- [+] added 2026-08-20 — which agents are necessary, which are missing -->

| agent | verdict | why |
| --- | --- | --- |
| ORCHESTRATOR | keep | the only agent allowed to `spawn`; owns the task DAG and `tasks.md` |
| SYS-ADMIN | keep | host/service ops + project structure; `python` stays (decided 2026-08-21) |
| RESEARCHER | keep | the only agent with web/browser tools |
| GIT-AGENT | keep | the only agent that commits; owns trailers + `shas.md` |
| FILE-MANAGER | folded into SYS-ADMIN (decided 2026-08-21) | becomes the PROJECT-STRUCTURE skill; CRUD is the `fs.*` group |
| MEMORY | keep | owns ingest/recall/consolidate; the only writer of `lessons.md`, `rules.md` |
| **CODER** (missing) | **add** | nothing in the roster EDITS code. `fs.read/write/edit`, `exec.shell` (build/test), `exec.go`, `lsp.*`, `memory.recall`, `eval.run`; no `git.commit` (hands off to GIT-AGENT), no `agent.spawn`. Maps to `icarus_coder` / `[models.agents.worker]` |
| **REVIEWER** (missing) | **add** | the REVIEWER INDEPENDENCE RULE (icarus CLAUDE.md) needs a structurally read-only role: `fs.read`, `fs.grep`, `fs.glob`, `exec.shell` (tests only — a `toolrules` pack), `eval.run`, `memory.recall`, `memory.remember(kind:lesson)`. Never shares a session with CODER. Maps to `fable-reviewer` (`Read, Grep, Glob, Bash`) |
| **EVALUATOR** (missing) | **add as a SKILL of REVIEWER**, not an agent | `make evals` / CUSTOM dispatcher is deterministic (D0); it emits `skill.eval_pass|fail`-shaped results and calls `icarus sessions fail …` (EVALS note). An agent here would add a model call to a pass/fail script |
| RA | not an agent | a harness component (hook + retrieval) — see RA |

Model pins per role reuse `[models.agents.<role>]` (`internal/config/models_config.go`;
`config.toml.example` ships `supervisor` → anthropic/claude-fable-5/high and `worker` →
local/low): orchestrator → cloud; coder, reviewer, researcher, memory → `sglang-monty`
`qwen3.8-27b` (175k window); git, sys-admin → local, effort `low`. Per-agent TOOL allowlists
are not expressible today (`SpawnRequest` carries `agent/prompt/description/provider/model/
effort` only; `--tools` / `tools.enabled` are process-wide) — TOOLS §MATRIX defines the field
and the agent definition that carries it: the **Claude Code agent format** (`.claude/agents/*.md`
YAML frontmatter), read by BOTH harnesses, so the coders you already drive from Claude Code are
the same agents icarus spawns (TOOLS §AGENT DEFINITIONS).

##### AI FLEET HOSTS:

** update model and runtimes on each host, after reconfig **

```json
{
  "hosts": {
    "monty": {
      "cpu": "AMD Ryzen Threadripper 7960X 24-Cores",
      "ram": "125Gi",
      "gpu": "RTX PRO 6000 96GB + RTX PRO 4500 32GB",
      "vram": "128GB",
      "disk_capacity": "98G (81G used, 12G free, 88%)",
      "host": "fleet hub desktop (always-on inference host)"
    },
    "cai": {
      "cpu": "AMD Ryzen 7 7800X3D 8-Core Processor",
      "ram": "30Gi",
      "gpu": "AMD Radeon AI PRO R9700 + AMD Raphael (iGPU)",
      "vram": "unknown (verify via rocm-smi)",
      "disk_capacity": "98G (54G used, 40G free, 58%)",
      "host": "remote (ssh n0ko@cai)"
    },
    "mac": {
      "cpu": "Apple M1 Max",
      "ram": "unknown — host unreachable (no route to host)",
      "gpu": "integrated (M1 Max)",
      "vram": "unified memory (shared)",
      "disk_capacity": "unknown — host unreachable",
      "host": "Mac local inference node (ssh n0ko@mac)"
    },
    "lewis": {
      "cpu": "AMD RYZEN AI MAX+ 395 w/ Radeon 8060S",
      "ram": "121Gi",
      "gpu": "Radeon 8060S (integrated, AI MAX+ 395)",
      "vram": "shared/unified (iGPU)",
      "disk_capacity": "92G (69G used, 18G free, 80%)",
      "host": "primary daily-driver laptop (Z13/dwm), WG 10.0.0.4"
    }
  }
}
```

###### AO:

<!-- [+] added 2026-08-20 — resolved from the live unit (ob1 had no entry for "AO") -->

A.O = the **A.O Autonomous Agent Supervisor**: `~/.config/systemd/user/ao-supervisor.service` →
`~/bin/ao-supervisor` (Go, `WatchdogSec=300`, `Restart=on-failure`,
`EnvironmentFile=~/.config/ao/env`, drives `claude` via `CLAUDE_CODE_PATH`, Discord bridge
`bridge.Start` — the only AFK/mobile reach in the fleet per `~/SPEC/SPEC.md`);
`ao-auth-refresh.timer` 04:00 on monty. Known trap (drop-in `network-guard.conf`): it starts
before DNS at boot and the Discord goroutine exits without reconnect.

- skills (A.O namespace — NOT stamped per project):
  - `push_mobile` (D0): Discord push when a loop fails, an eval blocks, or a task needs a human
    gate (EVENTS "Returns: … notification, user").
  - `afk_gate` (D0): hold a gated operation (`memory.forget`, `--force`, a destructive loop)
    until a Discord ack arrives; timeout ⇒ `Blocked:`.
  - `watchdog` (D0): the `sd_notify` pattern every long-running loop on monty reuses.

**DECIDE:** A.O as the fleet's mobile-push owner (callback server → A.O → Discord), or a direct
Discord webhook (`discord.go` in `~/SPEC/SPEC.md`)? The spec assumes A.O.

- A.O ⇄ icarus (added 2026-08-21): A.O is a CLIENT of `icarus serve` through the `/hook/*` routes
  (EVENTS) and the session API — it gains `ICARUS_SERVE_URL` next to `CLAUDE_CODE_PATH`, so
  what you can reach from a phone can drive icarus. Full picture: `# Big Picture`.

###### ICARUS:

<!-- [+] added 2026-08-20 -->

- skills (harness-only, never per project — the `<icarus_skills>` list under SKILLS):
  `self_healing`, `server_creation`, `recall` (the retrieval surface), `eval_environment`, `r2d2`
  (3-pane coms harness, `extensions/r2d2/`, preset `multi_agent_reasoning`), `cmp` (`POST /cmp`
  → `claude-hooks compact llm --dry-run`), `init` (this spec's scaffolder), `loop` (`icarus loop
  define|status|stop`).

###### MONTY:

- skills
  <!-- [+] added 2026-08-20 — monty = the loop host -->
  - `loop_runner` (D0): the systemd-timer executor for every `loops/*.toml` (LOOPS). Existing
    monty timers: `openbrain-pipeline` 5 m, `n0ko-ai-{harvest,inbox-sync,context-synth,eval-feed}`
    hourly, `ao-auth-refresh` 04:00 — no crontab, nothing runs autoresearch yet.
  - `memory_consolidate` (D1): nightly MEMORY §M5 pass on the local model (zero cloud cost).
  - `autoresearch` (D1): `~/Programs/autoresearch` Go engine (`Engine.Run`: propose → commit →
    run → parse → keep | revert; only `Propose` is a model call) as a loop whenever a project
    declares an objective.
  - `inference_health` (D0): `sglang-monty` :30000 and ob-mcp :8200 probes → `events.md`; the
    MTP crash is the known instability every loop must tolerate as a `crash` row.

#### CONTEXT

echo ## Context" > $MEMORY_DIR/context.md

##### CONTEXT_OUTCOMES

1. agents.md (will describe the memory system)
2. creates '.memory/'
3. creates 'agents.md':
   - explains the goal of the project
   - memory system

4. files include **append_only, unless gated removal is approved due to outdated
   information**:
   - lessons.md
   - shas.md
   - rules.md:
     1. when to log a new lesson: a. lesson formatting rules
     2. Spec rules (just as defined in claude code)
     3. General Housekeeping: Project Directory structure

<!-- [+] added 2026-08-20 — agents.md skeleton, the three context tiers, brownfield behaviour -->

5. `agents.md` skeleton (stamped by `[init]`; the TEMPLATE region between the markers is the
   only thing `--force` may rewrite):

```markdown
## AGENTS
<!-- init:begin -->
### Project
- name · goal (one paragraph — the user writes it; /init leaves `TODO`) · repo · default branch
- hosts this project runs on (from AI FLEET HOSTS) · local model pins
### Memory system
- tiers: T0 session (icarus JSONL) · T1 project (`memory/*.md`, git-tracked, append-only) ·
  T2 long-term (ob1, cross-project). What each file holds, who writes it, the record format.
- retrieval: Ra auto-injects ≤ 3 memories per turn; `memory.recall` for explicit lookups.
- gates: removal from lessons/rules/shas needs an approved `memory.forget` (RULES).
### Roster
- table: agent → role → tools → model pin → skills (from AGENTS)
### Rules
- pointer to `memory/rules.md` (coding style, data types, max_depth=3, composability)
### Existing instructions   (brownfield only)
- table: CLAUDE.md / AGENTS.md / agentic_instructions.md / SPEC/ → path · size · last commit
<!-- init:end -->
```

6. Context tiers — who writes, who reads, how much per turn:

| tier | store | written by | read by | budget per turn |
| --- | --- | --- | --- | --- |
| T0 session | `~/.icarus/sessions/<project>/<id>.jsonl` + `usage/<id>.jsonl` | `[harness]` | replay, `/cmp`, memory ingest | the model window (175k on `qwen3.8-27b`) |
| T1 project | `<repo>/memory/*.md`, `agents.md` | `[init]`; MEMORY / GIT agents through `projectmem` append seams | `[harness]` system-prompt assembly (NEW: `agents.md` ≤ 4 KiB + `rules.md` head), humans, git | ≤ 6 KiB |
| T2 long-term | ob1 items (pgvector) + TrustGraph | memory ingest (per turn), Ra decisions | Ra (auto, ≤ 3 items), `memory.recall` (explicit) | `ob1.inject_max_bytes` 2048 + `ra.max_bytes` 2048 |

7. Brownfield: `/init` never rewrites `CLAUDE.md` / `AGENTS.md`; `agents.md` LINKS them. The
   `[harness]` read order is `agents.md` → `CLAUDE.md` (if present) → `rules.md` head; today it
   reads none of them (verified) — the T1 read seam is ICARUS [harness] deliverable #1.

#### EVALS

##### GENERAL RULES

1.  Always use a build system for executing evals. (`make` until `just` migration)
2.  Pass/Fail -- there is no other option 2a. each eval should absolutely include a
    log in it's return signature. that log will follow the formatting rules as stated
    in the `####LOGS` section. Conventionally this means that all function exit routes
    must return a log.
3.  each eval suite should be a logical seperation of functionality, outside of the
    general suite, this can and should be tied to the logical commits that are made by
    the `git-agent`

```
    evals/
    ├── Makefile
    ├── general/
    │ ├── eval_1.go
    │ └── eval_2.go
    ├── suite_1/
    │ ├── eval_1.go
    │ └── eval_2.go
    ├── suite_2/
    │ ├── eval_1.go
    │ └── eval_2.go
    └── eval_1.go
    └── eval_2.go

```

eval execution script:

```bash
#!/usr/bin/env bash
EVAL="$1"
SESSION_ID="$2"
case "$EVAL" in
  eval_1)
    echo "Running eval_1"
    if [[ $? -eq 0 ]]; then
        echo "eval_1 passed"
    else
        echo "eval_1 failed"
        icarus session resume --id "$SESSION_ID" --status failed --eval eval_1
    fi
    exit 0
    ;;
  eval_2)
    echo "Running eval_2"
    if [[ $? -eq 0 ]]; then
        echo "eval_2 passed"
    else
        echo "eval_2 failed"
        icarus session resume --id "$SESSION_ID" --status failed --eval eval_2
    fi
    exit 0
    ;;
  # ...continue as needed
esac
```

Note:

- `<inference>` → resolved to `icarus session resume --id "$SESSION_ID" --status failed --eval <name>`, taking `session_id` as a second positional param, per the comment's own callout.
- Double-compilation callout (GENERAL rule under eval*2) doesn't apply to \_this* script — this file only runs `case`/eval dispatch, not `make build`; the double-compile guard belongs in `eval_2.go`'s Go body (`make evals` → its first function calls `make build`, and the eval itself must NOT shell out to `go build`/`make build` again — only check the already-built binary's exit/path).

##### GENERAL

eval_1:

1. linting
2. formatting
3. race condition - race condition detection: use Go's race detector during testing
   (`go test -race`) and ensure all concurrent operations are properly synchronized
   using channels, mutexes, or atomic operations. Avoid shared mutable state unless
   absolutely necessary, and prefer immutable data structures or message-passing
   patterns in concurrent workflows.
4. race condition - race condition detection: use Go's race detector during testing
   (`go test -race`) and ensure all concurrent operations are properly synchronized
   using channels, mutexes, or atomic operations. Avoid shared mutable state unless
   absolutely necessary, and prefer immutable data structures or message-passing
   patterns in concurrent workflows.

eval_2:

1. does updated code compile **be sure to not double compile** since you will be
   using the make system to compile code already. that should be considered part of
   the evaluation (i.e. `make evals` should run eval suite, with the first function
   calling `make build`
2. is the updated binary / executable in the $PATH (is there a $PATH collision?)

eval_3: **pass/fail exception** -- not all code can be 0 allocation

1. heap allocations = 0 (if possible) -- this is every developer's dream. not at the
   price of abnoxious complexity. i'd really like to run `unsafe` golang with the gc
   off to decrease runtime overhead. which means, all C footguns are plausible (well
   most of them). Also means that you MUST dereference in golang. we are skydiving
   with no parachute...
2. optimized data types (consider byte padding, struct formations in struct packing,
   memory layout, and alignment to minimize padding and improve cache efficiency)

##### CODING-AGENT EXPECTATIONS `[claude]` + `[harness]`

<!-- [+] added 2026-08-21 — the eval_1–3 expectations are NOT in the coding agents' definitions today; this block is the text to propagate -->

Audit (2026-08-20): `~/.claude/agents/{golang-coder,unix-coder,icarus_coder,cmdr_coder,pi_coder}.md`
carry tool allowlists and project scope but none of the eval_3 expectations (zero-allocation,
byte padding / struct packing, `unsafe`, GC-off) nor the RULES data-type and depth rules. The
block below is copied VERBATIM into each coder definition (section `## Performance contract`)
and into the icarus CODER / REVIEWER role definitions; `claude-hooks sync-agents` /
`fleet-sync-agents` propagate it. Every expectation has a deterministic check, so each is a
pass/fail eval, not a vibe.

| expectation | how it is measured (predicate) | eval |
| --- | --- | --- |
| hot paths allocate 0 | functions listed in `evals/hotpaths.txt` (or marked `// hot-path`) run under `go test -bench=. -benchmem -run='^$'`; `0 allocs/op` required; everything else in the package is exempt — the eval_3 "pass/fail exception" made binary | eval_3.1 (`output_pattern_match`) |
| struct layout minimises padding | `fieldalignment` analyzer (`golang.org/x/tools/go/analysis/passes/fieldalignment`) reports 0 findings under `internal/` and `pkg/`; fields ordered largest-first; a hot struct fits a cache line (≤ 64 B) or says why in a comment | eval_3.2 (`ast_check`) |
| no hidden escapes on hot paths | `go build -gcflags='-m -m' ./...` shows no `escapes to heap` for hot-path params/returns; `[]byte` over `string` for transient buffers; `sync.Pool` for reusable ones | eval_3.1 |
| `unsafe` is fenced | only in packages listed in `evals/unsafe-allow.txt`, each behind a `//go:build` tag with a `_test.go` exercising the aliasing; `go vet -unsafeptr` clean; every pointer cast carries a `// SAFETY:` comment (the dereference is yours) | eval_1 (`negation_check`) |
| GC-off is an experiment, not a default | `debug.SetGCPercent(-1)` / `GOGC=off` only inside a loop or skill that declares `max_rss_mib` and is killed at the ceiling; never in `icarus serve` | eval_1 (`contains_pattern`) |
| race-free | `go test -race ./...` clean; shared state behind a mutex OR a channel, never both | eval_1.3 |
| lint / format | `gofmt -s -l` empty; `go vet ./...` clean; `staticcheck` clean | eval_1.1–1.2 |
| depth ≤ 3, functions ≤ 40 lines, one purpose | `cmd/ → internal/<svc> → pkg/` only; a fourth hop or a > 40-line function is a review finding | eval_1 (`structural_check`) + REVIEWER |
| table-driven tests, wrapped errors | `ast_check` for `t.Run` tables in new `_test.go`; `%w` + `errors.Is/As`, never string matching | eval_1 |
| single compile | `make build` once; evals never re-`go build` (eval_2 rule) | eval_2 |

Propagation = a `[claude]` task (claude-agent): edit the five coder files + their `agents/.lazy`
siblings, run `sync-agents`, verify with `rg 'Performance contract' ~/.claude/agents`. The same
text is the body of the icarus `coder` / `reviewer` definitions (TOOLS §AGENT DEFINITIONS).

##### CUSTOM

feature specific evaluations. There is an option where the user can use

<!-- # Output -->

which will create evals to test off of. these are designed to be 'in-turn'
evaluations, where the evals executed in this section are designed to be far more
robust.

```bash
#!/usr/bin/env bash
# CUSTOM eval suite: user-defined feature evals sourced from an "# Output" block.
# Invoked the same way as the GENERAL dispatcher above, but $EVAL names come
# from the feature spec rather than a fixed set, so we discover them instead
# of hardcoding a case arm per name.
EVAL="$1"
SESSION_ID="$2"
CUSTOM_EVAL_DIR="./evals/custom"
eval_path="$CUSTOM_EVAL_DIR/$EVAL"
if [[ ! -x "$eval_path" ]]; then
    echo "custom eval not found or not executable: $eval_path"
    icarus session resume --id "$SESSION_ID" --status failed --eval "$EVAL"
    exit 1
fi
echo "Running custom eval: $EVAL"
"$eval_path" "$SESSION_ID"
if [[ $? -eq 0 ]]; then
    echo "$EVAL passed"
else
    echo "$EVAL failed"
    icarus session resume --id "$SESSION_ID" --status failed --eval "$EVAL"
fi
exit 0
```

Note: this dispatcher only shells out to the pre-built `$eval_path` binary/script — it never calls `go build`/`make build` itself, keeping the double-compilation guard intact (that guard lives in the eval's own Go body, per the eval_2 callout).

<!-- [+] added 2026-08-20 — contention: the resolved <inference> line names a command that does not exist -->

**Contention (verified 2026-08-20):** there is no `icarus session` subcommand — only
`icarus sessions list [--state active|detached|ended|stuck] [--json]`; session selection is by
root flags (`--resume/--continue/--session/--fork`). Keep both dispatchers as written and make
the call real: `icarus sessions fail --id "$SESSION_ID" --eval "$EVAL"` = a new `RunSessions`
op that mutates the registry record (`eval_done=false`, `eval_attempts++`, `next_retry_at`),
emits a `skill.eval_fail`-shaped bus event (`SkillEvalPayload{Skill:<eval>, SessionID,
PassRate:0, Detail}`) and appends `memory/evals.md` + `logs.md` (`eval_fail`). The registry's
teardown `Eval` side effect (`eval_done`/`eval_attempts`) is a noop slot today and
`icarus evolve retry|ack <session-id>` already manipulates the same teardown state, so the
plumbing exists.

Two functionality fixes for both scripts: (1) `$?` after `echo "Running …"` tests the echo, not
the eval — run the eval binary/`make` target and test ITS status; (2) every arm ends `exit 0`,
so `make evals` is green even when an eval fails — exit 1 on failure so the Makefile target is
itself pass/fail (EVALS rule 2).

#### EVENTS

- webhooks
- callback server:
  1. function calls
  2. agent calls
  3. structured return formatting rules
  4. Returns: where to (orchestrator, edge device, notification, user)
  5. asynchronous vs synchronous calls
  6. concurrency policies
  7. scaling

<!-- [+] added 2026-08-20 — callback server contract; the 7 bullets mapped to existing seams -->

`[harness]` has NO webhook/callback server today (verified). It has: an in-process typed bus
(~60 events, `internal/hooks/events.go`), `POST /notify` (loopback-only ingest → `notify.posted`),
`POST /session/{id}/message` (prompt injection), the coms-go hub (`10.0.0.4:8901`: unicast
prompt → response by handle, SSE per agent, `POST /v1/messages` + `/await`), and an MCP server
(:8199). computeCommander already has the OUTBOUND shape (`pkg/integrations/webhook`:
`Event{type, payload, timestamp}`, `Subscription{url, events}`, POST with 10 s timeout). The
callback server is the inbound half.

- callback server `[harness]` — `icarus serve` route group `/hook/*` (WG-bound like ob-mcp,
  bearer token from env):
  1. function calls — `POST /hook/tool/<name>` runs a registered tool out-of-band (same
     `Tool.Call` + `toolrules` + `ValidateArgs` path) and returns its result; lets a loop on
     monty reach a tool that exists only inside a session (`browser_*`).
  2. agent calls — `POST /hook/agent/<role>` = spawn-by-HTTP `{task_id, prompt, tools[],
     model?, reply_to}` → `spawnguard` admission → child runs → terse result POSTed to
     `reply_to` (or stored for `GET /hook/task/<task_id>`).
  3. structured return formatting — every callback returns the `/cd`-style envelope:
     `{success, command, task_id, agent_id, status: done|blocked|error, output_path?, detail,
     sha[]?, error?}`; `ParseTerseResult` lifts `Done. Output: <p>` into it.
  4. Returns — `reply_to` ∈ `orchestrator` (bus `task.done` on the parent session) · `edge`
     (coms-go `POST /v1/messages` to a seat) · `notification` (`POST /notify` → dunst / web
     strip; AFK ⇒ A.O `push_mobile`) · `user` (web SSE `/event`).
  5. async vs sync — sync for function calls under `timeout_ms` (default 30 s, the
     `mcp.call_timeout_ms` precedent); async for agent calls: `202 {task_id}`, completion by
     callback or poll.
  6. concurrency — `spawnguard.Limits` (`MaxConcurrent = NumCPU−1`, `PerAgentSubagentLimit` 8,
     `PerAgentMemMiB` 512, depth 3) is the ONLY admission policy; the callback server reuses
     it and never adds a second one.
  7. scaling — one serve per host; cross-host fan-out goes through coms-go (one seat per host);
     no queue until a measured need — the `events.md` arrival rate is the metric.
- event envelope (JSONL in `memory/events.md`, same shape as LOGS):
  `{"timestamp","event_type":"webhook_in|webhook_out|callback|loop_fail|eval_fail|gate_request|gate_ack","source","task_id","agent_id","details":{}}`
- feature requests / bug reports arrive as `webhook_in` (the `gh-mention-watch` timer on lewis
  is an existing source) → TASKS dedupe gate → `tasks.md`; a duplicate is auto-closed with its
  pointer and costs no agent.

#### GRAPH

- agent orchestration:
  1. sub-agent orchestration
  2. sub-agent task assignment
  3. sub-agent task completion
  4. sub-agent task reporting

<!-- [+] added 2026-08-20 — the orchestration graph: nodes, edges, lifecycle, signing, mapping, policies -->

Two graphs exist and must not be conflated: the **orchestration graph** (this section — agents
and tasks, per run) and the **knowledge graph** (MEMORY §M1/M2 — entities ↔ memories,
ob1/TrustGraph, permanent). `memory/graph.md` is the human projection of the orchestration graph.

##### NODES AND EDGES

- node `agent` — `{agent_id: "<role>-<4hex>" (pkg/identity), role, model pin, tools[], depth,
  parent, session_id}`; the base session is `base-0001`, depth 0.
- node `task` — `{task_id: "T<n>" scoped to its supervisor's run, description,
  agent_spec{role}, inputs{}, depends_on[], deadline, status}` (`pkg/orchestration.Task`).
- edge `assigns` (supervisor → task → worker) · `depends_on` (task → task: the DAG) ·
  `reports` (worker → supervisor, terse protocol) · `reviews` (REVIEWER → task; MUST originate in
  a different session than the author — REVIEWER INDEPENDENCE RULE) · `commits` (GIT-AGENT →
  sha[] → task).

##### LIFECYCLE (one state machine per task)

```
            assign              start                done             pass
  pending ─────────► assigned ─────────► running ──────────► review ────────► accepted
     ▲                  │                   │   │ blocked        │ fail
     │ retry ≤ limit    │ denied            │   └──► blocked     └──► rejected ──► lesson
     └──────────────────┴───────────────────┴── error ──► failed ──► escalate (A.O gate)
```

1. sub-agent orchestration — ORCHESTRATOR builds the DAG from `# Outcome`, orders it
   topologically (`Supervisor.Spawn` honours `DependsOn`; `max_workers` 8) and is the only
   agent that may call `agent.spawn` (depth ≤ 3).
2. sub-agent task assignment — `spawn{agent, prompt, task_id, tools[], provider?, model?,
   effort?}` → `spawnguard` admission (typed denials `depth_exceeded | concurrency_cap |
   memory_cap | subagent_limit | budget_exhausted | registry_unreachable`) → child session with
   `agent_id / supervisor_id / task_id` stamped on EVERY JSONL entry (the slots already exist
   in `session.Entry`; they are null today).
3. sub-agent task completion — the worker's last line is the terse protocol; GIT-AGENT commits
   with trailers `Task-Id` / `Agent` / `Session`; `task.done` carries `{task_id, output_path,
   sha[]}`.
4. sub-agent task reporting — `tasks.md` row updated, `logs.md` row appended, `graph.md` edge
   appended, ob1 `task` item updated; a failure also writes a `lesson` candidate (LESSONS) so
   the next run avoids it.

##### MAPPING TO WHAT EXISTS

| need | exists | gap |
| --- | --- | --- |
| task/agent types, terse protocol, `DependsOn` scheduler | `pkg/orchestration` (`Task`, `TaskResult{Done|Blocked|Error}`, `Supervisor.Spawn(tasks) <-chan TaskResult`, `ParseTerseResult`) | not bound to a model-backed worker |
| admission | `internal/spawnguard` (`Limits`, committed-memory ledger, `LogMux`) | production `Runner` = `deferredSpawnRunner` → "not yet wired" — **the Runner is deliverable #1** |
| identity | `pkg/identity` (`AgentID`, `TaskID`), JSONL `agent_id/supervisor_id/task_id`, `SessionCache.parent_id/spawn_depth` | never populated for children; no git trailers |
| events | bus `agent.spawn|stop|result`, `task.start|done|blocked|error` | no `graph.md` / `tasks.md` projection |
| richer model, if the in-process one proves insufficient | computeCommander: `Blueprint{depends_on, verify[], gates[], retry_limit, timeout, status}`, `gate` pipeline (lint/typecheck/test/security/format → `GateResult`), `mail` protocol (`dispatch/assign/worker_done/merge_ready/merged/escalation`, `@all/@builders`), `trace` causal chain, FIFO merge queue, watchdog nudges | a separate binary with its own SQLite — port the TYPES, not the daemon |

##### POLICIES

- depth 3, fan-out 8 per parent, `orchestration.max_turns` 50 and `turn_deadline_ms` per
  worker; a worker that trips `loopguard` is `failed` and is never retried with the same prompt
  (hash) — the retry carries the loop-break transcript path
  (`~/agentlogs/loop/<ts>-<sid>.jsonl`) as an input.
- worker input = task + "read `agents.md` and the local `agentic_instructions.md`"; nothing else.
- reviewer ≠ author (distinct session, no shared context); verdict is pass/fail with findings →
  `evals.md`.
- `graph.md` record (one line per edge change):
  `| run_id | task_id | parent | agent_id | role | depends_on | status | sha[] | started | ended |`

#### ICARUS [harness]

<!-- [+] added 2026-08-20 — what belongs to the icarus binary (as opposed to the /init skill) -->

The harness owns everything that needs a running session, a model, or a hook. The `/init`
binary owns nothing at runtime. `[harness]` deliverables this spec requires, in build order:

1. **T1 read seam** — on session start, if `<cwd>/agents.md` exists fold it (≤ 4 KiB) plus the
   head of `memory/rules.md` into the system prompt after `DefaultAgentSystemPrompt`
   (`internal/cli/agent_factory.go` assembly; stable-prefix invariant kept — it is part of the
   head, like the once-pinned ob1 block); brownfield `CLAUDE.md` next.
2. **T1 append seams** — one package `internal/projectmem`: `AppendJSONL(file, row)` /
   `AppendMD(file, line)` under a per-file flock, 5 MiB rotation, the ob1 id on every line. The
   ONLY code allowed to write `memory/{logs,events,loops,tasks,shas,lessons,rules,graph,evals}.md`
   and `memory/loops.json`; a `toolrules` pack `block`s `write`/`edit`/shell redirects into
   `memory/**` so no model edits them directly.
3. **Transcript enrichment** — stamp on `assistant_message`: `tokens{prompt, completion,
   cached}`, `model`, `provider`, `tool_calls[]{name, ms, ok}`, `sha[]`, `op_counter`. Today the
   JSONL holds only `text` + `stop_reason`; tokens/latency live write-only in
   `sessions/<project>/usage/<id>.jsonl` (`UsageRecord`). One struct change in
   `pkg/session/entry.go` (+ the writer mutex the LATENT race needs anyway).
4. **Memory ingest hook** — subscribe `message.assistant_end`, `compaction.done`, `session.end`,
   `task.done`, `skill.eval_fail` → MEMORY §M3, async, never on the turn path.
5. **Retrieval surfaces** — Ra (exists client-side: `internal/cli/context_injector.go` +
   `internal/hooks/handlers/steering.go`) moved onto the §M4 ranker; `memory.recall` tool; both
   read the same store with the same score.
6. **`/cmp --clear` as an operation** — `op_counter` on the session, the op table in the seed,
   REPL `/cmp` (web-only today), pointer seed ≤ 4 KiB (Data Flows [HARNESS]).
7. **Spawn Runner** — replace `deferredSpawnRunner` with an in-process `Worker` bound to a
   model-backed loop; thread `task_id` + `tools[]`; stamp identity on child JSONL (GRAPH).
8. **Eval hook** — `icarus sessions fail --id <sid> --eval <name>` (EVALS contention note) +
   `evals.md` append + `skill.eval_fail` event.
9. **Loop client** — `icarus loop define|status|stop|resume` writes `loops/*.toml` +
   `memory/loops.json` and installs the timer on monty over ssh (LOOPS); the runner is `[monty]`.
10. **Callback routes** — `/hook/*` (EVENTS).
11. **Agent definitions** — a loader for the **Claude Code agent format** (`~/.claude/agents/*.md`,
    `~/.claude/agents/.lazy/*.md`, `<project>/.claude/agents/*.md`: YAML frontmatter `name,
    description, tools, model, color, memory` + body), the way `internal/cccommands` already
    loads `.claude/commands/*.md`; icarus-only keys live in the same frontmatter under an
    `icarus:` block (tool groups, `max_turns`, `depth`, `skills`) that Claude Code ignores.
    Replaces the hard-coded `GET /agent` single entry; `[models.agents.<role>]` keeps the model
    pin; `agents.sources[]` + `agents.enabled[]` are the per-project configuration setting
    (TOOLS §AGENT DEFINITIONS). One file, both harnesses, shareable with contributors via git.

Config surface (every key lands in `docs/configuration.md` + a recipe — Continuous Docs Rule):
`project.read_agents_md` (true) · `project.append_memory` (true) · `memory.ingest{enabled,
provider, model, min_turn_tokens 400, max_candidates 8}` · `memory.recall{top_k 10, pool_min 60,
threshold 0.35, weights{vec 1, bm25 1, entity 0.5}, recency_half_life_days 30}` ·
`session.op_table` (true) · `spawn.runner` (`inprocess`) · `hooks.callback{enabled, bind,
token_env}` · `agents.sources[]` · `agents.enabled[]` · `agents.model_aliases{}` ·
`memory.models{embedder, extractor, reconciler, merger, rewriter, reranker}` (role → provider/model, MEMORY §M8.3).

##### REFACTOR MAP — what changes inside the harness (the refactor this spec implies)

<!-- [+] added 2026-08-21 -->

| today | becomes | why |
| --- | --- | --- |
| `pkg/session.Entry` — `assistant_message` is text + `stop_reason`; `UsageRecord` write-only in `usage/<id>.jsonl`; `jsonlWriter` unsynchronised | enriched entry (`tokens`, `model`, `provider`, `tool_calls[]`, `sha[]`, `op_counter`) + a mutex on the writer | the seed, the metrics and the LUT need it; closes the LATENT race |
| `internal/cli/context_injector.go` + `internal/hooks/handlers/{steering,ra}.go` — client-side Ra with its own ranker | `internal/memory` — ingest (§M3), recall (§M4), Ra as one surface of recall | one formula, one store, one dedupe |
| `internal/agent/clear.go` — carry = raw compact-index JSON clipped at 16 KiB | `internal/seed` — pointer seed + op table ≤ 4 KiB | pass by reference |
| `internal/web/cmp.go` — `/cmp` is web-only | one `/cmp` command registered for REPL + web | surface parity |
| `internal/cli/spawn_guard.go` — `deferredSpawnRunner` ("not yet wired"); `SpawnRequest` without `task_id`/`tools` | in-process Runner binding `pkg/orchestration.Worker` to a model loop; `task_id` + `tools[]` on the tool; identity stamped on child JSONL | multi-agent exists |
| `GET /agent` hard-coded single entry; roles only as `[models.agents.<role>]` pins | Claude-format agent loader + `icarus:` keys + `agents.sources/enabled/model_aliases` | shared agents across both harnesses and projects |
| `internal/skills` — frontmatter hints parsed, unenforced | `skill.toml` enforcement (`tools`, `max_calls`, `one_call_per_turn`, `max_result_bytes`) | loops can trust skills |
| `internal/cli/agent_factory.go` — system prompt without any project read | + T1 read seam (`agents.md`, `rules.md` head, `CLAUDE.md`) inside the stable head | projects steer the harness |
| `internal/hooks/handlers/context.go` — 60 / 90 % popups only | `internal/metrics/session_quality.go` — `ctx_pressure`, `latency_drift`, `quality_proxy` → compaction policy | compaction becomes a policy, not a popup |
| new packages | `internal/projectmem`, `internal/memory`, `internal/seed`, `cmd/icarus` `init` + `loop` + `sessions fail`, `/hook/*` routes | — |
| merged / removed | three Ra rankers → one; `tools.enabled` process-wide allowlist stays only as the OUTER bound of per-agent allowlists | fewer moving parts |

`memory/icarus.md` (the `[project]` file of this name) holds the per-repo harness facts: which
icarus version scaffolded it, the serve cwd, the model pins in force, the last `icarus init
--check` result.

#### LESSONS

- All systems will will self-improve, meaning learn from prior mistakes, bugs. There
  will be a specified format for lessons per each iteration.
- there will be a 2 stage trigger system for lesson injections:
  1. if there is a particular trigger for a lesson, it should a hook to invoke 'RA'
     should be created to inject the particular classification of lesson into the
     context window, prior to the model being called.
  2. Otherwise, RA will be responsible for injecting lessons into the context window,
     based on the lesson's classification, and the current context of the agent, and
     the ob1 clarity and classification and temporal relevance of the lesson to the
     current context of the agent. if there is an objective for the project that is
     listed, we should use kaparthy's autoresearch skill found at
     `~/Programs/autoresearch/`, which will live in the 'loops' directory.
- all lessons should also be logged into ob1. RA will be responsible for injection of
  the lesson into the context window. this will avoid the inefficiency of polling,
  and fully utilize the RA system to guide the agent to avoiding mistakes.

<!-- [+] added 2026-08-20 — the lesson record, when to log, the two triggers mapped to real mechanisms -->

- lesson record (one per iteration — `memory/lessons.md` JSONL + ob1 `observation` tagged
  `kind:lesson`):

```jsonl
{"id":"L-2026-08-20-017","ts":"2026-08-20T23:10:00Z","project":"icarus","agent_id":"worker-0a3f","task_id":"1755731400-9f3c2a1b/T7","session_id":"1755731400-9f3c2a1b","trigger":"eval_fail|user_correction|loop_break|review_reject|incident","classification":"build|test|git|memory|tooling|infra|style|spec","symptom":"go test -race: DATA RACE in pkg/session writer","cause":"jsonlWriter has no mutex; callers serialise ad hoc","rule":"serialise Append/Flush behind the writer's own mutex, not the caller's","evidence":["sha:4f2a1c9","log:~/agentlogs/loop/2026-08-20T23-09-00Z-1755731400.jsonl"],"confidence":0.8,"seen":1,"status":"candidate|confirmed|promoted|superseded","supersedes":null,"ob1":"019f…"}
```

  Human projection (what a reader sees first in `lessons.md`):
  `- [L-2026-08-20-017] (build · confirmed ×3) go test -race data race in jsonlWriter → serialise behind the writer's mutex · T7 · 4f2a1c9 · [ob1:019f…]`

- when to log a new lesson (RULES §1): ONLY the MEMORY agent's LESSON-LOG skill or the ingest
  pipeline writes one, and only from the five `trigger`s — never "I think I learned something".
  `seen` increments on hash-equal or cosine > 0.92 re-occurrence (the reconcile step, MEMORY §M3
  step 4); `seen ≥ 3` across ≥ 2 sessions ⇒ `promoted` ⇒ appended to `rules.md` (the consolidate
  loop, §M5).
- the 2-stage trigger, mapped to what exists:
  1. particular trigger ⇒ hook: a lesson whose `classification` has a deterministic detector
     becomes a **`toolrules` pack entry** (`~/.icarus/configs/toolrules/*.toml`, phase
     `pre-call`, action `inject | warn | block`, named Go detectors) or an **intent predicate**
     (`negation_check`, `contains_pattern`, `diff_validation`) — injected BEFORE the model call
     at zero retrieval cost. This is the "hook to invoke RA": deterministic detectors are Ra's
     fast path, and they are the only injection allowed to `block`.
  2. otherwise ⇒ Ra: the lesson is a memory like any other; Ra's per-turn ranked search
     (`handlers/steering.go`: threshold 0.4, cooldown 3 turns) moves onto the MEMORY §M4 ranker
     with `kind:lesson` weighted ×1.25 and `classification` matched against the turn's tool mix
     (build/test/git …). "ob1 clarity, classification and temporal relevance" = `s_vec`,
     classification match, and `s_time` (recency half-life + `last_verified_at`).
- objective present ⇒ autoresearch: `loops/autoresearch-<objective>.toml` (LOOPS) whose
  `results.jsonl` `discard | crash` rows are ingested as `lesson` candidates tagged
  `trigger:loop_break`.
- ob1 logging IS the ingest path (MEMORY §M3 step 5); Ra never polls — it is woken by the
  prompt hook chain (`message.user_start` / UserPromptSubmit).

#### LOGS

- Logs, and SHA lists should correlate with the tasks that are completed. This will
  allow for a more robust system of tracking what was completed, and what is in
  queue. This verification can and will count as a eval to ensure that tasks are
  being recorded in the system.

- formatting rules:

```jsonl
{
  "timestamp": "2024-06-01T12:00:00Z",
  "event_type": "task_completed",
  "task_type": "feature request|bug report| chore | research |  event | <other> ",
  "task_id": "12345",
  "agent_id": "agent_1",
  "details": {
    "description": "Completed task XYZ",
    "status": "< success|fail >"
  }
}
```

_under "other", we can have a list of other task types that are not feature requests
or bug reports, but are still relevant to the agent's operation. This could include
tasks like "data preprocessing ", "model training", "evaluation", etc. The goal is to
have a comprehensive log of all tasks that the agent is performing, so that we can
track progress and identify any issues that may arise._

<!-- [+] added 2026-08-20 — event_type enumeration, writers, the correlation eval -->

- `event_type` (closed set — adding one is a spec change): `init` ·
  `task_created|task_assigned|task_started|task_completed|task_blocked|task_failed` · `commit` ·
  `eval_pass|eval_fail` · `memory_add|memory_update|memory_supersede|memory_noop` · `recall` ·
  `lesson` · `loop_start|loop_done|loop_fail` · `compaction` (`/cmp`, `/cmp --clear`, with
  `op`) · `spawn|spawn_denied` · `webhook_in|webhook_out` · `gate_request|gate_ack`.
- `task_type` "other" list: `data_preprocessing`, `model_training`, `evaluation`, `memory`,
  `loop`, `infra`, `docs`, `spec`, `review`.
- every row also carries `session_id`, `run_id`, and (when relevant) `sha[]` and `ob1` — the
  pointers; numbers that already live elsewhere are NOT copied: per-call tokens stay in
  `~/.icarus/sessions/<project>/usage/<id>.jsonl` and are joined by `session_id` (pass by
  reference).
- writers: only `internal/projectmem` (`[harness]`, driven by bus events) and the `[init]`
  binary append `logs.md`; agents emit events, never write the file.
- the correlation eval (`general/logs_shas_tasks_consistent`, D0): every `task_completed` row
  has ≥ 1 `commit` row with the same `task_id` AND a `shas.md` line; every `shas.md` sha passes
  `git cat-file -e`; every `tasks.md` row in `accepted` has a `task_completed`; orphans ⇒ FAIL
  with the offending ids in the eval's log line.

#### LOOPS

- All systems will have the capability to run loops. These loops will will be
  determined during the course of use of the agent, and may be a suggestion from the
  agent, or a user defined loop.
- loop criteria:
  1. Loops are to be run as cronjobs, and ultimately should be executed on the host
     monty, which is also the inference host and always on.
  2. Loops should mostly be deterministic. There maybe some level of invariability,
     or probability to complete the loop's task, but there must be deterministic
     scaffolding that surrounds any model calls.
  3. Loops should be able to be run in the background, and not require user input.
     The user should be able to define a loop, and then have it run in the
     background, with the ability to check on its progress, or stop it if needed.
  4. Loops should execute skills, which will harness the deterministic scaffolding
     referenced in point 2. The skills will be defined in the 'skills' directory, and
     will be able to be called by the loop.
  - if there is an objective for the project that is listed, we should use kaparthy's
    autoresearch skill found at `~/Programs/autoresearch/`. This will be a loop
    executed at the optimal time, when usage of the agent is not needed.
  5. non-deterministic loops should be explicitly documented and justified, with clear exit conditions to prevent infinite looping.
  - each loop should have a defined interval, a trigger condition, and a completion criterion.
  - loop outputs should be logged and stored in `./memory/loops.md` for auditing and debugging.
  - loops may invoke other agents or scripts via `./scripts/loop_runner.sh`, which should handle error recovery, retries, and session persistence.
  - loop state should be persisted in `./memory/loops.json` to track execution history and prevent duplicate runs.
  - if a loop fails repeatedly, it should trigger a notification via webhook to the
    user or orchestrator, with details in the `logs.md` file and a corresponding entry in
    `events.md`.

<!-- [+] added 2026-08-20 — loop manifest, runner contract, state schema, the standard loops, existing timers -->

##### LOOP MANIFEST `[project]` — `loops/<name>.toml`

```toml
name        = "memory-consolidate"
skill       = "memory_consolidate"          # skills/<skill>/ (SKILLS contract) — a loop only ever calls a skill
host        = "monty"                       # always-on; lewis allowed only for dev runs
schedule    = "OnCalendar=*-*-* 03:45:00"   # systemd calendar, not cron (no crontab exists on either host)
trigger     = "schedule"                    # schedule | event:<event_type> | idle | manual
completion  = "exit0"                       # exit0 | metric:<name><op><value> | file:<path>
interval_ms = 0                             # trigger=idle: re-arm gap
max_runtime = "45m"                         # hard kill (systemd RuntimeMaxSec)
retries     = 2
determinism = "D1"                          # D0 pure | D1 bounded model call(s) | D2 non-deterministic (must justify)
justify     = ""                            # REQUIRED when determinism = "D2"; D2 loops are refused by default
notify_on   = ["fail", "repeated_fail"]     # → events.md + A.O push_mobile
git         = false                         # true ⇒ runner applies keep|revert around the skill (autoresearch shape)
model       = { provider = "sglang-monty", model = "qwen3.8-27b", effort = "low", max_calls = 20 }
```

##### RUNNER CONTRACT `[monty]` — `loop_runner`

(Go. The spec's `./scripts/loop_runner.sh` is the NAME; the artifact is a binary invoked by a
systemd service template `icarus-loop@<name>.service` + `.timer`.)

1. `icarus loop define loops/<name>.toml` validates the manifest, writes the `.timer`/`.service`
   pair to `~/.config/systemd/user/` on `host` (ssh BatchMode, never `-t`), `daemon-reload`,
   `enable --now`, and records the loop in `memory/loops.json`.
2. on fire: lock `memory/loops.json` → if `last_run.status == running` and its pid is alive ⇒
   `skipped` (duplicate-run guard) → set `running` → run the SKILL with `sd_notify` watchdog
   pings → capture exit code + metric → `done | fail` → append `memory/loops.md` (one human
   line) + `logs.md` (`loop_done | loop_fail`) → `fail` × `retries` ⇒ `notify_on` →
   `events.md` + A.O `push_mobile`; `consecutive_failures ≥ 3` ⇒ `suspended` until
   `icarus loop resume`.
3. state — `memory/loops.json`:

```json
{"memory-consolidate":{"manifest":"loops/memory-consolidate.toml","host":"monty","unit":"icarus-loop@memory-consolidate","runs":412,"last_run":{"started":"2026-08-20T03:45:00Z","ended":"2026-08-20T03:52:10Z","status":"done|fail|running|skipped|suspended","exit":0,"metric":{"merged":14,"expired":3,"promoted":1},"log":"~/agentlogs/loops/memory-consolidate/2026-08-20.jsonl"},"consecutive_failures":0}}
```

4. deterministic scaffolding around model calls is the SKILL's job (SKILLS contract); the runner
   enforces only `max_runtime`, `max_calls` (counts `provider.request` bus events) and, when
   `git = true`, the keep | revert discipline — exactly `~/Programs/autoresearch`
   `Engine.Run`: propose → commit → run → parse metric → `IsBetterByThreshold` ⇒ keep, else
   `RevertLastCommit`; crash ⇒ revert + `guard.RecordFailure()`
   (`max_consecutive_failures` 10, `max_total_crashes` 20).

##### STANDARD LOOPS

| loop | skill | schedule | det. | output |
| --- | --- | --- | --- | --- |
| `memory-consolidate` | `memory_consolidate` | 03:45 | D1 | MEMORY §M5 counts |
| `memory-embed-backfill` | `memory_embed_backfill` | hourly until the backlog is 0, then daily | D0 | embeds `embedding IS NULL` rows via the :8100 sidecar (10,529 today) |
| `autoresearch-<objective>` | `autoresearch` | idle window 00:30–06:00 | D1 (only `Propose` calls a model) | `results.jsonl`, advanced branch + shas, best metric, keep-rate |
| `autoresearch-ctx-threshold` | `autoresearch` | weekly | D1 | per-model `compact_at` (Data Flows [SESSION] step 6) |
| `evals-nightly` | `eval_environment` | 04:30 | D0 | `evals.md` pass/fail per suite |
| `host-inventory` | `host_inventory` | weekly | D0 | AI FLEET HOSTS JSON refresh |
| `agentlog-collector` → `agentlog-ingest` | (exist — lewis timers 02:00 → 03:00) | keep | D0 | `~/agentlogs` → ob1 `fleet-session` (today written with `embedding=NULL` — the backfill loop fixes that) |

Precedent to copy: `agentlog-collector.timer` (`Persistent=true`, 30 m ceiling, non-fatal per
host) and `ob-router-skill-eval.timer` (`OnFailure=` alert unit), both on lewis. monty today:
`openbrain-pipeline` 5 m (the Python sorter/bouncer/router/linker/compactor — INACTIVE on lewis
since July), `n0ko-ai-{harvest,inbox-sync,context-synth,eval-feed}` hourly, `ao-auth-refresh`
04:00. No crontab on either host; LOOPS standardise on timers.

#### MEMORY

<!-- [+] added 2026-08-20 — the memory system. Emulates the architecture taught in "Agent Memory
EXPLAINED – Complete Architecture" (Hugging Face, 2026-08-17, https://www.youtube.com/watch?v=aYfZN8t6AQs —
a dissection of Mem0), mapped onto the pieces that already exist (ob1, Ra, session JSONL,
compact-index, the ~/.claude auto-memory) and onto the project files /init stamps. -->

##### M0 — DEFINITIONS (what the video establishes, in our terms)

- **conversational memory** = the context window + the session JSONL (T0). LLMs are stateless;
  the harness re-sends history. `/cmp`, `/clear`, the digest bands (soft 140k / hard 165k on a
  175k window) and in-loop compaction are all T0 mechanics — they are NOT long-term memory.
- **long-term memory** = an EXTERNAL store, outside the session file, persistent across sessions
  and SHARED across agents (claude, pi, icarus, A.O) because it is centred on the USER and the
  PROJECT, not on one agent. Ours is ob1 (T2). The project files (T1) are its git-tracked,
  human-auditable projection for facts whose scope is one repo.
- **a memory** = one short declarative sentence + metadata — never a transcript chunk.
- **identity** = who a memory is about: `user` (preferences, history), `agent` (what it learned
  about the world, procedures), `project` (decisions, rules, shas), `run` (one task/loop
  execution — expires).
- **two moments of retrieval**: automatic on every turn (Ra) and explicit through a tool
  (`memory.recall`); one pipeline serves both.
- **ingestion** runs after every agent turn (at least every conversation) and is a SMALL model
  call + deterministic reconcile — never the main model, never on the turn path.

##### M1 — STORES (Mem0's three → ours)

| Mem0 store | role | ours (verified) | status / gap |
| --- | --- | --- | --- |
| main memory — vector DB of memory text + metadata: created/updated/expires, identity (user vs agent), creator attribution, content hash (exact-dedupe), lemmatised text (keyword search) | the memories | ob1 `items` table: Postgres + pgvector `vector(384)`, embeddings by `ob-embed-sidecar` :8100 (**all-MiniLM-L6-v2**, normalised); present columns: `id, item_type, priority, entities{tags[]}, confidence, status, routing_rule, corrections, captured_at, last_accessed_at, archived_at` + playbook `problem_signature, solution_pointer, working_example, last_verified_at` | exists. Missing: hash, lemmas/FTS, identity, `expires_at`, `supersedes`; `last_accessed_at` is never updated on read; confidence is hardcoded 1.0 on every MCP write; 10,529 rows unembedded; ~54 % of 65k items are bookkeeping (`file_change` 21.9k, `advisory-decision` 6.5k, `session-summary` 6.8k) |
| entity store — vector DB of entities, each linked to ≥ 1 memories; boosts retrieval, more when an entity has FEW memories | entity index | TrustGraph (`tg-bridge` forwards `project,task,observation,event` ≥ 20 chars; `ob.triples`, `ob.graph_search`, `ob.graph_query`) | reachable, but graph-rag returns EMPTY for project queries ("Knowledge graph: unavailable"); the Python Linker is dormant. v1 fallback = `entity:<slug>` tags inside ob1 + an in-process inverted index; v2 = TrustGraph once it answers |
| SQLite — (a) full change history of the vector store; (b) the last 10 messages sent to the pipeline (pronoun resolution) | history + recency | (a) `audit_entries{item_id, stage, action, details}` + `memory/logs.md` rows; (b) the session JSONL tail (`keep_recent_turns` 12 exists for the digest) | exists |

Project layer (beyond Mem0): `memory/*.md` — a repo must carry its own rules, lessons and shas
in git, reviewable in a PR, readable without ob1. Every write goes to ob1 FIRST (it mints the
id), then to the file projection with that id on the line.

##### M2 — THE MEMORY RECORD

```json
{
  "id": "01991c2e-…",
  "content": "icarus session JSONL never records tokens; per-call usage lives in sessions/<project>/usage/<id>.jsonl",
  "kind": "fact | preference | decision | lesson | procedure | rule | task | entity",
  "identity": "user | agent | project | run",
  "scope": {"project": "icarus", "host": "lewis", "agent_id": "base-0001", "task_id": null},
  "entities": ["icarus", "session-jsonl", "usage-jsonl"],
  "hash": "sha256(lower-cased, whitespace-collapsed content)",
  "lemmas": ["icarus", "session", "jsonl", "record", "token", "usage", "live"],
  "confidence": 0.9,
  "source": {"session_id": "1755731400-9f3c2a1b", "turn": 41, "sha": null, "ingest": "auto | explicit | loop"},
  "created_at": "…", "updated_at": "…", "expires_at": null, "last_verified_at": null,
  "last_accessed_at": null, "access_count": 0,
  "supersedes": null, "status": "active | superseded | expired | archived"
}
```

ob1 mapping — **v1, no schema migration** (everything above fits the existing row):
`kind`, `identity`, `scope.*`, `source.*`, `entities` → `entities.tags` (`kind:lesson`,
`identity:agent`, `project:icarus`, `host:lewis`, `session:<sid>`, `turn:41`, `entity:<slug>`,
`hash:<16hex>`); `expires_at` / `supersedes` / `lemmas` → `corrections` JSONB (written once ever
today — ours is the second writer); `status:superseded|expired` → `archived` + tag
`superseded_by:<id>` / `expired`. `item_type` per kind (ONLY valid labels — `ob.write` silently
downgrades unknown types to `reference`): lesson, fact, preference → `observation`; procedure →
`playbook` (+ `problem_signature`, `working_example`, `last_verified_at`); decision, rule →
`project`; task → `task`; entity → `reference`. **v2** (ob1-coder track, after v1 proves the
shape): first-class `memory` item_type with `hash`, `lemmas tsvector`, `expires_at`,
`superseded_by`, `access_count` columns and `last_accessed_at` maintained on read.

##### M3 — INGESTION (after every turn; `[harness]` hook → local model → ob1 → project files)

```
 turn ends ─► GATE ─► LOAD CONTEXT ─► EXTRACT ─► RECONCILE ─► PERSIST ─► AUDIT
  (bus)      deterministic  recent + relevant   small LLM,     ADD / UPDATE /   ob1 + T1      logs.md +
             skip rules     memories, last-10   JSON schema    SUPERSEDE / NOOP projection    memory-decision
```

1. **GATE** (deterministic, `[harness]`): skip when the turn has < `min_turn_tokens` (400) AND no
   tool call AND no correction marker (`no,` · `actually` · `don't` · `wrong` · `instead` · `stop`)
   — user corrections are the highest-value memories (the `feedback` type in the ~/.claude
   auto-memory); always run on `compaction.done`, `session.end`, `task.done`, `skill.eval_fail`.
2. **LOAD CONTEXT** (the extractor's prompt — exactly the video's list): role = memory
   extractor · user/project summary (`agents.md` `### Project` + ob1 `project` items) · the
   turn's messages (tool results elided to `name, ms, ok`) · recent memories (last 10 written
   for this project) · relevant memories (flatten the turn → embed → `/search` top-10) · last-10
   messages (session JSONL tail) · conversation date + current date.
3. **EXTRACT** (`memory.ingest.provider/model` = `sglang-monty` `qwen3.8-27b`, temperature 0,
   `guided_json` = the M2 schema minus server fields, `max_candidates` 8). The video's guidance:
   a 1–12B model is enough for extraction; ours is local and free. Output: `candidates[]`.
4. **RECONCILE** (the phase the video only gestures at; Mem0 paper arXiv 2504.19413 "update
   phase"): for each candidate fetch the top-5 similar ACTIVE memories (same identity + project)
   and decide with ONE small-model call: `ADD` (novel) · `UPDATE <id>` (same fact, more precise
   or newer) · `SUPERSEDE <id>` (contradiction — old item → `superseded`, link kept; NEVER a
   delete, per the append-only rule) · `NOOP` (hash-equal or cosine > 0.92). Deterministic
   overrides: hash-equal ⇒ NOOP before any model call; `kind:rule|lesson` can only be superseded
   through `memory.forget` with a gate ack (RULES). Note the Python Bouncer's cosine-0.95 dedupe
   is dormant — reconcile does its own.
5. **PERSIST**: `ob.write` (Go ob-mcp path: SERIALIZABLE insert + async embed via :8100) →
   entity linking (`entity:*` tags now; `ob.triples` when graph-rag answers) → T1 projection when
   `scope.project == cwd` and `kind ∈ {lesson, rule, decision, task}`: `projectmem.AppendMD`
   with `[ob1:<id>]` on the line.
6. **AUDIT**: one `logs.md` row per decision (`memory_add|update|supersede|noop`) and one ob1
   `memory-decision` observation (the shape of Ra's `advisory-decision`) — the labelled dataset
   for the M7 extraction eval and for fine-tuning the extractor later.

Budget: async, off the turn path; ≤ 2 model calls per turn (extract + reconcile); ≤ 3 s wall on
monty; ob1 unreachable ⇒ buffer to `~/.icarus/memory-outbox.jsonl` and drain next turn
(fail-open, like Ra). Never depends on the dormant Python pipeline.

##### M4 — RETRIEVAL (one pipeline, two surfaces)

Surfaces: **Ra** (automatic, per turn, ≤ `ra.max_advisories` 3 / `ra.max_bytes` 2048, identity
`project|agent|user`) and **`memory.recall`** (explicit: `{query, top_k 10, threshold 0.35,
identity?, kind?, project?}`), plus the session-boundary seed (Data Flows [SESSION]) = recall
with `query` = the seed summary.

```
 query ─► REWRITE (optional small-model call; off by default, on for recall with ≥ 12 words)
       ─► EMBED with the STORE's model (all-MiniLM-L6-v2 via :8100)
       ─► ANN pool = max(4·top_k, 60):  GET /api/v1/openbrain/search?q&limit&types&threshold
            (exclude_self_logs + exclude_tool_logs default ON — the ONLY read path that filters
             the file_change/advisory noise; never use /entries or ob.assist's semantic leg for this)
            + filters: status=active (not archived/superseded), identity, project
       ─► RERANK inside the pool only (nothing new enters):
            s_vec  ∈ [0,1]    cosine, min-max normalised over the pool
            s_kw   ∈ [0,1]    BM25 over lemmas(query) vs lemmas(memory), min-max normalised
            s_ent  ∈ [0,0.5]  entities(query) ∩ entities(memory):
                              boost = 0.5 / (1 + ln(1 + n_memories_linked_to_entity))
                              (ours — the video states "0–0.5, higher when the entity has fewer memories")
            s_time ∈ [0,1]    0.5^(age_days / recency_half_life_days 30); age from last_verified_at
                              when set, else updated_at                                   (ours)
            score  = (s_vec + s_kw + s_ent) / 2.5 · (0.7 + 0.3·s_time)
       ─► threshold ─► top_k ─► touch last_accessed_at / access_count (the forgetting signal)
       ─► return {id, content, kind, identity, age_days, confidence, source, verified} — NEVER embeddings
```

Wall time of recall — **no LLM is on this path.** Embed the query (MiniLM on the :8100 sidecar,
~5 ms), one pgvector ANN (~10–30 ms at 65k rows), an in-process rerank over ≤ 60 rows
(microseconds): budget ≤ 150 ms end to end, which is why Ra can afford it on EVERY turn. The
only optional model call (REWRITE) is a ≤ 40-token generation on the small tier (§M8.2), never
the 27B interactive model, off by default and never on Ra's path. Prefill overhead therefore
belongs to INGESTION (§M3), which is async and runs on its own endpoint (§M8.2).

Precedents already in the fleet: `claude-hooks context-inject` admits with
`0.70·sim + 0.20·tg + 0.10·kw ≥ 0.55 ∧ sim ≥ 0.45` (the same hybrid idea, different weights);
`ob.assist`'s reranker adds playbook +0.30, tag-match 0.10/hit (cap 0.30), recency on
`last_verified_at` (+0.15 < 30 d, +0.05 < 180 d), gate-align +0.10, duplicate −1.0; Ra's
reranker features are `[sem, tag_overlap, 1/(1+days/30), type_prior]`. M4 is the ONE formula
all three converge on (RA section). Defects it fixes: `ob_query` returns the 384-d vector
inline per hit (~1.5k tokens each); `ob.assist`'s semantic leg does not exclude tool logs; its
ob2 leg runs without `--project` (always searches openbrain's own stale index).

##### M5 — FORGETTING AND CONSOLIDATION (loop `memory-consolidate`, monty, 03:45, D1)

What exists: Python Compactor 90 d → archived, 180 d → compacted (dormant); `ob.cleanup`
(admin, `dry_run` default true); playbook `TouchLastVerified`; the context-router `Compressor`
computes snapshots but never writes them (`sweep()` lacks `client.Write`). The loop replaces
all of it with six deterministic passes (only #3 calls a model):

1. expire: `expires_at < now` ⇒ `expired` (all `identity:run` memories default to 14 d).
2. decay: `access_count == 0 ∧ age > 90 d ∧ kind ∈ {fact, preference}` ⇒ `archived`
   (never lessons, rules, decisions).
3. merge: pairs with cosine > 0.92 inside one identity+project ⇒ one small-model merge ⇒
   `UPDATE` the survivor, `SUPERSEDE` the other.
4. promote: `kind:lesson` with `seen ≥ 3` across ≥ 2 sessions ⇒ `rules.md` + tag `promoted`;
   `kind:procedure` used ≥ 3× ⇒ candidate skill ⇒ `memory/skills.md` (the "extract a skill from
   the transcript" path the video prefers over raw procedural memory).
5. verify: playbooks with `last_verified_at` older than 30 d whose `working_example` is a
   command ⇒ re-run it; failure ⇒ tag `needs_verification` (Ra shows it with a warning).
6. bookkeeping TTL: `file_change` observations > 30 d and `advisory-decision` rows > 90 d ⇒
   `archived` (they stay in `audit_entries`); honour `ra-curation verdict:stale` flags (written
   by `@ra`, read by nothing today) ⇒ `superseded`.
7. report: counts to `memory/loops.md` + `logs.md`.

##### M6 — WHAT THE SEED CARRIES (pass by reference)

The post-`/cmp --clear` seed is recall, not prose: `{window, compact_at, op_counter, op_table,
commits[] {subject, sha, ts, task_id}, turn→file table, top-K memory ids for the project}`; the
model fetches content by id (`memory.recall`, `git show`, `read`) only when needed. Budget
≤ 4 KiB (today's carry clips 16 KiB of compact-index JSON).

##### M7 — EVALS (pass/fail, EVALS rules)

| eval | metric | pass |
| --- | --- | --- |
| `memory/extract_precision` | labelled turns (from `memory-decision` audit rows, human-marked) | precision ≥ 0.8, recall ≥ 0.6 |
| `memory/dedupe` | re-ingesting the same turn | 0 ADDs |
| `memory/recall_hit` | gold (query, id) pairs from `@ra` affirmed/applied decisions; `memory.recall` top-3 contains the gold id | hit@3 ≥ 0.7 |
| `memory/cost` | tokens + wall per ingest | ≤ 6k tokens, ≤ 3 s |
| `memory/projection` | every T1 line has a resolvable `[ob1:id]` | 100 % |
| `memory/no_embeddings_on_wire` | recall responses | 0 vectors |
| `memory/backlog` | `embedding IS NULL` count | 0 after the backfill loop |

Ra's feedback corpus today is 6,306 `surfaced` / 112 `rejected` / 0 `affirmed` — untrainable.
The `# Steering` echo contract already makes every assistant say "applies" or "not relevant";
parsing that into `verdict:applied|dismissed` on the decision row is the positive label M7 needs.

##### M8 — MODELS: fully local, tiered, swappable

<!-- [+] expanded 2026-08-21 — answers: is it fully local? which model where? how do we swap when the hardware changes? -->

**8.1 Fully local by construction.** Mem0's defaults are cloud (GPT-5-mini extraction, OpenAI
embeddings); the video's closing advice is to run every stage on small open models — we do,
and the config enforces it: `memory.models.*` accept only providers tagged `local: true`
(a destination constraint, like `compaction.provider`: a pinned local provider that is down
makes the stage DECLINE, it never falls back to a cloud model). Where a model is called at all:

| step | model call? | tier | runs on | cost |
| --- | --- | --- | --- | --- |
| M3 GATE | no | — | harness | 0 |
| M3 LOAD CONTEXT (embed the flattened turn) | embedder | T-embed | `ob-embed-sidecar` :8100 | ~5 ms |
| M3 EXTRACT | yes — 1 call, ≤ ~3k-token prompt, JSON out | T-small | dedicated endpoint (8.2) | 0.3–1.5 s, async |
| M3 RECONCILE | yes — 1 call, ≤ ~1.5k tokens | T-small | same | 0.2–0.8 s, async |
| M3 PERSIST / AUDIT | no | — | ob-mcp + `projectmem` | ms |
| M4 RECALL (Ra + `memory.recall`) | embedder only; REWRITE optional | T-embed (+ T-small) | sidecar | ≤ 150 ms |
| M5 passes 1, 2, 4, 5, 6 | no | — | loop on monty | nightly |
| M5 pass 3 (merge) | yes — n pairs | T-small | dedicated endpoint | nightly |
| Ra surface, seed, evals | no | — | harness | 0 |

The INTERACTIVE model (`sglang-monty` `qwen3.8-27b`, the one you talk to) is never used by the
memory system — deliberately (8.2).

**8.2 Tiers — the decision.** Extraction/reconcile is a short, simple, schema-constrained task;
the video's guidance (1–12B is enough) matches the fleet's own measurements: a 27B MoE pays
prefill on every ingest prompt AND, when it shares the interactive model's sglang instance,
each ingest prompt competes for the prefix cache the session depends on (measured 2026-08-15:
a 28k-token prefill is 4.35 s cold vs 0.98 s cached — 4.4×). So the memory tier must be a
SEPARATE endpoint, and a smaller model is both cheaper and better isolated:

| option | host / device | model class | pros | cons |
| --- | --- | --- | --- | --- |
| **A (recommended)** | monty — the RTX PRO 4500 32 GB sitting next to the 6000 | Qwen3-8B-class instruct (bf16 or FP8), vLLM/sglang, 16–32k ctx, `guided_json` | no prefix-cache contention with the 27B; always-on; fast decode; headroom to batch ingest + merge + rewrite | a second serving process on monty to operate |
| B | cai — R9700 32 GB, llama.cpp :8090 | Qwen3-8B / 4B GGUF Q8 | zero impact on monty; a spare GPU | FA collapses on radv (known), slower prefill, one more host in the loop |
| C | lewis — Strix Halo iGPU (Vulkan) | Qwen3-4B / 1.7B GGUF | local to the harness, no network hop | eats the laptop's unified memory; slow prefill on the iGPU — dev fallback only |
| D | the 27B on the 6000 | qwen3.8-27b | nothing new to run | prefix-cache eviction on every ingest; 3–5× the prefill cost of A for no recall gain — rejected |

**DECIDE:** A vs B for T-small (C is the dev fallback, D is rejected). The spec assumes A.
Recall QUALITY is decided by the embedder and the reranker, not by T-small (8.3).

**8.3 Swap procedure (hardware is changing — a swap must be one line).** Every memory model is
a named ROLE resolved through the providers registry, never a literal model id in code:

```json
"memory": { "models": {
  "embedder":   { "provider": "ob-embed",     "model": "all-MiniLM-L6-v2", "dims": 384 },
  "extractor":  { "provider": "vllm-monty-s", "model": "qwen3-8b", "effort": "low" },
  "reconciler": "extractor",  "merger": "extractor",  "rewriter": "extractor",
  "reranker":   null
}}
```

- T-small roles — swapping = one provider/model pair (or an alias); nothing else moves because
  every prompt is schema-constrained (`guided_json`) and the M7 evals (`extract_precision`,
  `dedupe`, `cost`) are the acceptance gate: `icarus memory models` prints the live resolution,
  `icarus memory eval` runs the gate.
- EMBEDDER — the one expensive swap, designed for from day one: every item carries
  `embed:<model>@<dims>` (tag in v1, column in v2); queries filter on the CURRENT embedder; a
  swap starts `memory-embed-backfill` in dual-write mode (new vectors into a second pgvector
  column, old column kept); cut over when backlog = 0 and `recall_hit` on the new column ≥ the
  old; drop the old column one consolidation cycle later. Candidates when the hardware lands —
  all local, chosen by the `recall_hit` A/B, not by a leaderboard: bge-base-en-v1.5 (768-d),
  nomic-embed-text-v1.5 (768-d, long inputs), Qwen3-Embedding-0.6B (1024-d, multilingual).
  MiniLM-L6 (384-d) is the fastest and the weakest of the set.
- RERANKER (optional, D1) — a cross-encoder (bge-reranker-v2-m3 class) over the 60-row pool on
  T-small's GPU adds ~50–150 ms and is the highest-precision lever after the embedder; it is a
  role too, `null` today.
- `~/.claude/model_engines.toml` (the lcp engine registry) is the claude-side twin; the two
  registries share names through `agents.model_aliases`.

**8.4 Fine-tuning.** The `memory-decision` audit rows + the archived `/cmp` JSONL are the
dataset (LoRA roadmap); fine-tune T-small for extraction only once M7 is green on the base
model — the order the video recommends.

##### M9 — REPAIR BEFORE BUILD `[ob1]` (ob1-coder dispatch; blockers found 2026-08-20)

1. embed the backlog (10,529 NULL rows incl. every `fleet-session`) — Go backfill loop.
2. `ob-context-router` passes `--ob-socket` on a socket-less host (2.0 M poll errors, 0
   processed); its Compressor never persists (`sweep` missing `client.Write`).
3. `ob.assist`: exclude tool/self logs on the semantic leg; pass `--project` to ob2.
4. `ob.write`: refuse unknown types instead of silently writing `reference`; `ob.session.*`
   should write `session`, not `reference`.
5. warm-start "pending tasks" = 3 newest `task` items with no status filter inside a hidden 72 h
   `/entries` default — filter `status` and drop the window.
6. TrustGraph graph-rag returns empty for project queries although TCP-healthy — fix or demote
   the entity store to v1 tags until it answers.
7. **Cloud leak:** the dormant Python Sorter classifies through `litellm`
   `anthropic/claude-opus-4-8`. Re-enabling `openbrain-pipeline.timer` as-is would send every
   captured item to a cloud model — repoint the Sorter to T-small (§M8) or retire it: M3
   EXTRACT already classifies.

##### M10 — WHAT RA CAN SEE, AND WHEN

<!-- [+] added 2026-08-21 — "once ob1 is repaired and the distribution pipelines are back, does Ra see everything?" -->

Not automatically. Ra sees an item only if ALL of these hold — each is a separate fix:

| gate | today | after |
| --- | --- | --- |
| embedded | 10,529 rows NULL (every `fleet-session`) → invisible to semantic search | backfill loop (M9.1) ⇒ everything embedded; `memory/backlog` = 0 |
| item type allowed by Ra's query | `types=observation,project,task,event,contact,reference,idea,playbook` — `fleet-session`, `session`, future `memory` are NOT in the list | Ra moves onto the M4 ranker, which filters by `status=active` + `kind` + `identity`, not by item_type |
| not excluded as noise | `exclude_self_logs` / `exclude_tool_logs` drop `file_change`, `ra-steering`, `advisory-decision` (correct) | unchanged — and M5.6 archives them so they stop dominating the pool |
| survives the pool | `limit=10`, one best hit | pool `max(4k, 60)`, rerank, ≤ 3 advisories |
| not already surfaced | per-session set + 3-turn cooldown; nothing cross-session | cross-session dedupe by id (RA) |
| reachable | WG-only ob-mcp; the router's socket flag is broken | M9.2; hooks already fall through unix → `OB_SSE_URL` |
| graph leg | graph-rag answers empty | M9.6; until then `entity:*` tags carry the boost |
| distribution pipelines (Router → todoist / knowledge_base / contacts / calendar; TrustGraph forwarding) | dormant | re-enabling them changes DESTINATIONS, not Ra's visibility — Ra reads ob1 + the graph, never the destinations |

So: repaired ob1 (M9) + the M4 ranker ⇒ Ra can see every ACTIVE memory across claude, pi,
icarus and A.O sessions, ranked by one formula; `kind` and `identity` become the gates instead
of the item-type list.

#### RA

(Reactive Agent)

- RA is responsible for orchestrating agent behavior, managing context, and enforcing system rules.
- RA maintains a dynamic context window that adapts based on task complexity, session duration, and model performance metrics.
- RA is currently wired as a hook-driven advisory channel, not a standalone orchestrator: `ob-advisory-inject` runs on `UserPromptSubmit` and injects a `<ra-advisory>` block into session context when relevant; the assistant is contractually required to echo it (italic `> *☉ Ra: ...*` line) and state apply/dismiss before proceeding.
- RA's memory substrate is ob1 (OpenBrain), not local `logs.md`/`events.md`/`loops.json` files — those remain project-local artifacts for traceability, but RA's actual read/write path is `mcp__openbrain__ob_query`, `ob_read`, `ob_write`, and `ob.assist` (fan-out over ob1 + ob2 + graph, playbook-first re-ranked).
- RA dynamically injects lessons/advisories into the context window based on:
  1. Trigger conditions (hook firing at session start / prompt submit, not explicit per-task classification hooks)
  2. Temporal relevance and objective alignment, sourced from ob1 session summaries and recent observations
  3. ob1 clarity and task classification, resolved via `ob2_gate` (semantic vs structural routing) before querying `ob1` or `ob2`
- Session engagement is boundary-based, not continuous: RA surfaces once per turn (or once per session at warm-start) via hook injection rather than maintaining an always-on in-loop presence; every `ob.assist` call writes back a self-learning `assist-decision` observation to ob1 for future hit-rate tuning.
- RA ensures all agent outputs adhere to coding standards, data type optimization, and abstraction depth limits (max_depth=3).
- RA manages loop execution via `./scripts/loop_runner.sh`, including error recovery, retries, and state persistence in `./memory/loops.json`.
- RA enforces zero-heap-allocation principles where possible, leveraging unsafe Go and careful struct packing to minimize runtime overhead.
- RA validates SHA lists and logs to ensure every task has a corresponding record, enabling full traceability of system evolution.
- RA acts as the central intelligence for self-improvement, using feedback from logs,
  lessons, and evals to refine futureagent behavior, ensuring that every iteration of
  the system is more efficient, reliable, and aligned with project objectives. RA also
  ensures that all agent interactions are secure, auditable, and optimized for
  low-latency execution across distributed environments. RA continuously monitors the
  system for anomalies, inefficiencies, and deviations from defined objectives,
  triggering self-correction protocols when necessary. It ensures that all agent
  behaviors align with the project's goals, maintains strict adherence to coding and
  data standards, and dynamically adjusts context depth and task prioritization based
  on real-time performance feedback. By integrating with logs, events, and loop state,
  RA enables a closed-loop system of continuous improvement, where each iteration
  learns from past outcomes and refines future execution—ultimately achieving
  a self-sustaining, adaptive, and highly efficient agent ecosystem.

<!-- [+] added 2026-08-20 — Ra = the retrieval surface of MEMORY; three implementations → one contract -->

- Ra exists three times today, with three rankers: (1) claude-side `ob-turn-advise` (Stop:
  builds a prose query from project + prompt + touched files + `# Steering`, semantic search
  threshold 0.4 over 8 types, per-session surfaced set + 3-turn cooldown, one-shot pending
  seed) + `ob-advisory-inject` (UserPromptSubmit: renders `<ra-advisory>`, 200 ms budget,
  cold-start fallback) from `~/openbrain/hooks/go`; (2) icarus-side
  `internal/cli/context_injector.go` + `internal/hooks/handlers/steering.go` (one ranked
  pgvector search per turn, threshold 0.4, cooldown 3 turns, per-session dedupe); (3) coms-go
  embedded `serve --ra` (tap → per-agent window → tiered brain → SSE steering → ob1
  `advisory-decision`). ob-mcp has no server-side Ra. **Contract:** all three call the one
  MEMORY §M4 ranker server-side (`ob.assist` is the seam) so a lesson learned in a claude
  session surfaces in an icarus session with the same score. The LESSONS phrase "ob1 clarity,
  classification and temporal relevance" is exactly `s_vec`, `kind/classification` match, and
  `s_time`.
- budgets stay as configured: `ra.max_advisories` 3, `ra.max_bytes` 2048,
  `ob1.inject_max_bytes` 2048. Add a CROSS-session dedupe keyed by memory id (today the
  surfaced set is per session only, so a new session re-surfaces the same item immediately):
  an advisory is shown at most once per `{project, 7 days}` unless its `updated_at` changed or
  the user asked (`@ra`).
- every surfaced advisory writes `advisory-decision` (exists) AND touches `last_accessed_at` /
  `access_count` on the memory (new) — that is how Ra's hit-rate feeds forgetting (§M5) and the
  `recall_hit` eval (§M7). The `# Steering` echo ("applies" / "not relevant here because …")
  is parsed by `ob-reply`/`steering.go` into `verdict:applied|dismissed`; `@ra` `corrected`
  (`ra-curation verdict:stale`) must be READ by retrieval (today it is write-only).
- Ra's "dynamic context window" bullets map to real knobs it reads but does not own: digest
  bands (`harness.resident.context_digest` soft 140k / hard 165k), `compaction` thresholds, the
  [SESSION] threshold policy.
- `memory/ra.md` (`[project]`): the per-repo Ra ledger — advisories surfaced in this repo with
  verdicts, one line each (`ts · [ob1:id] · applied|dismissed|stale · turn`), so a PR reviewer
  can see what the agent was told.

#### RULES

- ob1
- ra
- agent usage (all of the following should be referenced in the actual agent
  available to project and/or system ):
  1. coding styles
  2. data types
  3. abstraction layers (max_depth=3)
  4. function composibility

<!-- [+] added 2026-08-20 — the gated-removal rule and the formatting rules as executable contracts -->

- append-only + gated removal: `lessons.md`, `rules.md`, `shas.md` are append-only. The ONLY
  removal path is `memory.forget(id, reason)` → `gate_request` event → human ack (web strip, or
  A.O Discord when AFK) → the line is NOT deleted: it is suffixed
  `⟂ superseded by <id> (<reason>, <ts>)` and the ob1 item becomes `superseded`. Eval
  `general/memory_files_append_only`: `git log -p` on those files shows no removed lines other
  than `⟂` rewrites.
- ob1 rule: every T1 line carries its `[ob1:<id>]`; ob1 is QUERIED, files are READ; a file
  without an id pointer is a bug (`memory/projection` eval).
- ra rule: advisories are echoed and acknowledged (`# Steering`), never silently dropped; the
  acknowledgement is data (RA).
- agent usage rules live in `memory/rules.md` and are injected through the T1 read seam:
  1. coding styles — gofmt/vet clean, wrapped errors, table-driven tests, no dead code.
  2. data types — struct packing (largest fields first), no `interface{}` on hot paths, fixed-size
     ints where the domain is bounded, `[]byte` over `string` for transient buffers.
  3. abstraction layers (`max_depth=3`) — `cmd/` → `internal/<service>` → `pkg/`; a fourth hop
     is a review finding.
  4. function composability — one purpose per function, ≤ 40 lines, pipelines via
     bitfield/script; side effects only at the edges (`projectmem`, providers).
- lesson format rule = the LESSONS record; spec rules = `SPEC/<name>/` layout (global
  CLAUDE.md); housekeeping = the `[init]` tree (`agents.md`, `memory/`, `evals/`, `loops/`,
  `skills/`, `.icarus/`) and nothing spec-shaped at the repo root.

#### SHAS

<!-- [+] added 2026-08-20 — record format, writer, regenerability -->

- `memory/shas.md` record (one line per logical commit, written by GIT-AGENT's SHA-LOG skill
  through `projectmem`, never by hand):
  `- <sha7> · <subject> · <ts> · <run_id>/T<n> · <agent_id> · <session_id> · [ob1:<id>]`
- the commit carries the same pointers as trailers (`Task-Id`, `Agent`, `Session`), so
  `git log --format='%h %s %(trailers:key=Task-Id,valueonly)'` regenerates the file — eval
  `general/shas_regenerable` diffs the two and FAILS on any line present in one but not the
  other.
- seed usage (Data Flows [SESSION] dependency 2): "work completed" = the `shas.md` lines since
  the last op, verbatim — `[logical commit subject – sha – timestamp]` is this line with the
  pointers dropped.
- the ~/.claude `lcp/findings.jsonl` ledger (`{session_id, task_id, round, diff_sha, verdict,
  severity, …}`) is the review-time cousin of this file; GIT-AGENT links them by writing the
  post-commit `sha` back onto the finding row's `diff_sha` (pre-commit) so review verdicts and
  landed commits are one chain.

#### SKILLS

<!-- [+] added 2026-08-20 — the skill contract; every category below is an instance of it -->

##### SKILL CONTRACT `[harness]` + `[project]`

What exists: a native skill is `<name>/skill.md` (YAML frontmatter `name, description, tools[],
guided_json, one_call_per_turn, max_result_bytes` + body), discovered from
`<repo>/.icarus/skills` → `$ICARUS_SKILLS_DIR` → `~/.config/icarus/skillsets`
(`skillsets/`, not `skills/` — that dir belongs to `icarus evolve`); `/name args` expands the
body into the outbound turn (`{{args}}`). The four affordance hints are parsed and NOT enforced
(`SPEC/skill_runtime` §9). The deterministic seams that DO exist are `.icarus/tools.toml`
custom tools (argv → schema'd tool, params via stdin JSON + `$ICARUS_PARAM_*`), `toolrules`
packs, `loopguard`, `orchestration.max_turns`, the `# Outcome` intent gate, and `icarus evolve`
(`eval.toml` case sets scored by six intent predicates). `/tf` shows the current ceiling: its
determinism is a bash harness embedded in the PROMPT BODY and its declared tool names do not
exist in the catalog.

A skill in this spec = **deterministic scaffolding + ≤ N bounded model calls + an eval + one log
line**. Manifest = the existing `skill.md` frontmatter PLUS a `skill.toml` sidecar that the
runtime ENFORCES (new `[harness]` work: make `tools`, `one_call_per_turn`, `max_result_bytes`
real, add the fields below):

```toml
# skills/<name>/skill.toml
name          = "memory_consolidate"
determinism   = "D1"                  # D0: no model call (pure Go/argv)   D1: bounded calls   D2: open-ended (justify)
max_calls     = 20                    # provider.request budget; the runtime aborts at N+1
model         = { provider = "sglang-monty", model = "qwen3.8-27b", effort = "low" }
tools         = ["memory.*", "ob1.*", "fs.append:memory/**"]   # ENFORCED allowlist (group | tool | tool:glob)
inputs        = ["project", "since"]                              # {{args}} names
outputs       = ["memory/loops.md", "memory/logs.md"]             # files the eval checks
entry         = { kind = "go", cmd = ["icarus-skill-memory-consolidate"] }   # D0/D1 code path; omit ⇒ prompt-only
eval          = "evals/custom/memory_consolidate"                # pass/fail script (EVALS CUSTOM dispatcher)
log_event     = "loop_done"                                       # LOGS event_type emitted on success
hosts         = ["monty"]
```

Rules: D0 skills MUST have `entry.kind = go | argv` and no prompt body; D1 skills put the model
call BEHIND code (the code calls the provider, not the reverse — the autoresearch `Propose`
shape); D2 skills are allowed only interactively, never in LOOPS; every skill has an eval; a
skill without `skill.toml` is a plain prompt skill (today's behaviour) and is barred from loops.
A skill's model pin is a destination constraint, like `compaction.provider`: if the pinned local
provider is down the skill declines, it does not fall back to a cloud model.

[g-suite]

- email
- calendar_scheduling
- navigation
- traffic_patterns

[harness]

- eval_environment
- r2d2
- server_creation: based on events icarus has the ability to:
  - create additional sub_domains on owned domains
  - route, via cloudflare, routes to created server, or additionally added endpoints
  - g2 features
  - create new virtual machines with pre-configured environments
  - deploy infrastructure-as-code (IaC) templates (Terraform/Ansible)
  - integrate with CI/CD pipelines for automated provisioning
  - monitor resource utilization and scale dynamically
  - rollback to previous configurations in case of failure
  - log all actions to `./memory/loops.md` and update `./memory/loops.json`
  - trigger notifications via webhook on successful deployment or failure
  - self_healing
- recall

[ai]
_these skill will need to be created, but never in context unless injected --
also developed as needed_

- model_training:
  - rl
  - sft
- data_preprocessing
- evaluation
- deployment:
  - kernel
  - runtime
- monitoring
- feedback_collection
- documentation_generation
- dependency_management
- security_scanning
- performance_benchmarking
- configuration_management
- version_control_integration
- artifact_storage
- environment_setup
- data_validation
- feature_engineering
- hyperparameter_tuning
- model_evaluation
- model_explanation
- data_augmentation
- model_versioning
- continuous_integration
- continuous_deployment
- automated_testing
- error_handling
- logging_and_monitoring
- resource_optimization
- scalability_testing
- fault_tolerance_testing
- backup_and_recovery
- compliance_checking
- audit_trail_generation
- user_access_control
- permission_management
- data_encryption
- secure_communication
- identity_verification
- threat_detection
- incident_response
- system_health_check
- load_testing
- stress_testing
- capacity_planning
- cost_optimization
- energy_efficiency_monitoring
- environmental_impact_assessment
- sustainability_reporting
- code_review
- technical_debt_management
- refactoring
- documentation_update
- knowledge_base_sync
- onboarding_assistance
- offboarding_cleanup
- license_compliance

<icarus_skills>
-- these skills are not to be generated per project, but soley for icarus --

- self*healing: \_networking -> router pass-through or reticulum node setup (fully
  internal network)*
- server_creation: based on events icarus has the ability to:
  - create additional sub_domains on owned domains
  - route, via cloudflare, routes to created server, or additionally added endpoints
  - g2 features
  - create new virtual machines with pre-configured environments
  - deploy infrastructure-as-code (IaC) templates (Terraform/Ansible)
  - integrate with CI/CD pipelines for automated provisioning
  - handle SSH key management and secure access control
  - monitor resource utilization and scale dynamically
  - rollback to previous configurations in case of failure
  - log all actions to `./memory/loops.md` and update `./memory/loops.json`
  - trigger notifications via webhook on successful deployment or failure
  - self_healing

<icarus_skills\>

[future functionality]

- robotic_controls: enables integration with physical systems via API endpoints,
  supporting real-time control of robots, sensors, and actuators (mech-arms,
  drones, mech-dog?, tesla - human_bot, automotive?)

consideration:

```
- model_trainingICARUS
- data_preprocessing
- evaluation
- deployment
- monitoring
- feedback_collection
- documentation_generation
- dependency_management
- security_scanning
- performance_benchmarking
- configuration_management
- version_control_integration
- artifact_storage
- environment_setup
- data_validation
- feature_engineering
- hyperparameter_tuning
- model_evaluation
- model_explanation
- data_augmentation
- model_versioning
- continuous_integration
- continuous_deployment
- automated_testing
- error_handling
- logging_and_monitoring
- resource_optimization
- scalability_testing
- fault_tolerance_testing
- backup_and_recovery
- compliance_checking
- audit_trail_generation
- user_access_control
- permission_management
- data_encryption
- secure_communication
- identity_verification
- threat_detection
- incident_response
- system_health_check
- load_testing
- stress_testing
- capacity_planning
- cost_optimization
- energy_efficiency_monitoring
- environmental_impact_assessment
- sustainability_reporting
- code_review
- technical_debt_management
- refactoring
- documentation_update
- knowledge_base_sync
- onboarding_assistance
- offboarding_cleanup
- license_compliance
```

<!-- [+] added 2026-08-20 — new categories (instances of the contract) + the agent → skills map -->

[memory] `[harness]` / `[project]` — the skills MEMORY defines

- recall (D1 — the optional rewrite call; §M4)
- remember (D1 — extract + reconcile; §M3)
- lesson_log (D1; LESSONS)
- memory_consolidate (D1, loop; §M5)
- memory_embed_backfill (D0, loop; §M9)
- forget (D0 + human gate; RULES)
- transcript_digest (D1; RESEARCHER DIGEST — `yt-dlp` / `webfetch` → summary → remember)

[init] `[init]`

- init (D0): the scaffolder (`icarus init`; `--check --dry-run --no-git --dir-instructions --force`)
- host_inventory (D0): refresh AI FLEET HOSTS

[evals] `[project]`

- eval_environment (D0): `make evals` → GENERAL + CUSTOM dispatchers → `evals.md`
- logs_shas_tasks_consistent (D0): the LOGS correlation eval
- memory_evals (D0/D1): the §M7 table

[loops] `[monty]`

- loop_runner (D0)
- autoresearch (D1): wraps `~/Programs/autoresearch` `Engine.Run`; inputs `script, metric,
  direction, target_file, run_command, timeout, iterations | budget, threshold, llm_backend /
  llm_model (local), branch tag`; outputs `results.jsonl` (`timestamp, iteration, channel,
  commit, metric_name, metric_value, best_metric, status keep|discard|crash|conflict,
  duration_seconds, description, patch_summary`), the advanced branch + shas, best metric,
  keep-rate
- inference_health (D0)

[agent → skills]

| agent | skills |
| --- | --- |
| ORCHESTRATOR | decompose, dispatch, reconcile |
| CODER | implement (D2, interactive only), build_test (D0: `make build && make evals`), lsp_blast_radius (D0) |
| REVIEWER | review (D1, read-only tools), eval_environment (D0), lesson_log (D1) |
| RESEARCHER | digest, compare, watch, transcript_digest |
| GIT-AGENT | logical_commit, sha_log, blame_attribute |
| MEMORY | recall, remember, lesson_log, memory_consolidate, forget |
| SYS-ADMIN | file_manager, host_inventory, service_check |
| (monty) | loop_runner, autoresearch, inference_health, memory_embed_backfill |

The `[ai]` list above is a REGISTRY OF NAMES: each becomes a skill only when a project needs it,
authored against the contract (never in context unless injected — as already stated). The
`consideration:` block is the same registry and can be collapsed into one list when you
decide which names survive.

#### TASKS

- Logs, and SHA lists should correlate with the tasks that are completed. This will
  allow for a more robust system of tracking what was completed, and what is in
  queue. This verification can and will count as a eval to ensure that tasks are
  being recorded in the system.
- task formatting rules
- inevitably, one of 'events' will be a callback for 'feature requests' and 'bug
  reports'. These will be logged into the 'tasks' file and used as a gate against
  repeat, redundant feature requests or bugs reports that can be automatically closed
  by an agent. **known issue: as this list grows it will be an artifact worth
  considering in the gitignore. it might also be worth calling out it's location and
  referencing in any collaboration rules, to check against as a human gate to
  redundant requests that eat wall time, and compute resources for no reason**

<!-- [+] added 2026-08-20 — task record, id scheme, the dedupe gate, the gitignore call-out resolved -->

- task record (`memory/tasks.md` JSONL + ob1 `task` item; the file is the queue humans read):

```jsonl
{"task_id":"1755731400-9f3c2a1b/T7","run_id":"1755731400-9f3c2a1b","parent":"T2","type":"feature request|bug report|chore|research|event|memory|loop|review","title":"thread task_id through spawn","hash":"sha256(normalised title + body)","status":"pending|assigned|running|blocked|review|accepted|rejected|failed|duplicate","agent_id":"worker-0a3f","depends_on":["T3"],"source":"user|webhook|loop|agent","created":"…","updated":"…","sha":["4f2a1c9"],"eval":"custom/spawn_task_id","duplicate_of":null,"ob1":"019f…"}
```

- id scheme: `T<n>` is scoped to the supervisor's run (`run_id` = the base session id); the
  globally unique key is `<run_id>/T<n>` — what trailers, `shas.md` and `graph.md` store.
- dedupe gate (the "repeat, redundant feature requests or bug reports" callout): a new task is
  checked (1) by `hash` against `tasks.md`, (2) by `memory.recall(kind:task, project)` cosine
  > 0.9 against open/accepted tasks, (3) by GIT-AGENT BLAME-ATTRIBUTE when it cites a file/line.
  A hit ⇒ the request is appended with `status:duplicate`, `duplicate_of:<task_id>` and auto-
  closed with the pointer — no agent is spawned. This runs inside ORCHESTRATOR DECOMPOSE, before
  any `spawn`.
- gitignore (the known issue): `tasks.md` stays TRACKED (it is the project's queue and the
  human gate); `events.md`, `logs.md`, `loops.json` and rotated `*.md.1` are ignored — `[init]`
  writes those lines into `.gitignore`, `projectmem` rotates at 5 MiB. Collaboration rule
  (stamped into `agents.md` `### Rules`): check `tasks.md` before filing; the dedupe gate is
  the machine half, the human glance is the other.

TODO:

#### TOOLS

<!-- NOTE: WALL TIME AND TOKEN USAGE BETWEEN JSON AND PYTHON IN RAW TOOL CALLS, WHEN TOOL -->
<!-- USAGE GOES OVER 26 RUNS. MAY NEED TO CONSIDER AN ADDITIONAL TOOLING FORMATTING FOR -->
<!-- PYTHON TO CALL PARAMS, AND TOOLS BE ACTUAL PYTHON FUNCTIONS.  -->

```python
tools_schema = [
{
    "type": "function",
    "function": {
        "name": "calculate_system_matrix",
        "description": "Computes structural parameters for a large matrix system.",
        "parameters": {
            "type": "object",
            "properties": {
                "matrix_size": {
                    "type": "integer",
                    "description": "The N x N size of the matrix.",
                },
            },
            "required": ["matrix_size"],
        },
    },
},
]
```

## Available Tools:

[orchestrator]

- `orchestrator`: Manage and coordinate complex workflows across
  multiple agents, tools, and systems. This tool enables high-level task planning,
  resource allocation, and execution tracking, ensuring efficient and reliable
  completion of multi-step, interdependent tasks in dynamic environments. It supports
  dynamic scaling, error recovery, and real-time monitoring of system state.

- `sub-agents`: Manage and coordinate specialized AI sub-agents for
  distributed problem-solving, enabling modular, scalable, and fault-tolerant workflows
  across complex tasks such as system orchestration, multi-step planning, and
  cross-domain reasoning. This tool allows dynamic instantiation and communication
  between sub-agents, facilitating advanced autonomous execution in large-scale
  environments.

  [sys-admin]

- `bash`: Execute shell commands directly for system-level operations, such as
  managing files, starting services, or checking system status. This tool enables
  direct interaction with the underlying OS, useful for deployment, debugging, and
  automation tasks.
- python : Execute Python scripts and code snippets for data processing, model
  training, or automation tasks. This tool supports complex logic, mathematical
  operations, and integration with external libraries, making it ideal for AI-driven
  workflows and system management.
  [researcher]
- websearch : Perform a web search to gather up-to-date information, research, or
  insights on a given topic. This tool is ideal for verifying facts, finding
  documentation, or exploring new technologies and best practices. Use it when external
  data is required to inform decisions or complete tasks.
  [git-agent]
- github: Interact with GitHub repositories to manage code, track issues, create
  pull requests, and handle repository settings. This tool enables seamless
  integration with version control, allowing for collaborative development, code
  reviews, and automated workflows directly from the system.
  [file-manager]
- file_manager: Manage files and directories, including creating, reading,
  updating, and deleting files. This tool supports advanced file operations such as
  copying, moving, and archiving, and is useful for organizing project structures,
  managing assets, and maintaining clean codebases.
  [memory]
- recall : Retrieve previously stored information from memory or context logs to
  inform current decisions or actions. This tool enables efficient access to
  historical data, ensuring continuity and consistency across sessions and tasks. Use
  it to reference past interactions, completed work, or system states when needed.

<!-- [+] added 2026-08-20 — the tool catalog (groups), the agent × tool matrix, code-mode, example schemas -->

##### CATALOG `[harness]`

Every tool is an `internal/tools.Tool` (`Name() / Description() / Call(ctx, jsonInput)`) with a
hand-written JSON schema (`Schema()`, `x-icarus-schema-version: 1`); the OpenAI-style
`tools_schema` above is exactly what `ToolSchemaJSON` emits, so this catalog needs no new
format. Groups are dotted names; an allowlist entry is a group (`fs.*`), a tool, or a
tool + path glob (`fs.append:memory/**`).

| group | tools | exists as | notes |
| --- | --- | --- | --- |
| `fs.read` | `read`, `ls`, `glob`, `find`, `grep` | yes (`fd`/`rg` engines, scan budgets) | read-only |
| `fs.write` | `write`, `edit`, `edit-diff` | yes | Go files gofmt'd on write |
| `fs.append` | `append` | NEW | the only write path into `memory/**`; path-globbed; backed by `projectmem` |
| `exec.shell` | `zsh`, `bash` | yes | `toolrules` pack applies (blocks file writes/search through the shell for SLMs) |
| `exec.go` | `go_run` | NEW (thin: `go run` / `go test -race` / `go vet`) | replaces `python` for scripting |
| `exec.python` | `python` | NEW — SYS-ADMIN (decided 2026-08-21) + `[ai]` skills | the Python ML/ops stack; Go stays the default for NEW artifacts |
| `exec.program` | `program` | NEW — code-mode, below | one model turn, many tool calls |
| `git.*` | `git_status, git_diff, git_log, git_blame, git_commit, git_branch, git_worktree, git_push` | partly (`internal/gitwork` behind `/gwp`) | `git_commit` enforces trailers |
| `web.*` | `websearch`, `webfetch` | yes | backends browser / searxng / brave / tavily |
| `browser.*` | `browser_tabs, browser_navigate, browser_read, browser_click, browser_type, browser_screenshot, browser_present` | yes (session-bound) | CDP on Vivaldi |
| `memory.*` | `memory_recall, memory_remember, memory_forget, memory_consolidate` | NEW (MEMORY) | over ob-mcp through the MCP bridge |
| `ob1.*` | `mcp__ob1__ob_query \| ob_read \| ob_write \| ob_assist \| ob_status` | yes (MCP bridge, `auth: tool_arg`) | raw; audits only |
| `agent.*` | `spawn` (+ `task_id`, `tools[]`), `agent_await`, `agent_message` | `spawn` admits only | Runner = GRAPH deliverable #1 |
| `task.*` | `task_create, task_update, task_complete` | NEW (`projectmem`) | writes `tasks.md` |
| `eval.run` | `eval_run` | NEW (`make evals` / CUSTOM dispatcher) | pass/fail + log |
| `event.emit` | `event_emit` | NEW | `events.md` + webhook out |
| `notify` | `notify` | yes (`POST /notify`, `icarus notify`) | dunst / web strip / A.O |
| `loop.*` | `loop_define, loop_status, loop_stop, loop_resume` | NEW | LOOPS |
| `coms.*` | `coms_send`, `coms_await` | yes (seat-bound) | a2a over coms-go |
| `lsp.*` | `lsp_definition, lsp_references, lsp_diagnostics` | NEW (gopls) | CODER / REVIEWER |
| `sys.*` | `service_status`, `host_card` | NEW (wraps `internal/metrics/host.go`) | SYS-ADMIN |
| custom | `[[tool]]` in `.icarus/tools.toml` | yes | argv → schema; the D0 skill seam |

##### MATRIX (✓ allowed · r read-only / narrowed · — denied)

| group | ORCH | CODER | REVIEWER | RESEARCHER | GIT | MEMORY | SYS-ADMIN | loops |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| fs.read | ✓ | ✓ | ✓ | ✓ | ✓ | r (`memory/**`) | ✓ | ✓ |
| fs.write | — | ✓ | — | r (`research/**`) | — | — | ✓ | — |
| fs.append | — | — | — | — | r (`shas.md`) | r (`memory/**`) | — | ✓ |
| exec.shell | — | ✓ | r (tests only) | — | — | — | ✓ | ✓ |
| exec.go | — | ✓ | ✓ | — | — | — | ✓ | ✓ |
| exec.python | — | — | — | — | — | — | ✓ | r (`[ai]` skills) |
| exec.program | — | ✓ | — | ✓ | — | ✓ | — | ✓ |
| git.* | r (log, blame) | r (status, diff) | r | — | ✓ | — | — | r (keep / revert) |
| web.* / browser.* | — | — | — | ✓ | — | — | — | — |
| memory.recall | ✓ | ✓ | ✓ | ✓ | — | ✓ | — | ✓ |
| memory.remember / forget | — | — | remember (lesson) | remember (fact) | — | ✓ | — | consolidate |
| ob1.* | — | — | — | — | — | ✓ | — | ✓ |
| agent.* | ✓ | — | — | — | — | — | — | — |
| task.* | ✓ | — | — | — | — | — | — | — |
| eval.run | ✓ | ✓ | ✓ | — | — | — | — | ✓ |
| event.emit / notify | ✓ | — | — | — | — | — | ✓ | ✓ |
| lsp.* | — | ✓ | ✓ | — | — | — | — | — |
| coms.* | ✓ | — | — | — | — | — | — | — |

Composability rules: (1) a child's allowlist ⊆ parent's allowlist ∩ the role row — `spawn`
denies otherwise with a new typed `Reason: tools_exceed_parent`; (2) groups are the unit of
grant, globs narrow them, nothing widens; (3) the matrix is DATA, not code — it lives in the
agent definitions and is mirrored into `agents.md` `### Roster` by `/init --check`.

##### AGENT DEFINITIONS — one format for Claude Code and icarus

<!-- [+] added 2026-08-21 — "the agents claude uses must be accessible to a project via a configuration setting; icarus needs all of claude's coding agents" -->

icarus has no agent-definition format today; Claude Code has one you already maintain
(`~/.claude/agents/*.md`: YAML frontmatter `name, description, tools, model, color, memory` +
body; project-level `<repo>/.claude/agents/*.md` is native to Claude Code too). Decision: that
format is CANONICAL for both harnesses — icarus loads it (a sibling of `internal/cccommands`,
which already loads `.claude/commands/*.md`) and adds its own keys under an `icarus:` block that
Claude Code ignores. One file, both harnesses, git-shareable with contributors.

```markdown
---
name: reviewer
description: structurally read-only review gate; never shares a session with the author
tools: Read, Grep, Glob, Bash          # Claude Code allowlist (what Claude sees)
model: qwen3.8-27b-failover            # Claude-side id (litellm); icarus maps it via agents.model_aliases
color: green
icarus:
  role: reviewer                       # binds [models.agents.reviewer] for the icarus model pin
  tools: [fs.read, "exec.shell:tests", exec.go, eval.run, memory.recall, "memory.remember:lesson", "lsp.*"]
  skills: [review, eval_environment, lesson_log]
  max_turns: 30
  depth: 1
---
## Performance contract        ← EVALS §CODING-AGENT EXPECTATIONS, verbatim
… body as today …
```

Sources and the configuration setting (`~/.icarus/settings.json`):

```json
"agents": {
  "sources": ["<project>/.claude/agents", "<project>/agents", "~/.claude/agents", "~/.claude/agents/.lazy"],
  "enabled": ["orchestrator", "coder", "reviewer", "researcher", "git-agent", "memory", "sys-admin", "icarus_coder", "golang-coder", "unix-coder", "fable-reviewer"],
  "model_aliases": { "qwen3.8-27b-failover": "sglang-monty/qwen3.8-27b", "claude-opus-4-8": "anthropic/claude-opus-4-8", "fable": "anthropic/claude-fable-5", "haiku": "anthropic/claude-haiku-4-5" }
}
```

- precedence: project sources win over user sources on a name clash (a project can specialise
  `coder`); `enabled` is the per-project allowlist — THE setting that makes "the agents claude
  uses" available to a project and to icarus; `/init` stamps `<repo>/.claude/agents/` with the
  roster from `agents.md` (role templates specialised per project — the `/agent-builder` shape),
  so a contributor who clones the repo gets the same agents in Claude Code AND icarus.
- Claude tool names map to icarus groups once, in code: `Read, Grep, Glob, LS → fs.read` ·
  `Write, Edit, MultiEdit, NotebookEdit → fs.write` · `Bash → exec.shell` · `WebFetch, WebSearch
  → web.*` · `LSP → lsp.*` · `Task / Agent → agent.spawn` · `mcp__*` passthrough. When
  `icarus.tools` is absent the mapped Claude list IS the allowlist, so every existing coder runs
  in icarus on day one with its current permissions.
- `tools` (Claude) and `icarus.tools` are both bounded by the role row of the MATRIX and by the
  parent's allowlist (composability rule 1); the TOML overlay `~/.config/icarus/agents/<role>.toml`
  is OPTIONAL, for icarus-only roles with no Claude counterpart (e.g. `loop_runner`).

##### CODE-MODE (`exec.program`) — the answer to the wall-time / token NOTE above

When a task needs more than ~10 tool round-trips (the NOTE's 26-run wall), the model writes ONE
program that calls tools as functions and returns a compact result; the harness executes it in
a sandbox with the session's allowlist enforced per call. This is "programmatic tool calling"
(Anthropic), "Code Mode" (Cloudflare), CodeAct (Wang et al., 2024). Shape:

- language: Go — `go run` in a temp module with a generated `tools` package whose functions
  marshal to the same `Tool.Call` path (every call still passes `toolrules`, `ValidateArgs`,
  budgets and `tool.pre_call / post_call` events); Python for SYS-ADMIN and `[ai]` skills.
- limits: `timeout_ms`, `max_tool_calls` (default 200), `max_result_bytes`; stdout is the tool
  result; loop-break transcripts apply.
- who: ORCHESTRATOR (no — it delegates), CODER, RESEARCHER, MEMORY, loops; never REVIEWER
  (read-only by structure) or GIT-AGENT.
- eval `tools/code_mode_saves`: a 30-file mechanical change completes in ≤ 2 model turns and
  ≤ 20 % of the token cost of the tool-per-turn baseline.

##### EXAMPLE SCHEMAS (same format as `tools_schema` above)

```python
tools_schema += [
  {"type": "function", "function": {
    "name": "memory_recall",
    "description": "Hybrid retrieval over long-term memory (vector + BM25 + entity boost + recency). Returns memories, never embeddings.",
    "parameters": {"type": "object", "properties": {
      "query": {"type": "string"},
      "top_k": {"type": "integer", "default": 10},
      "threshold": {"type": "number", "default": 0.35},
      "identity": {"type": "string", "enum": ["user", "agent", "project", "run"]},
      "kind": {"type": "string", "enum": ["fact", "preference", "decision", "lesson", "procedure", "rule", "task", "entity"]},
      "project": {"type": "string"}},
      "required": ["query"]}}},
  {"type": "function", "function": {
    "name": "memory_remember",
    "description": "Explicitly store a memory. Runs extraction + reconcile (ADD/UPDATE/SUPERSEDE/NOOP); writes ob1 and the project projection.",
    "parameters": {"type": "object", "properties": {
      "content": {"type": "string"},
      "kind": {"type": "string"},
      "identity": {"type": "string"},
      "entities": {"type": "array", "items": {"type": "string"}},
      "confidence": {"type": "number"},
      "expires_in_days": {"type": "integer"}},
      "required": ["content", "kind", "identity"]}}},
  {"type": "function", "function": {
    "name": "spawn",
    "description": "Spawn a sub-agent (depth ≤ 3, fan-out ≤ 8). Child tools ⊆ caller tools.",
    "parameters": {"type": "object", "properties": {
      "agent": {"type": "string"},
      "prompt": {"type": "string"},
      "task_id": {"type": "string", "pattern": "^[0-9]+-[0-9a-f]{8}/T[0-9]+$"},
      "tools": {"type": "array", "items": {"type": "string"}},
      "description": {"type": "string"},
      "provider": {"type": "string"}, "model": {"type": "string"}, "effort": {"type": "string"}},
      "required": ["agent", "prompt", "task_id"]}}},
  {"type": "function", "function": {
    "name": "eval_run",
    "description": "Run an eval suite through the build system; pass/fail only, with the log line.",
    "parameters": {"type": "object", "properties": {
      "suite": {"type": "string", "description": "general | custom/<name> | <suite_dir>"},
      "session_id": {"type": "string"}},
      "required": ["suite", "session_id"]}}},
]
```

TODO :

# Data Flows

all of the following tiers of context can be improved in icarus via pass by
reference:

1. session - engineering
2. harness - engineering
3. loop - engineering
4. graph - engineering

TODO:
[SESSION]

# Workflow

1. Session memory As context per session grows, define a threshold for context window
   against wall time and model output performance (accuracy) -- we will need a metric
   to help define what that is per model

# Dependencies

1. Model performance metrics - to evaluate the impact of context window size on
   accuracy and efficiency.
2. updated '/cmp --clear' command (along with jsonl artifact that is generated,
   absolutely still generate this file, as this is my dataset for evolving the
   system), the actual session seed file will now only reference:
   - the context window size
   - the session memory threshold
   - work completed - with git commit hash. the commit hashes will serve as pointers
     for completed work:
     1. each logical commit division in the seed file will have: [logical commit
        subject - git commit hash - timestamp]
   - table/associative array with session turn -> file
3. orchestrator updates: when spawning sub-agents, the orchestrator will need to be
   responsible for assigning each sub-agent a task specific id that is signed by that
   sub-agent during the code generation and commit process. Allowing tags to be added
   with the use of git-blame when completing coding work.

   ### Benefits:
   - bug report fixes
   - feature requests
   - de-duplication efforts requests

4. Updated seed file that will now only reference:
   - the context window size
   - the session memory threshold
   - '/cmp --clear' command event trigger

## Goal

the goal is to have the most robust seed files, and memory management system with the
smallest context window footprint, and highest fidelity of work completed against
what needs to be done.

## Considerations

use git blame to assist with code change identifications, which will also mean that
sub-agents will need to identify themselves not only by name, but by task

<!-- [+] added 2026-08-20 — [SESSION] resolved: what happens at each step, the threshold metric, dependencies -->

[SESSION] — resolved

# Workflow (what happens at each step)

1. turn starts → `[harness]` folds T1 (`agents.md`, `rules.md` head) + Ra advisories (T2, ≤ 3)
   into the prompt; the digest/compaction bands decide nothing yet.
2. model call → `usage/<id>.jsonl` row (`prompt_tokens` ≈ context occupancy, `ttft_ms`,
   `duration_ms`, `cached_tokens`) — exists, write-only; NEW: the same numbers are stamped on
   the `assistant_message` entry with `tool_calls[]` and `sha[]` (ICARUS [harness] #3).
3. turn ends → memory ingest (async, MEMORY §M3) → `logs.md` rows; GIT-AGENT commits carry
   `Task-Id`.
4. the threshold policy evaluates after every turn — three series, all derivable today:
   - `ctx_pressure  = prompt_tokens / window` (window = `providers.<id>.context_windows[model]`,
     175k for `qwen3.8-27b`);
   - `latency_drift = duration_ms / median(duration_ms of the first 5 turns at ctx_pressure < 0.3)`;
   - `quality_proxy = 1 − (eval_fail + user_corrections + loop_breaks) / turns` over a sliding
     window of 10 turns (the accuracy signal until evals give a direct one; user corrections =
     the GATE markers in §M3 step 1).
   A compaction op fires when `ctx_pressure ≥ compact_at` (default 0.60 — today's warn band)
   OR `latency_drift ≥ 2.0` OR `quality_proxy` drops ≥ 0.2 against the session's first window
   — whichever first; the hard band (`ctx_pressure ≥ 0.85` — today's danger band / digest hard
   165k) forces it.
5. op = `/cmp --clear`: distill (`claude-hooks compact llm --dry-run` → compact-index JSON,
   STILL archived — this is the dataset), build the SEED (pointers only: window, `compact_at`,
   `op_counter`, op table, `shas.md` lines since the last op as `[subject – sha – ts]`,
   turn → file table from `tool_calls[]`, top-K memory ids), `ClearHistory(seed)`.
6. the per-model threshold is CALIBRATED, not guessed: loop `autoresearch-ctx-threshold`
   replays archived sessions (`~/agentlogs`) per model, plots `quality_proxy` against
   `ctx_pressure`, and writes the knee to `memory/context.md` and to the model's
   `context_windows` sibling key `compact_at`.

# Dependencies (resolved)

1. model performance metrics — `usage/<id>.jsonl` (exists) + the three series above (NEW
   `internal/metrics/session_quality.go`, fed by bus events `provider.response`,
   `skill.eval_fail`, `message.user_start` (correction markers), loop-break transcripts).
2. `/cmp --clear` update — keep archiving the compact-index JSONL; the seed carries the listed
   fields; the `table/associative array turn → file` comes from `tool_calls[]` once the
   transcript records them.
3. orchestrator task ids — `spawn.task_id` + trailers (GRAPH); `git blame` → trailer → task →
   session JSONL + compact-index = the de-duplication and bug-fix pointer chain.
4. seed file — as listed; ≤ 4 KiB (today's carry clips 16 KiB of compact-index JSON); the
   `[claude]` seed (`composeSeed`: Summary → 5 decisions → 8 changes → 10 tasks → 5 context,
   8000 chars) gains the same op table so both harnesses read one format.

TODO:
[HARNESS]

Treat '/cmp --clear' as an operation. Include an op counter per session. if op>=1
then the harness now appends to the seed file that was generated a table entry. the
table

# Workflow

1.

# Dependencies

<!-- [+] added 2026-08-20 — [HARNESS] resolved: the op counter and the op table (finishes the sentence above) -->

[HARNESS] — resolved

`/cmp --clear` is an operation. `op_counter` lives on the session (the `session_start` entry
gains `op_counter`; it increments on every `icarus.context_clear` marker and survives resume).
When `op_counter ≥ 1` the harness appends one row to the seed's op table:

| op | ts | ctx_before | ctx_after | turns_folded | commits_since_last_op | memories_written | seed_path | index_path |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2026-08-20T23:41Z | 118,204 | 3,912 | 37 | 4f2a1c9 9be01d2 | 6 | `~/.icarus/seeds/<sid>/op1.md` | `~/agentlogs/icarus/<bucket>/local/<rel>/<sid>.compact-index.json` |

# Workflow

1. `POST /cmp {clear:true}` (web) or `/cmp --clear` (REPL — NEW; `/cmp` is web-only today) →
   `cmplog.Run` (distill + archive, as now).
2. `op_counter++`; build the row: `ctx_before` = last `prompt_tokens` from `usage/<id>.jsonl`,
   `commits_since_last_op` = `shas.md` lines newer than the previous op's `ts`,
   `memories_written` = ingest counter since the previous op.
3. write `~/.icarus/seeds/<sid>/op<n>.md` (the pointer seed, MEMORY §M6) with the op table
   appended; `ClearHistory(seed)` carries THIS file's content, not the raw 16 KiB index.
4. `logs.md` row `compaction{op, ctx_before, ctx_after}`; ob1 `observation` tagged
   `compaction, op:<n>, session:<sid>` (the context handler already writes one on
   `compaction.done`).
5. `ctx_after` is measured on the next model call and back-filled into the row (the seed file
   is rewritten once; the JSONL gets a second `icarus.context_clear`-adjacent marker
   `icarus.op_row{op, ctx_after}`).

# Dependencies

- ICARUS [harness] #3 (transcript enrichment) and #6; `shas.md` (GIT-AGENT); `projectmem`;
  the `[claude]` `composeSeed` parity change so `/tmp/claude_compact_seed.md` carries the same
  op table.

TODO:
[LOOP]

# Workflow

# Dependencies

<!-- [+] added 2026-08-20 — [LOOP] resolved (full contract in LOOPS) -->

[LOOP] — resolved

# Workflow

1. define: `icarus loop define loops/<name>.toml` → validate → install the timer on `host` →
   `memory/loops.json` entry.
2. fire: timer → `icarus-loop@<name>` → `loop_runner` → skill (D0/D1) → keep | revert when
   `git = true` → `loops.md` + `logs.md` + ob1 `observation` (`loop_done | loop_fail`).
3. observe: `icarus loop status [name]` reads `loops.json` (+ `systemctl --user list-timers` on
   the host); `icarus loop stop <name>` disables the timer and marks `stopped`.
4. fail × `retries` → `events.md` `loop_fail` → `notify` → A.O `push_mobile`;
   `consecutive_failures ≥ 3` ⇒ `suspended` until `icarus loop resume <name>`.

# Dependencies

- `loop_runner` binary on monty; ssh BatchMode from lewis (never `-t`); `projectmem`; skills
  with `skill.toml`; `~/Programs/autoresearch` (Go) for objective loops; the MTP instability on
  sglang is the known risk — loops set `max_calls`, tolerate provider errors as `crash` rows,
  and never hang (`max_runtime`).

TODO:
[GRAPH]

# Workflow

# Dependencies

<!-- [+] added 2026-08-20 — [GRAPH] resolved (full model in GRAPH) -->

[GRAPH] — resolved

# Workflow

1. ORCHESTRATOR reads `# Outcome` → task DAG (`Task{DependsOn}`) → TASKS dedupe gate →
   `tasks.md` rows (`pending`).
2. topological dispatch (`Supervisor.Spawn`, `max_workers` 8): `spawn{task_id, tools[]}` →
   `spawnguard` admission → child session stamped `agent_id / supervisor_id / task_id`.
3. worker → terse result → `task.done | blocked | error` → `tasks.md` + `graph.md` edge +
   `logs.md`.
4. REVIEWER (fresh session, read-only tools) → pass / fail → `accepted` | `rejected` (+ lesson).
5. GIT-AGENT → trailers + `shas.md`; ORCHESTRATOR reconciles and reports `Done. Output: <path>`.

# Dependencies

- the spawn Runner (`deferredSpawnRunner` → an in-process `Worker` bound to a model-backed
  loop); `task_id` / `tools[]` on the tool; `projectmem`; trailers in `gitwork`; the
  `agents/<role>.toml` definitions (TOOLS); computeCommander types (`Blueprint`, `GateResult`,
  mail protocol) only if the in-process model proves insufficient.

[Known Bottlenecks]

1. tool speed (specifically read and **find**)

- End state will be a lookup table (lut) of files, each with it's own table of k,v :=
  session(subject), git-commit. this decreases or space usage of context at the
  expense of increasing our time-complexity with multiple lookups due to sharding. if
  op_counter>={x} consider a cache depending on the prompt context **need to define
  'x', and based off of metric observation**

<!-- [+] added 2026-08-20 — more bottlenecks, the LUT + defining x; build order; open decisions; sources -->

2. retrieval payload — `ob_query` returns 384-d vectors inline (~1.5k tokens per hit) and ranks
   `file_change` bookkeeping first ⇒ projection + `status`/`kind` filters (MEMORY §M4) before
   anything else; only `/search` and `ob.query` exclude the noise today.
3. spawn — no child executes; everything multi-agent is behind the Runner.
4. transcript fidelity — no tokens / tool calls / SHAs in the JSONL ⇒ the seed cannot be built
   from it yet; `usage/<id>.jsonl` is written and never read.
5. the LUT (the end state named above): `~/.icarus/lut/<project>.json` keyed by file path →
   `[{subject, sha, task_id, session_id, turn}]`, built incrementally by `projectmem` on every
   `commit` row and every `tool_calls[]` entry (write/edit). Lookup is O(1) per file; the
   sharding cost is one extra read per distinct file a prompt names. **x** (the op counter at
   which a cache pays) = the op at which a session has touched more distinct files than the
   seed budget holds (4 KiB ≈ 60 rows): measure `distinct_files_per_op` over the first 50
   sessions after phase 1 lands and set `x` = p50; the cache = pin the top-N rows by
   `access_count` into the seed (the same signal memory uses for forgetting).
6. ob1 substrate health — dormant Python pipeline, 10,529 unembedded rows, the socket-less
   context-router, an empty-answering GraphRAG (MEMORY §M9) — repair is phase 0b, not a
   side quest.
7. MTP / sglang instability — every model-dependent loop tolerates `crash` rows.

# Big Picture

<!-- [+] added 2026-08-21 — three agent tiers, A.O ⇄ icarus, the path to your own mobile front end -->

## Three agent tiers, one memory

| tier | who | where the definitions live | memory identity | shared with |
| --- | --- | --- | --- | --- |
| **project agents** | agents specific to one repo (the two collaborative projects first) | `<repo>/.claude/agents/*.md` + the `agents.md` roster — git-tracked, stamped by `/init`, specialised per project | `project` | every contributor who clones the repo; Claude Code and icarus both read them |
| **icarus agents (personal)** | the agents that work for you under the icarus roof, over ALL your session logs and ob1 entries | `~/.claude/agents` + `.lazy` (today) = `agents.sources` for icarus; `~/.config/icarus/agents/*.toml` only for icarus-only roles | `user` + `agent` | you, across hosts (`fleet-sync-agents` already rsyncs them) |
| **fleet agent (A.O)** | the always-on supervisor you can reach from anywhere | `ao-supervisor.service` (+ its own roster) | `user` | you, from a phone |

The memory system is the spine across the tiers (the video's "same memory for multiple agents,
centred on the user"): `identity:user` memories are visible to all three; `identity:project`
only inside the repo; `identity:agent` to the agent family that wrote them. Nothing is
duplicated per tier — tiers differ in WHERE definitions live and WHO may read them.

## A.O ⇄ icarus

A.O is reachable anywhere; icarus needs a computer — so A.O becomes a CLIENT of icarus, not a
sibling: the `/hook/*` callback routes (EVENTS) are exactly A.O's ingress. `ao-supervisor` gains
`ICARUS_SERVE_URL` (WG) next to `CLAUDE_CODE_PATH` and can (1) open or continue an icarus
session (`POST /session`, `POST /session/{id}/message`, stream `/event`), (2) call a skill or a
tool out-of-band (`POST /hook/tool/<name>`), (3) dispatch an agent (`POST /hook/agent/<role>`
with `reply_to: edge`), and (4) receive `notify` / `gate_request` pushes to forward to you.
Today's Discord bridge stays as one transport; the bridge's job becomes "authenticated relay
between your device and icarus", whatever the transport.

## Your own front end (mobile chat with every native ability)

Target: a chat interface on your phone that reaches A.O, then icarus, with all skills and tools
available. The shortest path reuses what runs today; nothing in the build order below blocks
it, and the three interfaces it needs are named now so later phases do not paint it in:

1. **Transport:** the existing `n0kos` Cloudflare tunnel (or WG on the phone) → A.O bridge →
   `icarus serve`. No new network.
2. **Session API:** `icarus serve` already exposes sessions, messages, SSE events, notify,
   commands and skills — the SPA that IS your primary surface speaks it. Step one is that SPA
   served through the tunnel as a PWA behind A.O's auth: a chat interface with every slash
   command and skill, on day one, no app store. Step two is a thin native shell (push
   notifications, share-sheet, voice) over the same API.
3. **Identity + auth:** A.O owns the device session (a bearer per device, revocable); icarus sees
   `agent_id = ao-<device>` on every JSONL entry, so mobile turns are attributed like any other
   agent's and memory written from the phone carries `identity:user`.
4. **Push:** `notify` / `gate_request` → A.O → device (Discord today, APNs/FCM later) — the A.O
   skills `push_mobile` / `afk_gate` named under AI FLEET HOSTS.

Out of scope for the phases below (it is its own spec). In scope here: the callback routes, the
auth header on `/hook/*`, and `agent_id` stamping — the three things the app needs from the
harness.

# Build Order (the roadmap to an executable; each phase ships with its evals)

| phase | deliverable | owner | evals |
| --- | --- | --- | --- |
| 0a | `icarus init` (`/init`, `--check`, brownfield link), embedded templates, `projectmem`, T1 read seam, `.gitignore` lines | `[init]`, `[harness]` (icarus_coder) | `init_idempotent`, `memory_files_append_only` |
| 0b | ob1 repairs: embed backfill loop, `ob.assist` filters + `--project`, `ob.write` type refusal, pending-tasks filter, router socket flag, Compressor write, Sorter repointed off the cloud model | `[ob1]` (ob1-coder) | `memory/backlog` = 0, `no_embeddings_on_wire` |
| 0c | coding-agent expectations propagated into the five coder definitions + `.lazy` siblings (`sync-agents`); Claude-format agent loader + `agents.sources / enabled / model_aliases` | `[claude]` (claude-agent), `[harness]` | `rg 'Performance contract'` hits every coder; `GET /agent` lists the enabled roster |
| 1 | transcript enrichment + `usage` join; op counter + op table; pointer seed; REPL `/cmp --clear`; `session_quality` metrics | `[harness]` | `seed_is_pointers` (≤ 4 KiB, every sha resolves) |
| 2 | T-small endpoint (§M8.2, option A: Qwen3-8B-class on monty's RTX PRO 4500) + `memory.models` roles; memory ingest (gate → extract → reconcile → persist → audit) through the MCP bridge; T1 projection; `memory_remember` | `[harness]`, `[monty]` | §M7 `extract_precision`, `dedupe`, `projection`, `cost` |
| 3 | `memory.recall` ranker (in-process v1, server-side v2 in `ob.assist`); Ra on the same ranker; cross-session dedupe; `last_accessed_at` touch; `# Steering` verdict parse | `[harness]`, `[ob1]` | §M7 `recall_hit` |
| 4 | GIT-AGENT trailers + `shas.md` + `lcp/findings` link; `logs_shas_tasks_consistent` | `[harness]` | the LOGS eval, `shas_regenerable` |
| 5 | spawn Runner + `task_id` / `tools[]` + identity stamping; TOML overlay for icarus-only roles; `tasks.md` + dedupe gate; `graph.md`; `icarus sessions fail` | `[harness]` | `spawn_task_id`, reviewer-independence, EVALS dispatchers green/red for real |
| 6 | `loop_runner` + `icarus loop` + `memory-consolidate` + `memory-embed-backfill` + autoresearch wrapper + `autoresearch-ctx-threshold` | `[monty]` | `loop_no_duplicate_run`, §M5 counts, `compact_at` written |
| 7 | callback routes `/hook/*`, `events.md`, A.O `push_mobile` / `afk_gate` | `[harness]`, A.O | envelope conformance |
| 8 | `skill.toml` enforcement (tools / max_calls / one_call_per_turn / max_result_bytes) + `exec.program` code-mode | `[harness]` | `code_mode_saves` |

Phases 0a–1 need no model; 2–3 are the memory system proper and run on local models only
(T-small + the embedder — never the interactive 27B); nothing in 0–4 needs the spawn Runner, so
the memory system ships before multi-agent does.

# Open decisions (for the user — each is a **DECIDE:** line above)

Decided 2026-08-21 (recorded in place): `python` stays in SYS-ADMIN (Go remains the default for
NEW artifacts); FILE-MANAGER is folded into SYS-ADMIN as the PROJECT-STRUCTURE skill; agent
definitions use the Claude Code format for both harnesses.

1. T-small placement (MEMORY §M8.2): A = Qwen3-8B-class on monty's RTX PRO 4500 (recommended)
   vs B = cai. D (share the interactive 27B) is rejected.
2. A.O as the fleet's mobile-push owner vs a direct Discord webhook (AI FLEET HOSTS → AO);
   Big Picture assumes A.O stays the bridge and Discord becomes one transport of several.
3. ob1 schema: v1 tags-only mapping (no migration) then v2 first-class `memory` item_type with
   `hash`, `lemmas tsvector`, `expires_at`, `superseded_by`, `access_count`, `embed_model` —
   the spec assumes v1 → v2.
4. embedder: keep all-MiniLM-L6-v2 for v1; run the `recall_hit` A/B against bge-base /
   nomic-embed / Qwen3-Embedding-0.6B when the hardware lands (§M8.3) — the eval decides, not
   the leaderboard.
5. the `[ai]` name registry and the duplicate `consideration:` list — collapse into one?

# Sources

- "Agent Memory EXPLAINED – Complete Architecture", Hugging Face, 2026-08-17 (28:33), a
  dissection of Mem0: 03:28 persistent memory across agents · 04:26 the three stores · 08:06
  ingestion · 15:30 semantic retrieval · 19:19 BM25 + entity boosting · 25:38 open models.
- Mem0 paper: arXiv 2504.19413 — extraction phase + update phase (ADD / UPDATE / DELETE / NOOP).
- CodeAct (Wang et al., 2024); Anthropic "programmatic tool calling"; Cloudflare "Code Mode".
- Verified on 2026-08-20: icarus `feat/websearch-tool` @ 4fb175f (`pkg/session`,
  `internal/{cli,web,tools,skills,spawnguard,hooks,metrics,agent/contextdigest,gitwork}`,
  `pkg/orchestration`); `~/.claude/hooks/go` (`compact`, `capture`, `intent`, `context-inject`,
  `lcp`); `~/openbrain` (`mcp/`, `src/openbrain/pipeline`, `ob2/`, `hooks/go/internal/ra`) +
  live Postgres census; `~/Programs/autoresearch`; `~/Programs/ai/computeCommander`; coms-go;
  systemd user units on lewis (+ monty timer names); `~/.icarus/settings.json` (window 175k,
  digest 140k/165k, `ob1.inject_max_bytes` 2048); `ao-supervisor.service`.
