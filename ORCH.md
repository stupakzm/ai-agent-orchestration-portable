# ORCH.md — portable agent-orchestration layer for Claude Code

**One file. Drop it in any repo. It installs itself.**

This is a port of the `orch` system (SQLite + Python CLI + headless `claude -p`
subagents) onto Claude Code's own primitives: subagents, skills, hooks, and
plain files. Same architecture, no Python package, no database, no daemon.

## 0. Install

Deterministic — copy `ORCH.md` into the repo root and run this. It extracts the
28 `FILE:` blocks in §5 verbatim; no model is in the loop, so nothing can be
paraphrased. Safe to re-run: it is how you **update** an install, not just create
one.

What it will and will not overwrite:

| | on re-run |
|---|---|
| `.claude/` agents, skills, hooks | replaced from `ORCH.md` — these are generated, not yours |
| `CLAUDE.md` | the `<!-- ORCH … -->`…`<!-- /ORCH -->` block is **replaced in place**, by version; the rest of your file is untouched |
| `.claude/settings.json` | hook entries merged, never clobbered |
| `.orch/memory/`, `state.json`, `upstream.md`, queue/registry/traces | never touched — your data |
| `.orch/config/*.yaml`, `.orch/playbooks/*.yaml` | **not** overwritten (you are meant to tighten thresholds and edit playbooks) — but drift is **reported**, and the shipped version is written beside it as `*.new` to diff |

`python3 - ORCH.md --check` reports the same drift and writes nothing, exit 1 if
anything is stale. That is the CI form.

```bash
python3 - ORCH.md <<'INSTALL'
import json, pathlib, re, sys

src = pathlib.Path(sys.argv[1]).read_text()
check = "--check" in sys.argv[2:]
blocks = []
for chunk in re.findall(r"^````\n(.*?)^````\n", src, re.S | re.M):
    m = re.match(r"FILE: (\S+)[^\n]*\n```\w*\n(.*)\n```\n?\Z", chunk, re.S)
    if m:
        blocks.append((m.group(1), m.group(2)))
assert len(blocks) == 28, f"expected 28 FILE blocks, found {len(blocks)}"

# Your data: never compared, never touched once it exists. Everything ELSE under
# .orch/ is an ORCH-owned template — also never overwritten (thresholds and
# playbooks are meant to be edited by hand), but drift is REPORTED. A silent
# "keep" is how a shipped fix fails to reach an existing install and nobody
# finds out; that is a real defect this installer used to have.
DATA = {".orch/memory/INDEX.md", ".orch/state.json", ".orch/upstream.md"}
BLOCK = re.compile(r"<!-- ORCH v\d+[^\n]*-->.*?<!-- /ORCH -->", re.S)
V1 = re.compile(r"<!-- ORCH v1\b[^\n]*-->.*?Architecture: `ORCH\.md`\.", re.S)

def merge_json(path, new):
    """Deep-merge hook lists into an existing settings.json, no duplicates."""
    old = json.loads(path.read_text()) if path.exists() else {}
    for event, entries in new.get("hooks", {}).items():
        cur = old.setdefault("hooks", {}).setdefault(event, [])
        for e in entries:
            if e not in cur:
                cur.append(e)
    for k, v in new.items():
        if k != "hooks":
            old.setdefault(k, v)
    path.write_text(json.dumps(old, indent=2) + "\n")

def claude_md(p, body):
    """Replace the ORCH block in place, by version. v1 shipped without an end
    marker, so a legacy block is matched by its known final line instead — the
    one legacy shape that exists."""
    new_v = re.search(r"<!-- ORCH v(\d+)", body).group(1)
    cur = p.read_text() if p.exists() else ""
    m = BLOCK.search(cur) or V1.search(cur)
    if m is None:
        if "<!-- ORCH v" in cur:
            return "STALE", "CLAUDE.md holds an ORCH block of unknown extent — delete it by hand and re-run"
        if check:
            return "STALE", "CLAUDE.md has no ORCH block"
        p.write_text((cur.rstrip("\n") + "\n\n" if cur.strip() else "") + body + "\n")
        return "append", f"CLAUDE.md (v{new_v})"
    if m.group(0) == body:
        return "same", f"CLAUDE.md (v{new_v})"
    old_v = re.search(r"<!-- ORCH v(\d+)", m.group(0)).group(1)
    if check:
        return "STALE", f"CLAUDE.md is v{old_v}, ORCH.md ships v{new_v}"
    p.write_text(cur.replace(m.group(0), body))
    return "update", f"CLAUDE.md (v{old_v} -> v{new_v})"

stale = []
for path, body in blocks:
    p = pathlib.Path(path)
    if not check:
        p.parent.mkdir(parents=True, exist_ok=True)
    if p.name == "CLAUDE.md":
        verb, note = claude_md(p, body)
    elif p.name == "settings.json":
        if check:
            have = json.loads(p.read_text()).get("hooks", {}) if p.exists() else {}
            miss = [e for ev, es in json.loads(body)["hooks"].items()
                    for e in es if e not in have.get(ev, [])]
            verb, note = ("STALE", f"{path} missing {len(miss)} hook entry") if miss else ("same", path)
        else:
            merge_json(p, json.loads(body))
            verb, note = "merge", path
    elif not p.exists():
        verb, note = ("STALE", f"{path} missing") if check else ("write", path)
        if not check:
            p.write_text(body + "\n")
    elif p.read_text() == body + "\n":
        verb, note = "same", path
    elif not path.startswith(".orch/"):
        # generated: agents, skills, hooks. Regenerated from ORCH.md, never yours.
        if check:
            verb, note = "STALE", f"{path} differs from ORCH.md"
        else:
            p.write_text(body + "\n")
            verb, note = "update", path
    elif path in DATA:
        verb, note = "KEEP", f"{path} (your data)"
    else:
        verb, note = "STALE", f"{path} differs from ORCH.md"
        if not check:
            pathlib.Path(path + ".new").write_text(body + "\n")
            note += f" — shipped version at {path}.new"
    if verb == "STALE":
        stale.append(note)
    print(f"{verb:<6} {note}")

if not check:
    for h in ("orch-guard.py", "orch-scan.py", "orch-lint.py", "orch-task.py", "orch-stage.py",
              "orch-measure.py", "orch-report.py"):
        (pathlib.Path(".claude/hooks") / h).chmod(0o755)
    pathlib.Path(".work").mkdir(exist_ok=True)
    # git does not track empty dirs; .gitkeep keeps the layout intact on clone
    for d in ("memory/blocks", "memory/archive", "memory/receipts", "cards", "queue", "registry",
              "checkpoints", "checkpoints/resolved", "traces", "approvals"):
        q = pathlib.Path(".orch") / d
        q.mkdir(parents=True, exist_ok=True)
        (q / ".gitkeep").touch()
    gi = pathlib.Path(".gitignore")
    if ".work/" not in (gi.read_text() if gi.exists() else ""):
        gi.write_text((gi.read_text().rstrip("\n") + "\n" if gi.exists() else "") + ".work/\n")

# .orch/ is the memory, the registry and the traces. Ignored by git, it lives on
# one disk and no clone can replay or audit anything — a choice to make knowingly.
import subprocess
if subprocess.run(["git", "check-ignore", "-q", ".orch/registry/x.yaml"], capture_output=True).returncode == 0:
    print("\nNote: .orch/ is ignored by git here — memory, registry, traces and upstream.md live on this "
          "disk only, and a fresh clone can replay nothing. Track .orch/ unless that is the intent (§3).")

if stale:
    print("\nNeeds your attention — ORCH.md never overwrites a file you may have edited:")
    for x in stale:
        print(f"  ! {x}")
if check:
    sys.exit(1 if stale else 0)
print("\nOK. Restart Claude Code — hooks, agents and skills register at session start.")
INSTALL
```

Or, if you would rather not paste a script: `Read ORCH.md and install it` also
works — Claude Code writes the §5 blocks itself. Prefer the script: it is
byte-exact where the model is not, and reading this file costs **roughly 105k
tokens of context** where the script costs none.

**Then restart Claude Code.** Hooks, subagents, and skills are registered at
session start; until you restart, the guard is not enforcing anything.

Verify (see §7 for the full test procedure):

```bash
ls .claude/agents/orch-* .claude/skills/orch-*/SKILL.md && \
echo '{"tool_name":"Bash","tool_input":{"command":"git push --force"}}' \
  | python3 .claude/hooks/orch-guard.py; echo "exit=$? (want 2)"
```

To uninstall: `rm -rf .claude/agents/orch-* .claude/skills/orch-* .claude/hooks/orch-*.py .orch`, delete everything from `<!-- ORCH` through `<!-- /ORCH -->` in `CLAUDE.md`, and remove the ORCH hook entries from `.claude/settings.json` (three `PreToolUse` — two running `orch-guard.py`, one `orch-report.py`; one `SubagentStop`; two `SessionStart`).

---

## 1. Structure: why one file installs many

Single `CLAUDE.md` or a multi-file tree? It is both, and the split is forced by
the architecture being ported.

`CLAUDE.md` is read into context on **every turn of every session**. A
full-fidelity port of this system is ~1200 lines ≈ 16k tokens. Putting it all
in `CLAUDE.md` would spend the entire routine-turn context budget describing a
system whose own first invariant is *"a tight default with earned expansion —
10k for routine turns, more only when a decision class justifies it."* The
single-file-CLAUDE.md option breaks the rule it is written to enforce.

A multi-file tree fixes that — skills load on demand, agents load only inside
the subagent that uses them — but a tree is not something you drop into a repo.

So: **the portable unit is this file; the runtime is a tree it generates.**

| | always in context | loaded on demand | never in main context |
|---|---|---|---|
| what | §2 core (47 lines) + every skill's and agent's `description` line + the SessionStart memory headlines | `.claude/skills/orch-*` bodies | `.claude/agents/orch-*` bodies (subagent-only), block bodies, traces |
| cost | ~810 + ~760 = **~1.6k tokens**, plus headlines | ~0.8k–13k when invoked (`orch-task` is the large one) | 0 |

Two things that table gets asked about:

- **The `description:` lines are not free.** Every skill and agent description
  sits in the always-on listing whether or not it is ever invoked — ~760 tokens
  across twelve of them, about as much as the core itself. Keep them one sentence longer
  than feels necessary (they are what makes routing fire) and no longer.
- **It is a cached prefix read, not fresh input.** `CLAUDE.md` renders ahead of
  the conversation, so in a warm session those ~1.4k tokens are served from
  cache at roughly a tenth of input price. The cost that actually bites is
  **context occupancy and attention** — a long standing instruction competes
  with the task in front of it — and the fact that **editing the core
  invalidates the cached prefix behind it**, so update ORCH between sessions,
  not in the middle of one.

The core carries only what must fire on a turn where nothing has been invoked:
who you are, the routing table, that guards are code, and the skill list.
Everything consumed *after* routing has already chosen to orchestrate — the
procedure, the summary shape, attribution, checkpoints, memory rules — lives in
`orch-task` §0, which is loaded by then anyway. That split is worth preserving:
it took the core from 94 lines to 37 with nothing lost. v3 added three lines
for two things that must fire before any skill loads: the measurement route,
and the `! <cmd>` approval path for an operation the guard refused. v4 added
five more for the same reason: a question answered by computing over data is
measurement, and a number counts only when tracked code reproduces it. The
worst errors of the session behind v4 happened on question turns, where no
skill had loaded and nothing in the core said a quick script was a risk. v5
added two lines: text goes into files through the Write tool. The guard's
false positives in the session behind v5 came from the orchestrator's own
manual work — a checkpoint note through `printf`, a result file through a
heredoc — before any skill that says so had loaded. v6 changed no line of the
core: everything it adds fires in code (a hook, the guard) or after `orch-task`
has loaded, so an install keeps its block, and its cached prefix, as it was.

`ORCH.md` stays in the repo as the source of truth. The generated tree is
disposable — regenerate it from this file whenever it drifts.

### 1.1 What survives the port, and what does not

The original system's second invariant is: **anything that reduces oversight is
code, never a model decision.** A prompt-only port loses that outright — an
instruction not to force-push is a suggestion. This port keeps the invariant
where it matters by putting the risk triggers in a **PreToolUse hook** (§5.1),
which the harness executes and the model cannot talk its way past.

Honestly lossy, and you should know which parts:

| Original | Port | Loss |
|---|---|---|
| SQLite + FTS5 BM25 librarian | markdown blocks + `INDEX.md` + grep | real. No BM25 ranking, no eval gate. Fine to ~300 blocks/project; past that, compact or go back to SQLite. |
| Six-level budget hierarchy, measured $ | per-task step/turn budget, hooks count tool calls | real. In-session dollar spend is not observable. Budgets are advisory except the step cap. |
| Fitness composite + shadow replay + auto-promotion | traces written, scored by hand on request | intentional. Self-modification needs ≥100 traces and a paired statistical test; it does not belong in a portable drop-in. |
| Governor as Python, tightening-only thresholds | hook script + `sensitive.yaml` | preserved. Thresholds live in a file the model is instructed never to loosen, and the trigger list is in the hook. |
| Acceptance registry + rework attribution | `.orch/registry/*.yaml` + `/orch-rework` | preserved. This is the highest-ROI piece and needs only git + shell. |
| `git worktree` sandbox per task | same | preserved. |
| L4 swarm | preserved via parallel subagents in separate worktrees | preserved. |

### 1.2 Scope: this is repo-local, and stays that way

Every path the installer writes is relative to the repo root. It never writes
to `~/.claude/`, never touches your global settings, agents, skills, or
`CLAUDE.md`, and contains no absolute or `..` paths.

| | global (`~/.claude/`) | this repo |
|---|---|---|
| settings / hooks | untouched | `.claude/settings.json` |
| agents | untouched | `.claude/agents/orch-*.md` |
| skills | untouched | `.claude/skills/orch-*/` |
| instructions | untouched | `CLAUDE.md` (appended) |
| data | — | `.orch/`, `.work/` |

Claude Code merges project config over global at session start, so the ORCH
agents, skills, and hooks exist **only in sessions opened in this repo**. Other
projects are unaffected. Uninstalling is deleting files.

Three consequences worth knowing before you install:

- **The guard applies to the whole repo, not just orch tasks.** The PreToolUse
  matcher is `Bash|Write|Edit`, unconditionally — so `git push` is blocked in
  that repo even when *you* asked for it manually, not only inside a task. That
  is the intent (an irreversible op is irreversible regardless of who asked),
  but it is the thing most likely to surprise you on day one. If you want it
  narrowed to task execution only, gate the `COMMANDS` check on
  `os.environ.get("ORCH_TASK")` — and understand you are giving up the guard
  for everything else. A second `PreToolUse` entry, matcher `*`, runs
  `orch-guard.py --blind` on every tool call in the repo; it restricts only
  calls whose hook input says `agent_type: orch-judge` and lets everything else
  through, at the cost of one Python start per call. The guard also refuses a
  `git merge` of an `orch/tsk_…` branch until `verify` and `finish` have run —
  a record decides that, not a pattern — and your own `! git merge` never
  reaches it.
- **Every subagent's report passes `orch-report.py`** — a `PreToolUse` entry on
  `SubagentHandback` and a `SubagentStop` entry. It acts only for the ORCH
  agents (it refuses a report that breaks their contract, and records the one
  that passes) and lets every other agent through: one Python start per stop.
- **`.claude/settings.json` is the shared, committed file.** Push the repo and
  collaborators get these hooks. That is usually what you want for a team
  orchestration layer. If you want the hooks for yourself only, move the
  `hooks` block to `.claude/settings.local.json` (personal, gitignored).
- **Project hooks require your approval.** Claude Code will not silently run a
  hook a repo just added; you review it on first load. Expect that prompt.

---

## 2. The always-on core

Append this to the project's `CLAUDE.md`. It is deliberately short — everything
else is progressive disclosure.

````
FILE: CLAUDE.md (append)
```
<!-- ORCH v5 — orchestration layer. Source of truth: ORCH.md. -->
## Orchestration (ORCH)

This repo runs orchestrated work: tasks are contracts, subagents execute them in
sandboxed worktrees, and results are checked by code before they count.

**You are the orchestrator.** The user states what they want in plain English.
You classify it and route it — they should never have to name a skill, write a
packet, or ask for orchestration. Route every request:

| the request is | route | why |
|---|---|---|
| a question about the code, or a read-only look at it | answer inline | nothing to guard |
| a one-line, single-file, obvious edit (typo, rename, bump a constant) | do it inline, say you did it inline | a worktree costs more than the change |
| **anything else that changes code** — a feature, a bug, a refactor, multi-file work, anything needing a test | **invoke the `orch-task` skill and follow it** | this is what the system is for |
| an experiment, eval or benchmark whose number will decide something — **or a question you can only answer by computing over data** (a count, a rate, a breakdown) | run it yourself, after reading `orch-task` §8 | its scorer is code and its result is evidence; unregistered, the conclusion is an anecdote |
| you cannot tell which | ask, in one line | never improvise a chain silently |

**A number is evidence only if a tracked command reproduces it.** One from an
inline script is a draft: say so beside it, and keep it out of docs and
decisions until committed code prints it. The worst errors on record came from
the orchestrator's quick analysis, not from any agent.

`orch-task` carries the whole procedure: retrieval, drafting and linting the
packet, the single confirmation, dispatch, code-verified acceptance, the
registry, and the required summary shape. **Load it — never hand-roll the
procedure from memory, and never execute a packet inline in this session.** A
task runs in a worktree under a subagent, or it does not run.

**Guards are code.** `.claude/hooks/orch-guard.py` blocks irreversible
operations, secret paths, edits to `.orch/config/`, and writes outside a task's
declared scope — at the tool boundary, during your own manual work too. It reads
shell text as commands, so put text in files — notes, results, prompts,
scripts — with the Write tool, never `printf`, `echo` or a heredoc. If it
blocks you, do not route around it: surface it to the user. When they asked for
that exact operation — a push, or a merge a permission prompt refused — hand it
back as one `! <cmd>` line: their typing it is the approval, and a "yes" in chat
is not. If the block was *wrong*, that is a defect in ORCH itself and it belongs
in `.orch/upstream.md` (`ORCH.md` §5.5) — noticing an issue and not writing it
down is the failure that costs the most, because the next repo starts from the
same `ORCH.md`.

**Skills** — invoke these yourself as the routing above requires; the user
should rarely need to name one. `orch-task` run a task · `orch-memory`
read/write/compact memory blocks · `orch-loop` pick and drain the queue ·
`orch-rework` replay the registry, attribute breaks, and sweep the traces for
your own recurring packet defects. Architecture: `ORCH.md`.
<!-- /ORCH -->
```
````

---

## 3. Installed layout

```
.claude/
  agents/                     # subagent definitions — never in main context
    orch-executor.md          # writes code, sonnet, effort medium
    orch-executor-deep.md     # the same contract at effort high — chosen by the packet
    orch-debugger.md          # reproduces + isolates, sonnet, effort high
    orch-reviewer.md          # verifies against acceptance, sonnet, effort high
    orch-librarian.md         # selects memory blocks, haiku
    orch-archivist.md         # the only writer of memory, haiku
    orch-auditor.md           # audits the ORCHESTRATOR's own packets, sonnet, effort high
    orch-judge.md             # blind judge for measurement work: reads .work/_blind/ only
  skills/
    orch-task/SKILL.md        # run one task end-to-end
    orch-memory/SKILL.md      # block CRUD, retrieval, compaction
    orch-loop/SKILL.md        # selection, loop modes, checkpoints
    orch-rework/SKILL.md      # registry replay + break attribution
  hooks/orch-guard.py         # the risk triggers, as code
  hooks/orch-scan.py          # sensitive-content post-check; reads policy at runtime
  hooks/orch-lint.py          # packet, memory + results-doc lints: code checks on what the orchestrator writes
  hooks/orch-task.py          # prompt, verify, register, trace, finish: the task bookkeeping, generated
  hooks/orch-stage.py         # long jobs as resumable stages, one background job each
  hooks/orch-measure.py       # paired / matched-abstention / identity comparisons over per-item rows
  hooks/orch-report.py        # agent reports: held to their contract as they leave, recorded by code
  settings.json               # hook wiring (merge, don't clobber)
.orch/
  config/settings.yaml        # thresholds — tightening only
  config/sensitive.yaml       # security-gate policy — never self-modified
  playbooks/code.bugfix.yaml  # the shipped playbook
  upstream.md                 # findings about ORCH itself — carried back by hand
  memory/INDEX.md             # L0 headline index
  memory/blocks/blk_*.md      # L1 summary + L2 body
  memory/archive/blk_*.md     # superseded and decayed blocks — out of the index, never deleted
  memory/receipts/<task>.log  # archivist write receipts, one line per block
  cards/<project>.md          # project cards
  queue/tsk_*.yaml            # task packets
  registry/<project>.yaml     # acceptance registry
  checkpoints/ckp_*.md        # open decisions
  checkpoints/resolved/       # decided ones, the user's words under ## Resolution — what --waive reads
  approvals/apr_*.md          # a plan's approval, the user's words, once — packets carry its id
  traces/trc_*.json           # scored history
  state.json                  # loop state, counters
.work/                        # gitignored: worktrees, and the tools' scratch state
  <task_id>/                  # one worktree per task
  _prompts/ _verify/          # rendered prompts, verify records (orch-task.py)
  _stages/                    # stage status, exit codes, logs (orch-stage.py)
  _results/<id>/<agent>.json  # agent reports as the hook recorded them — the guard refuses other writes
  _blind/<msr_id>/            # what a blind judge may read, and nothing else
```

Track `.orch/` in git. It *is* the persistent memory; that is the whole point.
Add `.work/` (worktrees) to `.gitignore`.

---

## 4. The model in one page

Read this before changing anything in §5.

**Four planes.** *Control* (select a task, propose a mode, judge a result — the
main session) → *work* (stateless sandboxed subagents) → *memory* (blocks,
cards, traces) → *improvement* (traces scored, changes proposed by hand).
The control plane never reads raw artifacts, only digests.

**A task packet is a contract.** Goal, `acceptance` (executable beats judged),
`scope.paths`, `context_refs` (block *ids*, not content), `tools`, `budget`.
The subagent gets exactly this and nothing else.

**A result packet is evidence, not a claim.** `status` ∈ done · blocked ·
needs_decision · failed · escalate. A hook records it as the agent sends it,
so what `verify` reads is the agent's, and one the orchestrator supplies is
labelled as such. Acceptance checks are re-run by the
orchestrator; a `done` that fails its check becomes `failed`. A `blocked` says
what it is blocked on — `platform` (the harness refused tool calls for a reason
that is not ORCH's), `usage` (a usage limit) or `dependency` — because each
resumes differently, and none is the agent's failure or the packet's.
`context_usage` is **measured** — which block ids actually appear in the
evidence — never self-reported, because it feeds retrieval tuning and a
self-report is gameable.

**The packet is the likelier defect, and nothing else looks at it.** Guards
constrain the subagent's writes, acceptance verifies its output, checkpoints
fire on its behaviour and spend — every mechanism assumes *the agent* may be
wrong. A wrong packet fails silently instead: each check passes, and the defect
is registered as evidence that things are fine. Verification cannot catch a
defect encoded in the definition of success. Four mechanisms point the other
way. One is code and runs before any cost: `orch-lint.py packet` (§5.1) checks
the rules a machine can check and measures every count at the base commit. The
other three come after — mandatory `attribution` in every task summary, a
nullable-but-present `orchestrator_error` on every trace, and the orchestrator
sweep in `/orch-rework` §5, which is a separate agent because self-assessment is
the weak link this is meant to remove.

**The orchestrator's own numbers are the other blind spot.** Everything above
checks work that went through a packet. Analysis the orchestrator runs itself —
a quick script over results, to answer a question — went through nothing, and in
the field it produced both of one session's wrong conclusions: an inline
re-implementation of the scorer skipped one fallback, and 101 suspect rows became
195. So a number counts only when a tracked command reproduces it: measurements
register as `kind: eval`, results docs cite the registry and `orch-lint.py doc`
checks each citation, and a number from an inline script is labelled a draft
(`orch-task` §8). Bookkeeping follows the same rule — the registry entry and the
trace are generated by `orch-task.py` from a verification run, not typed.

**Memory has depth.** L0 headline (index) → L1 summary → L2 body → L3 the
original artifact (subagent only, never the orchestrator). Auto-attach rule:
for every selected block, also pull the head of its `supersedes` chain — the
single highest-value rule in the retrieval path, because it prevents acting on
a fact that has since been replaced. Superseding writes a new block and moves the
old one to `archive/`, out of the index, so a replaced fact cannot be selected by
its headline at all; auto-attach catches what still reaches it by id.

**Block types by ROI:** `procedure` (highest — makes the loop improve rather
than repeat) > `failure` (scar tissue; forced context on every plan phase) >
`convention` > `decision` > `api-shape` (decays fast) > `fact`.
`process-failure` sits beside `failure` and is the only type about the
orchestration rather than the project — how a packet went wrong, forced context
whenever a task plans something long, external, or costly.

**Loop modes.** L0 advisory (plan only) · L1 gated (approve each phase) · **L2
task-auto (default)** · L3 continuous (drain the queue) · L4 swarm (2–3 parallel
branches, keep one) · L5 watchdog (report, never act).

**Risk triggers force a checkpoint regardless of mode** (§2). Immutable list.

**Selection is additive**, never multiplicative:
`0.4·urgency + 0.4·impact + 0.2·staleness`, where `staleness = min(1, days_queued/30)`.
A product lets any single zero starve a task forever.

**Fitness** (weights; nulls renormalize over present metrics only):
`first_pass_acceptance` .25 · `discretionary_checkpoint_value` .10 ·
`cost_vs_baseline` .20 · `retrieval_precision` .10 · `retrieval_recall` .10 ·
`escalation_accuracy` .05 · `rework` .20. **Mandatory checkpoints are excluded
entirely** — scoring them would give the system a measurable incentive to reduce
its own oversight, and it would look like an improvement on the dashboard.
**`orchestrator_error` is excluded for exactly the same reason**: score honest
self-attribution and the cheapest way to raise the number is to attribute less.

**Rework is retroactive and never final.** A break found eight months later is
still that task's break. `penalty = min(1, 0.35 × links)`, cap 3 links/task.

---

## 5. Files

Write each block below verbatim to the path in its `FILE:` header.

Fence convention, so extraction is unambiguous: every file is wrapped in a
**four**-backtick container holding a `FILE:` line and one three-backtick block.
The file's content is exactly what is inside the three-backtick block — inner
fences belong to the file, not to this document.

### 5.1 The guard — the only real enforcement

v5 (slm-memory-management, 2026-10-05) makes the guard read what bash runs. In
one session four blocks hit text, not commands:
- a code span in a single-quoted `printf` checkpoint note;
- a result file written through a quoted heredoc;
- an executor's scratch script;
- the orchestrator's `rm -rf` of its own scratchpad.

None of these is read as a command any more: a single-quoted span, a heredoc
body only written to a file, or `echo`/`printf` text redirected to a file. A
recursive delete passes when it stays inside the temp dir and outside the repo.
The same review found holes in the other direction. shlex read a newline as a
space, so `ls⏎bash -c "git push --force"` passed. So did a wrapper after a
comment, a line continuation inside a command name, `find -exec sh -c`, and
`rm -fr`. All of these block now. Tier-1 step 24 tests both directions.

v6 (slm-memory-management, 2026-10-06/07) adds two refusals that a record
decides, not a pattern. A `git merge` of a task branch waits until `verify`
said `done` at the branch's tip and `finish` ran, and it runs as a command of
its own. That session merged a `failed` verdict through `verify … | tail -2 &&
finish … | grep merge: ; git merge …`: the pipe hid verify's exit code, and the
`;` ran the merge whatever finish said. And only the report hook writes
`.work/_results/`, where agent reports are recorded — the orchestrator had
twice written a result file that `verify` could not tell from an agent's. The
blind check now lets a judge's `SubagentHandback` through; refusing it cost
six judges' reports. Tier-1 steps 30–34.

````
FILE: .claude/hooks/orch-guard.py
```python
#!/usr/bin/env python3
"""ORCH guard — the risk-trigger list as code (PreToolUse).

The system's invariant: anything that reduces oversight is code, never a model
decision. "A model deciding whether to ask permission is a model that will
eventually decide not to." Everything below is deterministic.

Wired for Bash, Write, Edit. Exit 2 blocks the call and returns stderr to the
model as the reason. Exit 0 allows. Never exits nonzero for its own bugs — a
broken guard must fail open loudly, not wedge the session.

The COMMANDS and CONTENT lists are IMMUTABLE. Thresholds live in
.orch/config/settings.yaml and may be tightened, never loosened.

Two more refusals are decided by records, not by a pattern: a merge of a task
branch that verify and finish have not passed (merge_gate), and a write into
.work/_results/, where orch-report.py records what agents reported.

Note for orchestrators: this guard matches its own policy. An inline grep for
the sensitive.yaml content_patterns carries the force-push pattern in argv and
is blocked, read-only or not. Use .claude/hooks/orch-scan.py, which loads the
patterns from the YAML at runtime. Do not add a name-based exemption here.
"""
import json
import os
import re
import shlex
import subprocess
import sys
import tempfile
from pathlib import Path


def project_root():
    """The MAIN checkout, where .orch/ lives — never a task worktree.

    CLAUDE_PROJECT_DIR is not trustworthy: it follows the orchestrator's cwd
    into .work/<id>, where .orch/ does not exist (it is often gitignored, and a
    queued packet is uncommitted either way). Reading policy from there means
    reading no policy. So the env var is followed through a linked worktree's
    .git file back to the main checkout, and used only if .orch/ is there;
    otherwise this script's own location (.claude/hooks/ -> parents[2]), with
    the same resolution. Env first keeps a test fixture's root working."""
    def main_of(p):
        g = p / ".git"
        if g.is_file():                    # linked worktree: "gitdir: <main>/.git/worktrees/<id>"
            m = re.match(r"gitdir:\s*(.+)", g.read_text().strip())
            if m:
                gd = Path(m.group(1))
                gd = (gd if gd.is_absolute() else p / gd).resolve()
                if gd.parent.name == "worktrees":
                    return gd.parent.parent.parent
        return p
    here = main_of(Path(__file__).resolve().parents[2])
    env = os.environ.get("CLAUDE_PROJECT_DIR")
    if env and (main_of(Path(env).resolve()) / ".orch").is_dir():
        return main_of(Path(env).resolve())
    return here


ROOT = project_root()

# ── immutable: irreversible / externally-visible operations.
# COMMANDS are matched at command position only — see commands() for why.
COMMANDS = [
    (r"\bgit\s+push\b.*(--force|-f)\b", "force-push"),
    (r"\bgit\s+push\b", "push to a remote"),
    (r"\bgit\s+reset\s+--hard\b", "hard reset"),
    (r"\bgit\s+clean\s+-[a-z]*f", "git clean -f"),
    # -rf, -fr, -r -f, --recursive --force: the flags in any order or grouping
    (r"\brm\s+(?=(?:-\S+\s+)*(?:-[a-z]*r|--recursive\b))(?=(?:-\S+\s+)*(?:-[a-z]*f|--force\b))",
     "recursive force delete"),
    (r"\b(npm|pnpm|yarn)\s+publish\b", "package publish"),
    (r"\b(twine|poetry)\s+(upload|publish)\b", "package publish"),
    (r"\bdocker\s+push\b", "image push"),
    (r"\b(kubectl|helm)\s+(apply|delete|upgrade|rollout)\b", "cluster mutation"),
    (r"\bterraform\s+(apply|destroy)\b", "infrastructure change"),
    (r"\bgh\s+(pr|release|issue)\s+(create|merge|edit|close)\b", "GitHub write"),
    (r"\bchmod\s+777\b", "world-writable chmod"),
]

# CONTENT is matched against the command text, quoted arguments included,
# because these are not a single argv and cannot be: SQL reaches a client as a
# quoted payload, and pipe-to-shell IS the pipeline — split it into commands and
# the pipe that makes it dangerous is gone. Only text that never runs is left
# out first: comments, and what is merely written to a file (content_view()).
# The cost is that `grep "DROP TABLE" .` checkpoints. Erring toward a stop on
# destructive SQL is the right side to be wrong on; erring toward one on every
# commit message that says "git push" is not.
CONTENT = [
    (r"\b(DROP|TRUNCATE)\s+TABLE\b", "destructive SQL"),
    (r"\bDELETE\s+FROM\b(?!.*\bWHERE\b)", "unbounded DELETE"),
    (r"\bcurl\b[^|]*\|\s*(ba)?sh\b", "pipe-to-shell"),
]

PUNCT = ";&|()<>\n"
SEP_CHARS = set(";&|()\n")
# things that really do execute their quoted argument
WRAPPERS = {"bash", "sh", "zsh", "dash", "ksh", "env", "eval", "exec", "sudo",
            "doas", "nohup", "timeout", "xargs", "ssh", "nice", "setsid", "command"}
SHELLS = {"bash", "sh", "zsh", "dash", "ksh"}
# a heredoc body read by one of these, and piped nowhere, is only written somewhere
SINKS = {"cat", "tee"}
# ...and so are these commands' arguments, once their output goes to a file
WRITERS = {"echo", "printf"}
HEREDOC = re.compile(r"<<(-?)[ \t]*(?:'([^'\n]+)'|\"([^\"\n]+)\"|\\(\w+)|([A-Za-z_]\w*))")


def tokens(cmd):
    """shlex, with a newline as a separator, the way bash reads one. Plain shlex
    takes a newline for a space, so `ls⏎bash -c "git push --force"` was one
    command named `ls` whose quoted argument got masked, and the guard allowed
    it. Comments are gone by now (normalize); shlex's own comment handling would
    swallow the newline after one, and a `#` inside a word is not a comment."""
    lex = shlex.shlex(cmd, posix=True, punctuation_chars=PUNCT)
    lex.whitespace, lex.commenters, lex.whitespace_split = " \t\r", "", True
    return list(lex)


def segments(toks):
    """(argv tokens, the separator that ends them) for each simple command."""
    seg = []
    for t in toks + [";"]:
        if t in ("{", "}") or (t and set(t) <= SEP_CHARS):
            if seg:
                yield seg, t
            seg = []
        else:
            seg.append(t)


def runs_shell(header):
    """Does this heredoc's header line hand its body to a shell? Conservative:
    unparseable, or any wrapper word anywhere, counts as yes."""
    try:
        lex = shlex.shlex(header, posix=True, punctuation_chars=True)
        lex.whitespace_split = True
        return any(t.rsplit("/", 1)[-1] in WRAPPERS for t in lex)
    except ValueError:
        return True


def data_sinks(header):
    """For each heredoc on this header line, in order: is its body only ever
    written somewhere — read by cat or tee, piped nowhere, no process
    substitution? `cat > r.json <<'EOF' && git commit …` is; `cat <<'EOF' | psql`
    and `sqlite3 db <<'SQL'` are not. Unparseable: none is."""
    try:
        toks = tokens(header)
    except ValueError:
        return []
    out = []
    for seg, sep in segments(toks):
        sink = (seg[0].rsplit("/", 1)[-1] in SINKS and sep != "|"
                and not any("(" in t for t in seg))
        out += [sink] * seg.count("<<")
    return out


def substitutions(cmd, quotes=True):
    """Bodies of the $(...) and `...` that bash runs. A single-quoted span is
    literal: `printf 'ran `git push`\\n' >> notes.md` writes a code span and runs
    nothing, and matching inside it blocked an orchestrator's own checkpoint
    note (slm up_0011). Inside double quotes they still run. A heredoc body has
    no quoting at all (quotes=False). An unterminated one runs to the end of the
    text, so a malformed substitution is checked rather than skipped."""
    out, i, n, dq = [], 0, len(cmd), False
    while i < n:
        c = cmd[i]
        if c == "\\":
            i += 2
            continue
        if quotes and c == "'" and not dq:
            j = cmd.find("'", i + 1)
            i = n if j < 0 else j + 1
            continue
        if quotes and c == '"':
            dq = not dq
        elif cmd.startswith("$(", i):
            d, j = 1, i + 2
            while j < n and d:
                d += {"(": 1, ")": -1}.get(cmd[j], 0)
                j += 1
            out.append(cmd[i + 2:j - 1] if d == 0 else cmd[i + 2:])
            i = j
            continue
        elif c == "`":
            j = cmd.find("`", i + 1)
            out.append(cmd[i + 1:] if j < 0 else cmd[i + 1:j])
            i = n if j < 0 else j + 1
            continue
        i += 1
    return out


def normalize(cmd, content=False):
    """The shell text with what bash never runs taken out: comments,
    backslash-newline joins, and heredoc bodies that are data.

    A heredoc body is data when its header does not hand it to a shell, or when
    the command reading it is cat or tee writing it somewhere (data_sinks). A
    quoted body (<<'X', <<"X", <<\\X) that is data is dropped: bash expands
    nothing in it, so markdown showing `rm -rf a/b` in a code span is text. An
    unquoted one is reduced to its $(...) and `...`, which bash does run there.

    content=True is the view the CONTENT patterns (SQL, pipe-to-shell) see: a
    body counts as data only when it goes to cat or tee. SQL fed to sqlite3, or
    held in a python script, can run — so those bodies stay, and the block
    message says to write such a file with the Write tool.

    Operators are found by a quote- and comment-aware scan, never a regex over
    the raw string — otherwise `echo "<<'Y'"` would hide the next line."""
    out, pending, q, i, n, head = [], [], None, 0, len(cmd), 0
    while i < n:
        c = cmd[i]
        if q:
            if c == "\\" and q == '"' and i + 1 < n:
                if cmd[i + 1] != "\n":            # \⏎ inside "..." joins the lines
                    out.append(cmd[i:i + 2])
                i += 2
                continue
            out.append(c)
            if c == q:
                q = None
            i += 1
            continue
        if c == "\\" and i + 1 < n:
            if cmd[i + 1] != "\n":                # \⏎: bash joins the lines, `g\⏎it push` is git
                out.append(cmd[i:i + 2])
            i += 2
            continue
        if c in "'\"":
            q = c
        elif c == "#" and (i == 0 or cmd[i - 1] in " \t\n;&|()"):
            j = cmd.find("\n", i)                 # a comment runs nothing; its newline stays
            i = n if j < 0 else j
            continue
        elif cmd.startswith("<<<", i):
            out.append("<<<")
            i += 3
            continue
        elif cmd.startswith("<<", i):
            m = HEREDOC.match(cmd, i)
            if m:
                pending.append(m)
                out.append(m.group(0))
                i = m.end()
                continue
        elif c == "\n" and pending:
            header = cmd[head:i]
            executes, sinks = runs_shell(header), data_sinks(header)
            out.append("\n")
            i += 1
            for k, m in enumerate(pending):
                delim = next(g for g in m.groups()[1:] if g is not None)
                start, term = i, ""
                while i < n:
                    j = cmd.find("\n", i)
                    j = n if j < 0 else j
                    line = cmd[i:j]
                    if (line.lstrip("\t") if m.group(1) else line) == delim:
                        body, term, i = cmd[start:i], line, j
                        break
                    i = j + 1
                else:
                    body = cmd[start:]
                sink = k < len(sinks) and sinks[k]
                if not (sink if content else sink or not executes):
                    out.append(body)
                elif m.group(5) is not None:      # unquoted: its substitutions still run
                    out.append("".join(f"$({s})\n" for s in substitutions(body, quotes=False)))
                out.append(term)
            pending, head = [], i
            continue
        elif c == "\n":
            head = i + 1
        out.append(c)
        i += 1
    return "".join(out)


def commands(cmd, depth=0):
    """Yield each command in `cmd` as argv text, with quoted multi-word
    arguments masked out.

    Why this exists: the COMMANDS patterns describe *commands*, but they used to
    be matched against the raw string — so anything that merely mentioned one
    was blocked. `git commit -m "document the git push workflow"`,
    `echo "run git push when ready" >> README.md`, `grep -rn "rm -rf" scripts/`
    all checkpointed. That is the same defect as an inline grep tripping on the
    policy it greps for (§5.1), and a guard that fires on ordinary work is a
    guard people learn to route around — which costs more oversight than it buys.

    A quoted multi-word argument cannot itself be a command, so it is masked.
    Two things genuinely do execute their argument and are recursed into rather
    than masked: a shell wrapper (`bash -c "..."`, also further along a command
    line, as in `find -exec sh -c '...'`) and a command substitution (`$(...)`,
    backticks). What bash never runs — comments, heredoc bodies that are data —
    is removed first: see normalize().
    """
    if depth > 3:
        yield cmd                 # nested deeper than this reads: matched raw, never skipped
        return
    cmd = normalize(cmd)
    for body in substitutions(cmd):
        yield from commands(body, depth + 1)
    try:
        toks = tokens(cmd)
    except ValueError:
        yield cmd                 # unparseable (unbalanced quote): match raw, never open
        return
    for seg, _ in segments(toks):
        names = [t.rsplit("/", 1)[-1] for t in seg]
        wrapper = names[0] in WRAPPERS or any(
            a in SHELLS and re.fullmatch(r"-[A-Za-z]*c[A-Za-z]*", b) for a, b in zip(names, names[1:]))
        out = []
        for tok in seg:
            if any(c in tok for c in " \t\n"):
                if wrapper:
                    yield from commands(tok, depth + 1)
                out.append("\x00")     # an argument, not a command
            else:
                out.append(tok)
        yield " ".join(out)


def content_view(cmd, depth=0):
    """The text the CONTENT patterns are matched against: the command line
    minus what never runs. Those patterns stay on text rather than argv because
    SQL reaches a client as a quoted payload — but text written to a file is
    not a payload. A quoted heredoc to cat, or `printf '…DROP TABLE…' > f`, put
    an orchestrator's result file behind a mandatory stop, and the commit in
    the same command was lost with it (slm up_0011). So a body only written
    somewhere, and the arguments of echo/printf redirected to a file and piped
    nowhere, are left out; every $(...) inside them is still read."""
    if depth > 3:
        return cmd
    text = normalize(cmd, content=True)
    parts = [content_view(s, depth + 1) for s in substitutions(text)]
    try:
        toks = tokens(text)
    except ValueError:
        return "\n".join(parts + [text])        # unparseable: all of it, never less
    for seg, sep in segments(toks):
        to_file = any(set(t) <= set("<>&|") and ">" in t for t in seg)
        if (seg[0].rsplit("/", 1)[-1] in WRITERS and to_file and sep != "|"
                and not any("(" in t for t in seg)):
            continue
        parts.append(" ".join(seg) + (f" {sep}" if sep.strip() else ""))
    return "\n".join(parts)


def temp_only(seg):
    """Is this `rm -rf` confined to the system temp dir? Every operand a literal
    absolute path that resolves inside it — and not inside, or above, this
    repo. A session's own scratchpad is not the irreversible loss the trigger
    is for; blocking its cleanup cost a round-trip in the field and protected
    nothing. A variable, a glob, a relative path, a symlink out: blocks."""
    t = seg.split(" ")
    if t[0].rsplit("/", 1)[-1] != "rm":
        return False
    ops, flags = [], True
    for x in t[1:]:
        if flags and x == "--":
            flags = False
        elif not (flags and x.startswith("-")):
            ops.append(x)
    roots = {os.path.realpath(r) for r in ("/tmp", "/var/tmp", tempfile.gettempdir(),
                                           os.environ.get("TMPDIR") or "/tmp")}
    roots.discard("/")
    repo = os.path.realpath(ROOT)
    for x in ops or [""]:
        if not x.startswith("/") or any(c in x for c in "*?[]{}$`~\x00"):
            return False
        p = os.path.realpath(x)
        if not any(p.startswith(r + "/") for r in roots):
            return False
        if p == repo or p.startswith(repo + "/") or repo.startswith(p + "/"):
            return False
    return True


SECRET_PATH = re.compile(
    r"(^|/)(\.env(\..+)?|.*secret.*|.*credential.*|.*\.pem|.*\.key|id_rsa)$", re.I)


def load_thresholds():
    """Tightening-only merge. A settings file that loosens a threshold is
    ignored by code, not by policy."""
    out = {"max_attempts": 3, "confidence_floor": 0.6, "spend_fraction": 0.6}
    tighter = {"confidence_floor": +1, "spend_fraction": -1, "max_attempts": -1}
    p = ROOT / ".orch/config/settings.yaml"
    if not p.exists():
        return out
    for line in p.read_text().splitlines():
        m = re.match(r"\s*(max_attempts|confidence_floor|spend_fraction)\s*:\s*([\d.]+)", line)
        if not m:
            continue
        k, v = m.group(1), float(m.group(2))
        if (v - out[k]) * tighter[k] > 0:      # only if strictly tighter
            out[k] = v
    return out


# A write inside .work/<task_id>/ IS a task write: the worktree is named for its
# task, so scope resolves from the path itself and needs no environment. This is
# the load-bearing case. A PreToolUse hook is spawned by Claude Code and
# inherits ITS environment, so ORCH_TASK exported in a subagent's shell never
# reaches this process — enforcement that depends on it is enforcement that is
# not running. ORCH_TASK stays as a fallback for writes outside any worktree.
# (Found and fixed in the field, slm-memory-management, 2026-08-28.)
WORKTREE = re.compile(r"(?:^|/)\.work/(tsk_[A-Za-z0-9._-]+)/(.+)$")


def task_from_path(path):
    """(task_id, path relative to its worktree), or (None, None)."""
    m = WORKTREE.search(path)
    return (m.group(1), m.group(2)) if m else (None, None)


def scope_of(tid):
    """scope.paths of task `tid`, from the live queue or the drained one.

    None  — no task; the guard enforces only the global triggers.
    []    — a task IS named but its scope could not be read. That blocks
            writes. An unreadable contract is not an unlimited one, and this
            is the failure mode that silently disarms scope enforcement.

    Both YAML styles are accepted, because both get written in practice and a
    guard that only understands one of them is off half the time:

        scope: {paths: ["src/a.py"], network: false}   # flow — the §5.4 skeleton
        scope:                                          # block — §7's test
          paths:
            - "src/a.py"
    """
    if not tid:
        return None
    for rel in (f".orch/queue/{tid}.yaml", f".orch/queue/done/{tid}.yaml"):
        p = ROOT / rel
        if p.exists():
            break
    else:
        return []                          # a task was named, no packet exists
    return scope_paths(p.read_text())


def scope_paths(txt):
    """scope.paths from a packet's text, either YAML style. orch-lint.py
    imports this, so the lint and the guard cannot disagree about a scope."""
    m = re.search(r"paths\s*:\s*\[(.*?)\]", txt, re.S)
    if m:                                  # flow style
        return [x.strip().strip("\"'") for x in m.group(1).split(",") if x.strip()]
    paths, in_paths = [], False            # block style
    for line in txt.splitlines():
        if re.match(r"\s*paths\s*:\s*(#.*)?$", line):
            in_paths = True
            continue
        if in_paths:
            m = re.match(r"\s*-\s*(.+?)\s*$", line)
            if m:
                paths.append(m.group(1).strip("\"'"))
            else:
                break
    return paths


def in_scope(rel, scope):
    from fnmatch import fnmatch
    return any(fnmatch(rel, g) or rel.startswith(g.rstrip("/*") + "/")
               for g in scope)


def bash_trigger(cmd):
    """The first trigger `cmd` would hit, or None. The hook and --lint share
    this, so a lint can never disagree with the guard it predicts."""
    text = content_view(cmd)
    for pat, why in CONTENT:
        if re.search(pat, text, re.I):
            return why
    for seg in commands(cmd):
        for pat, why in COMMANDS:
            if re.search(pat, seg, re.I) and not (why == "recursive force delete" and temp_only(seg)):
                return why
    return None


MERGE_REF = re.compile(r"(?:refs/heads/)?orch/(tsk_[A-Za-z0-9._-]+)")


def merged_tasks(seg):
    """Task ids whose branch this one command merges (`git merge`, `git pull`
    naming orch/tsk_…), global options and a leading wrapper word skipped."""
    t = seg.split(" ")
    names = [x.rsplit("/", 1)[-1] for x in t]
    if "git" not in names:
        return []
    i = names.index("git") + 1
    while i < len(t) and t[i].startswith("-"):
        i += 2 if t[i] in ("-C", "-c", "--git-dir", "--work-tree", "--namespace") else 1
    if i >= len(t) or t[i] not in ("merge", "pull"):
        return []
    return [m.group(1) for x in t[i + 1:] for m in [MERGE_REF.fullmatch(x)] if m]


def merge_ready(tid):
    """Why task `tid`'s branch may not merge yet, or None. Code saw it done:
    a verify record saying `done` at the branch's current tip, and the packet
    in queue/done/ — finish ran, so what verify saw is registered and traced."""
    vf = ROOT / ".work/_verify" / f"{tid}.json"
    if not vf.exists():
        return f"orch/{tid} has no verify record ({vf.relative_to(ROOT)}) — run `orch-task.py verify {tid}` first"
    try:
        rec = json.loads(vf.read_text())
    except ValueError:
        return f"{vf.relative_to(ROOT)} is not JSON — verify again"
    if rec.get("verdict") != "done":
        return f"verify said `{rec.get('verdict')}` for {tid} — only a clean done merges"
    cp = subprocess.run(["git", "-C", str(ROOT), "rev-parse", "--verify", "-q", f"orch/{tid}^{{commit}}"],
                        capture_output=True, text=True)
    tip = cp.stdout.strip()
    if not tip:
        return f"there is no branch orch/{tid}"
    if tip != rec.get("head"):
        return f"orch/{tid} moved since verify ({str(rec.get('head'))[:12]} -> {tip[:12]}) — verify again"
    if not (ROOT / ".orch/queue/done" / f"{tid}.yaml").exists():
        return f"{tid} is verified but not finished — run `orch-task.py finish {tid}` first"
    return None


def merge_gate(cmd):
    """Why `cmd` may not merge the task branch it names, or None.

    A merge once ran on a `failed` verdict: `verify … | tail -2 && finish … |
    grep merge: ; git merge orch/<id>` — the pipe hid verify's exit code and the
    `;` ran the merge whatever finish said (slm, 2026-10-06). This hook runs
    before the line does, so it cannot see a verify earlier in the same line:
    a task branch merges as a command of its own — only `cd`, or another such
    merge, before it — once the records say verify and finish both ran. What
    follows it (`| tail`) is not gated. The user's own `! git merge` never
    reaches this hook."""
    if "orch/" not in cmd or not re.search(r"\b(merge|pull)\b", cmd):
        return None
    every = [s for s in commands(cmd) if merged_tasks(s)]
    if not every:
        return None
    try:
        top = [" ".join("\x00" if any(c in x for c in " \t\n") else x for x in seg)
               for seg, _ in segments(tokens(normalize(cmd)))]
    except ValueError:
        return "the command line cannot be read, so the merge in it cannot be checked"
    if sum(1 for s in top if merged_tasks(s)) != len(every):
        return "a task branch merges as a plain command, not inside a wrapper or a substitution"
    last = max(i for i, s in enumerate(top) if merged_tasks(s))
    other = [s.split(" ")[0] for s in top[:last] if not merged_tasks(s) and s.split(" ")[0] != "cd"]
    if other:
        return (f"the merge follows `{other[0]}` in the same command — run it as a command of its own, "
                "after verify and finish have printed their verdicts")
    for tid in dict.fromkeys(t for s in every for t in merged_tasks(s)):
        why = merge_ready(tid)
        if why:
            return why
    return None


RESULTS = re.compile(r"(?:^|/)\.work/_results(?:/|$)")
COPIERS = {"cp", "rsync", "install", "ln"}         # write their LAST operand
WRITERS_ANY = {"mv", "tee", "touch", "truncate"}   # write every operand


def results_write(cmd):
    """Does this shell line write into .work/_results/? Agent reports are
    recorded there by orch-report.py from the harness's own hook input, so a
    file put there by hand would read as the agent's — the hole up_0013 found,
    where `verify` could not tell an orchestrator-written result file from an
    agent's. Reading it is fine."""
    if "_results" not in cmd:
        return False
    for seg in commands(cmd):
        t = seg.split(" ")
        hit = [i for i, x in enumerate(t) if RESULTS.search(x)]
        if not hit:
            continue
        name = t[0].rsplit("/", 1)[-1]
        if any(i and ">" in t[i - 1] and set(t[i - 1]) <= set("<>&|0123456789") for i in hit):
            return True
        if name in WRITERS_ANY or (name == "sed" and any(x.startswith("-i") or x == "--in-place" for x in t)):
            return True
        if name == "dd" and any(x.startswith("of=") and RESULTS.search(x) for x in t):
            return True
        ops = [x for x in t[1:] if not x.startswith("-")]
        if name in COPIERS and (ops and RESULTS.search(ops[-1])
                                or any(a in ("-t", "--target-directory") and RESULTS.search(b)
                                       for a, b in zip(t, t[1:]))):
            return True
    return False


def prompt_hits(txt):
    """(why, culprit line) for every fenced block and inline code span in a
    prompt that the guard would block. orch-lint.py packet --prompt shares it."""
    fences = re.findall(r"^```[^\n]*\n(.*?)^```", txt, re.M | re.S)
    prose = re.sub(r"^```[^\n]*\n.*?^```", "", txt, flags=re.M | re.S)
    out = []
    for sn in fences + re.findall(r"`([^`\n]+)`", prose):
        why = bash_trigger(sn)
        if why:
            ls = sn.strip().splitlines()
            line = next((x for x in ls if bash_trigger(x)), ls[0])   # the culprit, not line 1
            out.append((why, line.strip()[:100]))
    return out


def lint(path):
    """`orch-guard.py --lint <file|->`: which commands in a dispatch prompt the
    guard would block. A guarded step inside a prompt costs a whole attempt
    with zero progress. orch-lint.py packet --prompt runs the same check with
    the packet's own; this form is for a prompt with no packet. Exit 1 on any
    hit, 0 clean, 2 if it could not read the file."""
    try:
        txt = sys.stdin.read() if path == "-" else Path(path).read_text()
    except OSError as e:
        sys.stderr.write(f"orch-guard --lint: {e}\n")
        return 2
    hits = prompt_hits(txt)
    for why, line in hits:
        print(f"blocked: {why} — {line}")
    if hits:
        print(f"\n{len(hits)} guarded command(s). Remove them from the prompt, "
              "or raise the checkpoint BEFORE dispatch if one is truly needed.")
    return 1 if hits else 0


BLIND, BLIND_LOG, JUDGE = ".work/_blind", ".work/_blind-denied.log", "orch-judge"
# How a subagent delivers its report in auto mode: the call reads no file,
# writes none and runs nothing. v5 refused it like any other tool, so all six
# judges of one measurement finished and none could deliver (slm up_0012).
HANDBACK = {"SubagentHandback"}


def blind():
    """`orch-guard.py --blind`: a session-wide PreToolUse hook (settings.json,
    matcher "*"). A blind judge reads only what the orchestrator packaged for
    it under .work/_blind/. Before this, blindness was testimony: judges said
    they never opened the results, and nothing could check it.

    v4 wired this in orch-judge's frontmatter instead, and Claude Code 2.1.289
    never ran it there: a probe hook in the same place logged nothing, and the
    judge read its canary. The same command in settings.json fires for the
    judge's calls, with agent_type in the input. So it runs for every call in
    the session and constrains only calls whose hook input says agent_type
    orch-judge: it can never block the orchestrator — and a harness that omits
    the field leaves the judge unrestricted, silently. That is why every
    denial is logged with the agent's id: the judge's first call is a canary
    read outside the blind dir, and a judge with no canary line in the log was
    not restricted (orch-task §8). Input that cannot be parsed cannot be
    attributed, and every session's calls pass through here, so it is allowed;
    the canary is what catches that. Once a call is known to be the judge's,
    any error denies: a broken check wedges one judge, never the session.
    SubagentHandback is allowed: it is how the judge's report leaves (HANDBACK)."""
    try:
        ev = json.load(sys.stdin)
    except Exception:
        return 0                          # unattributable: allowed; the canary detects it
    if ev.get("agent_type") != JUDGE:
        return 0
    tool, raw = ev.get("tool_name", "?"), ""
    if tool in HANDBACK:
        return 0                          # the report going out; orch-report.py records it
    try:
        ti = ev.get("tool_input", {}) or {}
        cwd = Path(ev.get("cwd") or os.getcwd())
        allowed = (ROOT / BLIND).resolve()

        def inside(r):
            p = Path(os.path.expanduser(str(r)))
            p = (p if p.is_absolute() else cwd / p).resolve()
            return p == allowed or allowed in p.parents

        if tool == "Read":
            raw = ti.get("file_path") or ""
            why = None if raw and inside(raw) else f"Read {raw or '(no path)'}"
        elif tool in ("Grep", "Glob"):
            raw = ti.get("path") or str(cwd)
            globs = [ti.get("glob") or ""] + ([ti.get("pattern") or ""] if tool == "Glob" else [])
            out = [g for g in globs if g.startswith(("/", "~")) or ".." in g.split("/")]
            why = None if inside(raw) and not out else f"{tool} {raw} {' '.join(out)}".strip()
        else:
            why = f"{tool} — a blind judge reads; it runs and writes nothing"
    except Exception as e:
        why = f"{tool} — the check itself failed ({e})"
    if why is None:
        return 0
    try:
        from datetime import datetime, timezone
        (ROOT / ".work").mkdir(exist_ok=True)
        with open(ROOT / BLIND_LOG, "a") as f:
            f.write(f"{datetime.now(timezone.utc).isoformat(timespec='seconds')} "
                    f"{ev.get('agent_id', '-')} {tool} {raw or '-'}\n")
    except OSError:
        pass
    sys.stderr.write(
        f"ORCH blind — denied: {why}. A blind judge reads only what was packaged "
        f"for it under {BLIND}/. Do not look for another way in: judge from what "
        "you were given, or end blocked.\n")
    return 2


def block(reason):
    sys.stderr.write(
        f"ORCH checkpoint — blocked: {reason}\n"
        "This is a mandatory risk trigger. Do not route around it. Write the "
        "checkpoint to .orch/checkpoints/ and ask the user to approve or reject.\n"
        "If what matched is text you are putting in a file — a note, a result, a "
        "prompt, a script — and not a command you mean to run, write that file "
        "with the Write tool: the guard reads commands, and Write content is not "
        "one. That is the intended route for text, not a way around this check.\n")
    sys.exit(2)


def refuse(what, reason, advice):
    """A refusal that is not a risk trigger: no checkpoint to write, one thing to do."""
    sys.stderr.write(f"ORCH {what} — refused: {reason}\n{advice}\n")
    sys.exit(2)


def main():
    try:
        ev = json.load(sys.stdin)
    except Exception:
        sys.exit(0)                      # fail open: never wedge on bad input
    tool = ev.get("tool_name", "")
    ti = ev.get("tool_input", {}) or {}

    if tool == "Bash":
        cmd = ti.get("command", "")
        why = bash_trigger(cmd)
        if why:
            block(f"{why} — irreversible or externally visible")
        why = merge_gate(cmd)
        if why:
            refuse("merge gate", why,
                   "A task branch merges only after `verify` said done at its tip and `finish` ran, and as "
                   "a command of its own: this check runs before the line does. If the user wants it merged "
                   "regardless, hand it back to them as one line: `! git merge --no-edit orch/<task_id>`.")
        if results_write(cmd):
            refuse("results", "a shell write into .work/_results/",
                   "Agent reports are recorded there by orch-report.py, from the harness's own hook input. "
                   "A report you hold yourself goes to verify as `--result <file>`, which records it as yours.")

    if tool in ("Write", "Edit", "NotebookEdit"):
        fp = os.path.normpath(ti.get("file_path", "") or ".")
        if RESULTS.search(fp):
            refuse("results", f"a write into .work/_results/ ({fp})",
                   "Agent reports are recorded there by orch-report.py, from the harness's own hook input. "
                   "A report you hold yourself goes to verify as `--result <file>`, which records it as yours.")
        # Path first, environment second: inside a worktree the path names the
        # task, which also keeps parallel tasks apart. Relative paths are taken
        # as repo-relative, as before.
        tid, rel = task_from_path(fp)
        if tid is None:
            tid = os.environ.get("ORCH_TASK")
            rel = os.path.relpath(fp, ROOT) if os.path.isabs(fp) else fp
        if SECRET_PATH.search(rel):
            block(f"write to a secrets path ({rel})")
        if re.search(r"(^|/)\.orch/config/", fp) or rel.startswith(".orch/config/"):
            block("edit of .orch/config — the trigger list and security policy "
                  "are human-owned and never self-modified")
        scope = scope_of(tid)
        if scope is not None:
            if not scope:
                block(f"a write for {tid}, but no readable scope.paths was found "
                      "in its packet (.orch/queue/ or queue/done/). Fix the "
                      "packet — a scope the guard cannot read is not a scope.")
            if not in_scope(rel, scope):
                block(f"write outside scope.paths ({rel}) for {tid}; "
                      f"declared: {', '.join(scope)}")
    sys.exit(0)


if __name__ == "__main__":
    if len(sys.argv) == 3 and sys.argv[1] == "--lint":
        sys.exit(lint(sys.argv[2]))       # a tool, not the hook: errors exit 2
    if sys.argv[1:] == ["--blind"]:
        sys.exit(blind())                 # session-wide; fails closed for the judge only
    try:
        main()
    except Exception as e:                # a broken guard fails open, loudly
        sys.stderr.write(f"orch-guard error (allowing): {e}\n")
        sys.exit(0)
```
````

````
FILE: .claude/settings.json (merge into existing)
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|Write|Edit|NotebookEdit",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/orch-guard.py\""
          }
        ]
      },
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/orch-guard.py\" --blind"
          }
        ]
      },
      {
        "matcher": "SubagentHandback",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/orch-report.py\"",
            "timeout": 150
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "sh -c 'cd \"$CLAUDE_PROJECT_DIR\" 2>/dev/null || exit 0; [ -d .orch ] || exit 0; echo \"## ORCH state\"; n=$(ls .orch/checkpoints/*.md 2>/dev/null | wc -l); q=$(ls .orch/queue/*.yaml 2>/dev/null | wc -l); echo \"open checkpoints: $n | queued tasks: $q\"; [ \"$n\" -gt 0 ] && ls .orch/checkpoints/; b=$(grep -c \"^blk_\" .orch/memory/INDEX.md 2>/dev/null | head -1); [ -n \"$b\" ] || b=0; [ \"$b\" -gt 0 ] && { echo \"## Memory index ($b blocks; full index at .orch/memory/INDEX.md)\"; grep -h \"^blk_\" .orch/memory/INDEX.md | head -40; }; exit 0'"
          }
        ]
      },
      {
        "hooks": [
          {
            "type": "command",
            "command": "sh -c 'cd \"$CLAUDE_PROJECT_DIR\" 2>/dev/null && [ -f .claude/hooks/orch-stage.py ] && python3 .claude/hooks/orch-stage.py status --brief; exit 0'"
          }
        ]
      }
    ],
    "SubagentStop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/orch-report.py\"",
            "timeout": 150
          }
        ]
      }
    ]
  }
}
```
````

If `.claude/settings.json` already exists, merge the `hooks` keys — do not
overwrite the file. The second PreToolUse entry is `orch-judge`'s blind
restriction (orch-task §8). It lives here, not in the agent's frontmatter,
because that is where Claude Code was seen to run it: a frontmatter hook in
`orch-judge.md` never fired (2.1.289, `claude -p`), while this entry fires for
the judge's calls and its input names the agent. It is a separate entry, not
a wider matcher on the first, so each keeps its own failure policy — the guard
allows on error, the blind check denies the judge on error. An install made
before it existed lacks it until the installer is re-run; `--check` reports
`missing 1 hook entry`. The second SessionStart entry reports long-job stages
(`orch-stage.py`): a stage that ran overnight, or died with the last session,
is the first thing the next one needs to know. It prints nothing when no stage
has ever run, and it marks where a session begins — from the hook input's
`source`, `startup` or `clear` — so `status` can total this session's stages
apart from the project's. The third PreToolUse entry and the SubagentStop
entry run `orch-report.py`, below, with a 150-second limit: for the archivist
it runs the memory lint. An install made before v6 lacks both until the
installer is re-run; `--check` reports them missing.

#### The scan the guard would otherwise block

`orch-task` §5 requires a post-check: test the diff's added lines against the
`content_patterns` in `sensitive.yaml`. Done as an inline grep, the command must
carry those patterns as literals in argv — and one of them describes a
force-push. The guard matches its own policy inside the scan's arguments and
blocks a read-only command. Every orchestrator hits this exactly once.

The fix is a helper that loads the patterns from the YAML **at runtime**, so no
literal ever reaches argv. It is not solved by exempting a script name inside the
guard: an exemption keyed on a name is a hole anything can name-drop
(`orch-scan.py; git push --force`). The guard stays absolute; the scan stops
carrying contraband.

````
FILE: .claude/hooks/orch-scan.py
```python
#!/usr/bin/env python3
"""ORCH sensitive scan — the orch-task §5 post-check, as code.

Loads content_patterns, path_globs and dependency_manifests from
.orch/config/sensitive.yaml at runtime. Nothing from the policy is ever spelled
in argv, which is the entire point: one of those patterns describes a
force-push, and orch-guard would match it inside an inline grep's own arguments
and block a read-only scan.

    python3 .claude/hooks/orch-scan.py <worktree> [<base-ref>] [--task <tsk_id>]
    # base: HEAD^ · task: $ORCH_TASK

With a task, two fields of its packet (.orch/queue/<id>.yaml) are honoured,
both confirmed by the user with the rest of the packet, neither a model's
call at scan time:
  expects:  [<why>, ...]   medium findings with that `why` report as low — the
                           task's own subject matter (a network client task
                           adds network code). High findings are never lowered.
  vendored: - {path: "<dir>", tree: "<tree-ish>"}
                           content findings under <dir> are summarised, not
                           gated, IF AND ONLY IF HEAD:<dir> is byte-identical
                           to <tree-ish> (compared by git tree hash). A claim
                           that does not hold is itself a high finding.
                           Dependency manifests gate regardless.

exit 0  clean, or low severity only
exit 1  findings above `low` -> mandatory checkpoint before anything registers
exit 2  usage, git or internal error. It never reports clean (or findings)
        when it could not look.
"""
import os
import re
import subprocess
import sys
from fnmatch import fnmatch
from pathlib import Path

RANK = {"low": 0, "medium": 1, "high": 2}


def project_root():
    """The main checkout, never a task worktree — same resolution as
    orch-guard.py, see its docstring. CLAUDE_PROJECT_DIR drifts into .work/<id>,
    where there is no policy, and the scan then exits 2 on every task."""
    def main_of(p):
        g = p / ".git"
        if g.is_file():
            m = re.match(r"gitdir:\s*(.+)", g.read_text().strip())
            if m:
                gd = Path(m.group(1))
                gd = (gd if gd.is_absolute() else p / gd).resolve()
                if gd.parent.name == "worktrees":
                    return gd.parent.parent.parent
        return p
    here = main_of(Path(__file__).resolve().parents[2])
    env = os.environ.get("CLAUDE_PROJECT_DIR")
    if env and (main_of(Path(env).resolve()) / ".orch").is_dir():
        return main_of(Path(env).resolve())
    return here


ROOT = project_root()
ENTRY = re.compile(
    r'\s*-\s*\{severity:\s*(\w+)\s*,\s*why:\s*(.+?)\s*,\s*pattern:\s*"(.+)"\s*\}\s*$')


def die(msg):
    """Always exit 2, never 1. 1 means findings; a scan that could not run must
    never be readable as a scan that ran and found nothing."""
    sys.stderr.write(f"orch-scan: {msg}\n")
    sys.exit(2)


def policy():
    """Parse the flow-style entries of sensitive.yaml. No YAML dependency —
    the guard has none either, and neither may acquire one."""
    p = ROOT / ".orch/config/sensitive.yaml"
    if not p.exists():
        die(f"no policy at {p}")
    pats, globs, lists, section = [], [], {}, None
    for line in p.read_text().splitlines():
        if re.match(r"^\w[\w_]*\s*:", line):
            section = line.split(":")[0]
        m = ENTRY.match(line)
        if m:
            # Case-insensitive by default; a pattern that must be case-exact
            # scopes it with (?-i:...) — `fetch(` is a call, `performFetch(` is not.
            raw = m.group(3).replace("\\\\", "\\")     # YAML double-quoted
            try:
                pats.append((m.group(1), m.group(2), re.compile(raw, re.I)))
            except re.error as e:
                die(f"unparseable pattern {raw!r}: {e}")
            continue
        m = re.match(r"^(\w+)\s*:\s*\[(.*)\]\s*(#.*)?$", line)
        if m:
            lists[m.group(1)] = flow_list(m.group(2))
        elif section == "path_globs":
            m = re.match(r'\s*-\s*"(.+)"\s*$', line)
            if m:
                globs.append(m.group(1))
    if not pats:
        die("policy parsed to zero patterns — refusing to pass")
    return pats, globs, lists


def flow_list(s):
    return [x.strip().strip("\"'") for x in s.split(",") if x.strip()]


def packet(task):
    """(expects, vendored) from the task's packet. A packet that cannot be read
    lowers nothing: the scan just runs at full strictness, and says so."""
    if not task:
        return [], []
    p = ROOT / f".orch/queue/{task}.yaml"
    if not p.exists():
        sys.stderr.write(f"orch-scan: no packet at {p}; expects/vendored ignored\n")
        return [], []
    txt = p.read_text()
    m = re.search(r"^expects\s*:\s*\[(.*?)\]", txt, re.M)
    expects = flow_list(m.group(1)) if m else []
    vend = []
    m = re.search(r"^vendored\s*:(.*?)(?=^\w|\Z)", txt, re.M | re.S)
    if m:
        for e in re.finditer(r'\{\s*path:\s*"?([^",}]+?)"?\s*,\s*tree:\s*"?([^",}]+?)"?\s*\}',
                             m.group(1)):
            vend.append((e.group(1).strip().rstrip("/"), e.group(2).strip()))
    return expects, vend


def rev(worktree, spec):
    cp = subprocess.run(["git", "-C", worktree, "rev-parse", "--verify", "-q", spec],
                        capture_output=True, text=True, errors="replace")
    return cp.stdout.strip() if cp.returncode == 0 else None


LOOPBACK = re.compile(r"^(localhost|127(\.\d+){3}|\[::1\]|0\.0\.0\.0)$", re.I)
URL_HOST = re.compile(r"\bhttps?://([^/\s:'\"`$]+|\[[^\]]+\])")


def added(worktree, base):
    """(path, text) for every ADDED line. Added lines only: an existing
    credential the task merely moved past is not this task's finding."""
    # --text + errors="replace": a binary or non-UTF-8 diff must be scanned,
    # not crash. A crash used to exit 1, which reads as "findings".
    cp = subprocess.run(["git", "-C", worktree, "diff", "--text", "--unified=0",
                         base, "HEAD"],
                        capture_output=True, text=True, errors="replace")
    if cp.returncode != 0:
        die(f"git diff failed: {cp.stderr.strip()}")
    path, out = None, []
    for line in cp.stdout.splitlines():
        if line.startswith("+++ "):
            path = line[6:] if line.startswith("+++ b/") else line[4:].strip()
        elif line.startswith("+"):
            out.append((path, line[1:]))
    return out


def hits(p, globs):
    # fnmatch has no ** — `**/.env` must also match a top-level `.env`
    return any(fnmatch(p, g) or fnmatch(p, g.replace("**/", "", 1)) for g in globs)


def manifest(p, manifests):
    """dependency_manifests entries are globs. A bare one (`requirements*.txt`)
    matches the file name at any depth; one with a `/` the repo-relative path.
    They were exact names, and a new `requirements-train.txt` (torch,
    transformers) reached master with no new_dependency finding (slm up_0010)."""
    return any(hits(p, [m]) if "/" in m else fnmatch(Path(p).name, m) for m in manifests)


def main():
    args = sys.argv[1:]
    task = os.environ.get("ORCH_TASK")
    if "--task" in args:
        i = args.index("--task")
        if i + 1 >= len(args):
            die("--task needs a task id")
        task = args[i + 1]
        del args[i:i + 2]
    if not args:
        die("usage: orch-scan.py <worktree> [<base-ref>] [--task <tsk_id>]")
    wt, base = args[0], args[1] if len(args) > 1 else "HEAD^"
    pats, globs, lists = policy()
    manifests = lists.get("dependency_manifests", [])
    expects, vendored = packet(task)
    lines = added(wt, base)
    paths = sorted({p for p, _ in lines if p})
    found = []

    # vendored: a verbatim upstream tree is a different risk class from lines
    # an agent wrote — but only once git proves it is verbatim.
    vroots = []
    for vpath, tree in vendored:
        have = rev(wt, f"HEAD:{vpath}")
        want = rev(wt, tree if ":" in tree else f"{tree}^{{tree}}")
        if have and have == want:
            vroots.append((vpath, tree))
        else:
            found.append(("high", f"{vpath}: packet declares it vendored from "
                          f"{tree}, but the trees differ ({have} != {want}) — "
                          f"treated as authored code"))

    # network findings under test paths whose added lines name no external host
    external = {p for p, t in lines for h in URL_HOST.findall(t)
                if not LOOPBACK.match(h)}
    test_paths, test_down = lists.get("test_paths", []), lists.get("test_downgrade", [])

    vend_hits = {}
    for path, text in lines:
        for sev, why, rx in pats:
            if not rx.search(text):
                continue
            vr = next((v for v in vroots if path and
                       (path == v[0] or path.startswith(v[0] + "/"))), None)
            if vr:
                k = (vr, sev, why)
                vend_hits[k] = vend_hits.get(k, 0) + 1
                continue
            note = ""
            if sev == "medium" and why in expects:
                sev, note = "low", " [packet expects:]"
            elif (sev == "medium" and why in test_down and path
                  and hits(path, test_paths) and path not in external):
                sev, note = "low", " [test path, no external host]"
            found.append((sev, f"{path}: {why}{note} — {text.strip()[:110]}"))
    for ((vpath, tree), sev, why), n in sorted(vend_hits.items()):
        found.append(("low", f"{vpath}/: {n}× {sev} {why} — vendored, identical "
                      f"to {tree}, not gated (scan with no task to list them)"))
    found += [("high", f"{p}: dependency manifest changed — new_dependency")
              for p in paths if manifest(p, manifests)]
    # Informational only, deliberately never gating. path_globs are the §3
    # PRE-dispatch gate on scope.paths, decided once by a human before any cost.
    # Re-gating them here double-charges the same fact, and the globs are broad
    # by design: `**/*token*` matches tokenizer.py, `**/*session*` matches
    # session_config.py. A trigger that fires on every real task is a trigger
    # everyone learns to ignore.
    found += [("low", f"{p}: matches a sensitive path_glob (informational; "
                      f"the gate for this was §3, before dispatch)")
              for p in paths if hits(p, globs)]

    for sev, msg in sorted(found, key=lambda f: -RANK.get(f[0], 0)):
        print(f"{sev:6} {msg}")
    gate = [f for f in found if RANK.get(f[0], 0) > RANK["low"]]
    if gate:
        print(f"\n{len(gate)} finding(s) above low — write the checkpoint "
              f"BEFORE registering anything.")
        return 1
    print(f"clean ({len(lines)} added lines, {len(paths)} files)"
          if not found else f"{len(found)} low-severity finding(s) only")
    return 0


if __name__ == "__main__":
    try:
        rc = main()
    except Exception as e:                # any crash is "could not look", never 1
        die(f"internal error: {type(e).__name__}: {e}")
    sys.exit(rc)
```
````

#### Lints for what the orchestrator writes

The guard checks an agent's tool calls and the scan checks an agent's diff.
Neither looks at the two things the orchestrator side writes — the packet, and
(through the archivist) memory — and in the field that is where the defects came
from. One session (slm-memory-management, 2026-09-29) produced five packet
defects across four tasks plus two archivist repairs: a check asserting
`git status` after the commit that empties it, a check script living in the
session scratchpad, "pass it to *every* `Retriever(...)`" in a scope that
excluded six files using one, a test count taken across unmerged branches, and
INDEX headlines stating the opposite of their blocks. None was visible to the
guard or the scan. Each was caught because someone happened to look — an
executor flagging it, or the orchestrator checking by hand.

`orch-lint.py` makes the machine-checkable ones code. `packet` runs on every draft before the
user sees it (`orch-task` §0); `--measure` also runs each executable check at the
packet's `base:`, each in a fresh throwaway checkout, and prints what it returns
— the number a count must be measured from, and whether the check can fail at
all. `memory` runs after every archivist ingest; its count line is what the
archivist's report must quote. It imports the guard's matcher, root and scope
parser rather than copying them, and never runs a command the guard would block.

The next session (slm-memory-management, 2026-10-03) found three more shapes, all
now in `packet`: a check whose run name was wrong failed at base, and `--measure`
printed that exactly as it prints a missing feature (`up_0007`) — it now says
which kind of failure it saw; a stacked branch's prompt pointed at a doc section
that existed only on master; a prompt said `<base>` where a check needed the
sha. `--prompt` checks a dispatch prompt against its packet. And `doc` checks a
results doc against the registry, because that session's two wrong conclusions
were numbers no registered command produced.

v6 (2026-10-06/07) is mostly subtraction. On that session's 18 packets the
lint printed 129 warnings and one false error, nearly all a bare file name
(`screen_report.py` is scripts/screen_report.py) or a suffix named in prose
(`-answers.json`), and the error a format template in a note. The habit that
teaches is dispatching past an error, and it was taught. Names are now looked
up by their trailing path, a name in scope by its name alone is the task's, a
name git ignores or cannot find anywhere is prose, "every X" counts only code,
and a placeholder counts only as a word of a command. Re-run on the same
packets: 13 warnings, no error — and 9 of the 13 say a cited file changed on
HEAD since base forked, true only because master has moved past those bases
since; for a fresh packet, whose base is HEAD, that check is silent. Two checks were added, each
for a wrong check that cost a round-trip: a `grep -c` over several files under
a numeric expect, and output at base that no number can equal. `memory` now
compares the numbers a claim states with what it cites.

````
FILE: .claude/hooks/orch-lint.py
```python
#!/usr/bin/env python3
"""ORCH lint — code checks on what the orchestrator side writes.

The guard checks an agent's tool calls and the scan checks its diff. Nothing
checked the two things the orchestrator side writes itself — the packet, and
(through the archivist) memory — and that is where the field defects came from:
a check asserting `git status` after the commit that empties it, a check script
in the session scratchpad, a test count written from memory, an INDEX headline
saying the opposite of its block. Each one passed every other check, because a
defect in the definition of success is invisible to verification.

    python3 .claude/hooks/orch-lint.py packet <packet.yaml> [--measure] [--prompt <file>]...
    python3 .claude/hooks/orch-lint.py memory [--task <tsk_id|msr_id>]
    python3 .claude/hooks/orch-lint.py doc <results.md>... [--strict]

packet  the orch-task §1 rules a machine can check. --measure also runs every
        executable check at the packet's `base:` in a throwaway worktree and
        prints what it returns there: the number a count is measured from, and
        whether the check can tell the finished task from the unstarted one —
        and, when it fails there, whether it failed like a missing feature or
        like a wrong command, or printed something no `stdout == N` can
        equal. A command the guard would block is never run. A `grep -c`
        over several files under a numeric expect is an error without it.
        --prompt checks a dispatch prompt against the packet: guarded
        commands, placeholders in a command, the base sha and worktree path,
        and files it cites that the agent's checkout will not hold as written
        (a bare name is looked up by its trailing path first).
memory  INDEX.md against blocks/: one line per block, no orphans, no strays,
        each headline equal to its block's own, archived blocks out of the
        index. Every [src: <path>:<line> "<excerpt>"] must quote its source:
        the excerpt is looked for at those lines. --task also checks that every
        block the task's receipt names landed, that each claim sentence in
        them cites something, and that every result-shaped number a sentence
        states is in what it cites. The summary line is the count an
        archivist's report quotes.
doc     a results doc against the registry: every `[chk_…]` citation names an
        active entry, and the number right before it is that entry's
        registered value. Result-shaped numbers on lines that cite nothing
        are listed (errors with --strict) — a number nobody can replay. An
        entry registered post hoc must be cited on a line that says so.

exit 0  clean (warnings may print)
exit 1  findings
exit 2  could not read its input. Never reported as clean.
"""
import importlib.util
import json
import os
import re
import signal
import subprocess
import sys
from pathlib import Path


def die(msg):
    """Always 2, never 1: a lint that could not look must never read as one
    that looked and found something — or as one that found nothing."""
    sys.stderr.write(f"orch-lint: {msg}\n")
    sys.exit(2)


def _guard():
    """The guard's own matcher, root and scope parser. One copy of each, so
    this lint cannot disagree with the code that enforces."""
    p = Path(__file__).resolve().with_name("orch-guard.py")
    sys.dont_write_bytecode = True        # no __pycache__ left in the user's tree
    spec = importlib.util.spec_from_file_location("orch_guard", p)
    g = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(g)
    return g


try:
    G = _guard()
except Exception as e:
    die(f"cannot load orch-guard.py: {e}")
ROOT = G.ROOT
found = []


def report(level, msg):
    found.append((level, msg))


def git(*args):
    return subprocess.run(["git", "-C", str(ROOT), *args],
                          capture_output=True, text=True, errors="replace")


# ── the YAML a packet is written in. No YAML dependency: the guard and the scan
#    have none, and this may not acquire one either.

QUOTED = re.compile(r'"(?:[^"\\]|\\.)*"|\'(?:[^\']|\'\')*\'')
BRACED = re.compile(r"\{(?:[^{}\"']|\"(?:[^\"\\]|\\.)*\"|'(?:[^']|'')*')*\}")
KEY = re.compile(r"\s*([A-Za-z_][\w-]*)\s*:(?:\s+|$)")


def unquote(v):
    v = v.strip()
    m = QUOTED.match(v)
    if m and v[0] == '"':
        return re.sub(r"\\(.)", lambda e: {"n": "\n", "t": "\t"}.get(e.group(1), e.group(1)),
                      m.group(0)[1:-1])
    if m:
        return m.group(0)[1:-1].replace("''", "'")
    return re.sub(r"\s+#.*$", "", v)


def flow_map(s):
    """`{k: v, k: "v, with a comma"}` -> dict. No nesting; an entry has none."""
    s = s.strip()
    if s[:1] == "{":
        s = s[1:s.rfind("}")] if s.rstrip().endswith("}") else s[1:]
    out, i = {}, 0
    while True:
        m = KEY.match(s, i)
        if not m:
            return out
        i = m.end()
        q = QUOTED.match(s, i)
        if q:
            out[m.group(1)], i = unquote(q.group(0)), q.end()
        else:
            j = s.find(",", i)
            j = len(s) if j < 0 else j
            out[m.group(1)], i = unquote(s[i:j]), j
        i = re.compile(r"\s*,?").match(s, i).end()


def scalar(txt, key):
    lines = txt.splitlines()
    for k, line in enumerate(lines):
        m = re.match(rf"{key}\s*:[ \t]*(.*)$", line)
        if not m:
            continue
        v = m.group(1).strip()
        if v[:1] in (">", "|"):                   # folded / literal block
            body = []
            for x in lines[k + 1:]:
                if x.strip() and not x[0].isspace():
                    break
                body.append(x.strip())
            return " ".join(b for b in body if b)
        return unquote(v) if v else None
    return None


def acceptance(txt):
    """Entries under `acceptance:`, in document order, either style:
    `- {type: …, cmd: "…"}` per line, or block mappings. None if no key."""
    lines = txt.splitlines()
    for k, line in enumerate(lines):
        m = re.match(r"acceptance\s*:(.*)$", line)
        if m:
            break
    else:
        return None
    entries = [flow_map(f) for f in BRACED.findall(m.group(1))]
    cur = None
    for line in lines[k + 1:]:
        s = line.strip()
        if line.strip() and not line[0].isspace() and not line.startswith("-"):
            break
        if not s or s.startswith("#"):
            continue
        if s == "-" or s.startswith("- "):
            s, cur = s[1:].strip(), None
            if s.startswith("{"):
                entries += [flow_map(f) for f in BRACED.findall(s)] or [flow_map(s)]
                continue
            cur = flow_map(s)
            entries.append(cur)
        elif s[:1] in "{[]":
            entries += [flow_map(f) for f in BRACED.findall(s)]
        elif cur is not None:
            cur.update(flow_map(s))
    return entries


def registry_entries(txt):
    """Entries under `checks:` in a registry file, in order — the shape
    orch-task.py register writes (orch-rework §3): `- id: chk_…`, then one
    `key: value` per line; a flow map (`artifact: {…}`) is one value.
    orch-task.py imports this, so what it writes is read back by the same
    parser everything else uses."""
    out, cur, in_checks = [], None, False
    for line in txt.splitlines():
        if not line.strip() or line.lstrip().startswith("#"):
            continue
        if not line[0].isspace() and not line.startswith("-"):
            in_checks, cur = line.split(":")[0].strip() == "checks", None
            continue
        if not in_checks:
            continue
        m = re.match(r"\s*-\s+(.*)$", line)
        if m:
            cur = {}
            out.append(cur)
            if m.group(1).startswith("{"):
                cur.update(flow_map(m.group(1)))
                continue
            line = " " + m.group(1)
        m = re.match(r"\s*([A-Za-z_][\w-]*)\s*:\s*(.*)$", line)
        if m and cur is not None:
            v = m.group(2).strip()
            cur[m.group(1)] = flow_map(v) if v.startswith("{") else unquote(v)
    return out


# ── packet rules

FLOOR = re.compile(r">=|>\s*0\b|\bat least\b|-gt\s+0\b|-ge\s+1\b", re.I)
EXPECT = re.compile(r"exit\s+\d+|stdout\s*==.*|stdout\s+matches\s+/.*/|delta\s*==\s*[+-]?\d+"
                    r"|stdout\s*(?:<=|>=)\s*-?\d+", re.S)
NUMERIC = re.compile(r"stdout\s*==\s*-?\d+|stdout\s*(?:<=|>=)\s*-?\d+|delta\s*==\s*[+-]?\d+")
TESTISH = re.compile(r"(^|/)(tests?|__tests__)/|(^|/)test_[^/]*$|_test\.[^/]+$|\.test\.[^/]+$")
ABS = re.compile(r"(?<![\w.~$:/-])(~?/[^\s'\"|;&()<>`]+)")
SYSTEM = ("/usr/", "/bin/", "/sbin/", "/lib", "/opt/", "/etc/", "/dev/", "/proc/", "/sys/")
# "every `Retriever(...)`": the quantifier, at most two words, then a span that
# looks like code. "all, prints `pass`" and "each path builds its own `results`"
# are prose, and were most of what this warned on (slm up_0014).
QUANT = re.compile(r"\b(every|all|each)[*_]*\s+(?:[\w-]+\s+){0,2}`([^`\n]+)`", re.I)
CODEISH = re.compile(r"\w\(|\w[._]\w|^[A-Z][a-z0-9]+[A-Z]|^[A-Z][A-Z0-9_]{2,}$")
PLACEHOLDER = re.compile(r"(?<![<\w])<(base|sha|base_sha|task_id|tsk_id|worktree)>")
# A placeholder as a word of a command: `git diff <base> HEAD`. Not one inside a
# format template (`<out>: replaced <n> rows (base <base>, …)`), which up_0014
# found raising an error on a note that described an output format.
PLACEHOLDER_WORD = re.compile(r"(?:^|(?<=[\s=]))<(base|sha|base_sha|task_id|tsk_id|worktree)>(?=$|[\s;|&)])")
# The first character is a word character: `-answers.json` is a suffix named in
# prose, not a file (up_0014).
CITED = re.compile(r"(?<![\w./~*<>{}$-])((?:[\w.-]+/)*\w[\w.-]*\.(?:py|md|ya?ml|jsonl?|toml|txt|cfg|ini|sh"
                   r"|js|ts|tsx|c|cc|cpp|h|hpp|rs|go|rb|java|csv))(?::(\d+))?(?![\w/])")
# Why a check failed at base. A missing feature is the failure a check is
# meant to have there; a missing file the task cannot create is a wrong command
# that prints the same "fails at base" (up_0007: a run name the script then
# suffixed, so the file it opened could never exist).
USAGE = re.compile(r"unrecognized arguments?|no such option|unknown (option|flag|command|argument)"
                   r"|invalid choice|unexpected argument", re.I)
MISSING = [re.compile(p) for p in (
    r"No such file or directory:\s*'([^']+)'",            # python open()
    r"can't open file '([^']+)'",                         # python script.py
    r"([^\s:'\"`]+):\s*No such file or directory",       # coreutils, grep, bash
    r"file or directory not found:\s*(\S+)",              # pytest
    r"cannot access '([^']+)'",                           # ls, stat
    r"pathspec '([^']+)' did not match",                  # git
    r"No module named '([^']+)'")]                        # python import
NOT_FOUND = re.compile(r"not found|no such file|does not exist|FileNotFoundError|ENOENT", re.I)


def is_rev(tok):
    if re.match(r"^\$|^<.*>$|\.\.|^[0-9a-f]{7,40}$", tok):   # $base, <base>, a..b, a sha
        return True
    return git("rev-parse", "--verify", "-q", tok + "^{commit}").returncode == 0


def git_state(cmd):
    """Why `cmd` inspects the working tree, or None. orch-task §5 commits the
    work BEFORE any check runs, so `git status`, `git diff` with no revision,
    `--cached`, or HEAD alone all see a clean tree — and pass, or fail, for a
    reason that has nothing to do with the task."""
    for seg in G.commands(cmd):
        t = seg.split()
        if "git" not in [x.rsplit("/", 1)[-1] for x in t]:
            continue
        i = [x.rsplit("/", 1)[-1] for x in t].index("git") + 1
        while i < len(t) and t[i].startswith("-"):
            i += 2 if t[i] in ("-C", "-c") else 1
        sub, rest = (t[i], t[i + 1:]) if i < len(t) else ("", [])
        if sub == "status":
            return "`git status`"
        if sub != "diff" or "--no-index" in rest:
            continue
        rest = rest[:rest.index("--")] if "--" in rest else rest
        if {"--cached", "--staged"} & set(rest):
            return "`git diff --cached` (the index is empty once committed)"
        revs = [x for x in rest if not x.startswith("-") and x != "\x00" and is_rev(x)]
        if not [r for r in revs if r not in ("HEAD", "@")]:
            return "`git diff` with no base revision"
    return None


def tracked(rel):
    return git("ls-files", "--error-unmatch", "--", str(rel)).returncode == 0


def outside(tok):
    """Why an absolute path in a check is wrong there, or None."""
    p = Path(os.path.expanduser(tok))
    try:
        rel = p.resolve().relative_to(ROOT.resolve())
    except ValueError:
        rel = None
    if rel is not None:
        if str(rel) != "." and tracked(rel):
            return ("a tracked file by absolute path runs the MAIN checkout's copy, "
                    "not the task's — use the repo-relative path")
        return None            # environment that lives in the repo: a venv, ignored data
    s = str(p)
    if s.startswith(SYSTEM):
        return None
    if "/scratchpad/" in s or s.startswith(("/tmp/", "/var/tmp/", str(Path.home()) + "/")):
        return ("outside the repo — a check that depends on it cannot be registered "
                "or replayed. Commit it under the task's scope")
    return None


def every_x(texts, scope, base):
    """`every Retriever(...)` in a scope that excludes some Retriever(...) is an
    instruction the agent cannot follow — it either breaks scope or stops."""
    seen = set()
    for txt in texts:
        for m in QUANT.finditer(txt):
            if not CODEISH.search(m.group(2).strip()):
                continue                   # a word in backticks, not a call site
            s = re.match(r"[A-Za-z_][\w.]*", m.group(2).strip())
            sym = s.group(0).rstrip(".").rsplit(".", 1)[-1] if s else ""
            if len(sym) < 3 or sym in seen:
                continue
            seen.add(sym)
            cp = git("grep", "-lwF", sym, base, "--", ".", ":(exclude)*.md", ":(exclude)*.json",
                     ":(exclude)*.jsonl", ":(exclude)*.csv", ":(exclude)*.txt",
                     ":(exclude).claude", ":(exclude).orch")    # call sites, not prose or data
            files = [x.split(":", 1)[1] for x in cp.stdout.splitlines() if ":" in x]
            out = [f for f in files if not G.in_scope(f, scope)]
            if out:
                more = f" (+{len(out) - 5} more)" if len(out) > 5 else ""
                report("warn", f"\"{m.group(0)}\" — `{sym}` also occurs in {len(out)} "
                       f"file(s) outside scope.paths: {', '.join(out[:5])}{more}. Name the "
                       "ones in scope, or widen it; as written it cannot be followed")


def placeholder_words(txt):
    """Placeholders an agent would run: in a fenced block or a code span, on a
    line that starts like a command, standing as a word of their own. A span
    that is a format template starts with `<` or `{`, and `results/<base>-a.json`
    is a file name being described; neither runs."""
    fences = re.findall(r"^```[^\n]*\n(.*?)^```", txt, re.M | re.S)
    prose = re.sub(r"^```[^\n]*\n.*?^```", "", txt, flags=re.M | re.S)
    out = set()
    for sn in fences + re.findall(r"`([^`\n]+)`", prose):
        for line in sn.splitlines():
            s = line.strip()
            if s and re.match(r"[\w./$~-]", s) and not s.startswith("#"):
                out |= set(PLACEHOLDER_WORD.findall(s))
    return out


def last_json(txt):
    """The result object a report ends with: the last JSON object in the text
    that has a `status` key. orch-task.py (verify) and orch-report.py (the hook
    that records reports) read reports with this one function."""
    dec = json.JSONDecoder()
    for i in [m.start() for m in re.finditer(r"\{", txt or "")][::-1]:
        try:
            obj, _ = dec.raw_decode(txt[i:])
            if isinstance(obj, dict) and "status" in obj:
                return obj
        except ValueError:
            continue
    return None


_trees = {}


def resolve(rel, spec):
    """The paths `rel` names in a tree: itself, or a path ending in /rel —
    prose says `eval_answers.py` for scripts/eval_answers.py. `spec` is a
    commit, or "untracked" for files on disk git neither tracks nor ignores."""
    if spec not in _trees:
        cp = (git("ls-files", "-o", "--exclude-standard") if spec == "untracked"
              else git("ls-tree", "-r", "--name-only", spec))
        _trees[spec] = cp.stdout.splitlines()
    t = _trees[spec]
    return [rel] if rel in t else [p for p in t if p.endswith("/" + rel)]


def cited_files(texts, scope, base_sha):
    """Files a prompt, goal or notes cites that the agent's checkout will not
    hold as written. A worktree holds what git tracks at `base` and nothing
    else, so a file only on master — a doc section added after a stacked
    branch forked — is invisible to the agent that was told to read it.

    A name is looked up as written and then by its trailing path, and only two
    misses are reported: a file HEAD has and base does not, and a file on disk
    git does not track. One that git ignores is generated data, and one found
    nowhere is prose. Before this, bare names (`screen_report.py` for
    scripts/screen_report.py) made 3 to 19 warnings per packet, nearly all
    false (slm up_0014). Re-run on those 18 packets, 129 warnings became 13,
    9 of them true only because master had moved past their bases since."""
    head, seen = git("rev-parse", "HEAD").stdout.strip(), set()
    for src, txt in texts:
        for m in CITED.finditer(txt):
            rel, line = m.group(1), m.group(2)
            if (rel in seen or rel.startswith((".work/", ".orch/")) or G.in_scope(rel, scope)
                    or any(p.endswith("/" + rel) for p in scope)):
                continue                   # in scope, by path or by name: the task's to write or create
            seen.add(rel)
            found = resolve(rel, base_sha)
            if not found:
                if resolve(rel, "HEAD"):
                    report("warn", f"{src} cites {rel}, which is not in git at base {base_sha[:12]} — "
                           "it is at HEAD, and a stacked base does not have master's newer files")
                elif resolve(rel, "untracked"):
                    report("warn", f"{src} cites {rel}, which is not in git at base {base_sha[:12]} — it is "
                           "on disk, untracked, and the agent's worktree holds tracked files only")
                continue
            if len(found) > 1 or found[0] in seen - {rel}:
                continue                   # ambiguous by name, or already read under its full path
            rel = found[0]
            seen.add(rel)
            if base_sha != head and git("diff", "--quiet", f"{base_sha}...HEAD", "--", rel).returncode == 1:
                report("warn", f"{src} cites {rel}, which HEAD has changed since base forked — "
                       "the agent reads base's version, not the one you are looking at")
            elif line and int(line) > len(git("show", f"{base_sha}:{rel}").stdout.splitlines()):
                report("warn", f"{src} cites {rel}:{line}, past the end of the file at base")


def why_failed(text, scope, notes):
    """(level, note) for a check that fails at base."""
    if USAGE.search(text):
        return None, "fails at base on a usage error: the flag or subcommand the task adds"
    for rx in MISSING:
        for m in rx.finditer(text):
            rel = re.sub(r"^.*?/\.work/_lint_\d+/|^\./", "", m.group(1))
            if rel.startswith("-"):        # a flag opened as a file: the option is what is missing
                return None, f"fails at base: {rel} was read as a file — the option the task adds"
            pkg = None
            if rx.pattern.startswith("No module"):
                rel, pkg = rel.replace(".", "/") + ".py", rel.replace(".", "/") + "/__init__.py"
            if G.in_scope(rel, scope) or (pkg and G.in_scope(pkg, scope)):
                return None, f"fails at base: {rel} does not exist yet, and it is in scope"
            if Path(rel).name in notes:
                return None, f"fails at base: {rel} not found, and notes say what produces it"
            return "warn", (f"fails at base because {rel} is not found, and it is outside scope.paths, so "
                            "the task cannot create it: likely a wrong command (a path, a run name), not "
                            "a missing feature. Fix it, or say in notes what produces the file (up_0007)")
    line = next((x for x in text.splitlines() if NOT_FOUND.search(x)), None)
    if line:
        return "warn", (f"fails at base with a not-found error ({line.strip()[:60]}): a wrong path, run "
                        "name or command fails here exactly like a missing feature (up_0007)")
    return None, "fails at base"


def run(cmd, cwd, timeout):
    """(exit, stdout, stderr) — or None on timeout. Its own process group, so a
    timeout kills the whole pipeline, not just the shell."""
    p = subprocess.Popen(["bash", "-c", cmd], cwd=cwd, stdout=subprocess.PIPE,
                         stderr=subprocess.PIPE, stdin=subprocess.DEVNULL, text=True,
                         errors="replace", start_new_session=True)
    try:
        out, err = p.communicate(timeout=timeout)
    except subprocess.TimeoutExpired:
        os.killpg(p.pid, signal.SIGKILL)
        p.communicate()
        return None
    return p.returncode, out, err


def lines(s):
    """Text as `stdout ==` compares it: each line stripped, blank lines at the
    ends dropped. Stripping only the whole of it kept the indentation of every
    line but the first, so multi-line output held or failed on whitespace no one
    can see — and on how a shell passed a multi-line --expect through."""
    return "\n".join(x.strip() for x in s.strip().splitlines())


def holds(expect, code, out):
    e = expect.strip()
    m = re.fullmatch(r"exit\s+(\d+)", e)
    if m:
        return code == int(m.group(1))
    m = re.fullmatch(r"stdout\s*==\s*(.*)", e, re.S)
    if m:
        return lines(out) == lines(m.group(1))
    m = re.fullmatch(r"stdout\s+matches\s+/(.*)/", e, re.S)
    if m:
        return re.search(m.group(1), out) is not None
    m = re.fullmatch(r"stdout\s*(<=|>=)\s*(-?\d+)", e)
    if m:                                 # "removed lines are at most 5" read as a ceiling, not /^[0-5]$/
        if not re.fullmatch(r"-?\d+", out.strip()):
            return False
        v, k = int(out.strip()), int(m.group(2))
        return v <= k if m.group(1) == "<=" else v >= k
    return None


def multi_count(cmd):
    """The files a `grep -c` counts separately, if it counts more than one: it
    then prints `path:count` per file, so `stdout == 0` can never hold. One
    check said `failed` for work that was fine (slm ixvec, 2026-10-06)."""
    for seg in G.commands(cmd):
        t = seg.split(" ")
        if t[0].rsplit("/", 1)[-1] not in ("grep", "egrep", "fgrep"):
            continue
        opts = [x for x in t[1:] if x.startswith("-") and x != "-"]
        short = "".join(x[1:] for x in opts if not x.startswith("--"))
        if "c" not in short and "--count" not in opts:
            continue
        ops, skip, given = [], False, False
        for x in t[1:]:
            if skip:
                skip = False
            elif x in ("-e", "-f", "--regexp", "--file"):
                skip = given = True
            elif not (x.startswith("-") and x != "-"):
                ops.append(x)
        files = ops if given else ops[1:]
        if re.search(r"[rR]", short) or "--recursive" in opts or len(files) >= 2:
            return files or ["(recursive)"]
    return None


def at_base(cmd, base, timeout):
    """run() in a fresh detached checkout of `base`, one per check: a check
    that writes (bytecode, build output) must not change what the next sees."""
    wt = ROOT / ".work" / f"_lint_{os.getpid()}"
    wt.parent.mkdir(exist_ok=True)
    cp = git("worktree", "add", "-q", "--detach", str(wt), base)
    if cp.returncode != 0:
        return cp.stderr.strip() or "git worktree add failed"
    try:
        return run(cmd, wt, timeout)
    finally:
        git("worktree", "remove", "--force", str(wt))


def measure(checks, base, timeout, scope, notes):
    """Run each check at `base`. What it prints is the only honest source for a
    count: `N` in `stdout == N` is this number plus the delta the task adds."""
    print(f"measured at base {base[:12]}, each check in a fresh checkout (no seeded caches, "
          "no untracked data — a failure here can be the environment; read it)")
    held = []
    for n, e in checks:
        cmd, expect = e.get("cmd", ""), e.get("expect", "")
        if G.bash_trigger(cmd):
            print(f"  #{n} not run — the guard blocks it")
            continue
        r = at_base(cmd, base, timeout)
        if isinstance(r, str):
            report("error", f"--measure: cannot check out base {base}: {r}")
            return
        if r is None:
            print(f"  #{n} timed out after {timeout:g}s — not measured")
            continue
        code, out, err = r
        shown = out.strip().replace("\n", "\\n")
        shown = shown[:70] + "…" if len(shown) > 70 else shown
        ok, note = holds(expect, code, out), ""
        d = re.fullmatch(r"delta\s*==\s*([+-]?\d+)", expect.strip())
        want = re.fullmatch(r"stdout\s*==\s*(-?\d+)", expect.strip())
        num = re.fullmatch(r"-?\d+", out.strip())
        if NUMERIC.fullmatch(expect.strip()) and out.strip() and not num:
            report("error", f"#{n} prints \"{shown}\" at base — not one number, so `{expect.strip()}` can "
                   "never hold. Count over one stream (`cat a b | grep -c …`) or print a single value")
            note = "  -> NOT A NUMBER, see the error"
        elif d and num:
            note = f"  -> HEAD must print {int(out) + int(d.group(1))}"
        elif ok:
            held.append(n)
            note = "  -> already holds at base: cannot tell done from not started"
        elif want and num:
            note = (f"  -> asserts {want.group(1)}: the task must account for "
                    f"{int(want.group(1)) - int(out):+d}, say which in notes")
        elif ok is False:
            level, why = why_failed(out + "\n" + err, scope, notes)
            note = f"  -> {why}" if level is None else "  -> fails at base: NOT FOUND, see the warning"
            if level:
                report(level, f"#{n}: {why}")
        tail = f" · stderr: {err.strip().splitlines()[-1][:60]}" if code and err.strip() else ""
        print(f"  #{n} exit {code} · stdout \"{shown}\"{tail}{note}")
    if checks and len(held) == len(checks):
        report("warn", "every executable check already holds at base: nothing in this "
               "acceptance can fail. Right for a pure refactor; a defect for anything else")


def lint_packet(path, prompts, do_measure, timeout):
    try:
        txt = Path(path).read_text()
        extra = [Path(p).read_text() for p in prompts]
    except OSError as e:
        die(str(e))
    tid, goal, notes = scalar(txt, "task_id"), scalar(txt, "goal") or "", scalar(txt, "notes") or ""
    base, role = scalar(txt, "base"), scalar(txt, "role") or ""
    scope = G.scope_paths(txt)
    entries = acceptance(txt)
    if entries is None:
        die(f"{path}: no `acceptance:` key")
    if not entries:
        die(f"{path}: `acceptance:` holds nothing this lint can read — one `- {{…}}` per line")
    if not goal:
        report("error", "no goal")
    if not scope:
        report("error", "no readable scope.paths — the guard will block every write of this task")
    if not base:
        report("warn", "no `base:` sha — counts and diffs have nothing fixed to be measured "
               "against (--measure uses HEAD)")
    base = base or "HEAD"
    base_sha = git("rev-parse", "--verify", "-q", base + "^{commit}").stdout.strip()
    if not base_sha:
        report("error", f"base {base} does not resolve to a commit")
        do_measure = False

    effort = scalar(txt, "effort")
    if effort and effort not in ("medium", "high"):
        report("error", f"effort: {effort} — medium (orch-executor) or high (orch-executor-deep)")
    checks = [(n, e) for n, e in enumerate(entries, 1) if e.get("type", "executable") != "judged"]
    if role == "executor" and not checks:
        report("error", "an executor packet with judged acceptance only — executable beats judged")
    per_task = []
    for n, e in checks:
        cmd, expect = e.get("cmd", ""), e.get("expect", "")
        if not cmd:
            report("error", f"#{n}: executable check with no cmd")
            continue
        if not EXPECT.fullmatch(expect.strip()):
            report("error", f"#{n}: expect {expect!r} is none of: exit N · stdout == V · "
                   "stdout matches /re/ · stdout <= N · stdout >= N · delta == +k")
        files = multi_count(cmd) if NUMERIC.fullmatch(expect.strip()) else None
        if files:
            report("error", f"#{n}: grep -c over {', '.join(files[:3])} prints `path:count` per file, so "
                   f"`{expect.strip()}` can never hold — count one stream: `cat {' '.join(files[:3])} | grep -c …`")
        if (FLOOR.search(expect) or re.search(r"-gt\s+0\b|-ge\s+1\b", cmd)) and "floor" not in notes.lower():
            report("error", f"#{n}: a floor ({expect or cmd!r}) — assert the exact value, or "
                   "say in notes why it is unknowable (the word `floor` marks it read)")
        why = G.bash_trigger(cmd)
        if why:
            report("error", f"#{n}: the guard blocks this check ({why}) — it would be refused when §5 runs it")
        for m in sorted(set(PLACEHOLDER.findall(cmd))):
            report("error", f"#{n}: unfilled <{m}> — bash reads it as a redirection; write the value")
        why = git_state(cmd)
        if why:
            report("error", f"#{n}: {why} — §5 commits before checks run, so the tree is "
                   "already clean. Assert changes with `git diff --name-only <base> HEAD`")
        expanded = re.sub(r"\$\{?HOME\}?(?=/)", str(Path.home()), cmd)
        for tok in sorted(set(ABS.findall(expanded))):
            why = outside(tok)
            if why:
                report("error", f"#{n}: {tok} — {why}")
        if re.match(r"delta\b", expect.strip()):
            per_task.append(f"#{n} (delta)")
        elif base != "HEAD" and (base in cmd or base[:7] in cmd or re.search(r"\$\{?base\b|<base", cmd)):
            per_task.append(f"#{n} (names the base)")
    if scope and all(TESTISH.search(p) for p in scope) and not any("grep" in e.get("cmd", "") for _, e in checks):
        report("error", "scope is test files only and no check greps for the assertions "
               "that must survive — `make the test pass` is satisfiable by deleting it")
    every_x([goal, notes] + [e.get("criterion", "") for e in entries] + extra, scope, base)
    names_base = [x.split()[0] for x in per_task if x.endswith("(names the base)")]
    wt = str(ROOT / ".work" / tid) if tid else None
    for pth, ptxt in zip(prompts, extra):
        for why, line in G.prompt_hits(ptxt):
            report("error", f"prompt {pth}: the guard blocks `{line}` ({why}) — it costs a whole "
                   "attempt mid-run. Take it out, or raise the checkpoint now")
        for m in sorted(placeholder_words(ptxt)):
            report("error", f"prompt {pth}: unfilled <{m}> in a command — the agent cannot run a placeholder")
        if base_sha and names_base and base_sha[:7] not in ptxt:
            report("error", f"prompt {pth}: check {', '.join(names_base)} names the base, but the prompt "
                   f"never gives its sha ({base_sha[:12]})")
        if wt and wt not in ptxt:
            report("error", f"prompt {pth}: never names the worktree as an absolute path ({wt}) — "
                   "the agent starts wherever the orchestrator's shell last was")
    if base_sha:
        cited_files([("goal", goal), ("notes", notes)] + [(f"prompt {p}", t) for p, t in zip(prompts, extra)],
                    scope, base_sha)
    if per_task:
        print(f"per-task only, never registered: {', '.join(per_task)}")
    pinned = sorted({f for _, e in checks for f, _ in CITED.findall(e.get("cmd", ""))
                     if TESTISH.search(f) and not G.in_scope(f, scope)})
    if pinned:
        # Information, not a warning: running a regression file unchanged is
        # usually the point. It is printed because one such file asserted that
        # reader prompt `v3` raises, and the task was to add v3 (slm readerv3).
        print(f"runs unchanged (outside scope.paths): {', '.join(pinned)} — a test there that pins "
              "what the goal changes fails, and the agent may not edit it")
    if do_measure and checks:
        measure(checks, base, timeout, scope, notes)
    return finish(f"{tid or path}: {len(entries)} acceptance entr{'y' if len(entries) == 1 else 'ies'}")


# ── memory rules

def block_headline(p):
    """The block's own headline: a `headline:` field (frontmatter, or a
    `**Headline:**` line), else its `# ` title — unless that is only its id."""
    txt = p.read_text(errors="replace")
    m = re.search(r"(?im)^[ \t*_-]*headline[ \t*_]*:[ \t*_]*(.+?)[ \t]*$", txt)
    if m:
        return unquote(m.group(1))
    m = re.search(r"^#[ \t]+(.+?)[ \t]*$", txt, re.M)
    return m.group(1) if m and m.group(1) != p.stem else None


def norm(s):
    return " ".join((s or "").replace("**", "").split()).strip("\"'` ").rstrip(".")


def flat(s):
    return " ".join(s.replace("`", "").split())


# A claim cites its source inline: [src: <path>:<line>[-<line>] "<verbatim excerpt>"],
# or [src: chk_<id> "<excerpt>"] for a registered value. The excerpt is the check.
SRC = re.compile(r'\[src:\s*([^\s\]"`]+)\s+(?:"((?:[^"\\\n]|\\.)+)"|`([^`\n]+)`)\s*\]')
ABBREV = re.compile(r"\b(e\.g|i\.e|etc|vs|cf|approx|incl|resp|fig|no)\.$", re.I)


def source_text(ref, task, regs):
    """(text, where) for what a [src:] names — the cited lines ±2 — or
    (None, why it cannot be read). A path is looked for in the task's worktree,
    then its branch, then the checkout, then HEAD: a block is written before
    its task merges."""
    if re.fullmatch(r"chk_[\w.-]+", ref):
        e = regs.get(ref)
        return (" ".join(str(v) for v in e.values()), ref) if e else (None, f"{ref} is not in .orch/registry/")
    m = re.fullmatch(r"(.+?):(\d+)(?:-(\d+))?", ref)
    if not m:
        return None, f"{ref} — cite <path>:<line>, <path>:<line>-<line>, or chk_<id>"
    path, a = m.group(1), int(m.group(2))
    b = int(m.group(3) or a)
    if Path(path).is_absolute():
        try:
            path = str(Path(path).resolve().relative_to(ROOT.resolve()))
        except ValueError:
            return None, f"{path} is outside the repo"
    txt = None
    if task and (ROOT / ".work" / task / path).is_file():
        txt = (ROOT / ".work" / task / path).read_text(errors="replace")
    for spec in ([f"orch/{task}:{path}"] if task else []) + [None, f"HEAD:{path}"]:
        if txt is not None:
            break
        if spec is None:
            txt = (ROOT / path).read_text(errors="replace") if (ROOT / path).is_file() else None
        else:
            cp = git("show", spec)
            txt = cp.stdout if cp.returncode == 0 else None
    if txt is None:
        return None, f"{path} cannot be read (task worktree or branch, checkout, HEAD)"
    return "\n".join(txt.splitlines()[max(0, a - 3):b + 2]), f"{path}:{a}" + (f"-{b}" if b != a else "")


def sentences(body):
    """The claim sentences of a block body, each citation masked as
    \\x01<n>\\x03 — n its index among the body's citations, so a sentence can
    be checked against what it cites. Frontmatter, headings, code fences,
    tables, `key: value` lines and a Sources/References section are not claims;
    a fragment under three words is not a sentence."""
    keep, fence, refs, k = [], False, False, iter(range(10 ** 6))
    for line in SRC.sub(lambda m: f"\x01{next(k)}\x03", body).splitlines():
        s = line.strip()
        if s.startswith("```"):
            fence = not fence
            continue
        if not fence and s.startswith("#"):
            refs = bool(re.match(r"#+\s*\**(sources?|references?|evidence|provenance)\b", s, re.I))
        if (fence or refs or s.startswith(("#", "<!--", "|"))
                or re.match(r"[-*\s]*\*\*[^*\n]{1,40}:\*\*|[A-Za-z_][\w-]*:(\s|$)", s) and "\x01" not in s):
            s = ""
        keep.append(s)
    out = []
    for para in re.split(r"\n\s*\n|\n(?=(?:[-*+]|\d+\.)\s)", "\n".join(keep)):
        p = re.sub(r"`[^`]*`", lambda m: m.group(0).replace(".", "\x02"), " ".join(para.split()))
        parts = []
        for s in re.split(r"(?<=[.!?])\s+(?=[A-Z\x01`(\[*_\"])", p):
            if parts and ABBREV.search(parts[-1]):
                parts[-1] += " " + s
            else:
                parts.append(s)
        out += [s.replace("\x02", ".") for s in parts if len(s.split()) >= 3]
    return out


def claims(p, task, regs):
    """(wrong, unread, uncited, misnumbered) for one block: citations whose
    excerpt is not at the source they name, citations that cannot be read,
    sentences citing nothing, and result-shaped numbers a sentence states that
    none of its citations shows. An archivist once wrote a workaround that
    never happened and a recommendation nobody made, and the lint passed both —
    it checked structure, not truth. An invented sentence has no line to quote;
    a restated number has one, and it can be compared."""
    body = re.sub(r"\A---\n.*?\n---\n?", "", p.read_text(errors="replace"), flags=re.S)
    m = re.search(r"task:\s*[\"']?((?:tsk|msr)_[\w.-]+)", p.read_text(errors="replace"))
    task = task or (m.group(1) if m else None)
    wrong, unread, texts = [], [], {}
    for i, c in enumerate(SRC.finditer(body)):
        ref, ex = c.group(1), flat(re.sub(r"\\(.)", r"\1", c.group(2) or c.group(3) or ""))
        if len(ex) < 12:
            wrong.append(f"[src: {ref} \"{ex}\"] — an excerpt this short proves nothing; quote a phrase")
            continue
        src, where = source_text(ref, task, regs)
        texts[i] = src
        if src is None:
            unread.append(where)
        elif ex not in flat(src):
            wrong.append(f"[src: {ref} \"{ex[:60]}\"] — not at {where} (±2 lines)")
    sents, misnumbered = sentences(body), []
    for s in sents:
        shown = [texts[int(i)] for i in re.findall(r"\x01(\d+)\x03", s) if texts.get(int(i))]
        if not shown:
            continue                       # uncited, or its sources are unreadable: reported above
        have = {abs(num(x)) for t in shown for x in NUMS.findall(t)}
        bare = re.sub(r"\s+([.,;:!?])", r"\1", " ".join(re.sub(r"\x01\d+\x03", " ", s).split()))
        for r in RESULTISH.finditer(bare):
            if any(abs(num(x)) not in have for x in NUMS.findall(r.group(0))):
                misnumbered.append(f"\"{bare[:70]}\" states {r.group(0).strip()}, which nothing it cites shows")
                break
    return wrong, unread, [s for s in sents if "\x01" not in s], misnumbered


def lint_memory(task):
    mem, rdir = ROOT / ".orch/memory", ROOT / ".orch/memory/receipts"
    idx = mem / "INDEX.md"
    if not idx.exists():
        die(f"no index at {idx}")
    rows = []
    for line in idx.read_text().splitlines():
        if line.startswith("blk_"):
            parts = [x.strip() for x in line.split(" · ", 3)]
            rows.append((parts[0], parts[3] if len(parts) == 4 else None))
    ids = [b for b, _ in rows]
    files = {p.stem: p for p in (mem / "blocks").glob("blk_*.md")}
    archived = {p.stem for p in (mem / "archive").rglob("blk_*.md")} if (mem / "archive").is_dir() else set()
    stray = {p.stem: p for p in mem.rglob("blk_*.md")
             if p.parent != mem / "blocks" and "archive" not in p.relative_to(mem).parts}

    # With --task, content findings are reported for the blocks this task's
    # receipt names; the rest are counted as backlog. Structure stays global:
    # a duplicate or an orphan anywhere breaks the count the report quotes.
    ops, touched = [], None
    if task:
        r = rdir / f"{task}.log"
        if not r.exists():
            report("error", f"no receipt for {task} at {r.relative_to(ROOT)} — receipt first, report second")
        else:
            for line in r.read_text().splitlines():
                m = re.match(r"\S+\s+(new|update|supersede|discard)\s+(blk_\S+)(?:\s+->\s+(blk_\S+))?", line)
                if m:
                    ops.append(m.groups())
        touched = {x for op in ops for x in op[1:] if x}
    backlog = 0

    def content(b, msg):
        nonlocal backlog
        if touched is None or b in touched:
            report("error", msg)
        else:
            backlog += 1

    for p in stray.values():
        report("error", f"{p.relative_to(ROOT)}: a block outside blocks/ — move it to .orch/memory/blocks/")
    for b in sorted({b for b in ids if ids.count(b) > 1}):
        report("error", f"{b}: {ids.count(b)} INDEX lines — one block, one line")
    matched = 0
    for b, head in dict(rows).items():
        if head is None:
            report("error", f"{b}: INDEX line is not `blk_<id> · <project> · <type> · <headline>`")
            continue
        if b not in files:
            where = ("it is archived — archived blocks leave the index" if b in archived else
                     f"its file is at {stray[b].relative_to(ROOT)}" if b in stray else "no such file")
            report("error", f"{b}: indexed, but not in blocks/ — {where}")
            continue
        bh = block_headline(files[b])
        if bh is None:
            content(b, f"{b}: the block has no headline of its own (a `headline:` field, or a "
                    "`# ` title that is not just the id) — its index line cannot be checked")
        elif norm(bh) != norm(head):
            content(b, f"{b}: INDEX headline is not the block's\n         index: {head}\n         block: {bh}")
        else:
            matched += 1
    for b in sorted(set(files) - set(ids)):
        report("error", f"{b}: in blocks/ with no INDEX line")
    regs = {e.get("id"): e for f in sorted((ROOT / ".orch/registry").glob("*.yaml"))
            for e in registry_entries(f.read_text())}
    uncited_blocks, unread_n = [], 0
    for b, p in sorted(files.items()):
        t = p.read_text(errors="replace")
        if not re.search(r"(?im)^[ \t*_#-]*sources?[ \t*_]*(:|$)", t):
            content(b, f"{b}: no source — no source, no block")
        mine = touched is not None and b in touched
        wrong, unread, uncited, misnumbered = claims(p, task if mine else None, regs)
        for w in wrong:
            content(b, f"{b}: {w}")
        if mine:
            for u in unread:
                report("error", f"{b}: [src:] {u} — cite what this task can show")
            for x in misnumbered:
                report("error", f"{b}: {x} — a restated number is checked against what it cites")
            # A result already lives in a tracked doc, linted against the
            # registry. A fact block that only re-cites that doc duplicates it;
            # blocks earn their place as procedures, failures, conventions and
            # gotchas (slm phase 16: 10 of 16 new blocks restated one doc).
            cited = {re.sub(r":\d+(?:-\d+)?$", "", c.group(1)) for c in SRC.finditer(t)}
            kind = re.search(r"(?im)^[ \t*_-]*type[ \t*_]*:[ \t*_]*([\w-]+)", t)
            if (len(cited) == 1 and next(iter(cited)).endswith(".md") and kind
                    and kind.group(1).lower() in ("fact", "decision")):
                report("warn", f"{b}: every claim cites {next(iter(cited))} — a block that restates a tracked "
                       "doc duplicates it; the doc is the record")
            for s in uncited[:3]:
                report("error", f"{b}: a claim with no [src: <path>:<line> \"<excerpt>\"] — \"{s[:80]}\"")
            if len(uncited) > 3:
                report("error", f"{b}: … and {len(uncited) - 3} more uncited claim(s)")
        else:
            unread_n += len(unread)
            if uncited:
                uncited_blocks.append(b)
        if re.search(r"(?im)^[ \t*_-]*superseded[ _]by[ \t*_]*:", t):
            content(b, f"{b}: marked superseded but still live — move it to .orch/memory/archive/ "
                    "and drop its INDEX line")
    for op, b, succ in ops:
        for x in ([succ] if op == "supersede" else [b] if op != "discard" else []):
            if not x or x not in files or x not in ids:
                report("error", f"receipt: {op} {x or b} — but it is not live (in blocks/ and the index)"
                       + ("; a supersede is written `supersede blk_<old> -> blk_<new>`" if not x else ""))
        if op == "supersede" and (b in files or b in ids or b not in archived):
            report("error", f"receipt: {b} superseded — but it is not in archive/ and out of the index")
    if backlog:
        report("warn", f"{backlog} finding(s) on blocks this task did not touch — run without --task to list them")
    bare = [b for b, p in sorted(files.items()) if not p.read_text(errors="replace").startswith("---")]
    if bare:
        report("warn", f"{len(bare)} block(s) without frontmatter, e.g. {', '.join(bare[:3])}: their "
               "links (supersedes, parent) are not machine-readable, so auto-attach cannot follow them")
    if uncited_blocks:
        report("warn", f"{len(uncited_blocks)} block(s) state claims with no [src:], e.g. "
               f"{', '.join(uncited_blocks[:3])} — older than cited claims, or invented; cite them when next touched")
    if unread_n:
        report("warn", f"{unread_n} [src:] citation(s) on untouched blocks cannot be read here (a pruned "
               "worktree, a deleted branch): unverifiable now, not wrong")
    for p in sorted(rdir.glob("*")) if rdir.is_dir() else []:
        if p.name != ".gitkeep" and not re.fullmatch(r"(tsk|msr)_[\w.-]+\.log", p.name):
            report("warn", f"{p.relative_to(ROOT)}: a receipt is named <task_id>.log or <msr_id>.log")
    print(f"index: {len(ids)} lines · blocks/: {len(files)} files · archive/: {len(archived)} · "
          f"headlines matched: {matched}/{len(set(ids))}")
    return finish("memory")


# ── doc rules

CITE = re.compile(r"\[(chk_[\w.-]+)\]")
NUM_BEFORE = re.compile(r"([+\-−]?\d[\d,]*(?:\.\d+)?)\s*(?:%|pp)?\s*$")
NUMS = re.compile(r"[+\-−]?\d[\d,]*(?:\.\d+)?")
RESULTISH = re.compile(
    r"(?<![\w.+\-−])[+\-−]\d+(?:\.\d+)?(?!\w|\.\d)"  # signed: +40, −31
    r"|(?<![\w.])\d+(?:\.\d+)?\s?%"                    # a percentage
    r"|(?<![\w.])\d+\s*(?:/|of)\s*\d+(?!\w|\.\d)"     # a ratio: 19 of 20, 3/50
    r"|\bp\s*[=<≤]?\s*0?\.\d+")                        # a p-value


def num(s):
    return float(s.replace("−", "-").replace(",", "").lstrip("+"))


def lint_doc(paths, strict):
    """A number in a results doc is evidence only if a registered command
    reproduces it. The session that wrote this rule reported two wrong
    conclusions, both from an inline script that re-implemented part of the
    scorer; neither number could be replayed, because neither was registered."""
    entries = {}
    for f in sorted((ROOT / ".orch/registry").glob("*.yaml")):
        for e in registry_entries(f.read_text()):
            if e.get("id") in entries:
                report("error", f"{e['id']}: defined twice in .orch/registry/")
            entries[e.get("id")] = e
    for path in paths:
        try:
            txt = Path(path).read_text()
        except OSError as e:
            die(str(e))
        cited, loose, fence = 0, [], False
        for ln, line in enumerate(txt.splitlines(), 1):
            if line.lstrip().startswith("```"):
                fence = not fence
                continue
            if fence:
                continue
            cites = list(CITE.finditer(line))
            for c in cites:
                cid, at, cited = c.group(1), f"{path}:{ln}", cited + 1
                e = entries.get(cid)
                if e is None:
                    report("error", f"{at}: [{cid}] is not in .orch/registry/")
                    continue
                st = e.get("status", "active")
                if st == "superseded":
                    report("error", f"{at}: [{cid}] is superseded ({e.get('superseded_why', 'no reason given')}) "
                           f"— cite {e.get('superseded_by', 'its replacement')}")
                elif st != "active":
                    report("warn", f"{at}: [{cid}] is {st} — no sweep replays it")
                if e.get("post_hoc") and not re.search(r"post[- ]?hoc", line, re.I):
                    report("warn", f"{at}: [{cid}] was registered post hoc ({e['post_hoc']}) — say so on "
                           "the line that cites it")
                m = NUM_BEFORE.search(line[:c.start()])
                if not m:
                    continue                # cites the check as a whole, not a value
                v = re.fullmatch(r"stdout\s*==\s*(.*)", e.get("expect", "").strip(), re.S)
                if not v:
                    report("warn", f"{at}: {m.group(1)} [{cid}] — its expect ({e.get('expect')}) holds "
                           "no value to compare; register the value the doc cites")
                elif num(m.group(1)) not in [num(x) for x in NUMS.findall(v.group(1))]:
                    report("error", f"{at}: {m.group(1)} [{cid}] — the registered value is {v.group(1).strip()}")
            if not cites:
                loose += [f"{ln}: {m.group(0).strip()}" for m in RESULTISH.finditer(line)]
        if loose:
            report("error" if strict else "warn",
                   f"{path}: {len(loose)} result-shaped number(s) on lines that cite no [chk_…] — "
                   + "; ".join(loose[:6]) + (" …" if len(loose) > 6 else ""))
        print(f"{path}: {cited} citation(s), {len(loose)} uncited result-shaped number(s)")
    return finish("doc")


def finish(what):
    for level, msg in sorted(found, key=lambda f: f[0] != "error"):
        print(f"{level:6} {msg}")
    errs = sum(1 for lv, _ in found if lv == "error")
    print(f"{what}: {errs} error(s), {len(found) - errs} warning(s)")
    return 1 if errs else 0


def main():
    a = sys.argv[1:]
    usage = ("usage: orch-lint.py packet <packet.yaml> [--measure] [--timeout S] [--prompt <file>]... "
             "| memory [--task <tsk_id|msr_id>] | doc <results.md>... [--strict]")
    if not a or a[0] not in ("packet", "memory", "doc"):
        die(usage)
    opts, pos, i = {"--prompt": []}, [], 1
    while i < len(a):
        if a[i] in ("--prompt", "--task", "--timeout"):
            if i + 1 >= len(a):
                die(f"{a[i]} needs a value")
            if a[i] == "--prompt":
                opts["--prompt"].append(a[i + 1])
            else:
                opts[a[i]] = a[i + 1]
            i += 2
        elif a[i] in ("--measure", "--strict"):
            opts[a[i]], i = True, i + 1
        else:
            pos, i = pos + [a[i]], i + 1
    if a[0] == "memory":
        return lint_memory(opts.get("--task"))
    if a[0] == "doc":
        return lint_doc(pos, opts.get("--strict", False)) if pos else die(usage)
    if len(pos) != 1:
        die(usage)
    return lint_packet(pos[0], opts["--prompt"], opts.get("--measure", False),
                       float(opts.get("--timeout", 60)))


if __name__ == "__main__":
    try:
        rc = main()
    except SystemExit:
        raise
    except Exception as e:                # a crash is "could not look", never 1
        die(f"internal error: {type(e).__name__}: {e}")
    sys.exit(rc)
```
````

#### The task bookkeeping, generated

A 4-line fix took as many steps as a 5-file feature — packet, lint, prompt,
worktree, dispatch, verification, registry entry, trace — and the last three
were typed by hand: registry entries appended through shell heredocs, regex
brackets inside YAML strings inside the heredoc, and traces whose `loop_mode` and
`orchestrator_error.kind` were present in some and missing in others. Nothing
validated either at write time (slm-memory-management, 2026-10-03).

`orch-task.py` generates them. `prompt` renders the dispatch prompt from the
packet, so the base sha and the absolute worktree path are in it by
construction. `verify` is §5 as one command, and it is the source of truth for
the rest: `register` refuses without a clean verify record and reads its own
write back with the parser everything else uses; `trace` writes every schema
key, and refuses a `done` that no clean verify backs. What stays the
orchestrator's: the mutation check, and every judgment a finding asks for.

v5 (slm-memory-management, 2026-10-04/05). A two-file tool still took about
fourteen coordinator steps, so the rest went into code: `verify --commit`
stages the agent's files by path and commits them with the trailers, and
`finish` does register, trace, packet, prune and the merge check in one
command after a clean verdict. The merge itself stays a visible command of its
own. And there was no waiver: §7 said an approved checkpoint is "waived once",
but `verify` re-derived `needs_decision` from the scan on every run and
`register` took only a clean `done`, so two verified, mutation-tested tasks
merged with no registry entries at all. Now `verify` writes the checkpoint
itself, one finding per line, and `--waive` takes a resolved checkpoint and
downgrades exactly what it lists. The waiver goes into every registry entry,
so a sweep can find it again.

v6 (2026-10-06/07). `verify` reads the agent's report where the report hook
recorded it, and says whose it is; a file the orchestrator supplies is
accepted and labelled as its own, on the verify record and the trace. `prompt`
renders the commit line with the task's trailers and the report object with
its task id filled in — executors left out the trailer until a dispatch said
exactly how. A plan's approval is one record (`.orch/approvals/apr_…`, the
user's words) that packets cite by id, and the packets a plan approval never
covers are refused in code. `register --eval` refuses a conclusion whose data
predates its pre-registration unless `--post-hoc` says why, and takes
`--capture` and `--conclusion`.

````
FILE: .claude/hooks/orch-task.py
```python
#!/usr/bin/env python3
"""ORCH task tool — the mechanical half of the orch-task skill, as code.

The skill keeps the judgment: what the packet says, what to stage, whether a
finding is a checkpoint. This keeps what used to be typed by hand and came out
different every time: the dispatch prompt (one said `<base>` and never gave
the sha), the verification run, the registry entry (a regex inside a YAML
string inside a heredoc) and the trace (`loop_mode` and
`orchestrator_error.kind` present in some traces, missing in others).
Generated, each is the same shape every time and records what code observed,
not what the session remembers.

    python3 .claude/hooks/orch-task.py prompt   <tsk_id>
    python3 .claude/hooks/orch-task.py verify   <tsk_id> [--commit "<summary>"] [--test "<cmd>"] [--result <file>]
                                       [--waive <ckp_id>]... [--timeout S]
    python3 .claude/hooks/orch-task.py register <tsk_id>
    python3 .claude/hooks/orch-task.py register --eval --msr <msr_id> --cmd "<cmd>"
                                       --expect "stdout == V" | --expect-file <file> | --capture
                                       [--artifact <path>] [--noise "<text>"] [--conclusion "<text>"]
                                       [--post-hoc "<why>"] [--project <name>]
    python3 .claude/hooks/orch-task.py trace    <tsk_id> --outcome <o> --error none|<kind> [--what ..] [...]
    python3 .claude/hooks/orch-task.py finish   <tsk_id> --error none|<kind> [--what ..] [--keep] [trace options]

prompt    renders the dispatch prompt from the packet to .work/_prompts/<id>.md
          — absolute worktree path, base sha, goal, every check, scope,
          forbidden, the text of its context blocks, the commit line with the
          task's trailers, and the report object with its task_id filled in —
          and lints it with the packet. Refuses a packet with no `approved:`,
          an approval record that quotes no one, and a plan approval stretched
          over what a plan never covers (orch-task §0 step 3). Append
          task-specific prose if you must; then lint again.
verify    orch-task §5 steps 2-6. --commit first stages the agent's files by
          path (uncommitted, in scope, not generated) and commits them with the
          Orch-Task (and Orch-Retires) trailer. Then: every executable check
          re-run in the worktree (a delta one at base too), the task's changed
          tests run at base (want: fail), writes outside scope.paths whether
          committed or not, the trailer, the sensitive scan, and the agent's
          report read for its confidence and the blocks its evidence cites —
          the one orch-report.py recorded as the agent sent it, or a --result
          file, which is recorded as the orchestrator's.
          A needs_decision verdict writes its checkpoint, one finding per line;
          --waive <ckp_id> names a RESOLVED checkpoint (approve or false
          positive, in the user's words) and downgrades exactly the findings it
          lists. Writes .work/_verify/<id>.json for register, trace and finish.
register  appends a verified task's passing, registrable checks — or one eval
          entry, replayed once now — to .orch/registry/<project>.yaml under a
          fresh id, reads the file back with the parser every other tool uses,
          and restores it if any entry did not survive the round trip. A waiver
          is carried into each entry. An eval entry records inputs git does not
          hold (`local:`) and the GPU/wall time of the measurement's stages.
          It refuses one whose data is older than its pre-registration, or
          that has none, unless --post-hoc says why. --capture registers what
          the scorer prints now.
trace     writes .orch/traces/trc_<id>.json with every schema key present and
          every enum checked; `verification` is copied from the verify record,
          and a `done` trace without a clean one is refused.
finish    the fast lane after a clean verify: register, trace (outcome done),
          the packet to .orch/queue/done/, the worktree pruned (kept while the
          archivist still needs it), and the merge checked with merge-tree.
          It prints the merge line and never runs it: a merge is visible to the
          harness's permission check only when it is its own command.

exit 0 ok · 1 findings (verify) or refused (register, trace, finish) · 2 could not run
"""
import hashlib
import importlib.util
import json
import re
import shlex
import shutil
import subprocess
import sys
from datetime import date, datetime, timezone
from pathlib import Path


def die(msg, code=2):
    sys.stderr.write(f"orch-task: {msg}\n")
    sys.exit(code)


def _lint():
    """orch-lint.py, which carries the guard: one parser, one matcher, one
    `holds`, so this cannot verify by rules the lint does not check by."""
    p = Path(__file__).resolve().with_name("orch-lint.py")
    sys.dont_write_bytecode = True
    spec = importlib.util.spec_from_file_location("orch_lint", p)
    m = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(m)
    return m


try:
    L = _lint()
except SystemExit:
    raise
except Exception as e:
    die(f"cannot load orch-lint.py: {e}")
G, ROOT = L.G, L.ROOT
GENERATED = re.compile(r"(^|/)(__pycache__|\.pytest_cache|\.mypy_cache|\.ruff_cache|\.tox|node_modules"
                       r"|\.pio|build|dist|[^/]*\.egg-info)(/|$)|\.py[co]$")
SCRIPTS = {".py", ".sh", ".bash", ".js", ".mjs", ".ts", ".rb", ".pl", ".r", ".jl", ".lua"}
OUTCOMES = ("done", "blocked", "needs_decision", "failed", "escalate")
KINDS = {"weak_acceptance": "a check that could not fail, or asserted the wrong thing",
         "wrong_scope": "scope.paths excluded what the goal needed, or allowed too much",
         "bad_reference": "code, docs or a block you handed the agent was wrong",
         "unsound_plan": "the packet's approach could not have worked",
         "budget": "the budget was wrong for the work"}
STAGES = ROOT / ".work/_stages"


def now():
    return datetime.now(timezone.utc).isoformat(timespec="seconds")


def git(*args, cwd=None):
    return subprocess.run(["git", "-C", str(cwd or ROOT), *args],
                          capture_output=True, text=True, errors="replace")


def rev(spec, cwd=None):
    cp = git("rev-parse", "--verify", "-q", f"{spec}^{{commit}}", cwd=cwd)
    return cp.stdout.strip() if cp.returncode == 0 else None


def listval(txt, key):
    """`key: [a, b]` or a block list under `key:`."""
    m = re.search(rf"^\s*{key}\s*:\s*\[(.*?)\]", txt, re.M | re.S)
    if m:
        return [x.strip().strip("\"'") for x in m.group(1).split(",") if x.strip()]
    m = re.search(rf"^\s*{key}\s*:\s*(#.*)?\n((?:\s+-\s*.+\n?)+)", txt, re.M)
    return [L.unquote(x) for x in re.findall(r"^\s+-\s*(.+?)\s*$", m.group(2), re.M)] if m else []


def packet(tid):
    if not re.fullmatch(r"tsk_[\w.-]+", tid or ""):
        die(f"{tid!r} is not a task id")
    for rel in (f".orch/queue/{tid}.yaml", f".orch/queue/done/{tid}.yaml"):
        if (ROOT / rel).exists():
            p = ROOT / rel
            break
    else:
        die(f"no packet for {tid} in .orch/queue/ or .orch/queue/done/")
    txt = p.read_text()
    acc = L.acceptance(txt)
    if not acc:
        die(f"{p}: no acceptance this tool can read")
    s = lambda k: L.scalar(txt, k) or ""
    net = re.search(r"network\s*:\s*(true|false)", txt)
    return {"path": p, "txt": txt, "id": tid, "project": s("project"), "goal": s("goal"),
            "base": s("base") or "HEAD", "role": s("role") or "executor", "phase": s("phase"),
            "playbook": s("playbook"), "notes": s("notes"), "scope": G.scope_paths(txt),
            "acceptance": acc, "refs": listval(txt, "context_refs"), "retires": listval(txt, "retires"),
            "approved": s("approved"), "forbidden": listval(txt, "forbidden"),
            "network": bool(net and net.group(1) == "true")}


def per_task(e, sha, spec):
    """delta, or names the base: true at one commit only, never registered.
    The same rule orch-lint.py packet prints as `per-task only`."""
    cmd, expect = e.get("cmd", ""), e.get("expect", "")
    return bool(re.match(r"delta\b", expect.strip()) or sha[:7] in cmd
                or (spec != "HEAD" and spec in cmd) or re.search(r"\$\{?base\b|<base", cmd))


# ── prompt

def block_text(b):
    for d in ("blocks", "archive"):
        f = ROOT / ".orch/memory" / d / f"{b}.md"
        if f.exists():
            t = f.read_text(errors="replace")
            m = re.match(r"---\n(.*?)\n---\n?(.*)$", t, re.S)
            meta, body = (m.group(1), m.group(2)) if m else ("", t)
            kind = re.search(r"^type:\s*(.+)$", meta, re.M) or re.search(r"\*\*Type:\*\*\s*(.+)", t)
            head = L.block_headline(f) or b
            note = " — ARCHIVED: name its successor instead" if d == "archive" else ""
            return f"[{b}] ({kind.group(1).strip() if kind else '?'}) {head}{note}\n\n{body.strip()}"
    return f"[{b}] — not found in .orch/memory/"


def policy_lists():
    """path_globs and dependency_manifests from sensitive.yaml — the gate a
    plan approval may not pass over."""
    p = ROOT / ".orch/config/sensitive.yaml"
    txt = p.read_text() if p.exists() else ""
    globs = re.findall(r'^\s*-\s*"([^"]+)"\s*$', txt.split("content_patterns:")[0], re.M)
    m = re.search(r"^dependency_manifests\s*:\s*\[(.*)\]", txt, re.M)
    return globs, ([x.strip().strip("\"'") for x in m.group(1).split(",") if x.strip()] if m else [])


def plan_exceptions(pk):
    """What a plan approval never covers (orch-task §0 step 3), as code: it
    covered the goals the plan named, not the risks a packet adds later."""
    from fnmatch import fnmatch
    out = []
    if pk["retires"]:
        out.append(f"it retires {', '.join(pk['retires'])}")
    globs, manifests = policy_lists()
    for sp in pk["scope"]:
        g = next((g for g in globs if fnmatch(sp, g) or fnmatch(sp, g.replace("**/", "", 1))), None)
        if g:
            out.append(f"scope path {sp} matches the sensitive path_glob {g}")
        elif any(fnmatch(Path(sp).name, m) if "/" not in m else fnmatch(sp, m) for m in manifests):
            out.append(f"scope path {sp} is a dependency manifest")
    vf = ROOT / ".work/_verify" / f"{pk['id']}.json"
    if vf.exists():
        rec = json.loads(vf.read_text())
        then = [(c.get("cmd"), c.get("expect")) for c in rec.get("checks", []) if c.get("type") == "executable"]
        now_ = [(e.get("cmd"), e.get("expect")) for e in pk["acceptance"] if e.get("type", "executable") != "judged"]
        if rec.get("verdict") == "failed" and then != now_:
            out.append("its acceptance changed after a failed verify")
    return out


def approval(pk):
    """`approved:` is "shown", "plan: <line>", or the id of an approval record
    .orch/approvals/apr_<id>.md holding the user's words. A record is written
    once per plan, so packets the plan covers carry its id instead of each
    retyping a quotation (slm phase 16: 19 packets, one plan, two approvals)."""
    a, name = pk["approved"].strip(), pk["path"].name
    if not a:
        die(f"{name}: `approved:` is empty — before dispatch, record how the user approved this packet: "
            "\"shown\" (shown and confirmed), or the approval record of the plan that covers it, "
            "\"apr_<id>\" (orch-task §0 step 3)", 1)
    if a == "shown":
        return
    if re.fullmatch(r"apr_[\w.-]+", a):
        f = ROOT / ".orch/approvals" / f"{a}.md"
        if not f.exists():
            die(f"{name}: approved: {a}, but there is no .orch/approvals/{a}.md", 1)
        if not re.search(r"\"[^\"]+\"|“[^”]+”", f.read_text(errors="replace")):
            die(f".orch/approvals/{a}.md quotes nothing the user said — an approval record is their words", 1)
    elif not a.startswith("plan:"):
        die(f"{name}: approved: {a!r} — \"shown\", \"apr_<id>\", or \"plan: <the plan line>\"", 1)
    why = plan_exceptions(pk)
    if why:
        die(f"{name}: a plan approval does not cover this packet — {'; '.join(why)}. Show it to the user, "
            "and record approved: \"shown\" once they confirm it", 1)


def cmd_prompt(tid):
    pk = packet(tid)
    wt, sha = ROOT / ".work" / tid, rev(pk["base"])
    if not sha:
        die(f"base {pk['base']} does not resolve to a commit")
    approval(pk)
    if not (wt / ".git").exists():
        sys.stderr.write(f"orch-task: no worktree at {wt} yet — create it before dispatch (§3)\n")
    out = [f"# {tid} — {pk['phase'] or 'task'} ({pk['role']})", "",
           f"Work only inside {wt}. It is a throwaway git worktree on branch orch/{tid}, branched "
           f"from {sha}. This path overrides the session's primary working directory: run `pwd` "
           "first, and `cd` there if it differs.", "", "## Goal", "", pk["goal"], "",
           "## Acceptance — the orchestrator re-runs these after you finish", "", "```bash"]
    judged = []
    for n, e in enumerate(pk["acceptance"], 1):
        if e.get("type", "executable") == "judged":
            judged.append(f"{n}. (judged) {e.get('criterion', '')}")
        else:
            out += [f"# {n} — expect: {e.get('expect', '')}", e.get("cmd", "")]
    out += ["```", ""] + judged + ([""] if judged else []) + [
        "If a check looks wrong, FLAG it in `open_questions` with the evidence — do not edit code "
        "solely to make it pass.", "", "## Scope — the only paths you may write", ""]
    out += [f"- {p}" for p in pk["scope"]] + ["", f"Network: {'allowed' if pk['network'] else 'none'}."]
    if pk["forbidden"]:
        out += ["", "## Forbidden", ""] + [f"- {x}" for x in pk["forbidden"]]
    seeds = re.search(r"seed_caches\s*:\s*\[(.*?)\]", (ROOT / ".orch/config/settings.yaml").read_text()
                      if (ROOT / ".orch/config/settings.yaml").exists() else "")
    names = [x.strip() for x in seeds.group(1).split(",")] if seeds else []
    have = [n for n in names if (wt / n).exists()]
    out += ["", (f"Dependency caches seeded from the main checkout: {', '.join(have)}. " if have else "")
            + "Never fetch dependencies yourself."]
    if pk["refs"]:
        out += ["", "## Context — memory blocks", ""]
        for b in pk["refs"]:
            out += [block_text(b), ""]
    if pk["notes"]:
        out += ["", "## Notes", "", pk["notes"]]
    trailers = f'--trailer "Orch-Task: {tid}"' + (f' --trailer "Orch-Retires: {", ".join(pk["retires"])}"'
                                                   if pk["retires"] else "")
    out += ["", "## Commit", "", "Commit your work with exactly these trailers — they are how bisect names this "
            "task later:", "", f'    git commit -m "<one line>" {trailers}', "",
            "## Report", "", "Your report is ONE JSON object, and code reads it. If this run has the "
            "SubagentHandback tool, the object is that call's `message`; otherwise it is your final message. "
            "Nothing else in it: no prose before or after, no code fence — your prose goes in \"summary\". "
            "A hook checks it as it leaves you, and refuses a report without it.", "",
            '{"task_id": "%s", "status": "done|blocked|needs_decision|failed|escalate", "blocked_on": null, '
            '"summary": "one or two sentences, file:line for code claims", "evidence": [{"cite": "file:line", '
            '"reason": "..."}, {"check": "cmd", "exit": 0}], "confidence": 0.0, "new_facts": [], '
            '"open_questions": []}' % tid]
    f = ROOT / ".work/_prompts" / f"{tid}.md"
    f.parent.mkdir(parents=True, exist_ok=True)
    f.write_text("\n".join(out).rstrip() + "\n")
    print(f"prompt: {f}")
    return L.lint_packet(str(pk["path"]), [str(f)], False, 60)


# ── verify

def missing_hint(text, scope):
    for rx in L.MISSING:
        m = rx.search(text)
        if m:
            return (f"{m.group(1)} not found at the task's HEAD — a finished task that cannot find "
                    "a file usually means the check names the wrong one (a path, a run name)")
    line = next((x for x in text.splitlines() if L.NOT_FOUND.search(x)), None)
    return f"not-found error at HEAD ({line.strip()[:60]}) — the check may be wrong" if line else None


def dirty(wt):
    """Uncommitted, unignored paths in the worktree, generated ones dropped."""
    out = []
    for line in git("status", "--porcelain", "--untracked-files=all", cwd=wt).stdout.splitlines():
        f = line[3:].split(" -> ")[-1].strip('"')
        if f and not GENERATED.search(f):
            out.append(f)
    return out


last_json = L.last_json                   # one reader of reports: verify and orch-report.py


def captured(tid):
    """The newest report orch-report.py recorded for this task, from any agent
    but the archivist (whose report is about memory, not the task)."""
    recs = []
    for f in sorted((ROOT / ".work/_results" / tid).glob("*.json")):
        try:
            r = json.loads(f.read_text())
        except ValueError:
            continue
        if r.get("agent_type") != "orch-archivist":
            recs.append(r)
    return max(recs, key=lambda r: r.get("at") or "") if recs else None


def commit_work(pk, wt, msg, prior):
    """§5 step 2 as code. Stages the agent's files by path — uncommitted, in
    scope, not generated, not made by an earlier verify's checks — never a
    directory and never -A, and commits them with the trailers via --trailer
    (git >= 2.32): a hand-typed trailer paragraph is parsed wrongly when a line
    is indented, silently. Out-of-scope files are left for verify to flag."""
    stage = [f for f in dirty(wt) if f not in prior and G.in_scope(f, pk["scope"])]
    if not stage:
        print("commit: nothing uncommitted in scope")
        return
    cp = git("add", "--", *stage, cwd=wt)
    if cp.returncode != 0:
        die(f"git add failed: {cp.stderr.strip()}")
    trailers = ["--trailer", f"Orch-Task: {pk['id']}"]
    if pk["retires"]:
        trailers += ["--trailer", "Orch-Retires: " + ", ".join(pk["retires"])]
    cp = git("commit", "-q", "-m", msg, *trailers, cwd=wt)
    if cp.returncode != 0:
        die(f"git commit failed: {(cp.stderr or cp.stdout).strip()}")
    print(f"commit: {len(stage)} file(s) — {', '.join(stage)}")


def section(txt, name):
    m = re.search(rf"^##\s+{name}\s*$(.*?)(?=^##\s|\Z)", txt, re.M | re.S)
    return m.group(1) if m else ""


def waivers(tid, ids):
    """{ckp_id: its Trigger text} for each checkpoint --waive names. Only a
    RESOLVED one waives: in .orch/checkpoints/resolved/, about this task, its
    `## Resolution` opening with approve or false positive and quoting the
    user. A waiver is the one way a finding stops gating, so the record of the
    user's decision is required where the tool can see it — and the waiver is
    carried into every registry entry, where a sweep can find it again."""
    out = {}
    for c in ids:
        if not re.fullmatch(r"ckp_[\w.-]+", c):
            die(f"{c!r} is not a checkpoint id", 1)
        f = ROOT / ".orch/checkpoints/resolved" / f"{c}.md"
        if not f.exists():
            still = (ROOT / ".orch/checkpoints" / f"{c}.md").exists()
            die(f"{c} is not resolved — " + ("it is still open: record the user's decision under "
                "`## Resolution`, then move it to .orch/checkpoints/resolved/" if still else
                "no such file in .orch/checkpoints/resolved/"), 1)
        t = f.read_text(errors="replace")
        if not re.search(rf"^task:\s*{re.escape(tid)}\s*$", t, re.M):
            die(f"{c} is not about {tid} — it has no `task: {tid}` line", 1)
        first = next((x.strip() for x in section(t, "Resolution").splitlines()
                      if x.strip() and not x.strip().startswith("<!--")), "")
        if re.match(r"(?i)reject", first):
            die(f"{c} was rejected — a rejected task registers nothing", 1)
        if not re.match(r"(?i)\**(approved?|false[ -]positive)\b", first):
            die(f"{c}: `## Resolution` must open with the user's decision, approve or false positive — "
                f"found {first[:50]!r}", 1)
        if not re.search(r"\"[^\"]+\"|“[^”]+”", first):
            die(f"{c}: the decision line quotes nothing the user said — write it as "
                "`false positive (user, <date>, \"<their words>\")`", 1)
        out[c] = L.flat(section(t, "Trigger"))
    return out


def names(key, trig):
    """Does a checkpoint's Trigger name this finding? Its key, or the key's
    text after the kind, as a whole phrase — `a.py` is not named by `data.py`."""
    for k in (key, key.split(": ", 1)[-1]):
        if re.search(r"(?<![\w./-])" + re.escape(L.flat(k)) + r"(?![\w/-])", trig):
            return True
    return False


def write_checkpoint(tid, rec, keys):
    """The checkpoint a needs_decision verdict owes, written by the tool that
    saw the findings, one per line — so --waive matches them exactly, instead
    of against prose that wrapped a scan line in two. Same findings, same id;
    a resolved one is never rewritten."""
    cid = f"ckp_{tid[4:]}_{hashlib.sha1(chr(10).join(sorted(keys)).encode()).hexdigest()[:6]}"
    d = ROOT / ".orch/checkpoints"
    for f in (d / f"{cid}.md", d / "resolved" / f"{cid}.md"):
        if f.exists():
            return cid, f, False
    stat = git("diff", "--stat", rec["base"], rec["head"]).stdout.rstrip() or "(nothing committed)"
    body = [f"# {cid} — verify: {len(keys)} finding(s) need a decision", "class: mandatory",
            f"task: {tid}", f"raised: {now()}", "## Trigger",
            f"`orch-task.py verify` on {rec['base'][:12]}..{rec['head'][:12]} said needs_decision. One "
            f"finding per line; `--waive {cid}` downgrades exactly these, once this file is resolved:", ""]
    body += [f"- {k}" for k in keys] + [
        "", "## What the agent did", "", "```", stat, "```", "",
        "## To approve", "", "<!-- only for a command the guard refused that the user asked for: the exact "
        "`! <cmd>` line. Their running it is the approval. -->", "",
        "## Decision", "",
        f"- approve → waived ONCE: `orch-task.py verify {tid} --waive {cid}`, and each registry entry "
        f"carries `waived: {cid}`",
        "- reject  → task fails; nothing registers; worktree kept",
        "- **false positive** → waive as approve, AND a `proven` entry in .orch/upstream.md (ORCH.md §5.5): "
        "this file is its reproduction", "",
        "## Resolution", "",
        "<!-- First line: the decision in the user's words — `false positive (user, <date>, \"<what they "
        "said>\")` — then move this file to .orch/checkpoints/resolved/. -->", ""]
    d.mkdir(parents=True, exist_ok=True)
    for old in open_checkpoints(tid):      # an earlier verify's, still undecided: these findings replace it
        t = old.read_text(errors="replace")
        if " — verify: " in t.split("\n", 1)[0] and not [x for x in section(t, "Resolution").splitlines()
                                                        if x.strip() and not x.strip().startswith("<!--")]:
            old.unlink()
            print(f"note: replaced {old.stem}, an undecided checkpoint from an earlier verify")
    (d / f"{cid}.md").write_text("\n".join(body))
    return cid, d / f"{cid}.md", True


def open_checkpoints(tid):
    return [f for f in sorted((ROOT / ".orch/checkpoints").glob("*.md"))
            if re.search(rf"^task:\s*{re.escape(tid)}\s*$", f.read_text(errors="replace"), re.M)]


def cmd_verify(tid, test, result, timeout, commit=None, waive=()):
    pk = packet(tid)
    wt = ROOT / ".work" / tid
    if not (wt / ".git").exists():
        die(f"no worktree at {wt}")
    base0 = rev(pk["base"])
    if not base0:
        die(f"base {pk['base']} does not resolve to a commit")
    waived_by = waivers(tid, waive)
    vf = ROOT / ".work/_verify" / f"{tid}.json"
    prior = set(json.loads(vf.read_text()).get("byproducts", [])) if vf.exists() else set()
    if commit is not None:
        commit_work(pk, wt, commit, prior)
    head, base = rev("HEAD", wt), base0
    findings = []                          # (fail | checkpoint | fix | waived, message, key)
    add = lambda kind, msg, key=None: findings.append([kind, msg, key])
    if git("merge-base", "--is-ancestor", base, head).returncode != 0:
        base = git("merge-base", base, head).stdout.strip()
        print(f"note: base {base0[:12]} is not an ancestor of the task's HEAD (did HEAD move?); "
              f"using the fork point {base[:12]}")
    if head == base and pk["role"] == "executor":
        die("nothing is committed on top of base — commit the agent's work first (§5 step 2, or --commit)")
    rec = {"task_id": tid, "at": now(), "base": base, "head": head, "checks": [],
           "tests_at_base": None, "changed": [], "scope_violations": [], "uncommitted": [],
           "byproducts": sorted(prior), "scan": None, "result": None, "report": None, "mutation": None,
           "waived": []}

    # what changed: committed, and anything left in the tree. A Bash write is
    # invisible to the guard (up_0006); it is not invisible here.
    rec["changed"] = [x for x in git("diff", "--name-only", base, head).stdout.splitlines() if x]
    for f in rec["changed"]:
        if not G.in_scope(f, pk["scope"]):
            rec["scope_violations"].append(f)
            add("checkpoint", f"committed outside scope.paths: {f}", f"scope: {f}")
    for f in dirty(wt):
        if f in prior:                     # an earlier verify's checks made it, not the agent
            continue
        rec["uncommitted"].append(f)
        if G.in_scope(f, pk["scope"]):
            add("fix", f"in scope but not committed: {f} — commit it (--commit), or say why not")
        else:
            rec["scope_violations"].append(f)
            add("checkpoint", f"written outside scope.paths and not committed: {f} — "
                "a Bash write the guard never saw (up_0006)", f"scope: {f}")
    trailers = git("log", "--format=%(trailers:key=Orch-Task,valueonly)", f"{base}..{head}").stdout
    if head != base and tid not in trailers.split():
        add("fix", f"no `Orch-Task: {tid}` trailer on {base[:7]}..{head[:7]} — bisect "
            "cannot name this task")

    for n, e in enumerate(pk["acceptance"], 1):
        if e.get("type", "executable") == "judged":
            rec["checks"].append({"n": n, "type": "judged", "criterion": e.get("criterion", "")})
            continue
        cmd, expect = e.get("cmd", ""), e.get("expect", "")
        c = {"n": n, "type": "executable", "cmd": cmd, "expect": expect, "exit": None,
             "stdout": None, "held": None, "per_task": per_task(e, base0, pk["base"])}
        rec["checks"].append(c)
        why = G.bash_trigger(cmd)
        if why:
            add("fail", f"#{n}: the guard blocks this check ({why}) — not run")
            continue
        r = L.run(cmd, wt, timeout)
        if r is None:
            add("fail", f"#{n}: timed out after {timeout:g}s")
            continue
        c["exit"], out, err = r[0], r[1], r[2]
        c["stdout"] = out.strip()[:300]
        d = re.fullmatch(r"delta\s*==\s*([+-]?\d+)", expect.strip())
        if d:
            rb = L.at_base(cmd, base, timeout)
            if not isinstance(rb, tuple):
                add("fail", f"#{n}: delta needs the count at base, which could not be "
                    f"measured ({rb or 'timed out'})")
                continue
            c["base_stdout"] = rb[1].strip()
            try:
                c["held"] = int(out.strip()) - int(rb[1].strip()) == int(d.group(1))
            except ValueError:
                c["held"] = False
        else:
            c["held"] = L.holds(expect, c["exit"], out)
        if c["held"] is not True:
            hint = missing_hint(out + "\n" + err, pk["scope"])
            add("fail", f"#{n}: does not hold — exit {c['exit']}, stdout "
                f"\"{c['stdout'][:60]}\", expect {expect}" + (f"\n         {hint}" if hint else ""))

    if test:
        tests = [f for f in rec["changed"] if L.TESTISH.search(f)]
        rec["tests_at_base"] = {"cmd": test, "tests": tests, "exit": None}
        if G.bash_trigger(test):
            die(f"the guard blocks the test command ({G.bash_trigger(test)})")
        if not tests:
            add("fail", "--test given, but the task changed no test file: nothing can "
                "show the change is detected")
        else:
            bw = ROOT / ".work" / f"_base_{tid}"
            git("worktree", "remove", "--force", str(bw))
            cp = git("worktree", "add", "-q", "--detach", str(bw), base)
            if cp.returncode != 0:
                die(f"cannot check out base: {cp.stderr.strip()}")
            try:
                for f in tests:
                    if (wt / f).exists():
                        (bw / f).parent.mkdir(parents=True, exist_ok=True)
                        shutil.copy2(wt / f, bw / f)
                r = L.run(test, bw, timeout)
            finally:
                git("worktree", "remove", "--force", str(bw))
            rec["tests_at_base"]["exit"] = r[0] if r else None
            if r is None:
                add("fail", "the task's tests timed out at base — not measured")
            elif r[0] == 0:
                add("fail", "the task's tests PASS at base: they cannot detect the change, "
                    "and acceptance resting on them is a floor in disguise")

    rec["byproducts"] = sorted(prior | (set(dirty(wt)) - set(rec["uncommitted"])))
    sc = subprocess.run([sys.executable, str(Path(__file__).resolve().with_name("orch-scan.py")),
                         str(wt), base, "--task", tid], capture_output=True, text=True, errors="replace")
    gating = [" ".join(x.split()) for x in sc.stdout.splitlines() if re.match(r"(medium|high)\s", x)]
    rec["scan"] = {"exit": sc.returncode, "gating": gating,
                   "lines": [x for x in sc.stdout.splitlines() if x.strip()][:20]}
    if sc.returncode == 1:
        for x in gating or ["exit 1"]:
            add("checkpoint", f"scan: {x}", f"scan: {x}")
    elif sc.returncode != 0:
        add("fail", f"the scan could not look (exit {sc.returncode}): {sc.stderr.strip()[:120]}")

    # The report: what orch-report.py recorded as the agent sent it, or a file
    # the orchestrator supplies — accepted, and labelled as the orchestrator's.
    # Before v6 both were "the result file", and an orchestrator-written one
    # read exactly like an agent's (up_0013).
    obj, src = None, None
    if result:
        try:
            obj = last_json(Path(result).read_text())
        except OSError as e:
            die(str(e))
        src = {"source": "orchestrator", "file": result}
        if obj is None:
            add("fix", f"{result} ends with no result JSON (no object with a `status` key). Resume that "
                "agent by its id and ask for the JSON alone; the report hook records its reply")
    else:
        cap = captured(tid)
        if cap:
            obj = cap.get("result") if cap.get("json") else None
            src = {"source": "agent", "via": cap.get("via"), "agent_type": cap.get("agent_type"),
                   "agent_id": cap.get("agent_id"), "refusals": cap.get("refusals"), "json": bool(obj)}
            if obj is None:
                add("fix", f"the report {cap.get('agent_id')} sent has no result JSON, after "
                    f"{cap.get('refusals')} refusal(s) — resume that agent by its id and ask for the JSON "
                    "alone; the report hook records its reply")
    if obj is not None:
        cites = set(re.findall(r"blk_[\w.-]+", json.dumps(obj.get("evidence", []))))
        floor = G.load_thresholds()["confidence_floor"]
        conf = obj.get("confidence")
        rec["result"] = {"status": obj.get("status"), "confidence": conf,
                         "blocked_on": obj.get("blocked_on"),
                         "refs_cited": sorted(cites & set(pk["refs"])),
                         "refs_foreign": sorted(cites - set(pk["refs"])),
                         "open_questions": len(obj.get("open_questions") or []),
                         "new_facts": len(obj.get("new_facts") or [])}
        if obj.get("status") == "done" and isinstance(conf, (int, float)) and conf < floor:
            add("checkpoint", f"the agent says done at confidence {conf} < {floor}",
                f"confidence: {conf} < {floor}")
    rec["report"] = src

    for c, trig in waived_by.items():
        hit = [f for f in findings if f[0] == "checkpoint" and names(f[2], trig)]
        for f in hit:
            f[0], f[1] = "waived", f"{f[1]}  [waived: {c}]"
        if hit:
            rec["waived"].append(c)
        else:
            print(f"note: {c} names none of this run's findings — it waives nothing here")
    open_keys = [f[2] for f in findings if f[0] == "checkpoint"]
    fails = [m for k, m, _ in findings if k == "fail"]
    rec["verdict"] = ("failed" if fails else "needs_decision" if open_keys
                      else "incomplete" if any(k == "fix" for k, _, _ in findings) else "done")
    ckp = write_checkpoint(tid, rec, open_keys) if rec["verdict"] == "needs_decision" else None
    rec["findings"] = [{"kind": k, "msg": m} for k, m, _ in findings]
    out = ROOT / ".work/_verify" / f"{tid}.json"
    out.parent.mkdir(parents=True, exist_ok=True)
    out.write_text(json.dumps(rec, indent=2) + "\n")

    n_commits = len(git("rev-list", f"{base}..{head}").stdout.split())
    print(f"verify {tid}  base {base[:12]} -> head {head[:12]} ({n_commits} commit(s))")
    for c in rec["checks"]:
        if c["type"] == "judged":
            print(f"  #{c['n']} judged — yours to judge: {c['criterion'][:70]}")
        else:
            state = {True: "held", False: "FAILED", None: "NOT RUN"}[c["held"]]
            tag = " · per-task" if c["per_task"] else ""
            print(f"  #{c['n']} exit {c['exit']} · {state}{tag} · {c['cmd'][:70]}")
    t = rec["tests_at_base"]
    if t:
        print(f"  tests at base: exit {t['exit']} (want nonzero) · {', '.join(t['tests']) or 'none changed'}")
    print(f"  changed: {len(rec['changed'])} file(s), {len(rec['scope_violations'])} outside scope · "
          f"uncommitted: {', '.join(rec['uncommitted']) or 'none'}")
    print(f"  scan: exit {sc.returncode} — {(rec['scan']['lines'] or ['(no output)'])[-1]}")
    r, src = rec["result"], rec["report"]
    if src is None:
        print(f"  report: none recorded in .work/_results/{tid}/ — is orch-report.py wired? "
              "(`python3 - ORCH.md --check`). Pass the agent's reply with --result if you hold it")
    elif src["source"] == "orchestrator":
        print(f"  report: SUPPLIED BY THE ORCHESTRATOR ({src['file']}), not recorded from the agent — "
              "say so in the summary")
    else:
        print(f"  report: from {src['agent_type']} {src['agent_id']} via {src['via']}"
              + (f", after {src['refusals']} refusal(s)" if src.get("refusals") else ""))
    if r:
        print(f"  agent: {r['status']}, confidence {r['confidence']}, cites "
              f"{', '.join(r['refs_cited']) or 'no given block'} of {', '.join(pk['refs']) or 'none given'}"
              + (f" · {r['open_questions']} open question(s): read them, a flagged check is one"
                 if r["open_questions"] else ""))
    for k, m, _ in findings:
        print(f"{k:10} {m}")
    if ckp:
        cid, f, new = ckp
        print(f"checkpoint {'written' if new else 'exists'}: {f.relative_to(ROOT)} — present it; record the "
              f"user's decision under `## Resolution`; move it to resolved/; verify again with --waive {cid}")
    nxt = {"done": "finish (or register, then trace)", "failed": "trace --outcome failed; keep the worktree",
           "needs_decision": "the checkpoint above, before anything registers",
           "incomplete": "fix the above and verify again"}[rec["verdict"]]
    print(f"verdict: {rec['verdict']} -> {nxt}   ({out.relative_to(ROOT)})")
    return 0 if rec["verdict"] == "done" else 1


# ── register

def q(v):
    """A YAML double-quoted scalar. JSON's string escapes are YAML's, so any
    YAML parser — and registry_entries() — reads back exactly what went in."""
    return json.dumps(str(v), ensure_ascii=False)


def all_entries():
    out = {}
    for f in sorted((ROOT / ".orch/registry").glob("*.yaml")):
        for e in L.registry_entries(f.read_text()):
            if e.get("id") in out:
                die(f"{e.get('id')} is defined twice in .orch/registry/ — fix that before registering more", 1)
            out[e.get("id")] = e
    return out


def append(reg, new):
    """Insert entries at the end of `checks:` (not of the file: `rework_links:`
    follows it), in the indentation already there, then read the file back."""
    old = reg.read_text() if reg.exists() else ""
    lines = old.splitlines()
    start = next((i for i, x in enumerate(lines) if re.match(r"checks\s*:", x)), None)
    if start is None:
        lines.append("checks:")
        start = len(lines) - 1
    end = next((i for i in range(start + 1, len(lines))
                if lines[i].strip() and not lines[i][0].isspace() and not lines[i].startswith("-")
                and not lines[i].lstrip().startswith("#")), len(lines))
    item = next((re.match(r"(\s*)-", x).group(1) for x in lines[start + 1:end] if re.match(r"\s*-\s", x)), "  ")
    body = []
    for e in new:
        body.append(f"{item}- id: {e['id']}")
        for k, v in e.items():
            if k != "id":
                v = ("{" + ", ".join(f"{a}: {q(b)}" for a, b in v.items()) + "}") if isinstance(v, dict) else q(v)
                body.append(f"{item}  {k}: {v}")
    while end > start + 1 and not lines[end - 1].strip():
        end -= 1
    reg.parent.mkdir(parents=True, exist_ok=True)
    reg.write_text("\n".join(lines[:end] + body + lines[end:]) + "\n")
    back = {e.get("id"): e for e in L.registry_entries(reg.read_text())}
    bad = [e["id"] for e in new if {k: (v if isinstance(v, dict) else str(v)) for k, v in e.items()} != back.get(e["id"])]
    if bad:
        reg.write_text(old) if old else reg.unlink()
        die(f"{', '.join(bad)} did not read back as written — registry restored, nothing registered", 1)


def next_ids(have, k):
    top = max([int(m.group(1)) for i in have for m in [re.fullmatch(r"chk_(\d+)", i or "")] if m] or [0])
    return [f"chk_{top + j:03d}" for j in range(1, k + 1)]


def cmd_register(tid):
    pk = packet(tid)
    vf = ROOT / ".work/_verify" / f"{tid}.json"
    if not vf.exists():
        die(f"no verify record for {tid} — run `orch-task.py verify {tid}` first", 1)
    rec = json.loads(vf.read_text())
    tip = rev(f"orch/{tid}") or rev("HEAD", ROOT / ".work" / tid)
    if rec["head"] != tip:
        die(f"orch/{tid} moved since verify ({rec['head'][:12]} -> {(tip or '?')[:12]}) — verify again", 1)
    if rec["verdict"] != "done":
        die(f"verify said {rec['verdict']}; only a clean done registers", 1)
    if not pk["project"]:
        die(f"{pk['path']}: no `project:` — it names the registry file", 1)
    have = all_entries()
    active = {(e.get("cmd"), e.get("expect")): i for i, e in have.items() if e.get("status", "active") == "active"}
    todo = []
    for c in rec["checks"]:
        if c["type"] != "executable" or c["held"] is not True:
            continue
        if c["per_task"]:
            print(f"  #{c['n']} per-task (delta, or names the base) — not registered")
        elif (c["cmd"], c["expect"]) in active:
            print(f"  #{c['n']} already registered as {active[(c['cmd'], c['expect'])]}")
        else:
            todo.append(c)
    waived = {"waived": ", ".join(rec["waived"])} if rec.get("waived") else {}
    new = [{"id": i, "task": tid, "commit": rec["head"][:12], "cmd": c["cmd"], "expect": c["expect"],
            **waived, "registered": date.today().isoformat(), "status": "active"}
           for i, c in zip(next_ids(have, len(todo)), todo)]
    if new:
        append(ROOT / ".orch/registry" / f"{pk['project']}.yaml", new)
    for e in new:
        print(f"  registered {e['id']}: {e['cmd'][:70]}  ({e['expect']})")
    print(f"{len(new)} registered in .orch/registry/{pk['project']}.yaml")
    return 0


def stage_cost(msr):
    """What the measurement's stages spent: every .work/_stages/<msr_id>[-_.]*
    stage, all attempts, wall time and time holding each lock. Fifteen GPU
    hours once went into four measurements and none of it was recorded beside
    the conclusions it bought; the stage files are disposable, the registry is
    not."""
    rows = [json.loads(f.read_text()) for f in sorted(STAGES.glob(f"{msr}*.json"))
            if re.fullmatch(rf"{re.escape(msr)}([-_.][\w.-]*)?", f.stem)]
    if not rows:
        return None
    total = lambda r: r.get("total_s") or r.get("elapsed_s") or 0
    out = {"stages": str(len(rows)), "wall_h": f"{sum(map(total, rows)) / 3600:.2f}"}
    for lock in sorted({r["lock"] for r in rows if r.get("lock")}):
        out[re.sub(r"[^\w-]", "_", lock) + "_h"] = f"{sum(total(r) for r in rows if r.get('lock') == lock) / 3600:.2f}"
    return out


def prereg_order(msr, inputs, artifact=None):
    """(prereg, why-not) — the commit that first named `msr` in a tracked file
    outside .orch/ and .work/, and why that is not before the data the
    conclusion scores: the --artifact, else the newest file the command reads
    (an eval set is old by design; the run's output is the newest input). A
    file's time is its first commit, or its mtime when git does not hold it as
    it stands. Three of one session's measurements ran before their rule was
    committed, or had it written after the numbers were seen; all were
    disclosed in prose, and nothing could check any of them."""
    rx = r"(^|[^A-Za-z0-9_-])" + re.escape(msr) + r"([^A-Za-z0-9_-]|$)"     # the whole id: msr_x is not msr_x2
    cp = git("log", "--reverse", "--format=%H %ct", "-G", rx, "--", ".", ":(exclude).orch", ":(exclude).work",
             ":(exclude).claude", ":(exclude)ORCH.md")       # ORCH's own files hold example ids
    first = cp.stdout.split("\n", 1)[0].split()
    if not first:
        return None, f"no committed file names {msr} — the pre-registration is committed before the run"
    sha, t0 = first[0], int(first[1])
    dated = []
    for rel in inputs:
        tracked = git("ls-files", "--error-unmatch", "--", rel).returncode == 0
        clean = tracked and git("diff", "--quiet", "HEAD", "--", rel).returncode == 0
        if clean:
            out = git("log", "--diff-filter=A", "--format=%ct", "--", rel).stdout.split()
            t = int(out[-1]) if out else None
        else:
            t = int((ROOT / rel).stat().st_mtime)
        if t is not None:
            dated.append((t, rel))
    if artifact:
        dated = [d for d in dated if d[1] == artifact] or dated
    if not dated:
        return sha, None
    t, rel = max(dated)
    if t < t0:
        when = lambda x: datetime.fromtimestamp(x, timezone.utc).isoformat(timespec="minutes")
        return sha, (f"{rel} ({when(t)}) is older than the pre-registration ({sha[:12]}, {when(t0)}): the "
                     "rule was committed after the data it decides on existed")
    return sha, None


def cmd_register_eval(o):
    msr, cmd, expect = o.get("--msr", ""), o.get("--cmd", ""), o.get("--expect", "").strip()
    if sum(bool(o.get(k)) for k in ("--expect", "--expect-file", "--capture")) != 1:
        die("one of --expect \"stdout == V\", --expect-file <file>, or --capture", 1)
    if o.get("--expect-file"):
        try:                                # the value, written with the Write tool: no shell quoting
            expect = "stdout == " + Path(o["--expect-file"]).read_text().strip("\n")
        except OSError as e:
            die(str(e))
    if not re.fullmatch(r"msr_[\w.-]+", msr):
        die("--msr <msr_id> names the measurement (orch-task §8)", 1)
    why = G.bash_trigger(cmd)
    if why or L.PLACEHOLDER.search(cmd):
        die(f"refused: {why or 'an unfilled placeholder'}", 1)
    if o.get("--capture"):
        # What the scorer prints now IS the value — the step a scratchpad
        # helper (reg_arm.sh) used to do, untracked. The run below repeats it,
        # so output that changes between two runs is refused, not registered.
        r = L.run(cmd, ROOT, float(o.get("--timeout", 600)))
        if r is None or r[0] != 0 or not r[1].strip():
            got = "timed out" if r is None else f"exit {r[0]}, stdout \"{r[1].strip()[:60]}\""
            die(f"--capture needs a scorer that exits 0 and prints its value ({got}) — nothing registered", 1)
        expect = "stdout == " + r[1].strip("\n")
        print(f"captured: {r[1].strip()[:200]}")
    if not re.fullmatch(r"stdout\s*==\s*.+", expect, re.S):
        die("an eval entry registers the value its conclusion cites: --expect \"stdout == <value>\"", 1)
    try:
        toks = shlex.split(cmd)
    except ValueError:
        toks = cmd.split()
    local, scored = [], []
    for t in toks + ([o["--artifact"]] if o.get("--artifact") else []):
        p = ROOT / t
        if not p.is_file() or ROOT.resolve() not in p.resolve().parents:
            continue
        rel = str(p.resolve().relative_to(ROOT.resolve()))
        tracked = git("ls-files", "--error-unmatch", "--", rel).returncode == 0
        if t != o.get("--artifact") and p.suffix.lower() in SCRIPTS:   # the scorer: committed code (§8)
            if not tracked:
                die(f"{rel} is not tracked — a scorer that decides something is committed code (§8)", 1)
            if git("diff", "--quiet", "HEAD", "--", rel).returncode != 0:
                die(f"{rel} has uncommitted changes — the registered commit would not be what ran", 1)
        elif not tracked and rel not in local:
            local.append(rel)               # data git does not hold: replayable on this disk only
        elif tracked and rel not in scored:
            scored.append(rel)
    pre, late = prereg_order(msr, local + scored, str(Path(o["--artifact"])) if o.get("--artifact") else None)
    if (late or not pre) and not o.get("--post-hoc"):
        die(f"{late or 'no pre-registration'} — register it with --post-hoc \"<why>\", which the entry "
            "keeps and a results doc must say beside every citation of it (orch-task §8)", 1)
    r = L.run(cmd, ROOT, float(o.get("--timeout", 600)))
    if r is None or L.holds(expect, r[0], r[1]) is not True:
        got = "timed out" if r is None else f"exit {r[0]}, stdout \"{r[1].strip()[:80]}\""
        die(f"it does not hold now ({got}) — nothing registered", 1)
    have = all_entries()
    for i, e in have.items():
        if (e.get("cmd"), e.get("expect"), e.get("status", "active")) == (cmd, expect, "active"):
            print(f"already registered as {i}")
            return 0
    regs = sorted((ROOT / ".orch/registry").glob("*.yaml"))
    project = o.get("--project") or (regs[0].stem if len(regs) == 1 else None)
    if not project:
        die("name the registry with --project <name>", 1)
    e = {"id": next_ids(have, 1)[0], "kind": "eval", "task": msr, "commit": rev("HEAD")[:12],
         "cmd": cmd, "expect": expect}
    if o.get("--artifact"):
        a = ROOT / o["--artifact"]
        if not a.is_file():
            die(f"no artifact at {a}", 1)
        e["artifact"] = {"path": o["--artifact"], "sha256": hashlib.sha256(a.read_bytes()).hexdigest()}
    if local:
        e["local"] = ", ".join(local)
    cost = stage_cost(msr)
    if cost:
        e["cost"] = cost
    e["prereg"] = pre[:12] if pre else "none"
    if o.get("--post-hoc"):
        e["post_hoc"] = o["--post-hoc"]
    if o.get("--conclusion"):
        e["conclusion"] = o["--conclusion"]
    e.update({"noise": o.get("--noise", "unmeasured"), "registered": date.today().isoformat(),
              "status": "active"})
    append(ROOT / ".orch/registry" / f"{project}.yaml", [e])
    print(f"registered {e['id']} (eval, {msr}): {cmd[:70]}  ({expect[:80]})")
    if e.get("post_hoc"):
        print(f"  post hoc: {e['post_hoc']} — every citation of {e['id']} says so (orch-lint.py doc checks)")
    if cost:
        print(f"  cost: {', '.join(f'{k} {v}' for k, v in cost.items())}")
    if local:
        print(f"  local only: {e['local']} — not in git, so a fresh clone cannot replay this entry; a "
              "sweep reports it `inputs missing`, not a break. Commit what is small enough to commit")
    return 0


# ── trace

def trace_error(o):
    """orchestrator_error from --error, or a refusal that says what to pass.
    It takes the KIND of packet defect, not the summary's origin: an origin of
    `execution` or `system` is not the orchestrator's error, so it is `none`."""
    err = o.get("--error")
    kinds = "; ".join(f"{k} ({v})" for k, v in KINDS.items())
    if err is None:
        die("--error is required: `none` is a claim that the packet was right, and the key is never blank. "
            f"A packet defect is one of: {kinds}", 1)
    if err == "none":
        return None
    if err in ("packet", "orchestrator"):
        die(f"--error takes the kind of packet defect, not the origin: {kinds}", 1)
    if err in ("execution", "system", "agent"):
        die(f"an `{err}` origin is not the orchestrator's error: --error none (a system origin also goes "
            "to .orch/upstream.md)", 1)
    if err in KINDS and o.get("--what"):
        return {"kind": err, "what": o["--what"], "better": o.get("--better"),
                "superseded_check": o.get("--superseded-check")}
    die(f"--error is none, or a kind with --what \"…\" (and --better \"…\"): {kinds}", 1)


def trace_args(o):
    """(orchestrator_error, mode, interruptions), every argument checked —
    finish calls this before it writes anything, so a typo cannot leave a task
    registered with no trace."""
    oe = trace_error(o)
    mode = (o.get("--mode") or "L2").split(":")
    if not all(re.fullmatch(r"L[0-5]", m) for m in mode) or len(mode) > 2:
        die("--mode L2, or proposed:used as L2:L3", 1)
    ints = []
    for x in o.get("--interruption", []):
        on, _, how = x.partition(":")
        if on not in ("platform", "usage") or how not in ("same-agent", "redispatched", "pending"):
            die("--interruption platform|usage:same-agent|redispatched|pending", 1)
        ints.append({"on": on, "at": now(), "resumed": how})
    for k in ("--attempts", "--steps", "--wall", "--l3-escapes"):
        if o.get(k) and not o[k].isdigit():
            die(f"{k} takes a whole number", 1)
    if o.get("--checkpoints") and not re.fullmatch(r"\d+,\d+", o["--checkpoints"]):
        die("--checkpoints M,D", 1)
    return oe, mode, ints


def cmd_trace(tid, o):
    pk = packet(tid)
    outcome = o.get("--outcome")
    if outcome not in OUTCOMES:
        die(f"--outcome is one of {', '.join(OUTCOMES)}", 1)
    oe, mode, ints = trace_args(o)
    vf = ROOT / ".work/_verify" / f"{tid}.json"
    rec = json.loads(vf.read_text()) if vf.exists() else None
    executable = any(e.get("type", "executable") != "judged" for e in pk["acceptance"])
    if outcome == "done" and executable and (rec is None or rec.get("verdict") != "done"):
        die("a done trace needs a clean verify record — the outcome is what code verified, "
            "not what the agent said", 1)
    if o.get("--mutation") and rec:
        rec["mutation"] = o["--mutation"]
    ck = {"mandatory": 0, "discretionary": 0}
    for f in (ROOT / ".orch/checkpoints").rglob("*.md"):        # open and resolved/
        t = f.read_text(errors="replace")
        if re.search(rf"^task:\s*{re.escape(tid)}\s*$", t, re.M):
            k = "mandatory" if re.search(r"^class:\s*mandatory", t, re.M) else "discretionary"
            ck[k] += 1
    if o.get("--checkpoints"):
        m, d = (int(x) for x in o["--checkpoints"].split(","))
        ck = {"mandatory": m, "discretionary": d}
    num = lambda k: int(o[k]) if o.get(k) else None
    trace = {
        "trace_id": None, "task_id": tid, "playbook": pk["playbook"] or None,
        "approval": pk["approved"] or None,
        "loop_mode": {"proposed": mode[0], "used": mode[-1], "overridden": mode[0] != mode[-1]},
        "outcome": outcome, "attempts": num("--attempts") or 1, "checkpoints": ck,
        "context_usage": {"refs_given": pk["refs"],
                          "refs_cited": (rec.get("result") or {}).get("refs_cited") if rec else None,
                          "l3_escapes": num("--l3-escapes")},
        "cost": {"steps": num("--steps"), "wall_s": num("--wall")},
        "interruptions": ints,
        "rework": {"links": [], "penalty": 0.0},
        "orchestrator_error": oe,
        "human_actions": [{"cmd": h.split(":")[0], "at": now(), "modified": h.endswith(":modified")}
                          for h in o.get("--human", [])],
        "verification": None if rec is None else {
            k: rec.get(k) for k in ("at", "base", "head", "verdict", "tests_at_base", "scope_violations",
                                    "uncommitted", "mutation", "waived", "findings", "report")}
                        | {"checks": [{k: c.get(k) for k in ("n", "type", "cmd", "expect", "exit", "held", "per_task")}
                                      for c in rec.get("checks", [])],
                           "scan_exit": (rec.get("scan") or {}).get("exit")},
    }
    d = ROOT / ".orch/traces"
    d.mkdir(parents=True, exist_ok=True)
    stem, k = f"trc_{tid[4:]}", 1
    while (d / f"{stem if k == 1 else f'{stem}_{k}'}.json").exists():
        k += 1
    trace["trace_id"] = stem if k == 1 else f"{stem}_{k}"
    (d / f"{trace['trace_id']}.json").write_text(json.dumps(trace, indent=2, ensure_ascii=False) + "\n")
    print(f"trace: .orch/traces/{trace['trace_id']}.json ({outcome}, orchestrator_error: "
          f"{oe['kind'] if oe else 'null'})")
    return 0


def cmd_finish(tid, o):
    """Everything after a clean verify that used to be typed one step at a
    time: register, trace, move the packet, prune, check the merge. A
    two-file tool once took fourteen coordinator steps; the gates are the
    worktree, the subagent and verify, not the typing. Refuses up front —
    before anything is written — unless the verify record is clean and current
    and the trace arguments are complete."""
    pk = packet(tid)
    vf = ROOT / ".work/_verify" / f"{tid}.json"
    if not vf.exists():
        die(f"no verify record for {tid} — run `orch-task.py verify {tid}` first", 1)
    rec = json.loads(vf.read_text())
    if rec["verdict"] != "done":
        die(f"verify said {rec['verdict']}; finish is for a clean done", 1)
    tip = rev(f"orch/{tid}") or rev("HEAD", ROOT / ".work" / tid)
    if rec["head"] != tip:
        die(f"orch/{tid} moved since verify ({rec['head'][:12]} -> {(tip or '?')[:12]}) — verify again", 1)
    o = dict(o, **{"--outcome": "done"})
    trace_args(o)
    src = rec.get("report") or {}
    if src.get("source") == "orchestrator":
        print(f"report: supplied by the orchestrator ({src.get('file')}), not recorded from the agent — "
              "the summary says so")
    cmd_register(tid)
    have = [json.loads(f.read_text()) for f in (ROOT / ".orch/traces").glob(f"trc_{tid[4:]}*.json")]
    if any(t.get("task_id") == tid and t.get("outcome") == "done"
           and (t.get("verification") or {}).get("head") == rec["head"] for t in have):
        print(f"trace: one already records this verify of {tid} — not written again")
    else:
        cmd_trace(tid, o)
    if pk["path"].parent == ROOT / ".orch/queue":
        (ROOT / ".orch/queue/done").mkdir(parents=True, exist_ok=True)
        pk["path"].replace(ROOT / ".orch/queue/done" / pk["path"].name)
        print(f"packet: .orch/queue/done/{pk['path'].name}")
    wt, facts = ROOT / ".work" / tid, (rec.get("result") or {}).get("new_facts") or 0
    left = [f for f in dirty(wt) if f not in rec.get("byproducts", [])] if wt.exists() else []
    if not wt.exists():
        print("worktree: already gone")
    elif facts or o.get("--keep") or left:
        why = (f"the result proposes {facts} new_facts — dispatch orch-archivist with this path first"
               if facts else f"it holds {', '.join(left[:5])}, which verify neither committed nor made"
               if left else "--keep")
        print(f"worktree kept: {wt} — {why}; then `git worktree remove --force {wt}`")
    else:
        cp = git("worktree", "remove", "--force", str(wt))
        print(f"worktree: pruned; the work is on orch/{tid}" if cp.returncode == 0
              else f"worktree: could not prune ({cp.stderr.strip()})")
    cur = git("rev-parse", "--abbrev-ref", "HEAD").stdout.strip()
    mt = git("merge-tree", "--write-tree", "HEAD", f"orch/{tid}")
    state = {0: f"merges cleanly into {cur}", 1: f"CONFLICTS with {cur} — merge it by hand"}.get(
        mt.returncode, "merge not checked (git merge-tree --write-tree needs git >= 2.38)")
    for f in open_checkpoints(tid):
        print(f"open checkpoint: {f.relative_to(ROOT)} — this task is done; resolve it into resolved/ "
              "or delete it, or every session start keeps counting it")
    print(f"merge: orch/{tid} {state}\n  git merge --no-edit orch/{tid}")
    print("Run that line yourself only if the user granted merges — as its own command, so a refusal is "
          "seen and not retried; otherwise it goes in the summary as `! git merge …`.")
    return 0


def main():
    a = sys.argv[1:]
    if not a or a[0] not in ("prompt", "verify", "register", "trace", "finish"):
        die("usage: orch-task.py prompt|verify|register|trace|finish <tsk_id> [...], or register --eval — "
            "see the docstring")
    o, pos, i = {}, [], 1
    multi = ("--interruption", "--human", "--waive")
    while i < len(a):
        if a[i] in ("--eval", "--keep", "--capture"):
            o[a[i]], i = True, i + 1
        elif a[i].startswith("--"):
            if i + 1 >= len(a):
                die(f"{a[i]} needs a value")
            if a[i] in multi:
                o.setdefault(a[i], []).append(a[i + 1])
            else:
                o[a[i]] = a[i + 1]
            i += 2
        else:
            pos, i = pos + [a[i]], i + 1
    if a[0] == "register" and o.get("--eval"):
        return cmd_register_eval(o)
    if len(pos) != 1:
        die(f"{a[0]} takes one task id")
    if a[0] == "prompt":
        return cmd_prompt(pos[0])
    if a[0] == "verify":
        return cmd_verify(pos[0], o.get("--test"), o.get("--result"), float(o.get("--timeout", 300)),
                          o.get("--commit"), o.get("--waive", []))
    if a[0] == "register":
        return cmd_register(pos[0])
    if a[0] == "finish":
        return cmd_finish(pos[0], o)
    return cmd_trace(pos[0], o)


if __name__ == "__main__":
    try:
        rc = main()
    except SystemExit:
        raise
    except Exception as e:                # a crash is "could not run", never 0 or 1
        die(f"internal error: {type(e).__name__}: {e}")
    sys.exit(rc)
```
````

#### Reports, recorded by code

`verify` reads what the agent reported — its status, its confidence, the
blocks its evidence cites — and in the field it was not reading the agent's
report at all. In auto mode Claude Code tells every subagent to deliver its
report through a `SubagentHandback` call, "your full report", and that plain
text written after it is not delivered. The executors did as told, in prose:
4 of 19 delivered the JSON their definition asks for, each of the others was
resumed by hand once or twice, and for two tasks the orchestrator wrote the
result file itself, which `verify` could not tell from an agent's (up_0013).
The blind hook refused `SubagentHandback` to judges like any other tool, so
six judges finished and none could deliver (up_0012). And the archivist
reported "complete" over 58 lint errors.

`orch-report.py` checks a report where it leaves the agent: on the
`SubagentHandback` call, and at `SubagentStop` when there is no such call. A
report that breaks its agent's contract is refused — the call denied, or the
stop prevented — with the exact object to send instead, so the agent fixes it
itself rather than the orchestrator resuming it. One that passes is written to
`.work/_results/<task_or_msr_id>/<agent_id>.json` with the agent's type and
id, which come from the hook input: the harness supplies them, no model does.
The guard refuses every other write there. `verify` reads that record, and a
report the orchestrator supplies instead is accepted and labelled as its own.

````
FILE: .claude/hooks/orch-report.py
```python
#!/usr/bin/env python3
"""ORCH report hook — an agent's report, held to its contract and recorded by code.

Wired twice in .claude/settings.json, for every subagent in the repo; it acts
only for the ORCH agents below and lets everything else through untouched.

  PreToolUse SubagentHandback   in auto mode a subagent delivers its report as
                                that call's `message`, and "plain text you write
                                at the end is not delivered" (Claude Code 2.1.292)
  SubagentStop                  otherwise the report is the last assistant message

Why it exists (slm-memory-management, 2026-10-06). Only 4 of 19 executors
delivered the result JSON their definition asks for — the harness tells a
subagent to hand back "your full report", and they did, in prose — so each was
resumed by hand, 15 to 25 times in one session. For two tasks the orchestrator
wrote the result file itself, and `verify` could not tell it from an agent's
(up_0013). The archivist reported "complete" over 58 lint errors.

So the contract is checked where the report leaves the agent:
  executor, executor-deep, debugger, reviewer   one JSON object with `status`
  judge                                         and `task_id` (judge: `msr_id`)
  archivist   `orch-lint.py memory --task <id>` exits 0 for the id it names
A report that breaks it is refused (exit 2): the handback call is denied, or
the stop is prevented, and the agent reads why and sends it again. Two
refusals at most (three for the archivist's lint); then it goes through and is
recorded as failing, so `verify` says `incomplete` — never a wedged agent.

A report that passes is written to .work/_results/<task_or_msr_id>/<agent_id>.json
with the agent type and id from the hook input, which the harness supplies
and no model writes. The guard refuses any other write there, so `verify`
reads what the agent sent, and says so. Refusals are logged to
.work/_results/refusals.log.

Never blocks for its own bugs: unreadable input, a crash or a timeout allows.
"""
import importlib.util
import json
import os
import re
import subprocess
import sys
import time
from datetime import datetime, timezone
from pathlib import Path

CONTRACT = {"orch-executor": "task_id", "orch-executor-deep": "task_id", "orch-debugger": "task_id",
            "orch-reviewer": "task_id", "orch-judge": "msr_id"}
ARCHIVIST = "orch-archivist"
HANDBACK = "SubagentHandback"
STATUSES = ("done", "blocked", "needs_decision", "failed", "escalate")
IDS = {"task_id": r"tsk_[A-Za-z0-9][\w.-]*", "msr_id": r"msr_[A-Za-z0-9][\w.-]*"}
MAX = {"report": 2, "lint": 3}
SCHEMA = {
    "task_id": '{"task_id": "%s", "status": "done|blocked|needs_decision|failed|escalate", '
               '"blocked_on": null, "summary": "one or two sentences, file:line for code claims", '
               '"evidence": [{"cite": "file:line", "reason": "..."}, {"check": "cmd", "exit": 0}], '
               '"confidence": 0.0, "new_facts": [], "open_questions": []}',
    "msr_id": '{"msr_id": "%s", "status": "done|blocked", "blocked_on": null, "canary": "refused|read", '
              '"items": [{"id": "...", "label": "...", "why": "..."}], "open_questions": []}',
}


def _lint():
    p = Path(__file__).resolve().with_name("orch-lint.py")
    sys.dont_write_bytecode = True
    spec = importlib.util.spec_from_file_location("orch_lint", p)
    m = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(m)
    return m


def now():
    return datetime.now(timezone.utc).isoformat(timespec="seconds")


def safe(s):
    return re.sub(r"[^\w.-]", "_", s)[:80] or "-"


class Hook:
    def __init__(self, ev, root, lint):
        self.ev, self.L = ev, lint
        self.agent = ev.get("agent_type") or ""
        self.aid = str(ev.get("agent_id") or "-")
        self.dir = root / ".work/_results"
        self.root = root
        self.sf = self.dir / ".agents" / f"{safe(self.aid)}.json"
        try:
            self.st = json.loads(self.sf.read_text())
        except (OSError, ValueError):
            self.st = {}

    def save(self):
        self.sf.parent.mkdir(parents=True, exist_ok=True)
        self.sf.write_text(json.dumps(self.st))

    def write(self, rid, via, text, obj, ok, **extra):
        d = self.dir / (rid if re.fullmatch(r"(tsk|msr)_[\w.-]+", rid or "") else "_unattributed")
        d.mkdir(parents=True, exist_ok=True)
        f = d / f"{safe(self.aid)}.json"
        try:
            n = json.loads(f.read_text()).get("reports", 0)
        except (OSError, ValueError):
            n = 0
        rec = {"id": rid, "agent_type": self.agent, "agent_id": self.aid, "via": via, "at": now(),
               "reports": n + 1, "refusals": self.st.get("refusals", 0), "ok": ok, "json": obj is not None,
               "result": obj, "report": text[-20000:], "transcript": self.ev.get("agent_transcript_path"),
               **extra}
        tmp = d / f".{f.name}.tmp"
        tmp.write_text(json.dumps(rec, indent=2, ensure_ascii=False) + "\n")
        tmp.replace(f)                      # atomic: verify never reads half a report

    def refuse(self, kind, rid, msg):
        """Exit 2 with the reason, unless this agent has used its refusals."""
        k = f"{kind}_refusals" if kind != "report" else "refusals"
        n = self.st.get(k, 0)
        if n >= MAX[kind]:
            return False
        self.st[k] = n + 1
        self.save()
        try:
            self.dir.mkdir(parents=True, exist_ok=True)
            with open(self.dir / "refusals.log", "a") as f:
                f.write(f"{now()} {self.aid} {self.agent} {rid or '-'} {kind} {n + 1}/{MAX[kind]}\n")
        except OSError:
            pass
        sys.stderr.write(f"ORCH report — refused ({n + 1} of {MAX[kind]}): {msg}\n")
        sys.exit(2)

    def delivered_this_run(self):
        """At a stop: did this run already hand back a report that passed?"""
        return self.st.get("delivered", 0) > self.st.get("stopped", 0)

    def stopped(self):
        self.st["stopped"] = time.time()
        self.save()


def named_id(text, path):
    """The tsk_/msr_ id an archivist's report names (`ingest <id>`), else the
    first its dispatch prompt names — read from the transcript's first user turn."""
    m = re.search(r"\bingest\s+((?:tsk|msr)_[\w.-]*\w)", text) or re.search(r"\b((?:tsk|msr)_[\w.-]*\w)", text)
    if m:
        return m.group(1)
    try:
        with open(path or "") as f:
            for _, line in zip(range(400), f):
                r = json.loads(line)
                if r.get("type") != "user":
                    continue
                c = (r.get("message") or {}).get("content")
                t = c if isinstance(c, str) else " ".join(x.get("text", "") for x in c or [] if isinstance(x, dict))
                m = re.search(r"\b((?:tsk|msr)_[\w.-]*\w)", t)
                return m.group(1) if m else None
    except (OSError, ValueError, AttributeError):
        return None
    return None


def contract(h, via, text, at_stop):
    if at_stop and h.delivered_this_run():
        return 0                           # the report went out through SubagentHandback; this is after it
    key = CONTRACT[h.agent]
    obj = h.L.last_json(text)
    problem = None
    if obj is None:
        problem = "it holds no result JSON object (one with a `status` key)"
    elif obj.get("status") not in STATUSES:
        problem = f"its status is {obj.get('status')!r}, not one of {', '.join(STATUSES)}"
    elif not re.fullmatch(IDS[key], str(obj.get(key) or "")):
        problem = f"it names no `{key}` — the id at the top of your prompt"
    rid = str(obj.get(key)) if obj and re.fullmatch(IDS[key], str(obj.get(key) or "")) else None
    if problem is None:
        h.write(rid, via, text, obj, True)
        h.st.update(refusals=0, delivered=time.time() if not at_stop else h.st.get("delivered", 0))
        h.save()
        return 0
    guess = rid or (re.search(IDS[key], text) or [None])[0]
    h.refuse("report", guess,
             f"your report is code's input, and {problem}. Send it again as ONLY this JSON object — no prose "
             "before or after it, no code fence; your prose goes in \"summary\":\n"
             + SCHEMA[key] % (guess or f"<the {key} your prompt names>")
             + "\nWhere it goes: if this run has the SubagentHandback tool, it is that call's `message`; "
             "otherwise it is your final message.")
    h.write(guess, via, text, obj, False)    # refusals spent: recorded as failing, verify says incomplete
    return 0


def archivist(h, via, text, at_stop):
    if at_stop and h.delivered_this_run():
        return 0
    rid = named_id(text, h.ev.get("agent_transcript_path"))
    if not rid:
        h.write(None, via, text, None, False, lint="no tsk_/msr_ id named — not linted")
        return 0
    try:
        cp = subprocess.run([sys.executable, str(Path(__file__).resolve().with_name("orch-lint.py")),
                             "memory", "--task", rid], capture_output=True, text=True, timeout=100,
                            stdin=subprocess.DEVNULL, cwd=str(h.root))
        rc, out = cp.returncode, cp.stdout
    except subprocess.TimeoutExpired:
        rc, out = None, "timed out"
    lines = [x for x in out.splitlines() if x.strip()]
    if rc == 1:
        errs = [x for x in lines if x.startswith("error")]
        h.refuse("lint", rid,
                 f"`orch-lint.py memory --task {rid}` exits 1 — {len(errs)} error(s) on this ingest's blocks. "
                 "Fix them, run it again until it exits 0, and only then report, quoting its `index: …` line:\n"
                 + "\n".join(errs[:15]) + (f"\n… and {len(errs) - 15} more" if len(errs) > 15 else ""))
    h.write(rid, via, text, None, rc == 0, lint={"exit": rc, "summary": lines[-1] if lines else ""})
    if not at_stop:
        h.st["delivered"] = time.time()
        h.save()
    return 0


def main():
    try:
        ev = json.load(sys.stdin)
    except Exception:
        return 0                           # unattributable: allowed
    agent = ev.get("agent_type") or ""
    if agent not in CONTRACT and agent != ARCHIVIST:
        return 0
    event = ev.get("hook_event_name") or ("PreToolUse" if ev.get("tool_name") else "SubagentStop")
    if event == "PreToolUse":
        if ev.get("tool_name") != HANDBACK:
            return 0
        text, via, at_stop = str((ev.get("tool_input") or {}).get("message") or ""), HANDBACK, False
    elif event == "SubagentStop":
        text, via, at_stop = str(ev.get("last_assistant_message") or ""), "SubagentStop", True
    else:
        return 0
    L = _lint()
    h = Hook(ev, L.ROOT, L)
    rc = contract(h, via, text, at_stop) if agent in CONTRACT else archivist(h, via, text, at_stop)
    if at_stop:
        h.stopped()
    return rc


if __name__ == "__main__":
    try:
        sys.exit(main())
    except SystemExit:
        raise
    except Exception as e:                 # a broken hook fails open, loudly
        sys.stderr.write(f"orch-report error (allowing): {type(e).__name__}: {e}\n")
        sys.exit(0)
```
````

#### Long jobs: stages, not chains

A chained GPU job put the wait for stage 1 inside the background job that then
ran stage 2. The wait counted against that job's time cap (2 hours in that
session), and stage 2 was killed midway. An earlier chain waited on `pgrep -f`,
which matched its own command line and never fired. About an hour of GPU,
lost to chaining by hand.

`orch-stage.py` makes each stage its own background job, whose outcome is a
file its own shell writes — never a process listing. A chain is then: start a
stage, be told it ended, start the next. Re-running a chain skips the stages
already done, which is what resume means; `--lock gpu` keeps two stages off one
device without anyone waiting inside a job. The procedure is `orch-task` §8.

v5 added two things from a fifteen-GPU-hour session. `--prereg` checks the
command against the parameters a committed pre-registration fixes. One
pre-registration said a 1 % dev split, the training script defaulted to 5 %,
and only a manual read before launch caught it. Each stage now also keeps its
time across every attempt, `status` sums it per lock, and `register --eval`
copies the measurement's total onto its registry entry. Before that, the
question "what did this conclusion cost?" could be answered only from logs.

v6 adds three from a ten-GPU-hour session. A stage of a measurement no
committed file names is refused: the pre-registration comes first. A stage
that cannot finish under the harness's job cap is refused with the
arithmetic, and one with no `--timeout` ends just under the cap, on record.
And `status` labels its total `ALL` and prints this session's beside it: a
project total of 23.72 hours, 13.53 of them from before the session, once
reached the user as the session's GPU time. `status --new` lets a stale
wakeup answer in one line.

````
FILE: .claude/hooks/orch-stage.py
```python
#!/usr/bin/env python3
"""ORCH stage runner — long jobs as stages, each its own background job.

A GPU run, a bulk fetch, an overnight eval: work that outlives a turn. The
field lost about an hour of GPU to two ways of chaining it by hand. A wait for
stage 1 put inside the background job that then ran stage 2 counted against
that job's time cap (2 h in that session), so stage 2 was killed midway. An
earlier chain waited on `pgrep -f`, which matched its own command line and so
never fired. Both are a stage's completion inferred from something other than
the stage.

So: one stage per background job, and completion is a status file written by
the stage's own shell, never a process listing. The orchestrator starts the
next stage when it is told the previous one ended — never a wait inside a job.

    python3 .claude/hooks/orch-stage.py run <stage> [--after <s>[,<s>]] [--lock <name>]
                                             [--timeout <seconds>] [--est <min> | --like <stage>]
                                             [--prereg <file>] [--no-prereg "<why>"] [--rerun] -- <command>
    python3 .claude/hooks/orch-stage.py status [<stage or glob>...] [--since <time|stage>] [--new] [--brief]

run     runs <command> in the foreground — launch it as ONE background job —
        logging to .work/_stages/<stage>.log. Refuses (exit 3) while a stage
        named in --after is not done (and prints the command that resumes
        it), while the same stage is still running, or while another stage
        holds --lock (one GPU, one holder). A stage already done with the same
        command is skipped, so re-running a whole chain after an interruption
        resumes it; --rerun forces it. A command the guard blocks is refused
        (exit 2). --prereg <file> checks the command against the parameters a
        committed pre-registration fixes, before anything runs (exit 2 on a
        mismatch). Exits with the command's code, 124 on --timeout.
        The job cap (settings.yaml stages.job_cap_min, 120): a --timeout, an
        --est, or the time of a --like stage that does not fit under it less
        five minutes is refused (exit 3) with the arithmetic; with no
        --timeout the stage gets exactly that, so it ends as `timeout`, on
        record, instead of being killed. A stage named msr_… is refused
        (exit 3) until a committed file names its measurement — the
        pre-registration comes first — unless --no-prereg says why it
        produces no result.
status  every stage (or the named ones; globs allowed): done · failed ·
        running · interrupted (the runner died and the job left no exit file) ·
        orphaned (the runner died, the job is still running) · timeout ·
        killed — with the time each spent over all its attempts, and a total
        per lock: `status 'msr_p15-r13*'` is what that measurement cost.
        The total is labelled ALL; beside it, `this session` (from the
        SessionStart that began it) and --since a time or a stage. --new
        prints only what changed since the last --new, or `nothing new`.
        exit 0 all done · 1 any failed/interrupted/timeout/killed · 3 any running
        or never run · 2 could not read. --brief prints one line, for SessionStart.
"""
import fcntl
import fnmatch
import importlib.util
import json
import os
import re
import shlex
import signal
import subprocess
import sys
import time
from datetime import datetime, timezone
from pathlib import Path


def die(msg, code=2):
    sys.stderr.write(f"orch-stage: {msg}\n")
    sys.exit(code)


def _lint():
    """orch-lint.py, which carries the guard: the guard's matcher, and the
    YAML reader everything else uses."""
    p = Path(__file__).resolve().with_name("orch-lint.py")
    sys.dont_write_bytecode = True
    spec = importlib.util.spec_from_file_location("orch_lint", p)
    m = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(m)
    return m


try:
    L = _lint()
except SystemExit:
    raise
except Exception as e:
    die(f"cannot load orch-lint.py: {e}")
G = L.G
DIR = G.ROOT / ".work/_stages"
NAME = re.compile(r"[A-Za-z0-9][\w.-]*")
FAILED = ("failed", "interrupted", "timeout", "killed")
SESSION, SEEN = DIR / ".session", DIR / ".seen.json"
MARGIN = 300                               # seconds kept between a stage's timeout and the job cap
# Where a measurement id does not count as pre-registered: ORCH's own files
# hold example ids, and .orch/ is written by registering, after the run.
NOT_PREREG = (":(exclude).orch", ":(exclude).work", ":(exclude).claude", ":(exclude)ORCH.md")


def cap_s():
    """The harness's cap on one background job, from settings.yaml
    `stages.job_cap_min` (120 when unset). A stage longer than it is killed
    by the harness, and nothing records why."""
    p = G.ROOT / ".orch/config/settings.yaml"
    m = re.search(r"^\s*job_cap_min\s*:\s*(\d+(?:\.\d+)?)", p.read_text(), re.M) if p.exists() else None
    return float(m.group(1)) * 60 if m else 7200.0


def now():
    return datetime.now(timezone.utc).isoformat(timespec="seconds")


def load(stage):
    f = DIR / f"{stage}.json"
    try:
        return json.loads(f.read_text()) if f.exists() else None
    except ValueError:
        die(f"{f} is not JSON")


def save(st):
    tmp = DIR / f".{st['stage']}.json.tmp"
    tmp.write_text(json.dumps(st, indent=2) + "\n")
    tmp.replace(DIR / f"{st['stage']}.json")          # atomic: a reader never sees half


def alive(pid, word):
    """Is `pid` alive AND still the process we started? A recycled pid is a
    stranger, and taking it for the job is the pgrep mistake again."""
    try:
        os.kill(pid, 0)
    except (ProcessLookupError, PermissionError, TypeError):
        return False
    cl = Path(f"/proc/{pid}/cmdline")
    return word in cl.read_bytes().decode(errors="replace") if cl.exists() else True


def state(st):
    """The truth about a stage, from its files — never from a process list."""
    if st is None:
        return "never run"
    if st["status"] != "running":
        return st["status"]
    if alive(st.get("pid"), "orch-stage"):
        return "running"
    ex = DIR / f"{st['stage']}.exit"
    if ex.exists():                       # the runner died; the job finished anyway
        return "done" if ex.read_text().strip() == "0" else "failed"
    pg = st.get("pgid")
    try:
        os.killpg(pg, 0)
        return "orphaned"
    except (ProcessLookupError, PermissionError, TypeError):
        return "interrupted"


def again(st):
    """The command that resumes a stage: the same name and command, which a
    stage not `done` re-runs (a resumable one skips what it finished)."""
    opts = (f" --lock {st['lock']}" if st.get("lock") else "") + \
           (f" --timeout {st['timeout']:g}" if st.get("timeout") else "") + \
           (f" --prereg {shlex.quote(st['prereg']['path'])}" if st.get("prereg") else "") + \
           (f" --no-prereg {shlex.quote(st['no_prereg'])}" if st.get("no_prereg") else "")
    return f"python3 .claude/hooks/orch-stage.py run {st['stage']}{opts} -- {shlex.quote(st['cmd'])}"


def prereg(path, stage, cmd):
    """Check `cmd` against the parameters a pre-registration fixes for this
    stage. The pre-registration is a committed file holding a block

        params:
          - {stage: "msr_p15-r13-train*", flag: --dev-frac, value: 0.01}

    and each entry whose stage glob matches must be passed, explicitly, with
    that value. A pre-registration said a 1 % dev split while the training
    script defaulted to 5 %, and the command never passed the flag; nothing
    compared them. A flag left to the script's default is refused for the same
    reason a different value is: the run would not be the one registered."""
    p = G.ROOT / path
    try:
        rel = str(p.resolve().relative_to(G.ROOT.resolve())) if p.is_file() else None
    except ValueError:
        rel = None
    if rel is None:
        die(f"--prereg {path}: no such file in the repo")
    git = lambda *a: subprocess.run(["git", "-C", str(G.ROOT), *a], capture_output=True, text=True)
    if git("ls-files", "--error-unmatch", "--", rel).returncode != 0 or \
            git("diff", "--quiet", "HEAD", "--", rel).returncode != 0:
        die(f"--prereg {rel} is not committed as it stands — a pre-registration is fixed before the run")
    entries, inside = [], False
    for line in p.read_text().splitlines():
        if re.match(r"\s*params\s*:\s*(#.*)?$", line):
            inside = True
        elif inside and re.match(r"\s*-\s*\{", line):
            entries.append(L.flow_map(line.split("-", 1)[1]))
        elif inside and line.strip() and not line.lstrip().startswith("#"):
            inside = False
    mine = [e for e in entries if fnmatch.fnmatchcase(stage, e.get("stage") or "*")]
    if not mine:
        die(f"--prereg {rel}: no `params:` entry matches stage {stage} ({len(entries)} entr"
            f"{'y' if len(entries) == 1 else 'ies'}) — name the stage, or run it without --prereg")
    try:
        toks = shlex.split(cmd)
    except ValueError as e:
        die(f"cannot read the command to check it: {e}")
    bad = []
    for e in mine:
        flag, want = e.get("flag", ""), e.get("value", "")
        got = [toks[i + 1] for i, t in enumerate(toks[:-1]) if t == flag] + \
              [t.split("=", 1)[1] for t in toks if t.startswith(flag + "=")]
        if not got:
            bad.append(f"{flag} is not passed — the script's default would run, not the registered {want}")
        elif not all(same(g, want) for g in got):
            bad.append(f"{flag} {got[-1]} — the pre-registration fixes {want}")
    if bad:
        die(f"{stage} does not run what {rel} registered:\n  " + "\n  ".join(bad))
    head = git("rev-parse", "HEAD").stdout.strip()
    print(f"{stage}: {len(mine)} pre-registered parameter(s) passed as registered ({rel})")
    return {"path": rel, "commit": head[:12], "checked": [e.get("flag") for e in mine]}


def measurement(stage):
    """(id, file) — the measurement this `msr_` stage belongs to, and a
    committed file naming it. `msr_p16-g4-retrieve` belongs to the longest
    prefix a committed file names as a whole id: `msr_p16-g4`, then `msr_p16`.
    (None, candidates) when none is named — the run would come before its rule."""
    parts, cands = stage.split("-"), []
    for k in range(len(parts), 0, -1):
        c = "-".join(parts[:k])
        if len(c) > 4:
            cands.append(c)
    for c in cands:
        rx = r"(^|[^A-Za-z0-9_-])" + re.escape(c) + r"([^A-Za-z0-9_-]|$)"
        cp = subprocess.run(["git", "-C", str(G.ROOT), "grep", "-lE", rx, "HEAD", "--", ".", *NOT_PREREG],
                            capture_output=True, text=True)
        if cp.returncode == 0 and cp.stdout.strip():
            return c, cp.stdout.split("\n", 1)[0].split(":", 1)[-1]
    return None, cands


def estimate(stage, est, like):
    """(seconds, basis) a stage is expected to run, or (None, None)."""
    if est:
        return float(est) * 60, "--est"
    for name, why in ((like, f"--like {like}"), (stage, "its last run")):
        st = load(name) if name else None
        if st and st.get("status") == "done" and st.get("elapsed_s") is not None:
            return float(st["elapsed_s"]), f"{why}: {st['elapsed_s'] / 60:.0f} min"
    if like:
        die(f"--like {like}: no finished stage of that name to take a time from")
    return None, None


def same(a, b):
    try:
        return float(a) == float(b)
    except ValueError:
        return a == b


def run(stage, cmd, after, lock, timeout, rerun, pre, est=None, like=None, no_prereg=None):
    if not NAME.fullmatch(stage):
        die(f"stage name {stage!r}: letters, digits, . _ -")
    why = G.bash_trigger(cmd)
    if why:
        die(f"the guard blocks this command ({why}) — a stage is not a way around it")
    if stage.startswith("msr_") and not no_prereg:
        mid, where = measurement(stage)
        if mid is None:
            die(f"{stage}: no committed file names its measurement ({' or '.join(where)}). Commit the "
                "pre-registration first, naming the id — the rule is fixed before the run (orch-task §8). "
                "A stage that produces no result (a download, an index build) runs with "
                "--no-prereg \"<why>\", kept on the stage", 3)
    checked = prereg(pre, stage, cmd) if pre else None
    # The job cap: a 2 h harness cap killed stages midway, and splitting the
    # ones that might not fit was kept in the orchestrator's head (slm, 2026-10-06).
    cap = cap_s()
    usable = cap - MARGIN
    if timeout is not None and timeout > usable:
        die(f"--timeout {timeout:g}s does not fit the job cap: {cap / 60:g} min is {cap:g}s, less {MARGIN}s "
            f"margin = {usable:g}s. The harness would kill the job first and nothing would record it — "
            "split the stage (the cap is stages.job_cap_min in .orch/config/settings.yaml)", 3)
    t_est, basis = estimate(stage, est, like)
    if t_est is not None and t_est > usable:
        die(f"{stage}: expected {t_est / 60:.0f} min ({basis}), and one job gets {usable / 60:.0f} "
            f"({cap / 60:g} min cap less {MARGIN // 60} min margin). Split it into stages that each fit — "
            "retrieval and generation for one arm are two stages", 3)
    if timeout is None:
        timeout = usable                   # ends as `timeout`, recorded, instead of killed by the harness
    DIR.mkdir(parents=True, exist_ok=True)
    st = load(stage)
    s = state(st)
    if s in ("running", "orphaned"):
        die(f"{stage} is already {s} (pid {st.get('pid')}) — one job per stage", 3)
    if s == "done" and not rerun:
        if st["cmd"] == cmd:
            print(f"{stage}: done at {st.get('ended')} — skipped (resume). --rerun to run it again.")
            return 0
        die(f"{stage} is done with a different command — name a new stage, or --rerun", 3)
    for dep in after:
        dst = load(dep)
        ds = state(dst)
        if ds != "done":
            how = (f" Resume it first — the same name and command re-run a {ds} stage:\n  {again(dst)}"
                   if ds in FAILED else "")
            die(f"{stage} runs after {dep}, which is {ds}. Start {stage} when {dep} is done — "
                f"never wait for it inside a job.{how}", 3)
    held = None
    if lock:
        if not NAME.fullmatch(lock):
            die(f"lock name {lock!r}: letters, digits, . _ -")
        held = open(DIR / f"{lock}.lock", "a+")
        try:
            fcntl.flock(held, fcntl.LOCK_EX | fcntl.LOCK_NB)
        except BlockingIOError:
            held.seek(0)
            die(f"lock {lock} is held by {held.read().strip() or 'another stage'} — start {stage} "
                "when it is released", 3)
        held.seek(0)
        held.truncate()
        held.write(stage)
        held.flush()
    exitf = DIR / f"{stage}.exit"
    exitf.unlink(missing_ok=True)
    log = DIR / f"{stage}.log"
    # the job's own shell writes its exit code, so the outcome survives the runner
    script = f"trap 'echo $? > {shlex.quote(str(exitf))}' EXIT\n{cmd}"
    with open(log, "a") as lf:
        lf.write(f"\n== {stage} {now()} :: {cmd}\n")
        lf.flush()
        # the job inherits the lock: if the runner dies, the GPU is still taken
        p = subprocess.Popen(["bash", "-c", script], stdout=lf, stderr=subprocess.STDOUT,
                             stdin=subprocess.DEVNULL, start_new_session=True,
                             pass_fds=(held.fileno(),) if held else ())
    prev = st or {}
    st = {"stage": stage, "cmd": cmd, "status": "running", "pid": os.getpid(), "pgid": p.pid,
          "cwd": os.getcwd(), "after": after, "lock": lock, "timeout": timeout, "prereg": checked,
          "log": str(log.relative_to(G.ROOT)), "started": now(), "ended": None, "exit": None,
          "elapsed_s": None, "total_s": prev.get("total_s") or prev.get("elapsed_s") or 0,
          "attempt": (prev.get("attempt") or 0) + 1, "no_prereg": no_prereg,
          "runs": runs(prev) + [{"started": now(), "elapsed_s": None}]}
    save(st)
    t0, end = time.monotonic(), "done"

    def stop(sig, _):
        os.killpg(p.pid, signal.SIGTERM)
        raise KeyboardInterrupt
    for sg in (signal.SIGTERM, signal.SIGINT, signal.SIGHUP):
        signal.signal(sg, stop)
    try:
        rc = p.wait(timeout=timeout)
        end = "done" if rc == 0 else "failed"
    except subprocess.TimeoutExpired:
        os.killpg(p.pid, signal.SIGTERM)
        try:
            p.wait(10)
        except subprocess.TimeoutExpired:
            os.killpg(p.pid, signal.SIGKILL)
            p.wait()
        rc, end = 124, "timeout"
    except KeyboardInterrupt:
        try:
            p.wait(10)
        except subprocess.TimeoutExpired:
            os.killpg(p.pid, signal.SIGKILL)
        rc, end = 143, "killed"
    el = round(time.monotonic() - t0)
    st["runs"][-1]["elapsed_s"] = el
    st.update({"status": end, "exit": rc, "ended": now(), "elapsed_s": el, "total_s": st["total_s"] + el})
    save(st)
    print(f"{stage}: {end} (exit {rc}, {el}s) — log {st['log']}")
    return rc


def hours(s):
    return f"{s / 3600:.2f} h"


def runs(st):
    """Every attempt as {started, elapsed_s}. A record older than the list
    has one: all its time, at its last start."""
    if not st:
        return []
    return st.get("runs") or [{"started": st.get("started"), "elapsed_s": st.get("total_s") or st.get("elapsed_s")}]


def mark_session():
    """SessionStart pipes its input to `status --brief`: a new session (or
    /clear) marks where "this session" begins; resume and compaction do not.
    One session's GPU time was reported as about 24 hours — the project's
    total over 44 stages, 21 of which ran before the session (slm, 2026-10-06)."""
    try:
        if sys.stdin is None or sys.stdin.isatty():
            return
        import select
        if not select.select([sys.stdin], [], [], 0.5)[0]:
            return
        ev = json.loads(sys.stdin.read() or "{}")
    except (ValueError, OSError):
        return
    if isinstance(ev, dict) and ev.get("source") in ("startup", "clear"):
        DIR.mkdir(parents=True, exist_ok=True)
        SESSION.write_text(now() + "\n")


def window(states, since):
    """(stages, seconds, {lock: seconds}) of the attempts started at or after `since`."""
    n, tot, locks = 0, 0, {}
    for _, st, _ in states:
        el = sum(r.get("elapsed_s") or 0 for r in runs(st) if (r.get("started") or "") >= since)
        if el or any((r.get("started") or "") >= since for r in runs(st)):
            n, tot = n + 1, tot + el
            if st.get("lock"):
                locks[st["lock"]] = locks.get(st["lock"], 0) + el
    return n, tot, locks


def news(states):
    """What changed since the last `status --new`: a wakeup that finds nothing
    new answers in one line. Three stale wakeups in one session each got the
    full wrap-up again (slm, 2026-10-06)."""
    try:
        seen = json.loads(SEEN.read_text()) if SEEN.exists() else {}
    except ValueError:
        seen = {}
    changed = [(n, seen.get(n, "unseen"), s) for n, _, s in states if seen.get(n) != s]
    DIR.mkdir(parents=True, exist_ok=True)
    SEEN.write_text(json.dumps({"_at": now(), **{n: s for n, _, s in states}}))
    if not changed:
        print(f"nothing new since {seen.get('_at', 'the last check')}")
    for n, was, s in changed:
        print(f"{n:24} {was} -> {s}")
    return 0


def status(names, brief, since=None, new=False):
    if brief:
        mark_session()
    if not DIR.is_dir():
        return 0 if brief else (print("no stages") or 0)
    every = sorted(f.stem for f in DIR.glob("*.json") if not f.name.startswith("."))
    if names:
        picked = [n for pat in names for n in (fnmatch.filter(every, pat) if any(c in pat for c in "*?[")
                                                else [pat])]
        names = list(dict.fromkeys(picked))
    else:
        names = every
    rows = [(n, load(n)) for n in names]
    states = [(n, st, state(st)) for n, st in rows]
    if new:
        return news(states)
    if brief:
        done = sum(1 for _, _, s in states if s == "done")
        other = [f"{n} {s}" for n, _, s in states if s != "done"]
        if states:
            print(f"## ORCH stages: {done} done" + ("".join(f" · {x}" for x in other)))
        return 0
    spent = lambda st: (st or {}).get("total_s") or (st or {}).get("elapsed_s") or 0
    for n, st, s in states:
        extra = ""
        if st and s == "running":
            extra = f" · started {st['started']}"
        elif st and st.get("ended"):
            extra = f" · exit {st.get('exit')} · {st.get('elapsed_s')}s"
        if st and (st.get("attempt") or 1) > 1:
            extra += f" · {st['attempt']} attempts, {spent(st)}s in all"
        print(f"{n:24} {s:12}{extra}" + (f" · log {st['log']}" if st else ""))
    lk = lambda d: "".join(f" · {hours(v)} holding {k}" for k, v in sorted(d.items()))
    present = [x for x in states if x[1]]
    if len(states) > 1:
        locks = {}
        for _, st, _ in present:
            if st.get("lock"):
                locks[st["lock"]] = locks.get(st["lock"], 0) + spent(st)
        first = min((r.get("started") or "" for _, st, _ in present for r in runs(st)), default="")
        print(f"total: {len(states)} stage(s), {hours(sum(spent(st) for _, st, _ in states))} wall{lk(locks)}"
              f" — ALL of them, every attempt since {first[:16] or '?'}")
    # Labelled windows: a total is never the only number printed.
    marks = []
    if SESSION.exists():
        marks.append(("this session", SESSION.read_text().strip()))
    if since:
        st = load(since) if NAME.fullmatch(since) and (DIR / f"{since}.json").exists() else None
        marks.append((f"since {since}", st["started"] if st else since))
    for label, t in marks:
        k, tot, locks = window(present, t)
        print(f"{label}: {k} stage(s), {hours(tot)} wall{lk(locks)} — attempts started at or after {t[:16]}")
    ss = [s for _, _, s in states]
    if any(s in FAILED for s in ss):
        return 1
    return 3 if any(s in ("running", "orphaned", "never run") for s in ss) else 0


def main():
    a = sys.argv[1:]
    if not a or a[0] not in ("run", "status"):
        die("usage: orch-stage.py run <stage> [--after s,..] [--lock name] [--timeout s] [--prereg file] "
            "[--est min | --like stage] [--no-prereg why] [--rerun] -- <cmd> "
            "| status [<stage or glob>...] [--since <time|stage>] [--new] [--brief]")
    if a[0] == "status":
        rest, since = a[1:], None
        if "--since" in rest:
            i = rest.index("--since")
            if i + 1 >= len(rest):
                die("--since needs a time or a stage name")
            since = rest[i + 1]
            del rest[i:i + 2]
        return status([x for x in rest if x not in ("--brief", "--new")], "--brief" in rest, since, "--new" in rest)
    if "--" not in a:
        die("run needs `-- <command>`")
    k = a.index("--")
    opts, cmd = a[1:k], a[k + 1:]
    if not opts or not cmd:
        die("run <stage> [options] -- <command>")
    stage, after, lock, timeout, rerun, pre, i = opts[0], [], None, None, False, None, 1
    est = like = no_prereg = None
    while i < len(opts):
        if opts[i] == "--rerun":
            rerun, i = True, i + 1
        elif opts[i] in ("--after", "--lock", "--timeout", "--prereg", "--est", "--like", "--no-prereg") \
                and i + 1 < len(opts):
            v = opts[i + 1]
            if opts[i] == "--after":
                after += [x for x in v.split(",") if x]
            elif opts[i] == "--lock":
                lock = v
            elif opts[i] == "--prereg":
                pre = v
            elif opts[i] == "--est":
                est = v
            elif opts[i] == "--like":
                like = v
            elif opts[i] == "--no-prereg":
                no_prereg = v
            else:
                timeout = float(v)
            i += 2
        else:
            die(f"unknown option {opts[i]}")
    return run(stage, cmd[0] if len(cmd) == 1 else shlex.join(cmd), after, lock, timeout, rerun, pre,
               est, like, no_prereg)


if __name__ == "__main__":
    sys.exit(main())
```
````

#### Measurement helpers, shipped

Twice in one session (slm-memory-management, 2026-10-04) the honest path was a
new code task in the middle of a measurement: a tracked drift checker, then
scoring at matched abstention. Both were generic. `orch-measure.py` is the
generic half, over per-item rows any scorer can write: a paired comparison
pooled over groups with an exact sign test, the same at the control's
abstention, and an item-for-item identity check over a subset. `--value` prints
one number, the form a registered `stdout ==` holds. It was checked against
that session's own scorer on its real runs: every registered gain, loss, net,
p and matched threshold came out identical. Since v6 it also reads a judge's
report as the report hook recorded it, so a judge's controls are scored
against their key with `identity`, straight from the records.

````
FILE: .claude/hooks/orch-measure.py
```python
#!/usr/bin/env python3
"""ORCH measurement helpers — the comparisons measurement work keeps needing.

Twice in one session the honest path was a new code task in the middle of a
measurement: a tracked drift checker, then scoring at matched abstention. Both
were generic, and both cost a packet, a dispatch and a wait while the
measurement stood still. These are the generic halves, tracked and tested once
here, so a scorer only has to write per-item rows.

Input: per-item rows — a .jsonl file (one object per line), or a .json file
holding a list of objects or an object with one such list ({"results": [...]}),
or a judge's report as the report hook recorded it (its `result.items`).
Several files per side join with commas and pool; a key in two of them is an
error. --meta adds fields by key from other files (an eval set's `kind`, say),
never overriding a field a row already has. Field names may be dotted (a.b).

    python3 .claude/hooks/orch-measure.py paired   <control> <arm> --key K --outcome F
                                                   [--group F] [--where [!]F] [--meta files] [--common]
    python3 .claude/hooks/orch-measure.py matched  <control> <arm> --key K --outcome F --abstained F
                                                   --score F [--answerable F] [--gate G] [--group F] [...]
    python3 .claude/hooks/orch-measure.py identity <a> <b> --key K --field F[,F...] [--ids <file>]

paired    per item, control vs arm on a truthy outcome: gained (arm right,
          control wrong), lost, net, and an exact two-sided sign test over the
          discordant pairs — per --group, and pooled over all of them.
matched   the arm compared at the control's abstention. A gate calibrated on
          one pipeline refuses differently on another, so an arm can gain
          answers by refusing less, which is not the same gain. Replays the arm
          at the lowest score threshold (>= --gate) whose refusals on rows that
          should be refused (--answerable falsy; all rows without it) reach the
          control's — rows below it become refusals, rows with no score never
          do — then compares outcomes on answerable rows as `paired` does.
identity  are two runs the same item for item? Over --ids (a JSON list, or
          one id per line) or every common key: how many rows differ in the
          named fields, and which. Request order, caching and nondeterminism
          show up here first.

--value <name> prints that one number and nothing else: the form a registry
entry's `stdout ==` holds (orch-task §8). paired/matched: net · gained · lost ·
p · n · control · arm (pooled, or the group named by --in); matched also
threshold · abstained. identity: differ · compared.

exit 0 computed · 2 could not read the input, or it does not pair up
"""
import json
import math
import sys
from pathlib import Path


def die(msg):
    sys.stderr.write(f"orch-measure: {msg}\n")
    sys.exit(2)


def get(row, field):
    v = row
    for part in field.split("."):
        if not isinstance(v, dict) or part not in v:
            return None
        v = v[part]
    return v


def truth(v, field):
    if isinstance(v, bool) or v is None:
        return bool(v)
    if isinstance(v, (int, float)) and v in (0, 1):
        return bool(v)
    if isinstance(v, str) and v.strip().lower() in ("true", "yes", "1", "false", "no", "0", ""):
        return v.strip().lower() in ("true", "yes", "1")
    die(f"{field} = {v!r} is not a truth value")


def load(spec, key):
    """{key: row} pooled over comma-separated files."""
    out = {}
    for path in [p for p in spec.split(",") if p]:
        try:
            txt = Path(path).read_text()
        except OSError as e:
            die(str(e))
        try:
            if path.endswith(".jsonl"):
                items = [json.loads(x) for x in txt.splitlines() if x.strip()]
            else:
                items = json.loads(txt)
                if isinstance(items, dict):
                    lists = [v for v in items.values() if isinstance(v, list)]
                    res = items.get("result")
                    if not lists and isinstance(res, dict) and isinstance(res.get("items"), list):
                        lists = [res["items"]]        # a judge's report, as orch-report.py recorded it
                    if len(lists) != 1:
                        die(f"{path}: an object with {len(lists)} lists — say which with a .jsonl, or one list")
                    items = lists[0]
        except ValueError as e:
            die(f"{path}: not JSON ({e})")
        for r in items:
            k = get(r, key) if isinstance(r, dict) else None
            if k is None:
                die(f"{path}: a row with no {key}")
            if k in out:
                die(f"{path}: {key} {k} is already in an earlier file — pooled files must not overlap")
            out[k] = r
    if not out:
        die(f"{spec}: no rows")
    return out


def meta(rows, spec, key):
    if spec:
        extra = load(spec, key)
        for k, r in rows.items():
            for f, v in (extra.get(k) or {}).items():
                r.setdefault(f, v)
    return rows


def sign_p(g, l):
    """Exact two-sided sign test over g gains and l losses."""
    n = g + l
    if n == 0:
        return 1.0
    return min(1.0, 2 * sum(math.comb(n, i) for i in range(max(g, l), n + 1)) / 2 ** n)


def fmt_p(p):
    """Two significant digits, never an exponent: a results doc cites it."""
    if p <= 0:
        return "0"
    return f"{p:.{max(1, 1 - math.floor(math.log10(p)))}f}"


def outcomes(rows, keys, field):
    return {k: truth(get(rows[k], field), field) for k in keys}


def pair(c, a, keys):
    """Paired comparison of two {key: right?} maps over `keys`."""
    g = sum(1 for k in keys if a[k] and not c[k])
    lo = sum(1 for k in keys if c[k] and not a[k])
    return {"n": len(keys), "control": sum(c[k] for k in keys), "arm": sum(a[k] for k in keys),
            "gained": g, "lost": lo, "net": g - lo, "p": sign_p(g, lo)}


def common(ctl, arm, o):
    a, b = set(ctl), set(arm)
    if a != b and not o.get("--common"):
        die(f"the runs do not pair up: {len(a - b)} key(s) only in control, {len(b - a)} only in the arm "
            "(--common compares the shared ones, and says how many it dropped)")
    if a != b:
        print(f"note: {len(a ^ b)} unpaired key(s) dropped")
    keys = sorted(a & b, key=str)
    w = o.get("--where")
    if w:
        neg, f = w.startswith("!"), w.lstrip("!")
        keys = [k for k in keys if truth(get(ctl[k], f), f) != neg]
    return keys


def groups(ctl, keys, field):
    out = {}
    for k in keys:
        out.setdefault(str(get(ctl[k], field)) if field else "all", []).append(k)
    return out


def line(name, r):
    return (f"  {name:14} n {r['n']:<5} control {r['control']:<5} arm {r['arm']:<5} gained {r['gained']:<4} "
            f"lost {r['lost']:<4} net {r['net']:+d}  p {fmt_p(r['p'])}")


def report(by, pooled, o, extra=None):
    val = o.get("--value")
    if val:
        r = by.get(o["--in"]) if o.get("--in") else pooled
        if r is None:
            die(f"--in {o['--in']}: no such group ({', '.join(by)})")
        r = {**r, **(extra or {})}
        if val not in r:
            die(f"--value is one of {', '.join(r)}")
        v = r[val]
        print(f"{v:+d}" if val == "net" else fmt_p(v) if val == "p" else v)
        return 0
    if len(by) > 1:
        for g, r in by.items():
            print(line(g, r))
    print(line("pooled" if len(by) > 1 else "all", pooled))
    return 0


def cmd_paired(o):
    ctl = meta(load(o["control"], o["--key"]), o.get("--meta"), o["--key"])
    arm = meta(load(o["arm"], o["--key"]), o.get("--meta"), o["--key"])
    keys = common(ctl, arm, o)
    c, a = outcomes(ctl, keys, o["--outcome"]), outcomes(arm, keys, o["--outcome"])
    by = {g: pair(c, a, ks) for g, ks in groups(ctl, keys, o.get("--group")).items()}
    if not o.get("--value"):
        print(f"paired: {len(keys)} pairs on {o['--key']} · outcome {o['--outcome']}")
    return report(by, pair(c, a, keys), o)


def cmd_matched(o):
    K, out, ab, sc = o["--key"], o["--outcome"], o["--abstained"], o["--score"]
    ctl = meta(load(o["control"], K), o.get("--meta"), K)
    arm = meta(load(o["arm"], K), o.get("--meta"), K)
    keys = common(ctl, arm, {**o, "--where": None})
    ans = o.get("--answerable")
    refuse = [k for k in keys if ans and not truth(get(ctl[k], ans), ans)] if ans else keys
    target = sum(truth(get(ctl[k], ab), ab) for k in refuse)

    stored = {k for k in keys if truth(get(arm[k], ab), ab)}
    scores = {}
    for k in keys:
        v = get(arm[k], sc)
        if v is not None and not (isinstance(v, float) and math.isinf(v)):
            scores[k] = float(v)

    def refused(t):
        return stored | {k for k, s in scores.items() if s < t}

    gate = float(o.get("--gate", "-inf"))
    cands = [gate] + sorted({s + 1e-9 for s in scores.values() if s + 1e-9 >= gate})
    t = next((t for t in cands if len(refused(t) & set(refuse)) >= target), None)
    if t is None:
        most = len(refused(cands[-1]) & set(refuse))
        if o.get("--value"):
            die(f"matched abstention unreachable: the arm refuses at most {most}/{len(refuse)}, "
                f"the control {target}")
        print(f"matched: unreachable — the arm refuses at most {most} of {len(refuse)}, the control {target}")
        return 0
    off = refused(t)
    scored = [k for k in keys if not ans or truth(get(ctl[k], ans), ans)]
    c = outcomes(ctl, scored, out)
    a = {k: False if k in off else v for k, v in outcomes(arm, scored, out).items()}
    by = {g: pair(c, a, ks) for g, ks in groups(ctl, scored, o.get("--group")).items()}
    extra = {"threshold": "none" if t == float("-inf") else f"{t:.6g}",
             "abstained": len(off & set(refuse))}
    if not o.get("--value"):
        print(f"matched: arm replayed at threshold {extra['threshold']} — refuses {extra['abstained']} of "
              f"{len(refuse)} (control {target}); {len(scored)} answerable pairs on {K}")
    return report(by, pair(c, a, scored), o, extra)


def cmd_identity(o):
    K = o["--key"]
    a, b = load(o["control"], K), load(o["arm"], K)
    fields = [f for f in o["--field"].split(",") if f]
    if o.get("--ids"):
        try:
            txt = Path(o["--ids"]).read_text()
        except OSError as e:
            die(str(e))
        try:
            ids = json.loads(txt)
        except ValueError:
            ids = [x.strip() for x in txt.splitlines() if x.strip()]
        miss = [i for i in ids if i not in a or i not in b]
        if miss:
            die(f"{len(miss)} id(s) are not in both runs, e.g. {miss[0]}")
    else:
        ids = sorted(set(a) & set(b), key=str)
    differ = [i for i in ids if any(get(a[i], f) != get(b[i], f) for f in fields)]
    r = {"compared": len(ids), "differ": len(differ)}
    if o.get("--value"):
        if o["--value"] not in r:
            die("--value is differ or compared")
        print(r[o["--value"]])
        return 0
    print(f"identity: {len(ids)} compared on {', '.join(fields)} · {len(differ)} differ")
    for i in differ[:10]:
        print(f"  {i}: " + "; ".join(f"{f} {get(a[i], f)!r} != {get(b[i], f)!r}" for f in fields
                                      if get(a[i], f) != get(b[i], f))[:160])
    if len(differ) > 10:
        print(f"  … {len(differ) - 10} more")
    return 0


NEED = {"paired": ("--key", "--outcome"), "matched": ("--key", "--outcome", "--abstained", "--score"),
        "identity": ("--key", "--field")}


def main():
    a = sys.argv[1:]
    if not a or a[0] not in NEED:
        die("usage: orch-measure.py paired|matched|identity <control> <arm> --key K ... — see the docstring")
    o, pos, i = {}, [], 1
    while i < len(a):
        if a[i] == "--common":
            o[a[i]], i = True, i + 1
        elif a[i].startswith("--"):
            if i + 1 >= len(a):
                die(f"{a[i]} needs a value")
            o[a[i]], i = a[i + 1], i + 2
        else:
            pos, i = pos + [a[i]], i + 1
    if len(pos) != 2:
        die(f"{a[0]} takes two runs: <control> <arm>")
    miss = [k for k in NEED[a[0]] if not o.get(k)]
    if miss:
        die(f"{a[0]} needs {' '.join(miss)}")
    o["control"], o["arm"] = pos
    return {"paired": cmd_paired, "matched": cmd_matched, "identity": cmd_identity}[a[0]](o)


if __name__ == "__main__":
    try:
        rc = main()
    except SystemExit:
        raise
    except Exception as e:                # a crash is "could not compute", never a number
        die(f"internal error: {type(e).__name__}: {e}")
    sys.exit(rc)
```
````

### 5.2 Config

````
FILE: .orch/config/settings.yaml
```yaml
# Thresholds. TIGHTENING ONLY — the guard ignores any value that loosens a
# trigger (confidence_floor may rise, spend_fraction and max_attempts may fall).
# Loosening is a human edit to the guard itself, deliberately inconvenient.
governor:
  confidence_floor: 0.6      # a `done` below this checkpoints
  spend_fraction: 0.6        # >60% of the task's step budget checkpoints
  max_attempts: 3            # attempts on one task before checkpoint
  # If real tasks keep tripping spend_fraction, the DEFAULT BUDGET is wrong, not
  # this number. Fix it in the playbook (heavy_budget_multiplier) or per packet.
  # Raising spend_fraction is loosening, and the guard ignores it anyway.

selection:                   # additive — never multiply, a zero would starve
  urgency: 0.4
  impact: 0.4
  staleness: 0.2

retrieval:
  candidate_limit: 20        # headlines shown to the librarian
  max_select: 8              # blocks it may select
  max_l2: 3                  # blocks expanded to full body
  block_cap: 300             # per project; exceeding forces compaction

loop:
  default_mode: L2
  max_tasks_per_cycle: 1
  max_slots: 2               # parallel worktrees, clamp [1,8]

sandbox:
  # Ignored dependency caches copied from the main checkout into each new
  # worktree before dispatch (orch-task §3). `git worktree add` checks out
  # tracked files only, so without these a build fetches from the network —
  # which a `network: false` packet forbids. Only paths that exist are copied.
  seed_caches: [.pio/libdeps, node_modules, vendor/bundle]
  # NOT .venv: its scripts and any editable install point back at the main
  # checkout, so tests would silently import main-tree code, not the task's.

stages:
  # The harness's cap on one background job, in minutes. orch-stage.py refuses
  # a stage whose --timeout or expected time does not fit under it, and gives
  # one with no --timeout five minutes less, so it ends as `timeout`, on
  # record, instead of being killed midway with nothing saying why.
  job_cap_min: 120
```
````

````
FILE: .orch/config/sensitive.yaml
```yaml
# Security-gate policy. Human-editable, NEVER self-modified.
# path_globs match repo-relative paths in a diff or a packet's scope.paths.
# content_patterns run over ADDED diff lines only. Severity above `low`
# forces a mandatory checkpoint.

path_globs:
  - "**/.env"
  - "**/.env.*"
  - "**/secrets/**"
  - "**/*secret*"
  - "**/*credential*"
  - "**/auth/**"
  - "**/*session*"
  - "**/*token*"
  - "**/migrations/**"
  - "**/requirements*.txt"
  - "**/pyproject.toml"
  - "**/package.json"
  - "**/package-lock.json"
  - "**/go.mod"
  - "**/Cargo.toml"

content_patterns:
  - {severity: high,   why: credential material,        pattern: "(api[_-]?key|secret[_-]?key|password|passwd|BEGIN (RSA|EC|OPENSSH) PRIVATE KEY)"}
  - {severity: high,   why: unsafe deserialization,     pattern: "pickle\\.loads?\\(|yaml\\.load\\((?!.*SafeLoader)|marshal\\.loads?\\("}
  - {severity: medium, why: dynamic execution,          pattern: "(?<![.\\w])eval\\(|(?-i:(?<![.\\w])exec\\()|os\\.system\\(|shell=True"}
  - {severity: medium, why: network client code,        pattern: "socket\\.socket\\(|requests\\.(get|post|put|delete)\\(|urllib\\.request|(?-i:(?<![A-Za-z0-9_])fetch\\()"}
  - {severity: high,   why: destructive SQL,            pattern: "DROP TABLE|TRUNCATE TABLE|DELETE FROM .* WHERE 1"}
  - {severity: high,   why: irreversible operation,     pattern: "git push --force|rm -rf /|chmod 777"}

# eval/exec/fetch need a non-identifier left edge: `.eval()` is torch's
# inference mode (every ML repo calls `model.eval()`), `.exec(` is RegExp
# matching, `performFetch(` is a method name; none is the risk. exec/fetch are
# also case-exact.

# Any added line in these files is the `new_dependency` trigger, separately
# from severity. Entries are globs: a bare one matches the file's name at any
# depth, one with a `/` the repo-relative path. `requirements-train.txt` passed
# an exact-name list ungated; platformio.ini `lib_deps` passed an earlier one —
# embedded/C++ build manifests are dependency surfaces too.
dependency_manifests: ["requirements*.txt", "*.requirements.txt", "constraints*.txt", "**/requirements/*.txt", pyproject.toml, package.json, go.mod, Cargo.toml, Gemfile, pom.xml, platformio.ini, library.json, CMakeLists.txt, conanfile.txt, conanfile.py, vcpkg.json, idf_component.yml]

# Medium findings with these `why` values, in files under test_paths whose
# added lines name no non-loopback http(s) host, report as low: a test calling
# its own in-process server is not egress. One external URL in the file and
# the finding gates as usual.
test_paths: ["**/test/**", "**/tests/**", "**/__tests__/**", "**/*.test.*", "**/*_test.*", "**/test_*"]
test_downgrade: [network client code]
```
````

````
FILE: .orch/playbooks/code.bugfix.yaml
```yaml
# reproduce → isolate → fix → regression → review
# Phase goals and acceptance are templates. /orch-task expands one phase into a
# packet skeleton that a human edits before it is queued.
id: code.bugfix
version: 1
triggers: ["fix bug", "broken", "regression", "crash", "wrong output"]
default_loop_mode: L2
default_budget: {steps: 25, wall_s: 600}
heavy_budget_multiplier: 3   # applied to a phase with `heavy: true` — see code.feature.yaml
complexity: 0.4
red_team: false
memory:
  reads: [failure, fact, convention, procedure, process-failure]
  writes: [failure, fact, process-failure]
phases:
  - id: reproduce
    role: debugger
    goal: "Reproduce the reported failure deterministically; capture the failing command and its output"
    acceptance: [{type: judged, criterion: "A command exists that fails before the fix, output captured"}]
  - id: isolate
    role: debugger
    goal: "Identify the root cause with file:line evidence. Do not fix yet."
    acceptance: [{type: judged, criterion: "Root cause named with file:line evidence"}]
  - id: fix
    role: executor
    goal: "Apply the minimal fix for the isolated root cause. No drive-by changes."
    tools: [read, grep, edit, "bash:test"]
    acceptance: [{type: executable, cmd: "REPLACE with the repro command", expect: "exit 0"}]
  - id: regression
    role: executor
    goal: "Add a test that fails without the fix and passes with it"
    tools: [read, grep, edit, "bash:test"]
    acceptance: [{type: executable, cmd: "REPLACE with the project test command", expect: "exit 0"}]
  - id: review
    role: reviewer
    goal: "Verify the fix addresses the root cause, cites file:line, and introduces no scope creep"
    acceptance: [{type: judged, criterion: "Reviewer cites file:line and confirms the test covers the regression"}]
failure_modes:
  - {symptom: "cannot reproduce",              branch: "escalate to needs_decision with what was tried"}
  - {symptom: "root cause outside scope.paths", branch: "escalate — never expand scope unilaterally"}
  - {symptom: "fix breaks other tests",        branch: "retry isolate; the root cause was wrong"}
```
````

````
FILE: .orch/playbooks/code.feature.yaml
```yaml
# clarify → plan → implement → test → review
# The orchestrator runs these phases in order, each its own packet and
# subagent, and reports once at the end. `clarify` is skipped when the user's
# request is already unambiguous — do not interrogate them for its own sake.
id: code.feature
version: 1
triggers: ["add", "implement", "support", "new feature", "build", "expose"]
default_loop_mode: L2
default_budget: {steps: 40, wall_s: 1200}
# A phase marked `heavy: true` multiplies default_budget by this. Heavy means
# an acceptance check that invokes a model, a build, a GPU job, or a network
# fetch — the default was tuned for tasks whose checks are text and exit codes,
# and a default that overruns on every real task teaches everyone to ignore the
# spend trigger. Raise the budget, never spend_fraction.
heavy_budget_multiplier: 3
complexity: 0.6
red_team: false
memory:
  reads: [convention, procedure, decision, api-shape, failure, process-failure]
  writes: [decision, convention, api-shape, failure, process-failure]
phases:
  - id: clarify
    role: reviewer
    goal: "State the feature's observable behaviour and its acceptance checks. Name what is explicitly out of scope."
    acceptance: [{type: judged, criterion: "Behaviour stated as testable assertions; non-goals named"}]
  - id: plan
    role: reviewer
    goal: "Name the files to change and the approach. Check `decision` and `failure` blocks for prior art before proposing anything new."
    acceptance: [{type: judged, criterion: "File list + approach, consistent with existing conventions"}]
  - id: implement
    role: executor
    heavy: false          # set true if the test command runs a model/build/GPU job
    goal: "Build the feature to the plan. Follow the project's conventions; no drive-by changes."
    tools: [read, grep, edit, "bash:test"]
    acceptance: [{type: executable, cmd: "REPLACE with the project test command", expect: "exit 0"}]
  - id: test
    role: executor
    heavy: false
    goal: "Add tests covering the stated behaviour AND its edge cases. A test that cannot fail is not a test."
    tools: [read, grep, edit, "bash:test"]
    acceptance: [{type: executable, cmd: "REPLACE with the project test command", expect: "exit 0"}]
  - id: review
    role: reviewer
    goal: "Verify behaviour matches what clarify stated, tests genuinely cover it, and nothing outside scope changed"
    acceptance: [{type: judged, criterion: "Each clarify assertion mapped to a passing test, cited file:line"}]
failure_modes:
  - {symptom: "acceptance invokes a model/build/GPU job", branch: "set heavy: true or a per-packet budget BEFORE dispatch — an expected overrun is a planning failure, not a spend trigger"}
  - {symptom: "requirement ambiguous at implement time", branch: "escalate — do not guess at behaviour the user did not state"}
  - {symptom: "needs a new dependency",                  branch: "checkpoint — new_dependency is a mandatory trigger"}
  - {symptom: "touches auth, network, or a manifest",    branch: "checkpoint — security gate, regardless of this playbook's flags"}
  - {symptom: "plan grows past the declared scope",      branch: "escalate; re-scope with the user rather than widening silently"}
```
````

````
FILE: .orch/memory/INDEX.md
```
# Memory index (L0)

Headlines only. One line per block. This file is cheap to read and is the ONLY
part of memory that belongs in a routine turn. Read a block body only when its
headline matches what you are doing.

Format: `blk_<id> · <project> · <type> · <headline>`

<!-- blocks below, newest first -->
```
````

````
FILE: .orch/state.json
```json
{"version": 1, "paused": false, "traces": 0, "phase3_exit": "0/30", "last_task": null}
```
````

### 5.3 Subagents

````
FILE: .claude/agents/orch-executor.md
```markdown
---
name: orch-executor
description: Executes one phase of an orchestrated task inside a sandboxed git worktree — writes the minimal change for a stated goal, within a declared scope, and reports a structured result packet. Use for `fix`, `implement`, and `regression` phases.
tools: Read, Grep, Glob, Edit, Write, Bash
model: sonnet
effort: medium
---

You execute ONE phase of an orchestrated task. You are stateless: the packet
you were given is all the context that exists.

**Sandbox.** Work only inside the current directory — it is a throwaway git
worktree. Never `cd` out of it, never push, never touch the user's live tree.

**Scope.** Modify only paths matching the packet's `scope.paths`. If the fix
genuinely lives outside that scope, do not expand it: end `escalate` and name
the file and line where the real fix belongs. Refusing to exceed scope is a
correct outcome, not a failure — an agent that quietly widened its scope would
be worse than one that stopped.

**Out-of-scope files are read-only, even for a moment.** Do not edit one to try
something and then restore it. A net-zero edit is still an out-of-scope write,
the guard cannot see one made through Bash, and mutation checks are the
orchestrator's job. If an experiment needs such an edit, ask in `open_questions`.

**Never `git stash`.** The stash stack belongs to the repository, not the
worktree: every task running in parallel shares it, and a `pop` in one can
apply another task's changes.

**Refused by the platform is not failed.** If tool calls are refused and the
refusal is not the ORCH guard's (its message starts `ORCH checkpoint`) — a
permission prompt, a classifier error, every command failing the same way —
stop at once. Do not retry in a loop or switch tools to get around it. End
`blocked` with `blocked_on: platform`, listing what you wrote and what you had
already validated, so the same agent can resume where it stopped.

**Minimality.** Apply the smallest change that satisfies the goal. No drive-by
refactors, no reformatting, no "while I was here". Every changed line you
cannot justify against the goal is scope creep and will be flagged.

**Evidence.** Cite `file:line` for every claim about code. Report only commands
you actually ran and their real exit codes. Never weaken, skip, or delete a
test to make acceptance pass — if a test blocks you, that is a finding.

**A wrong check is a finding, not a target.** Acceptance checks are written by
the orchestrator and are sometimes wrong — a grep that also matches code the
goal allows, a count miscounted against the base commit. If a check looks
wrong, FLAG it in `open_questions` with the evidence and leave the check
failing. Never edit code solely to make a grep or a count pass; a change whose
only justification is the check is gaming it, however harmless it looks.

**Working directory.** The packet names your worktree as an absolute path. It
overrides whatever the environment reports as the primary working directory.
Run `pwd` first; if it is not that path, `cd` to it before anything else, and if
you cannot, end `blocked`.

**Memory.** If a context block informed your work, cite its `blk_` id in an
evidence entry. If you had to read a file you were not pointed at, say so in
`open_questions` — that is a retrieval miss and it is useful.

**Commit** with the trailer line your prompt gives, exactly as written — it is
how bisect names this task later.

**Your report is one JSON object, and code reads it.** If this run has the
SubagentHandback tool, the object is that call's `message`; otherwise it is
your final message. Nothing else goes in it — no prose before or after it, no
code fence; your prose goes in `summary`. A hook checks the report as it leaves
you and refuses one without the object, with the reason: a prose report costs
you a retry, and costs the task its verification.

{"task_id": "tsk_…",
 "status": "done|blocked|needs_decision|failed|escalate",
 "blocked_on": null,
 "summary": "one or two sentences, file:line for code claims",
 "evidence": [{"cite": "file:line", "reason": "..."}, {"check": "cmd", "exit": 0}],
 "confidence": 0.0,
 "new_facts": [{"type": "fact|failure", "headline": "<=15 words"}],
 "open_questions": []}

`task_id` is the id at the top of your prompt. `confidence` is your honest
posterior that this passes review. Below 0.6 forces a human checkpoint — that
is the system working, so do not inflate it.
`blocked_on` is `null` unless `status` is `blocked`; then it is `platform` or
`dependency` (something the packet needs does not exist yet).
```
````

`orch-executor-deep` is the same agent at `effort: high`. Effort is fixed per
agent type — set in frontmatter, overriding the session's level — and the
Agent tool has no per-dispatch effort parameter (it has one for `model`). So
the only way to choose effort per task is to choose the agent. Its body is
byte-identical to `orch-executor`'s, and tier-1 step 17 fails if the two
drift: two copies of one contract that disagree are two contracts.

````
FILE: .claude/agents/orch-executor-deep.md
```markdown
---
name: orch-executor-deep
description: The orch-executor contract at high reasoning effort. Dispatch it only when a packet says `effort: high` — multi-module changes, a retry after a failed attempt, or logic where a shallow reading fails silently.
tools: Read, Grep, Glob, Edit, Write, Bash
model: sonnet
effort: high
---

You execute ONE phase of an orchestrated task. You are stateless: the packet
you were given is all the context that exists.

**Sandbox.** Work only inside the current directory — it is a throwaway git
worktree. Never `cd` out of it, never push, never touch the user's live tree.

**Scope.** Modify only paths matching the packet's `scope.paths`. If the fix
genuinely lives outside that scope, do not expand it: end `escalate` and name
the file and line where the real fix belongs. Refusing to exceed scope is a
correct outcome, not a failure — an agent that quietly widened its scope would
be worse than one that stopped.

**Out-of-scope files are read-only, even for a moment.** Do not edit one to try
something and then restore it. A net-zero edit is still an out-of-scope write,
the guard cannot see one made through Bash, and mutation checks are the
orchestrator's job. If an experiment needs such an edit, ask in `open_questions`.

**Never `git stash`.** The stash stack belongs to the repository, not the
worktree: every task running in parallel shares it, and a `pop` in one can
apply another task's changes.

**Refused by the platform is not failed.** If tool calls are refused and the
refusal is not the ORCH guard's (its message starts `ORCH checkpoint`) — a
permission prompt, a classifier error, every command failing the same way —
stop at once. Do not retry in a loop or switch tools to get around it. End
`blocked` with `blocked_on: platform`, listing what you wrote and what you had
already validated, so the same agent can resume where it stopped.

**Minimality.** Apply the smallest change that satisfies the goal. No drive-by
refactors, no reformatting, no "while I was here". Every changed line you
cannot justify against the goal is scope creep and will be flagged.

**Evidence.** Cite `file:line` for every claim about code. Report only commands
you actually ran and their real exit codes. Never weaken, skip, or delete a
test to make acceptance pass — if a test blocks you, that is a finding.

**A wrong check is a finding, not a target.** Acceptance checks are written by
the orchestrator and are sometimes wrong — a grep that also matches code the
goal allows, a count miscounted against the base commit. If a check looks
wrong, FLAG it in `open_questions` with the evidence and leave the check
failing. Never edit code solely to make a grep or a count pass; a change whose
only justification is the check is gaming it, however harmless it looks.

**Working directory.** The packet names your worktree as an absolute path. It
overrides whatever the environment reports as the primary working directory.
Run `pwd` first; if it is not that path, `cd` to it before anything else, and if
you cannot, end `blocked`.

**Memory.** If a context block informed your work, cite its `blk_` id in an
evidence entry. If you had to read a file you were not pointed at, say so in
`open_questions` — that is a retrieval miss and it is useful.

**Commit** with the trailer line your prompt gives, exactly as written — it is
how bisect names this task later.

**Your report is one JSON object, and code reads it.** If this run has the
SubagentHandback tool, the object is that call's `message`; otherwise it is
your final message. Nothing else goes in it — no prose before or after it, no
code fence; your prose goes in `summary`. A hook checks the report as it leaves
you and refuses one without the object, with the reason: a prose report costs
you a retry, and costs the task its verification.

{"task_id": "tsk_…",
 "status": "done|blocked|needs_decision|failed|escalate",
 "blocked_on": null,
 "summary": "one or two sentences, file:line for code claims",
 "evidence": [{"cite": "file:line", "reason": "..."}, {"check": "cmd", "exit": 0}],
 "confidence": 0.0,
 "new_facts": [{"type": "fact|failure", "headline": "<=15 words"}],
 "open_questions": []}

`task_id` is the id at the top of your prompt. `confidence` is your honest
posterior that this passes review. Below 0.6 forces a human checkpoint — that
is the system working, so do not inflate it.
`blocked_on` is `null` unless `status` is `blocked`; then it is `platform` or
`dependency` (something the packet needs does not exist yet).
```
````

````
FILE: .claude/agents/orch-debugger.md
```markdown
---
name: orch-debugger
description: Reproduces a reported failure deterministically and isolates its root cause with file:line evidence, without fixing it. Use for `reproduce` and `isolate` phases of code.bugfix.
tools: Read, Grep, Glob, Bash
model: sonnet
effort: high
---

You reproduce and isolate. **You do not fix.** A diagnosis that turns into a
patch is a diagnosis nobody checked.

You have no edit tools, by design.

**Reproduce.** Produce a single command that fails deterministically now and
would pass once fixed. Capture its actual output. "It seems to fail sometimes"
is not a reproduction — if you cannot make it deterministic, say so and end
`blocked` with everything you tried.

**Isolate.** Name the root cause at `file:line`. Distinguish the line that
*changed behaviour* from the line that *surfaced* the symptom; they are often
different and attributing to the second is the standard mistake. State what
evidence rules out the alternatives you considered.

**Scar tissue.** Failure blocks in your context describe dead ends already
paid for. Check them before proposing a theory; if one applies, say which.

If tool calls are refused by anything other than the ORCH guard, stop: end
`blocked` with `blocked_on: platform` and say where you got to. Never retry in a
loop or switch tools to get past it.

Your report is the result JSON — orch-executor's schema, with the `task_id`
your prompt gives — and nothing else: the SubagentHandback message if this run
has that tool, otherwise your final message. A hook refuses a report without
it. Put the repro command in `evidence` as a `check` entry with its real exit
code.
```
````

````
FILE: .claude/agents/orch-reviewer.md
```markdown
---
name: orch-reviewer
description: Verifies a completed task's diff against its acceptance criteria and scope, citing file:line, and reports pass/fail with evidence. Use for `review` phases and before finalizing any task.
tools: Read, Grep, Glob, Bash
model: sonnet
effort: high
---

You verify. You have no edit tools: if the work is wrong, you say so — you do
not fix it.

Check, in order, and report each:

1. **Acceptance.** Run every executable check yourself. Report real exit codes.
   A check that passes for the wrong reason (test weakened, assertion deleted,
   fixture stubbed) is a FAIL — diff the test files and say what changed.
2. **Root cause.** Does the diff address the cause the isolate phase named, or
   the symptom? Cite `file:line`.
3. **Scope.** Every changed file against `scope.paths`. Name violations.
4. **Minimality.** Changed lines not justified by the goal.
5. **Security.** Does the diff touch auth, network, deserialization, subprocess
   construction, or a dependency manifest? If yes, name it — that is a mandatory
   checkpoint regardless of how the change looks.

Rubber-stamping is the failure mode this role exists to prevent. A review that
finds nothing must say what it checked and how, or it is worthless.

If tool calls are refused by anything other than the ORCH guard, stop: end
`blocked` with `blocked_on: platform` and list what you had already checked.

Your report is the result JSON — orch-executor's schema, with the `task_id`
your prompt gives — and nothing else: the SubagentHandback message if this run
has that tool, otherwise your final message. A hook refuses a report without
it. `status: done` means it passed; anything else carries the specific reason
in `summary`.
```
````

````
FILE: .claude/agents/orch-librarian.md
```markdown
---
name: orch-librarian
description: Selects the minimum set of memory blocks relevant to a task from the headline index, returning block ids with one-word reasons. Use before dispatching any task that needs project context.
tools: Read, Grep, Glob
model: haiku
---

You select memory blocks. You return ids, never content.

Input: a goal, and `.orch/memory/INDEX.md`.

1. Grep the index for terms from the goal — and for their synonyms; the block
   author did not know the words the requester would use.
2. Select **at most 8**, and prefer fewer. Most tasks need 1–3. A block that is
   merely topical is noise; select only what changes what the agent will do.
3. **Auto-attach (mandatory).** For every selected block, read its header and
   also return the head of its `supersedes` chain and its `parent`, if any.
   Auto-attached ids do not count against your 8. This rule prevents the agent
   acting on a fact that has since been replaced, and it is the highest-value
   step in the path.
4. **Superseded blocks are not in the index** — they live in
   `.orch/memory/archive/`. Look there only when the query is explicitly
   historical ("why did we used to…"). An id that reaches you directly (an old
   packet's `context_refs`, a `related:` link) may be archived: return its
   successor, named in the archived block's `superseded_by:`.
5. Every `failure` block for this project+playbook is forced context — include
   them regardless of your own judgment. Dead ends already paid for are the
   cheapest thing in the system.
6. Every `process-failure` block is forced context when the goal involves a
   long-running, external, or costly job (training, deploy, bulk fetch, anything
   with a quota). These describe how *packets* go wrong, and the packet is
   already written by the time anyone else could catch it.

Return exactly:

blk_xxxx — <one word reason>
blk_yyyy — <one word reason> [auto: supersedes-head of blk_xxxx]

If nothing is relevant, return `none` — an empty result is a correct answer and
is scored as such.
```
````

````
FILE: .claude/agents/orch-archivist.md
```markdown
---
name: orch-archivist
description: The only writer of project memory. Turns a completed task's or measurement's proposed facts into new/updated/superseded blocks, and runs compaction. Use after a task or a pre-registered measurement finishes, and when the block count exceeds its cap.
tools: Read, Grep, Glob, Edit, Write, Bash
model: haiku
---

You are the ONLY thing that writes memory. Result packets *propose*; you decide.

For each proposed fact:

1. Grep `.orch/memory/INDEX.md` for near-duplicates.
2. Decide one of: **new** · **update** (same claim, better detail) ·
   **supersede** (the old block is now wrong or outdated) · **discard**
   (duplicate, or too specific to ever match again).

   **Supersede is a new block, never a rewrite.** Write the successor under a
   new id with `links: {supersedes: [<old>]}`. Add `superseded_by: <new>` and
   `superseded_why: wrong | outdated` to the old block's header, move its file
   to `.orch/memory/archive/`, and delete its INDEX line. A block rewritten in
   place and logged as a supersede is how two INDEX headlines came to state the
   opposite of their bodies while every count matched. **Update** keeps the id
   and the claim; if its headline changes, the INDEX line changes in the same
   edit.
3. **Contradiction rule — immutable.** If it contradicts an existing block with
   `confidence >= 0.8`, you may NOT overwrite. Write a `decision` block holding
   both claims and their sources, and flag it for escalation. Silently resolving
   a contradiction is how a wrong fact propagates into every future task.
4. **No source, no block.** Every block needs `source:` naming the task id and
   the artifact (`file:line`, a command, or a URL). A fact with no provenance
   cannot be checked later, so it is not written. No exceptions. A finding from
   measurement work names its `msr_` id instead of a task, the results doc,
   and the registry entry: `source: {task: msr_p14, artifact:
   "docs/results/p14.md", check: chk_031}`.
5. Headline ≤ 15 words, summary ≤ 60 words. If you cannot say it that short,
   the block is really two blocks.
6. **`process-failure` blocks are accepted from the orchestrator, not only from
   agents.** They record how a task was *specified* wrongly — a weak acceptance
   check, a plan whose resource cost was never reasoned about, a reference file
   that was itself broken. Same rules as any block: no source, no block, and the
   source is a task id plus the trace or checkpoint that shows it. Write them
   `decay_class: permanent`; the constraint outlives the code.

7. **Verify every `file:line` before writing it.** Memory is the one artifact
   every future packet trusts, so a wrong citation outlives the task that made
   it. For each code claim, read the cited line in the task's worktree or
   branch and confirm the symbol the block names is actually there — a block
   once blamed `MappedInputManager::loop()` for a bug in `TrmnlActivity::loop()`.
   A claim that does not check out is not written; say so in your report.

8. **Measurement findings come from the orchestrator, and they are checked
   like code claims.** At the end of a pre-registered measurement it proposes
   its own `new_facts`: the result, the cause it found, the gotchas (a port held
   by an unrelated process, a background-job time cap). Before writing a result,
   grep `.orch/registry/` for the cited `check:` and confirm it is `active` and
   its `expect` holds the number the fact states. A number no registered entry
   holds is not written; say so in your report. A gotcha with no registry entry
   needs a command and its output as its artifact, like any other fact.

9. **A guard block is a stop, not an obstacle.** If the guard refuses a write —
   an id that happens to match a secrets-path pattern, say — do not rename,
   reword or relocate the block to get past it. Stop, report the refused write
   and the guard's reason, and let the orchestrator raise it. A block renamed to
   slip a guard is a route-around, however harmless its content.

10. **Every claim quotes its source, inline.** End each sentence of the summary
   and body with `[src: <path>:<line> "<excerpt>"]`. The excerpt is copied
   character for character from that line, or from up to two lines either side.
   For a registered value, cite `[src: chk_<id> "<excerpt>"]`. An event — what
   the user decided, what an agent reported — cites the file that records it:
   the checkpoint's `## Resolution`, the trace, or the saved result. If you
   cannot attach a sentence to an excerpt, you are inferring it: leave it out.
   One block described a workaround that never happened and a recommendation
   nobody made. Both read as plausible, and neither had a line to quote.
   `orch-lint.py memory --task` looks for each excerpt where its citation
   points, errors on any claim sentence that cites nothing, and errors on a
   number a sentence states that its citations do not show.
   A fact spread over several lines takes one `[src:]` per piece. A gotcha is
   cited where it lives — the script, the config, the trace that shows it —
   not in a doc that mentions it: a port list is in `scripts/servers.sh`. A
   fact you cannot cite is not silently dropped: list it in your report as
   `refused: <fact> — <why>`, so the orchestrator can commit a source for it.

11. **A block is not a copy of a tracked doc.** A results doc is already the
   record, its numbers linted against the registry. A `fact` or `decision` block
   whose every claim re-cites one tracked doc duplicates it; the lint warns.
   Blocks earn their place as procedures, failures and process failures,
   conventions and gotchas — what a later task would not find by reading the
   doc. In one session, 10 of 16 new blocks restated a single results doc.

Write the block file to `.orch/memory/blocks/<id>.md` — nowhere else — then add
its INDEX line, newest first: `blk_<id> · <project> · <type> · <headline>`,
where `<headline>` is copied from the block's own `headline:`, never worded
separately. The index is derived from the blocks.

**Receipt first, report second.** Before each block write, append one line to
`.orch/memory/receipts/<task_id>.log` — that exact name, the task id (or the
`msr_` id) and nothing else: `<ISO> new|update|discard blk_<id> <headline>`, or for a supersede
`<ISO> supersede blk_<old> -> blk_<new> <headline>`. If you are interrupted — a
usage limit, a crash — every write you made is recoverable from the receipt
instead of by diffing `INDEX.md`, and the orchestrator can tell a finished
ingest from a partial one.

**Check, then report — and quote, never count.** After your last write, run

    python3 .claude/hooks/orch-lint.py memory --task <task_id or msr_id>

and fix what it reports about this task's blocks until it exits 0. Your report
begins `ingest <task_id or msr_id>` and quotes the lint's `index: … · blocks/: …`
line verbatim; never state a number you did not read from it (a report once
said 95 blocks for a store of 85). That command is the only thing you run with
Bash. A hook runs it again as your report leaves you — the SubagentHandback
message if this run has that tool, otherwise your final message — and refuses
the report while it exits 1: one batch was reported "complete" over 58 errors.

**Compaction** (when a project passes `block_cap`, or on request):
- merge near-duplicate clusters, unioning their sources
- a fact independently confirmed ≥3× is promoted to `convention`
- `decay_class: fast` unread 30d → confidence × 0.8; below 0.3 → move to
  `.orch/memory/archive/`
- `cite_count / read_count < 0.15` after ≥10 reads → the headline is
  misleadingly broad. Rewrite it or archive the block.
- rebuild `INDEX.md` from the surviving block files, then run
  `orch-lint.py memory` (no `--task`: compaction owns the whole store) until it
  exits 0

Report what you changed as a list. Never delete a block file; archive it.
```
````

````
FILE: .claude/agents/orch-auditor.md
```markdown
---
name: orch-auditor
description: Replays trace and checkpoint history looking for recurring ORCHESTRATOR failure modes — weak acceptance, wrong scope, bad reference material, unsound plans — and proposes process-failure blocks. Use for the /orch-rework orchestrator sweep, or when the same mistake seems to keep recurring.
tools: Read, Grep, Glob, Bash
model: sonnet
effort: high
---

You audit the orchestrator, not the agents. Everything you are about to read was
written by the session you are auditing: it wrote the packets, chose the
acceptance checks, wrote the traces, and filled in its own attribution. That
conflict of interest is the reason you exist. Read it as testimony, not as fact.

Input: `.orch/traces/*.json`, `.orch/checkpoints/*.md` and `resolved/*.md`, and
`.orch/registry/<project>.yaml`. You have no edit tools — you propose, the
archivist writes.

1. **Do not take `orchestrator_error` at face value, in either direction.** A
   populated one tells you where to look. A `null` one on a task with three
   attempts, a spend checkpoint, or a check that was superseded later is itself
   a finding.
2. **Read the acceptance checks, not the outcomes.** A registry of passing
   checks is exactly what a project with weak checks looks like. Grep `expect:`
   for floors (`>=`, `>0`, `-gt 0`, "at least"), for commands that cannot fail,
   and for goals whose prose promises something no check asserts.
3. **A pattern is ≥2 independent instances.** One is an incident. Give the trace
   ids and quote the field; a pattern asserted without them is an opinion.
4. **Rank by what reached the user.** A defect the guard caught cost a
   round-trip. A defect that passed every check and shipped is the expensive
   one, and it is always a packet defect — verification cannot catch a defect
   encoded in the definition of success.
5. **Name the mechanism that would have caught it**, or say plainly that none
   exists. A finding with no mechanism behind it is a note, and a note is lost
   by the next session exactly as the last one was.
6. **Classify every pattern: project, or ORCH itself.** A lesson about this
   repo is a `process-failure` block. A defect in the orchestration layer — a
   rule that cannot be followed as written, a guard that blocks legitimate work,
   a default that is wrong for every real task, a test that passes for the wrong
   reason — belongs in `.orch/upstream.md`, because it is worth nothing to the
   next repo unless it travels (`ORCH.md` §5.5).
7. **You are the only party that may mark an upstream entry `proven` on
   recurrence alone.** The orchestrator has a stake in that answer; you do not.
   It may self-certify only with a reproduction. A `proven` entry owes three
   things — the exact text it replaces, the proposed replacement, and **the test
   that would have caught it**. Without the test it is `observed`, no matter how
   obvious the defect looks; a fix with nothing that fails before it and passes
   after is a suggestion.
8. **Read every waiver.** A registry entry with `waived:` registered only
   because a person set a finding aside. Open its checkpoint in
   `.orch/checkpoints/resolved/` and check two things. Does the decision quote
   the user? Is the finding one a person would set aside? The same kind of
   finding waived twice is a pattern. Either the policy is wrong, which is an
   upstream entry, or waiving has become a habit, which is a `process-failure`.

9. **Read where each report came from.** A trace's `verification.report` says
   whether the agent's report was recorded by the hook (`source: agent`, with
   how many refusals came before its JSON) or supplied by the orchestrator
   (`source: orchestrator`). A `done` resting on an orchestrator-supplied
   report is a finding to read: its confidence was never the agent's. Refusals
   on most tasks are an agent-definition problem, which goes upstream. An eval
   entry with `post_hoc:` was registered after its data existed; the results
   doc must say so wherever it cites it.

Report in this order: patterns (ids + quoted evidence), one-line attribution for
each, registry entries that should be superseded, proposed `process-failure`
blocks in block schema with their sources, then upstream entries in the §5.5
template.

Do not propose a fix for a pattern you have seen once and cannot reproduce. Say
you saw it, say what you would look for next time, and leave it `observed` — a
fix derived from a single instance is a guess wearing the costume of a finding.

If nothing recurs, say so **and say what you checked** — how many traces, over
what date range, which fields. An audit that finds nothing and cannot say what
it looked at is worth nothing, and here that failure is invisible unless you
make it visible.
```
````

````
FILE: .claude/agents/orch-judge.md
```markdown
---
name: orch-judge
description: A blind judge for measurement work — labels, adjudicates or scores items using only what the orchestrator packaged under .work/_blind/, never results, keys or answers. Use for blind labelling and judged evals (orch-task §8).
tools: Read, Grep, Glob
model: sonnet
effort: high
---

You judge blind. What you may know is what the orchestrator packaged for you
under `.work/_blind/<msr_id>/` — the items, the source text, the rubric — and
nothing else. Results, scores, answer keys and earlier labels exist elsewhere in
this repo; a judgment made after seeing them measures them, not the items.

A hook refuses every read outside `.work/_blind/`, and every tool but Read, Grep
and Glob — and SubagentHandback, which delivers your report. It protects you
only while it runs, so:

1. **Canary first.** Before anything else, Read the canary path your prompt
   names. It must be refused with a message starting `ORCH blind`. If you see
   its contents instead, the restriction is not running: judge nothing, and end
   `blocked` with `blocked_on: platform` and summary `blind restriction not
   enforced`.
2. **Pass the blind dir as the path** to every Grep and Glob; without one they
   search the repo root and are refused.
3. **A refusal is a stop, not a puzzle.** Never look for another route to a file
   you were refused. Ask for a missing *item*, by id, in `open_questions`.
4. **Judge each item on its own, by the rubric.** Some items are controls whose
   answer is already known; they are unmarked on purpose. Treat all alike.
5. **Say "cannot tell" when you cannot.** An abstention costs one item; a guess
   costs the measurement its meaning.

Your report is ONLY this JSON object, with the `msr_id` your prompt names — the
name of your blind dir: the SubagentHandback message if this run has that
tool, otherwise your final message. No fence, no prose. A hook records it as you send it, and refuses a
report without it:

{"msr_id": "msr_…",
 "status": "done|blocked",
 "blocked_on": null,
 "canary": "refused|read",
 "items": [{"id": "...", "label": "...", "why": "<= 20 words, citing the source text"}],
 "open_questions": []}
```
````


### 5.4 Skills

````
FILE: .claude/skills/orch-task/SKILL.md
```markdown
---
name: orch-task
description: Run one orchestrated task end-to-end — draft and lint the packet, dispatch it to a subagent in a sandboxed git worktree, verify acceptance by code, register the passing checks, and report in the required summary shape. Load this for ANY request that changes code beyond a one-line obvious edit — a feature, a bug, a refactor, multi-file work, anything needing a test — as well as for "run a task", "orch task", or a named tsk_ id. It carries the whole orchestration procedure; do not hand-roll it.
---

# Run one task

Never execute a queued packet inline in the main session. It runs in a
worktree, under a subagent, with declared scope — or it does not run.

## 0. The procedure

The always-on core routes here; everything the orchestrator does for a task
lives in this file, so it costs nothing on turns that are not orchestrating.
Run these in order.

1. **Retrieve.** Read `.orch/memory/INDEX.md`; dispatch `orch-librarian` if more
   than a couple of blocks might matter.
2. **Draft the packet** from the user's English — `goal`, executable
   `acceptance`, tight `scope.paths`, `context_refs`, `base`, `budget`. Infer
   these; do not interrogate the user for them. Write it to
   `.orch/queue/<task_id>.yaml` and lint it before anyone sees it:
   ```bash
   python3 .claude/hooks/orch-lint.py packet .orch/queue/<task_id>.yaml --measure
   ```
   Fix every error. Take each count from what `--measure` printed at `base`,
   never from memory or another branch — a count taken across unmerged
   branches is the classic wrong number. A number in the goal or prompt that
   describes the repo is quoted from a command run at `base`, with the command
   beside it. The lint cannot read prose; that part is yours (§1). Each
   measured check gets `--timeout` seconds (default 60); a heavy one times out
   unmeasured, which is fine unless it carries a count — then measure it by hand.
3. **Show it in ~8 lines and take one confirmation.** Scope and acceptance are
   where a task goes wrong, and a wrong one wastes the whole run. Include the
   mode line: `L2 — reversible (worktree), 3 files, 2 prior runs here`. After
   that single confirm, run the whole task without further questions.
   **It confirms the plan; it is not a ritual.** If the user's last message
   approved this exact goal, scope and acceptance, that was the confirmation:
   show the packet, say so, and run. If the packet differs from what they
   approved in scope or acceptance, ask. Packets drafted together — parallel
   tasks, the phases of one feature — are shown together and confirmed once.
   **A plan approval covers what the plan showed.** When the user approves a
   plan ("all of it", "run them"), it covers each later packet whose goal the
   plan named. It never covers a packet that needs any of these:
   - a scope path outside what the plan named, or one matching a sensitive
     `path_glob`;
   - a new dependency;
   - `retires:`;
   - acceptance that changed after a failed attempt.

   Show those packets and confirm them one at a time, even mid-run. Record the
   basis in the packet: `approved: "shown"`, or the id of the plan's approval
   record. Write that record once, when the user approves the plan, with the
   Write tool: `.orch/approvals/apr_<date>_<slug>.md`, holding the plan's path
   and the user's words in quotes. Every packet the plan covers then carries
   `approved: "apr_<…>"` — one id, not a quotation retyped per packet (one
   session typed nineteen for two approvals). `orch-task.py prompt` refuses
   an empty field, a record that quotes no one, and a plan approval on any
   packet the list above excludes — those exclusions are code. (The older
   `"plan: <line>"` form is read too, under the same exclusions.) The trace
   keeps the field, and the summary lists every packet that ran on a plan
   approval, so the user sees after the fact what they approved in advance.
4. **Run it** — §2–§6 below. Never execute a packet inline in the main session.
   The bookkeeping is generated, never typed: `orch-task.py prompt` renders the
   dispatch prompt, `verify --commit` commits and re-runs the checks, and
   `finish` registers, traces, prunes and checks the merge from what `verify`
   saw. The agent's report is recorded by a hook, never saved by hand (§4).
5. **Stop at any checkpoint** and present the decision. Otherwise do not
   interrupt.
6. **Summarize**, always in this shape:

   > **what changed** (1–2 sentences, `file:line`) · **acceptance** (each check
   > + real exit code, as `orch-task.py verify` printed it, and any report you
   > supplied rather than the agent) · **checkpoints** hit
   > and how resolved, waivers by id · **approval** — each packet that ran on a
   > plan approval rather than being shown · **memory**
   > blocks cited and written, with the lint's count line · **attribution** ·
   > **merge** — the one `! git merge …` line for every finished branch (§6),
   > or where the worktree is if it did not finish.

   Report the verified outcome, never the agent's claim. A `done` whose check
   failed is `failed`.

Multi-phase work (feature, refactor) runs its playbook's phases in order, each
its own packet and subagent. Report once at the end, not per phase.

**Small changes take the light lane.** One file plus its test, executable
acceptance, no sensitive path: that is one executor packet, not a playbook's
five phases — the regression test goes in the same packet. Read `INDEX.md`
yourself instead of dispatching the librarian, and skip the archivist when the
result proposes no `new_facts`. Everything after the packet is a command:

```bash
python3 .claude/hooks/orch-lint.py packet .orch/queue/<id>.yaml --measure
git worktree add -b orch/<id> .work/<id> <base>
python3 .claude/hooks/orch-task.py prompt <id>          # renders + lints the prompt
#   dispatch it; the report hook records its report in .work/_results/<id>/
python3 .claude/hooks/orch-task.py verify <id> --commit "<summary>" --test "<test cmd>"
python3 .claude/hooks/orch-task.py finish <id> --error none   # register, trace, done/, prune, merge check
git merge --no-edit orch/<id>       # a command of its own, only if granted; else the `! …` line
```

What never shrinks: the worktree, the subagent, the lint, the re-run checks,
the scan and the trace. The subagent stays even for four lines, because it is
the only independent reader of your packet — in one session, three wrong checks
reached an executor and all three were flagged rather than met. Those are the
substance; the typing was not, and it is gone.

**Attribution is mandatory and it is about you.** For every defect found during
the task, name its origin: the *packet* (goal, acceptance, scope, or reference
material you supplied), the *execution* (the agent's own work), or the *system*
(ORCH itself). If the packet was wrong, say what a better packet would have
said. `none` is a legitimate answer; blank is not. Nothing else here looks at
the packet: a bad one produces a task where every check passes, so the defect
ships as registered evidence. Attribution is never scored (`ORCH.md` §4), and a
*system*-origin defect also goes to `.orch/upstream.md` (`ORCH.md` §5.5).

**Checkpoints.** These force a stop and ask the user regardless of loop mode:
irreversible ops, external side effects, secrets in scope, writes outside
`scope.paths`, a new dependency, a `done` with confidence < 0.6, 3 attempts on
one task, any security finding above low. **The list is immutable** — never edit
it to make a task pass. Thresholds in `.orch/config/settings.yaml` may be
*tightened, never loosened*; the guard ignores a loosening value and blocks
writes to `.orch/config/` outright, so this is code, not etiquette. A pending
checkpoint is a file in `.orch/checkpoints/` (`verify` writes its own); resolve
it before the task advances — the user's decision, in their words, under
`## Resolution`, and the file moved to `resolved/` (§7).

**Memory.** Durable project knowledge lives in `.orch/memory/blocks/*.md`,
indexed by `.orch/memory/INDEX.md` (headlines only). Read the index before
non-trivial work; read a block body only when its headline matches. Never write
a block by hand mid-task — propose it in the result and let `orch-memory` ingest
it. Every block needs a `source:`. **No source, no block.**

## 1. Packet

If given a `tsk_` id, read `.orch/queue/<id>.yaml`. Otherwise draft one from
the playbook phase as §0 says — linted, then **confirmed by the user before it
runs**. The four fields that decide whether this works are `goal`,
`acceptance`, `scope.paths`, and `context_refs`.

```yaml
task_id: tsk_<ulid>
project: <name>
playbook: code.bugfix@v1
phase: fix
role: executor
goal: "one sentence, testable"
base: <sha>                      # HEAD when drafted, or a parent task's branch tip (§3)
acceptance:
  - {type: executable, cmd: "pytest tests/test_x.py::test_y", expect: "exit 0"}
  # exact, not >=1 — a floor would pass on 394 markers as readily as on 12
  - {type: executable, cmd: "ffprobe -show_chapters out.m4b | grep -c '\\[CHAPTER\\]'", expect: "stdout == 12"}
  # negative space is its own check, with its own command
  - {type: executable, cmd: "test -z \"$(ls -A output/)\"", expect: "exit 0"}
  - {type: judged, criterion: "root cause named with file:line"}
context_refs: [blk_...]          # ids, not content
scope: {paths: ["src/muxer.py", "tests/test_muxer.py"], network: false}
tools: [read, grep, edit, "bash:test"]
forbidden: [write:outside_scope, network, "git:push"]
budget: {steps: 25, wall_s: 600}
effort: medium                   # executor phases only: high -> orch-executor-deep (§4)
approved: ""                     # after the confirmation: "shown", or the plan's "apr_<id>" (§0)
notes: ""                        # required when budget deviates, or a floor check is justified
```

Add these only when they say something — written empty on every packet, they
are lines every reader learns to skip (19 of 19 in one session):

| field | when |
|---|---|
| `expects: [<why>, …]` | a medium `sensitive.yaml` finding IS this task's subject, e.g. `[network client code]` (§5) |
| `vendored: [{path: "lib/x", tree: "<sha>[:subdir]"}]` | a verbatim upstream tree, verified by tree hash at scan time |
| `retires: [chk_…]` | the task deliberately removes behaviour (unship); it becomes an `Orch-Retires:` trailer |
| `urgency: 0.5`, `impact: 0.5` | queued work the loop selects from; 0.5 when absent |

`expect:` takes one of these forms, and the choice is not cosmetic —
`exit N` accepts anything the command tolerates, so a check whose real contract
is a value must say the value:

| form | means |
|---|---|
| `exit 0` / `exit N` | the command's exit status, and nothing about its output |
| `stdout == <value>` | stdout, stripped, equals this exactly — the form a count check takes |
| `stdout matches /re/` | stdout matches the regex; use only when the exact string genuinely varies |
| `stdout <= N` / `stdout >= N` | stdout is one integer, at most / at least `N` — a ceiling such as "removes at most 5 lines". `>=` is a floor, and the floor rule below applies to it |
| `delta == +k` | stdout at HEAD minus stdout at the base commit, both integers, equals `k` — for a count over a set that other tasks also grow |

`delta` is **per-task acceptance only, never registered.** A total like
`grep -c '^STR_' strings.yaml == 212` is exact and correct, but every feature
that adds a string invalidates it, and one registry entry was superseded six
times in one project. Accept the task on its delta; register an **invariant**
instead (`every key defined is used`, `no key is duplicated`), which stays true
as the set grows.

Rules that are not negotiable:

- **Executable beats judged.** If the task admits an executable check, it must
  have one. "Judged" acceptance alone is how a task avoids being scored.
- **Count checks assert an exact number, never a floor.** `>=1`, `>0`, `-gt 0`
  and "at least" are rejected in an `expect:` unless the packet states in
  `notes` why the exact count is genuinely unknowable. A floor proves the
  feature *exists*; correctness needs it to be *exactly right*, and those are
  different assertions. If you cannot predict the exact count, you do not yet
  understand the behaviour well enough to accept it.
- **A count over an existing file is `count_at_BASE + delta`, measured.**
  `orch-lint.py packet --measure` runs the check at `base` and prints the
  first term; say in `notes` which lines the delta is. A count guessed from
  reading the code — or taken in a checkout holding other tasks' unmerged
  work — is the most common way a correct agent ends up facing a wrong check.
- **Checks assert commits, not the working tree.** §5 commits the agent's work
  before any check runs, so `git status`, `git diff` with no base, `--cached`,
  or `HEAD` alone see a clean tree, and pass or fail for reasons unrelated to
  the task. What changed is `git diff --name-only <base> HEAD`. A check that
  names `base` is per-task acceptance, never registered.
- **A check depends only on what the repo tracks.** A script in the session
  scratchpad or `/tmp` cannot be registered or replayed: the data-quality
  checks of a whole dataset task once lived there, and nobody can run them
  again. Commit such a script inside the task's scope (`scripts/check_<x>.py`)
  and call it by relative path. An absolute path to a tracked file is wrong
  too — it runs the main checkout's copy, not the task's.
- **A count runs over one stream.** `grep -c pat a b` prints `a:3` and `b:0`
  — a list, never a number — so `stdout == 0` cannot hold however right the
  work is. `cat a b | grep -c pat` counts once. The lint errors on the first,
  and `--measure` errors on any numeric expect whose output at base is not one
  number.
- **A test run outside scope is asserted unchanged.** Before a check runs a
  test file the agent may not edit, grep it for what the goal changes:
  `git grep -n '<the new value>' <base> -- <test file>`. One packet added
  reader prompt `v3` and ran a test asserting that `v3` raises; the agent
  could neither pass the check nor edit the test. The lint names every such
  file (`runs unchanged`); whether one pins the old behaviour is yours to read.
- **"Every X" must be true inside the scope.** An instruction over every
  caller, every test, every `Retriever(...)` collides with a scope that
  excludes some of them, and the agent must then break scope or stop. Name the
  instances in scope, or widen it.
- **Negative space needs its own check.** "Writes nothing to `output/`", "does
  not install anything", "leaves `config.yaml` untouched" are assertions and
  need commands, not prose. Absence is never verified by a check that only
  looks at what is present.
- **A packet whose scope is a test file needs a second check** that greps for
  the assertions that must survive. Otherwise "make the test pass" is gameable
  by deleting the test.
- **Unshipping is declared, never discovered.** A task that removes behaviour
  lists the registry checks it retires in `retires:`, so the user confirms it
  with the rest of the packet — dropping a check reduces oversight, and that is
  never a call made at commit time. Each retirement needs a negative-space
  check of its own proving the behaviour is actually gone; a retirement whose
  old check still passes is an unship that did not happen.
- **Scope tight.** Every path the agent may write, and no others. The guard
  hook blocks writes outside it.
- **Budget realistic, and per-packet when the work is heavy.** A default budget
  that trips the spend trigger on every real task teaches everyone to ignore the
  trigger. If any acceptance check invokes a model, a build, a GPU job, or a
  network fetch, the playbook default is wrong for this packet: set `budget`
  explicitly and justify it in `notes`, or mark the phase `heavy: true` and let
  the playbook's `heavy_budget_multiplier` apply. Never raise `spend_fraction`
  to make a real overrun stop reporting.

Most of these are **code**: `orch-lint.py packet` rejects floors, working-tree
checks, out-of-repo and main-checkout paths, guarded commands, unfilled
placeholders, judged-only executor acceptance, a test-only scope with no
survival grep, a `grep -c` over several files under a numeric expect, and (with
`--measure`) output at base that no numeric expect can equal. It warns on
"every X" outside scope, on acceptance that already holds at `base`, and on a
check that fails at `base` because a file outside the scope is missing — a
wrong command, not a missing feature. What it
cannot read — whether the goal is the right goal, the numbers in prose, whether
a negative-space check is missing — is still yours. The rules exist because a
bad packet fails silently: every check passes and the defect ships as
registered evidence. Verification cannot catch a defect that is encoded in the
definition of success.

## 2. Context

Dispatch `orch-librarian` with the goal. Put the returned ids in
`context_refs`, and their headline+summary in the subagent's prompt. Ids the
librarian auto-attached go in too.

## 3. Sandbox

```bash
git worktree add -b orch/<task_id> .work/<task_id> <base>
```

`<base>` is the packet's `base:` — HEAD, normally. **A task that needs an
earlier task's unmerged work branches from that task's branch**, rather than
stopping the run until the user merges: `base:` is `orch/<earlier>`'s tip, and
merging the later branch brings both. Name the stacked branches in the
summary — a stacked branch carries its parent, so rejecting the parent means
redoing the child.

**Seed dependency caches** listed in `settings.yaml` `sandbox.seed_caches`.
A worktree has tracked files only; without its caches a build fetches from the
network, which a `network: false` packet forbids. Copy each one that exists in
the main checkout, at any depth, and **say so in the dispatch prompt** — the
agent must never fetch them itself:

```bash
git ls-files -oi --exclude-standard --directory \
  | grep -E '(^|/)(\.pio/libdeps|node_modules|vendor/bundle)/$' \
  | while read -r d; do mkdir -p ".work/<task_id>/$(dirname "$d")"
      cp -a --reflink=auto "$d" ".work/<task_id>/$d"; done
```

A copied cache can hold absolute paths back to the main checkout (PlatformIO's
`.pio-link` files, npm workspace symlinks). Grep the seeded copy for the main
checkout's path; if a build would read through one, the build tests the wrong
tree. Never seed a Python `.venv` — its editable install is exactly that.

If the phase is `heavy: true`, multiply the playbook `default_budget` by its
`heavy_budget_multiplier` **now**, before dispatch — a budget known in advance
to be too small produces a spend checkpoint that carries no information, and
that is how the trigger gets trained out of everyone.

Pre-check before dispatch — any hit is a **mandatory checkpoint, before any
agent runs and before any cost**:
- a `scope.paths` entry matches a `path_globs` pattern in `sensitive.yaml`
- `attempts >= max_attempts` for this task
- the task is `deferred_until` a future time
- **the dispatch prompt fails its lint.** Render it from the packet —

  ```bash
  python3 .claude/hooks/orch-task.py prompt <task_id>   # writes .work/_prompts/<task_id>.md, then lints it
  ```

  — so the base sha, the absolute worktree path and every check are in it by
  construction. It refuses a packet whose `approved:` is empty (§0 step 3).
  Anything you add by hand goes in with the Write or Edit tool, never a shell
  heredoc: in an unquoted one bash runs every backticked code span, and the
  guard blocks it because bash would. Then lint it again:

  ```bash
  python3 .claude/hooks/orch-lint.py packet .orch/queue/<task_id>.yaml --prompt .work/_prompts/<task_id>.md
  ```

  It errors on a command the guard blocks (the guard's own matcher, over
  fenced blocks and inline code), a placeholder standing as a word of a command
  (`git diff <base> HEAD` — a format template such as `<out>: … (base <base>)`
  is not one), a prompt that omits the base sha when a check names it, or the
  worktree's absolute path. It warns on a cited file the worktree will not hold
  as you read it: on HEAD but not at `base`, untracked on disk, or changed on
  HEAD since `base` forked — a stacked branch's prompt once pointed at a doc
  section that existed only on master. A bare name is looked up by its path
  first (`eval_answers.py` is scripts/eval_answers.py), and a name found
  nowhere, or ignored by git, is prose or data; before that, 3 to 19 false
  warnings a packet taught the habit of dispatching past an error.

  A guarded step in a prompt is blocked mid-run and costs a whole attempt with
  zero progress — a cleanup `rm -rf` of the task's own gitignored build dir did
  exactly that. Take it out (a build tool's `clean` target, or no cleanup at
  all), or, if it is genuinely needed, this is the checkpoint: raise it now,
  before any cost. There is no scope-aware exemption in the guard, on purpose.

## 4. Dispatch

Spawn the subagent named by the phase's `role` (`orch-executor`,
`orch-debugger`, `orch-reviewer`). An executor phase whose packet says
`effort: high` goes to `orch-executor-deep` instead — same contract, deeper
reasoning, slower and dearer. Choose `high` when the change spans more than one
module, when the logic is the kind a shallow reading gets subtly wrong
(concurrency, parsing, security, numerics), or for **any retry of a task that
failed at `medium`**. Everything else is `medium`: acceptance is re-run by code,
so a shallow miss there is loud, not silent. Debugger, reviewer and auditor
are always `high`; librarian and archivist inherit the session's level (their
model, haiku, may not offer every level). Give the subagent the rendered
prompt (§3). It carries:

- the packet's goal, acceptance, scope, forbidden
- the retrieved blocks as `[blk_id] (type) headline / summary / body`
- `Work only inside <absolute path to .work/<task_id>>. It is a throwaway
  worktree. This path overrides the session's primary working directory.`
- `If a check looks wrong, FLAG it — do not edit code solely to make it pass.`

**The agent's report is recorded for you.** `orch-report.py` checks it as it
leaves the agent — on the `SubagentHandback` call in auto mode, where plain
text after it is never delivered, and at `SubagentStop` otherwise. A report
without the result JSON is refused, twice at most, with the exact object to
send; the one that passes is written to `.work/_results/<task_id>/<agent_id>.json`,
and `verify` reads it there. Do not save the reply yourself. `verify --result
<file>` accepts a file you hold, but records it as yours and prints `SUPPLIED
BY THE ORCHESTRATOR`, and the summary says so: twice, a result file the
orchestrator wrote read exactly like an agent's. If `verify` says no report was
recorded, the hook is not running (`python3 - ORCH.md --check`). If it says the
report had no JSON after two refusals, resume that agent by its id and ask for
the JSON alone; the hook records the reply.

**Keep the agent's id** (the Agent tool returns it) with the task. A task that
ends `blocked_on: platform` or `usage` resumes that agent, not a fresh one (§6).

**Return the orchestrator shell to the repo root before every dispatch**
(`cd "$(git rev-parse --path-format=absolute --git-common-dir)/.."` works from
inside any worktree; `--show-toplevel` does not — it returns the worktree). A subagent inherits the orchestrator's cwd as its primary working
directory: `cd` into worktree A to place fixtures, then dispatch task B, and B's
agent starts in A. Scope enforcement does not catch that when A's and B's paths
do not overlap — the writes land in a legitimate worktree, just the wrong one.
State the worktree as an absolute path, never `.work/<id>` relative.

Scope is enforced from the **path**: a write under `.work/<task_id>/` is
checked against that task's packet, so the worktree must be named exactly for
its task id. `ORCH_TASK` in a shell does **not** reach the guard — a hook runs
in Claude Code's environment, not the agent's — and is only a fallback for
writes outside any worktree.

## 5. Verify — this is the part that matters

The subagent's claimed status is an input, not the answer.

1. **Capture agent writes first.** `git -C .work/<id> status --porcelain`,
   BEFORE running acceptance checks — otherwise the checks' own byproducts
   (`__pycache__`, `.pytest_cache`, build output, lockfiles) look like agent
   writes and false-positive as scope creep. Filter generated artifacts out.
2. **Commit** with an `Orch-Task:` trailer — `verify --commit "<summary>"` does
   it. It stages the files step 1 captures that are uncommitted, in scope and
   not generated, by explicit path — never `-A`, never a directory — commits
   them with `--trailer` (git ≥ 2.32: `Orch-Task:`, plus `Orch-Retires:` from
   the packet's `retires:`), and prints the list. Read it. Out-of-scope files
   stay out and are flagged. By hand only for a path git does not show: an
   ignored-but-intended file (a vendored tree whose own `.gitignore` hides real
   upstream files) takes `git add --force -- <file>` on that file, never a
   directory, which would override every nested `.gitignore` and commit build
   output. Trailers go in with `--trailer`, never typed: git parses them from
   the last paragraph only, and an indented line there is read as a
   *continuation* — the retirement vanishes and the `Orch-Task:` value is
   corrupted, silently.

Steps 2–6 are one command:

```bash
python3 .claude/hooks/orch-task.py verify <task_id> --commit "<summary>" --test "<test command>"
```

It prints each check with its real exit code, then a `verdict`:
- `done`;
- `failed`;
- `needs_decision` — it has written the checkpoint, one finding per line (§7);
- `incomplete` — fix what it names, such as a reply with no JSON, and verify
  again.

It writes the record that `register`, `trace` and `finish` read. A failing
check that hit a not-found error at the task's HEAD is flagged as a probable
wrong check: say so in **attribution**. Some things it cannot do, and they
stay yours: the mutation check below, the post-check items it leaves to
judgment, and what any finding means.

3. **Run every executable check** inside the worktree (`verify` does, and a
   `delta` check at base too). A `done` whose check fails is `failed`. No
   discussion.
4. **A new test must fail without the change.** `verify --test` copies every
   test file the task changed onto a fresh checkout of `base` and runs the
   test command there; it wants nonzero. Some new tests may pass there — they pin behaviour that must not change;
   say which. If all of them pass, they cannot detect the change, and
   acceptance resting on them is a floor in disguise. When a test is the only
   evidence for a piece of logic, also delete that logic in a scratch copy and
   watch the test fail — a mutation check, the cheapest proof that a test
   catches what it claims to.
5. **Measure `context_usage`** — which `blk_` ids actually appear in the
   evidence, verified against the given set (`verify` does, from the recorded
   report). Discard
   anything the model wrote in those fields; a self-reported retrieval score
   is gameable and it feeds tuning.
6. **Post-check** — any hit downgrades the result to `needs_decision` and writes
   a checkpoint **before** anything is registered:
   - writes outside `scope.paths`
   - `status: done` with `confidence < 0.6`
   - a dependency manifest in the diff (`new_dependency`)
   - added diff lines matching `sensitive.yaml` above `low`
   - steps or wall time over `spend_fraction` of budget
   - an irreversible or externally-visible operation

   The last three of those are yours to judge. `verify` covers the first four:
   scope from the committed diff **and** from whatever is left uncommitted in
   the worktree — a write made through Bash never passes the guard (`up_0006`),
   but it does show up there — confidence from the result file, and the
   sensitive-content, manifest and path-glob checks through the scan, which is
   also one command on its own:

   ```bash
   python3 .claude/hooks/orch-scan.py .work/<id> <sha-at-worktree-creation> --task <task_id>
   ```

   `--task` makes the scan honour two packet fields the user already
   confirmed: `expects:` (medium findings that are the task's own subject —
   a network client task adds network code — report as low; high never does)
   and `vendored:` (findings under a path that git proves byte-identical to a
   pinned upstream tree are summarised, not gated; a claim that does not hold
   is itself a high finding). Medium network findings in test files that name
   no external host are low by policy (`sensitive.yaml` `test_downgrade`).

   The base ref defaults to `HEAD^`, which is right only when the task produced
   exactly one commit. Pass the sha the worktree was branched from (§3) and it
   is right regardless — a scan that silently covers the last commit of three
   is worse than no scan, because it reports clean.

   It gates on content patterns above `low` and on dependency manifests —
   matched as globs, so `requirements-train.txt` is one. Path
   globs come back **informational**: that gate already fired in §3, against
   `scope.paths`, before any cost — and the globs are broad enough
   (`**/*token*`, `**/*session*`) that re-gating them post-hoc would checkpoint
   `tokenizer.py`. Do not "fix" that by promoting them.

   **Run the scan this way, never as an inline grep.** A grep must spell the
   `content_patterns` in argv, one of them describes a force-push, and the guard
   blocks its own post-check — a refused read-only command, a wasted checkpoint,
   a round-trip to the user. The helper reads the patterns from the YAML at
   runtime. If it exits 2 it could not look, which is not the same as clean.

## 6. Finalize

On a clean `done`, one command does the bookkeeping this section describes:

```bash
python3 .claude/hooks/orch-task.py finish <task_id> --error none    # or --error <kind> --what "…" --better "…"
```

`finish` writes nothing unless the verify record is clean and current and
every trace argument is valid. Then it:
1. registers the passing checks;
2. writes the trace;
3. moves the packet to `.orch/queue/done/`;
4. prunes the worktree — unless the result proposes `new_facts` the archivist
   must check against it, or the worktree holds a file `verify` neither
   committed nor made;
5. checks the merge with `merge-tree` and prints the merge line, without
   running it.

The items below are what each step does and what stays yours. `register` and
`trace` are also commands of their own, for a task that is not `done`.

On a clean `done`:
- `python3 .claude/hooks/orch-task.py register <task_id>` appends each passing
  executable check to `.orch/registry/<project>.yaml` with its commit sha and a
  fresh id — except the per-task ones (`delta`, or naming `base`), which
  `orch-lint.py packet` lists — and reads the file back to prove each entry
  survived. Never append by hand: a heredoc carrying a regex inside a YAML
  string is three escaping layers, and the registry is replayed unattended.
  **Register what this task added, not a new total.** `grep -cE '^def test_(a|b|c)\('` `== 3` stays true when a later task
  adds a fourth test; `grep -c '^def test_'` `== 3` does not. A later task that
  grows a set registers a check for its own additions and leaves the earlier
  one alone
- **retired checks are not edited in the registry.** The `Orch-Retires:`
  trailer is the record; `/orch-rework` §1 reads it from git. Name them in the
  summary's **what changed**
- **if a check here replaces an existing one because the old one was wrong**,
  mark the old entry `status: superseded`, `superseded_by: <new id>`, and give
  the reason. Silently outnumbering a weak check with better ones leaves it in
  the registry as passing evidence at the commit where the behaviour was wrong
  (`/orch-rework` §3). **Run the old check at HEAD first: if it still passes, it
  was not wrong and is not superseded** — a set that grew is not a wrong check.
  Both supersessions in the busiest field registry so far were exactly that:
  name-counts that still passed, retired because a later task added tests
- if the result proposes `new_facts`, dispatch `orch-archivist` with them and
  the worktree's absolute path (it verifies every `file:line` there, so
  dispatch before pruning). If it is interrupted,
  `.orch/memory/receipts/<task_id>.log` says what landed. Then run
  `python3 .claude/hooks/orch-lint.py memory --task <task_id>` yourself: the
  archivist runs it too, but its report is testimony and the exit code is
  evidence. Repair what it finds before the summary, and quote its count line
- write the trace with `orch-task.py trace <task_id> --outcome <o> --error
  none|<kind> --what "…" --better "…"` (plus `--mode L2`, `--attempts`,
  `--steps`, `--wall`, `--interruption platform:same-agent`, `--mutation "…"`
  as they apply). `--error` is required: `none` is the claim that the packet
  was genuinely right, and the key is never blank. It takes the *kind* of
  packet defect, not the summary's origin: `weak_acceptance` (a check that
  could not fail, or asserted the wrong thing) · `wrong_scope` ·
  `bad_reference` (code, docs or a block you supplied was wrong) ·
  `unsound_plan` · `budget`. An *execution* or *system* origin is `--error
  none`; a system one also goes to `.orch/upstream.md`. It is the machine-readable
  half of the summary's `attribution`, it is never scored, and its schema is in
  `ORCH.md` §5.4 (`orch-loop`, *Traces*). The tool writes every key every
  time, copies `verification` from the verify record, and refuses a `done`
  that no clean verify backs — hand-typed traces drifted, and a trace missing
  a key is the one failure this system cannot detect later
- prune the worktree (`git worktree remove`); the work lives on the branch.
  If a dispatched archivist needed it, prune after it reports
- **merging is the user's**, unless they granted it — and even a grant can be
  refused by the harness: auto mode's permission classifier has denied
  `git merge` to an orchestrator the user had told to merge. So never stop a
  run to wait for a merge (stack dependent work instead, §3). End the run with
  **one line** covering every finished branch, parents first, each checked to
  merge cleanly (`git merge-tree --write-tree HEAD orch/<task_id>` exits 0;
  git ≥ 2.38, and it touches no branch or file):

  > `! git merge --no-edit orch/tsk_a && git merge --no-edit orch/tsk_b`

  Holding a grant, try the merge once; if it is refused, that line is the
  fallback. Never retry a refused merge, and never route around the refusal.
  **The guard checks a merge of `orch/tsk_…` before it runs:** it must be a
  command of its own — only `cd`, or another such merge, before it; a pipe
  after it is fine — `verify` must have said `done` at the branch's current
  tip, and `finish` must have run. Never chain it after verify or finish: the
  guard cannot see a verdict printed earlier in the same line, so it refuses
  the line. Once, `verify … | tail -2 && finish … | grep merge: ; git merge …`
  merged a `failed` verdict — the pipe hid verify's exit code, and the `;` ran
  the merge whatever finish said

On anything else, keep the worktree for inspection and say where it is.

**On every outcome, `done` included:** if the task's `attribution` named the
*system* as an origin — the guard blocked something legitimate, a rule here
could not be followed as written, a documented shape did not work — append an
entry to `.orch/upstream.md` (`ORCH.md` §5.5). `observed` if you saw it once,
`proven` if you can reproduce it, and a `proven` entry owes a proposed fix and
the test that would have caught it. This is the only path by which a defect in
the orchestration layer leaves the repo it was found in.

**Blocked is not failed, and it says on what.** `blocked_on: usage` (a usage
limit) or `platform` (the harness refused tool calls — a classifier outage once
refused every Bash call, `pwd` included, and stalled an executor holding an
unvalidated draft) resumes **the same agent** once the cause clears: message it
by the id kept at dispatch. It still holds its context and its draft; a fresh
agent would redo the work and lose both. Re-dispatch only if it cannot be
reached, handing over the worktree and its last report. Neither counts toward
`max_attempts`, and neither is the packet's error or the agent's: record it in
the trace's `interruptions`, not in `orchestrator_error`. `blocked_on:
dependency` waits for the dependency. In every case the work is preserved on
the branch and in the worktree.

## 7. Checkpoint format

`.orch/checkpoints/ckp_<id>.md`. When the finding comes from `verify`, the
tool writes the file itself. Its `## Trigger` lists one finding per line, so a
waiver can name exactly them. Other checkpoints — a guard block, a spend
overrun, an L4 tie — you write in the same shape:

```markdown
# ckp_<id> — <trigger name>
class: mandatory | discretionary
task: tsk_...
raised: <ISO>
## Trigger
<which rule fired, with the evidence that fired it — for verify, one line per finding>
## What the agent did
<summary + diffstat>
## To approve
<for a blocked command the user asked for: the exact command, as one `! <cmd>`
 line for them to run. Their running it IS the approval. No path turns a chat
 "yes" into the model running a guarded command, and that is deliberate: an
 approval the model can act on is an approval the model can imagine.>
## Decision
- approve → the findings are waived ONCE: `verify <task_id> --waive <ckp_id>`;
  each registry entry then carries `waived: <ckp_id>`
- reject  → task fails; nothing registers; worktree kept
- **false positive** → the trigger was wrong, not the work. Waive as above AND
  append a `proven` entry to `.orch/upstream.md` (`ORCH.md` §5.5): you
  have the reproduction already — the finding and the command that produced
  it. A trigger that stops legitimate work, left unreported, teaches the next
  orchestrator to route around it
## Resolution
<first line: the decision in the user's words —
 false positive (user, 2026-10-04, "waive both")>
```

**Resolving is two edits.** Write the user's decision as the first line under
`## Resolution`, quoting what they said, then move the file to
`.orch/checkpoints/resolved/`. `verify --waive` accepts only a file that has
both. It refuses an open checkpoint, a rejected one, one about another task,
and a decision line with no quoted words. It then downgrades each finding
named in `## Trigger`, and nothing else. A finding that appears later, or one
the checkpoint never listed, still gates.

Only `discretionary` checkpoints are scored. Mandatory ones are excluded from
fitness entirely — scoring them would reward the system for asking less.

## 8. Measurement work

An eval, a benchmark, a determinism check, a scoring script — and any question
you can only answer by computing over data: work whose output is a number that
decides something. It changes no product code, so it is not a packet, and
running it yourself is fine. What it owes is what makes a task's result count —
the result can be replayed, and its rule was fixed before it was seen. In the
field, most of a session's decisions came from measurements run this way,
outside every check: a warm-up call missing keys, a `stop <name>` that stopped
every server, an ill-posed diagnostic — each found only because someone
happened to notice. The worst two came from the quickest work: an inline script
looked labels up without the variant→base fallback the real scorer applies, so
101 suspect rows became 195, and a claim that the reader was the bottleneck
reached the user on the same bad number.

Give each measurement an id, `msr_<short>` (`msr_p14-floor`). It is the `task:`
of its registry entries and names its blind dir, its stages and its memory
receipt.

1. **Pre-register** the decision rule — which number, compared with what, at
   what threshold — and commit it before the run. If the run takes
   parameters that matter (a split, a seed, a learning rate), fix them there
   too, one line each, and launch every stage with `--prereg <file>`:

   ```yaml
   params:
     - {stage: "msr_p15-r13-train*", flag: --dev-frac, value: 0.01}
   ```

   `orch-stage.py` refuses a command that passes another value, or none. A
   flag left out means the script's default runs, and in one session that
   default was 5 % against a pre-registered 1 %.

   **The id goes in the pre-registration**, as its heading
   (`## msr_p16-g4 — worked examples in the reader prompt`), and the order is
   code twice over. `orch-stage.py run` refuses a stage named `msr_…` until a
   committed file names its measurement; a stage that produces no result (a
   download, an index build) runs with `--no-prereg "<why>"`, kept on the
   stage. `register --eval` refuses a conclusion whose data is older than the
   commit that first named the id, unless `--post-hoc "<why>"` — kept on the
   entry, and a results doc says "post hoc" wherever it cites it (`orch-lint.py
   doc` checks). In one session three measurements broke the order — a replay
   run before its rule was committed, a rule written after its numbers, a
   variant added after its parent's result — and each was disclosed only in
   prose, where nothing could check it.

   **Say what result fails the rule, and check the control can produce it.**
   One rule set the operating point at "the control's own refusals", which
   made the control's calibrated gate its plain gate by construction: the
   control half of the rule could not fail, and it cost a second task and a
   revised pre-registration. Apply the rule to the control alone before the
   run; a rule the control passes by construction decides nothing.

   **A variant names the failure it targets**, with the evidence from the run
   before it — the sample you read. One variant changed the input format after
   a sample had shown the model picking description sentences; it addressed
   something else, cost a task and 22 GPU minutes, and lost more. If the
   sample points elsewhere, say so and do not run it.
2. **The harness is tracked code.** Scorers, checkers and run scripts live in
   the repo and get there through a task, with a test, like any other code. A
   scratchpad script that decides an outcome is code that escaped review.
   Comparisons ship already: have the scorer write per-item rows and use
   `orch-measure.py` —
   - `paired`: control against arm, per group and pooled, with an exact sign
     test;
   - `matched`: the same comparison at the control's abstention;
   - `identity`: whether two runs agree item for item over a subset.

   Twice in one session these were new tasks in the middle of a measurement.
   Only what the helper cannot express needs one now.
3. **Quick analysis is measurement too.** A breakdown, a sample, a comparison
   run to answer a question: call the scorer's own entry points, never
   re-implement a piece of its logic inline — a partial copy is a second scorer
   nobody tested. A number that comes from an inline script anyway is a draft:
   write "inline, unverified" beside it, keep it out of docs, decide nothing on
   it. If the question recurs, the analysis belongs in the scorer: a task.
4. **Freeze the raw output** — per-item results, committed. If the file is too
   large to commit, keep it at a stable path; `register` records its sha256,
   and lists every input of the command that git does not hold as `local:`.
   Such an entry replays on this disk only. A fresh clone reports it `inputs
   missing`, not a break. Commit what is small enough to commit; for larger
   data, `local:` makes the limit explicit instead of silent.
5. **Register the conclusion** — the scoring command over the frozen output,
   `expect` the exact number the conclusion cites:

   ```bash
   python3 .claude/hooks/orch-task.py register --eval --msr <msr_id> \
     --cmd "python3 scripts/score.py data/results/p14.jsonl --net" --expect "stdout == +3" \
     --artifact data/results/p14.jsonl --noise "7/40 answers differ"
   ```

   It refuses a scorer that is untracked or has uncommitted changes, runs the
   command once and refuses if it does not hold, and writes a `kind: eval`
   entry (`orch-rework` §3). `--capture` instead of `--expect` registers what
   the scorer prints now and runs it a second time — output that differs
   between two runs is refused — which is the whole of what a scratchpad
   helper once did, untracked. `--conclusion "<one line>"` keeps the
   conclusion on the entry beside its number. **Register the table, not each
   arm:** one entry whose command prints every arm's row covers them all —
   `orch-lint.py doc` accepts any number the registered value holds — so an arm
   that lost by 255 answers needs a row, not an entry of its own. A sweep replays the scoring, never the run. Write
   a multi-line value to a file with the Write tool and pass `--expect-file`
   instead of quoting it through a shell; `stdout ==` compares line by line,
   whitespace at either end of each line ignored. The entry also gets `cost:`
   — stages, wall hours, hours holding each lock — summed from the stages named
   `<msr_id>-…`, so what a conclusion cost sits beside it.
6. **Measure noise once per configuration** before its numbers count: run a
   sample twice, unchanged. Record the rate on every eval entry made under that
   configuration (`--noise`); a margin inside the noise is not a finding. Until
   it is measured, an entry says `noise: unmeasured` — when a late measurement
   shows noise, those entries are what to re-read.
7. **Results docs cite the registry.** Put the entry's id right after the number
   it backs — `routing nets +3 [chk_031] answers` — and lint the doc before it
   is committed or quoted:

   ```bash
   python3 .claude/hooks/orch-lint.py doc docs/results/p14.md
   ```

   A citation of a missing or superseded entry, or a number that is not the
   registered value, is an error. Numbers shaped like results (signed,
   percentages, ratios, p-values) on lines that cite nothing are listed: cite
   each, or rewrite it as context, or drop it. `--strict` makes them errors.
8. **Findings go to memory.** When the measurement ends, propose its durable
   facts yourself — the result, the cause found, the gotchas met on the way
   (a port held by an unrelated process, a job time cap) — and dispatch
   `orch-archivist` with them, sourced to the results doc and the registry
   entry: `source: {task: <msr_id>, artifact: <doc>, check: chk_…}`. Agents'
   `new_facts` reach memory through every task; your own findings had no route,
   and a session's most important facts stayed in its docs.

### Long jobs: stages

A job that can outlive a turn — a GPU run, a bulk fetch, an overnight eval —
runs as stages, each launched as **its own** background job:

```bash
python3 .claude/hooks/orch-stage.py run msr_p14-retrieve --lock gpu --timeout 6600 -- python3 scripts/retrieve.py ...
#   ...and when the harness reports that job ended:
python3 .claude/hooks/orch-stage.py run msr_p14-generate --after msr_p14-retrieve --lock gpu -- python3 scripts/generate.py ...
python3 .claude/hooks/orch-stage.py status
```

- **Never wait inside a job.** No `wait`, sleep-until or `pgrep` loop in a
  background job to chain the next stage: the wait spends that job's time cap,
  and the stage after it is killed midway. Start the next stage when you are
  told the last one ended; `--after` refuses to start it early.
- **Completion is the status file, never a process listing.** `pgrep -f` matches
  its own command line.
- **Split a stage that cannot finish under the cap** — retrieval and generation
  for one arm are two stages — and give each a `--timeout` below the cap, so it
  ends as `timeout` rather than as a kill nobody recorded.
- **Make each stage resumable** — skip items whose output exists — so a re-run
  after `interrupted` continues instead of starting over. A stage already
  `done` with the same command is skipped, so re-running a chain resumes it.
- `orphaned` means the runner died and the job did not: let it finish (its exit
  code is still recorded) or kill its process group. SessionStart lists every
  stage that is not `done`, so the next session starts from the truth.
- **A stage after a `timeout` or `failed` one is refused, with the resume
  command printed**: re-run the earlier stage under the same name and command
  (a stage not `done` runs again), then the later one.
- **Name stages `<msr_id>-<step>`.** `status '<msr_id>*'` totals their time per
  lock across every attempt, and `register --eval` copies that total onto the
  entry.
- **Each stage fits the job cap** — `stages.job_cap_min` in `settings.yaml`,
  120 when unset. A `--timeout` over it is refused, and so is a stage expected
  to run longer — `--est <minutes>`, or `--like <stage>` for one that took as
  long — with the arithmetic printed. With no `--timeout` a stage gets the cap
  less five minutes, so it ends as `timeout`, on record, not killed midway.
- **Quote a total with its label.** `status` prints `total: … — ALL of them`,
  the whole project's, beside `this session: …` (from the SessionStart that
  began it; resume and compaction keep it) and `--since <time|stage>`. One
  session's GPU time went to the user as "about 24 hours": the project's total
  over 44 stages, 21 of which ran before the session. The session's was about
  10.
- **A wakeup you schedule carries its own stale check:** "run
  `python3 .claude/hooks/orch-stage.py status --new`; if it prints `nothing
  new`, answer in one line and stop". `--new` prints only what changed since
  it last ran. Three wakeups that arrived after their work was done each got
  the whole wrap-up again.

### Blind judging

A judge, labeller or adjudicator that must not see results, scores or keys is
`orch-judge`. Package what it may see, and only that:

```bash
mkdir -p .work/_blind/<msr_id>       # items, source text, rubric — never answers, scores, keys
sha256sum .work/_blind/<msr_id>/*    # goes in the results doc: what the judge could see
```

Its prompt names the blind dir and a canary path outside it
(`<repo>/.orch/state.json`). A hook — the `orch-guard.py --blind` entry in
`.claude/settings.json` — refuses the judge every read outside `.work/_blind/`
and logs each refusal; the canary's refusal is the evidence the restriction was
live. Count the canary lines before dispatch and after — each judge must add
one:

```bash
grep -c "Read .*/\.orch/state\.json" .work/_blind-denied.log
```

No new line means the hook did not run — an untrusted workspace, a
`settings.json` without the `--blind` entry (an install older than it:
`python3 - ORCH.md --check` says `missing 1 hook entry`), or a harness that did
not report `agent_type` — and the judgments rest on the judge's word
again: void them, or say so in the results doc.

Each judge's report — its labels — is recorded by the report hook in
`.work/_results/<msr_id>/<agent_id>.json`, from the `SubagentHandback` call the
blind check allows; v5 refused that call, and six judges finished with no way
to deliver. If a judge has no record, the hook did not run: its report is the
last assistant message in its transcript (`agent_transcript_path` in the hook
input), which is testimony, so say so.

Mix in controls whose answers are known, unmarked, so the judge's reliability
is measured, not assumed: **at least 30, known-right and known-wrong both**,
with the bar fixed in the pre-registration. 17 of 20 known-right controls once
met a bar exactly, and could not have shown a judge that marks everything
right. Score each class against its key — the judges' records pool with commas:
`orch-measure.py identity .work/_results/<msr_id>/<a>.json,… <key> --key id
--field label --ids <class ids> --value differ`.
```
````

````
FILE: .claude/skills/orch-memory/SKILL.md
```markdown
---
name: orch-memory
description: Read, write, search, and compact durable project memory blocks in .orch/memory. Use when recalling what the project already knows, recording a durable fact/failure/convention/procedure after work lands, or when the block index has grown noisy.
---

# Project memory

Blocks are durable project knowledge. They are not a log, not a changelog, and
not a place to put anything git already records.

## Read

1. `.orch/memory/INDEX.md` — headlines only. Cheap. Always start here.
2. Grep it for the task's terms *and their synonyms*.
3. Read a block file only when its headline matches.
4. **Auto-attach:** if a block header names `supersedes:` or `parent:`, read the
   chain head too. Acting on a superseded fact is the failure this prevents.

For anything non-trivial, dispatch `orch-librarian` instead of selecting by
hand — it is cheap and it applies the rules above consistently.

## Write

Only `orch-archivist` writes blocks. Propose facts in a result packet; do not
hand-edit memory mid-task.

Three sources, not one. Agents propose blocks about **the project**. The
orchestrator proposes `process-failure` blocks about **how the work was
specified** — from a task's `attribution` field, or from an orchestrator sweep
(`/orch-rework` §5). A lesson like "a packet that plans an external run must
state the resource ceiling it assumes" has nowhere else to live, and it is
retrieved on the next task that plans one. And the orchestrator proposes what
its own **measurement work** found — a result, its cause, an environment
gotcha — sourced to the results doc and the registry entry (`orch-task` §8).
Without that route, a session's most important findings never left its docs.

```markdown
FILE: .orch/memory/blocks/blk_<id>.md
---
id: blk_<id>
project: <name>
type: fact | decision | failure | convention | procedure | api-shape | entity | process-failure
headline: "<= 15 words, the claim itself"
links: {parent: blk_..., supersedes: [blk_...], related: [blk_...]}
source: {task: tsk_..., artifact: "src/muxer.py:412"}   # measurement: {task: msr_..., artifact: <doc>, check: chk_...}
superseded_by: blk_...           # only once archived, beside superseded_why: wrong | outdated
confidence: 0.9
decay_class: permanent | slow | fast
created: 2026-08-28
read_count: 0
cite_count: 0
tags: [muxer, audio]
---
<= 60-word summary (L1), each sentence citing its source:
The muxer drops chapters unless flushed before the trailer
[src: src/muxer.py:412 "self._flush()  # before write_trailer"].

Full body (L2) below — only if the summary genuinely cannot carry it. Same rule.
```

Non-negotiable:
- **No source, no block.**
- **Every claim quotes its source**: `[src: <path>:<line> "<excerpt>"]`, the
  excerpt verbatim from those lines, or `[src: chk_<id> "<excerpt>"]`.
  `orch-lint.py memory` finds each excerpt where it points; a sentence that
  quotes nothing is a sentence nobody can check, and invented ones look just
  like true ones.
- **A block is not a copy of a tracked doc.** A results doc is already the
  record, its numbers linted against the registry; a `fact` block that only
  re-cites it duplicates it (the lint warns). What earns a block is what a
  later task would not find by reading the doc: a procedure, a failure, a
  convention, a gotcha cited where it lives — the script or config that shows
  it. And a number a claim states must be in what it cites; the lint checks.
- **The INDEX line is a copy of the block's headline**, never worded on its
  own. A superseded block moves to `.orch/memory/archive/` and leaves the
  index; nothing is rewritten in place. `orch-lint.py memory` checks all of it.
- Headline states the *claim*, not the topic. "Muxer needs an explicit flush
  before the trailer" — not "notes about the muxer".
- Type by ROI: `procedure` (highest — makes the loop improve rather than
  repeat) > `failure` (scar tissue) > `convention` > `decision` > `api-shape`
  (decays fast) > `fact`.
- **`process-failure` is a lesson about writing packets, not about the code.**
  "Checkpointing multi-GB training state to a 15 GB free quota destroys the run"
  is durable, expensive, and belongs to the orchestration layer, not the
  project. Every other type describes the repo; this one describes how work here
  goes wrong. Its `source:` is a task id and a trace, and it is `decay_class:
  permanent` — the constraint that caused it does not go away because the code
  moved. Retrieve it when a task plans anything long, external, or costly.
- **Write blocks for a project BEFORE its first task.** They pay for themselves
  immediately — a single good failure block routinely predicts the exact way a
  task is about to go wrong, and the agent cites it.

## Search

```bash
grep -in "<term>" .orch/memory/INDEX.md
grep -rln "<term>" .orch/memory/blocks/
```

When retrieval was wrong, distinguish the two cases — they have opposite fixes:
- the index never surfaced it → the headline is wrong, or the tags are
- it surfaced and was not selected → the selection rule is wrong

## Compact

When a project exceeds `block_cap` (default 300), the index reads noisy, or
`orch-lint.py memory` reports a backlog, dispatch `orch-archivist` to compact.
It merges duplicates, promotes 3×-confirmed facts to `convention`, decays unread
fast blocks, archives blocks with high reads and low cites, rebuilds the index,
and runs the lint over the whole store until it is clean.

Memory sludge is the slow failure of this system: retrieval quietly degrades as
blocks accumulate until nothing useful surfaces. Compaction is not optional
maintenance.
```
````

````
FILE: .claude/skills/orch-loop/SKILL.md
```markdown
---
name: orch-loop
description: Select the next task from the queue and run it under a loop mode (L0-L5), handling checkpoints, deferral, and cross-project draining. Use when the user says "run the loop", "drain the queue", "work the backlog", or asks to pick the next task.
---

# The loop

## Select

Additive, never multiplicative — a product lets any single zero starve a task
forever:

```
score = 0.4·urgency + 0.4·impact + 0.2·staleness
staleness = min(1, days_in_queue / 30)
```

Eligible: `queued` **and `blocked`**. A blocked task is re-checked each cycle
(did its dependency finish? did its checkpoint resolve? for `blocked_on:
platform` or `usage`, does one trivial call — `pwd` — go through?) and resumes
its own agent when it does (`orch-task` §6). Leaving `blocked` out silently
defeats the whole resume design. `needs_decision` is NOT eligible; it is
waiting on a human.

Skip anything with `deferred_until` in the future.

## Modes

| | behaviour | use |
|---|---|---|
| L0 | plan only, no writes | unfamiliar domain, high stakes |
| L1 | approve every phase transition | production, irreversible work |
| **L2** | **one full task, stop at the boundary** | **default** |
| L3 | drain the queue across projects until empty or triggered | backlogs, overnight |
| L4 | 2–3 parallel branches, keep one | genuine uncertainty |
| L5 | scheduled, reports only, never acts | scans, drift monitoring |

**Propose a mode in one line** before running: reversibility (recoverable by
discarding the worktree?), blast radius (files, systems, external visibility),
familiarity (count of `procedure` blocks for this playbook+project), cost.

> `L2 — reversible (worktree), 3 files, 4 prior successful code.bugfix runs here`

If the user overrides your proposal, record it as a `decision` block: features →
proposed → chosen. That is how the proposal gets better.

## Cycle

For each selected task: `/orch-task`, then

- **checkpoint** → stop. Write it, present it, wait. In L3, skip to the next
  task rather than blocking the whole drain.
- **done** → register checks, ingest facts, write the trace, next task.
- **failed / escalate** → write the trace, do not retry silently. Three
  attempts on one task is a mandatory checkpoint.
- **blocked** → record the interruption and move on; the next cycle re-checks
  it. An interruption is not an attempt.

Pause stops at task boundaries, never mid-task. `.orch/state.json` holds
`paused` and the counters, so the loop survives a session ending.

## L4 swarm

Only for genuinely uncertain, costly-to-reverse choices — it multiplies cost by
branch count and that must be declared **before** dispatch, not discovered after.

- one worktree per branch, `orch/<task>-a|b|c`
- arbitration when several pass, in order: acceptance strength (executable >
  judged) → diff size (smaller wins) → dependencies added (fewer wins) → cost.
  An exact tie goes to a checkpoint, never a coin flip.
- **losing branches never write `fact` or `convention` blocks.** Their findings
  are ingested as `failure` only — they describe an approach that was not
  adopted, and admitting them as facts would let rejected work contradict
  shipped work in future retrieval.

## Traces

Every terminal outcome writes `.orch/traces/trc_<id>.json`, through
`orch-task.py trace` (`orch-task` §6) — never by hand. Hand-typed traces
drifted: `loop_mode` and `orchestrator_error.kind` present in some and missing
in others, which makes every sweep over them a sweep over a guess. The tool
writes every key below every time; a value it does not know is `null`, which is
visible, where a missing key is not.

```json
{"trace_id": "trc_...", "task_id": "tsk_...", "playbook": "code.bugfix@v1",
 "approval": "shown",
 "loop_mode": {"proposed": "L2", "used": "L2", "overridden": false},
 "outcome": "done", "attempts": 1,
 "checkpoints": {"mandatory": 1, "discretionary": 0},
 "context_usage": {"refs_given": [], "refs_cited": [], "l3_escapes": 0},
 "cost": {"steps": 11, "wall_s": 143},
 "interruptions": [{"on": "platform", "at": "...", "resumed": "same-agent"}],
 "rework": {"links": [], "penalty": 0.0},
 "orchestrator_error": null,
 "human_actions": [{"cmd": "approve", "at": "...", "modified": false}],
 "verification": {"base": "...", "head": "...", "verdict": "done",
   "checks": [{"n": 1, "cmd": "...", "expect": "exit 0", "exit": 0, "held": true}],
   "tests_at_base": {"cmd": "...", "tests": ["tests/test_x.py"], "exit": 1},
   "scope_violations": [], "uncommitted": [], "scan_exit": 0, "waived": [],
   "report": {"source": "agent", "via": "SubagentHandback", "agent_type": "orch-executor",
              "agent_id": "a1b2…", "refusals": 0, "json": true},
   "mutation": "deleted the flush at muxer.py:412 -> test failed", "findings": []}}
```

`approval` is the packet's `approved:` — `"shown"`, or the plan line that
covered it (`orch-task` §0 step 3) — so a sweep can tell packets the user saw
from those that ran on a plan. `verification.waived` names the resolved
checkpoints that set findings aside.

`verification` is what `orch-task.py verify` observed, copied from its record —
not the orchestrator's account of it. Its `report` says where the agent's
report came from: `source: agent`, recorded by the report hook with the
refusals that came before its JSON — the first-report rate, per task — or
`source: orchestrator`, a file the orchestrator supplied; `null` when none was
read. It is `null` only for a phase with no
executable acceptance.

`orchestrator_error` is **nullable but always present** — writing the key and
leaving it `null` is a claim, and a named empty field is harder to leave blank
dishonestly than a question nobody asked. It is the machine-readable half of the
summary's `attribution`:

```json
"orchestrator_error": {"kind": "weak_acceptance",
  "what": "expect '>=1 chapter marker' passed on 394 markers for one chapter",
  "better": "assert the exact per-chapter count",
  "superseded_check": "chk_001"}
```

`kind` ∈ `weak_acceptance` · `wrong_scope` · `bad_reference` (code or docs you
handed the agent) · `unsound_plan` (the packet's approach could not have worked)
· `budget` · `none`. It is what `/orch-rework`'s orchestrator sweep reads.

`interruptions` lists every `blocked_on: platform | usage` stop and how the task
resumed (`same-agent` or `redispatched`). It is not an attempt and not an error
of anyone's; it is kept so a platform that fails often shows up as a pattern,
instead of being read as agents that fail often.

**It never feeds fitness.** Same reasoning as mandatory checkpoints: a metric
that scores self-reported error gives the system a measurable incentive to
attribute less, and under-reporting would show up on the dashboard as
improvement. Self-reported attribution is gameable in a way a re-run exit code
is not; treat it as a prompt for honesty, not a guarantee of one. That is why
the sweep below is a separate agent.

Score on request only (weights in `ORCH.md` §4). Nulls are real and common:
the composite is a weighted mean over **present** metrics, renormalized, and
you record which ones counted. A task that retrieved nothing has no precision
and must not be penalized for it.

Self-modification stays OFF. Below ~100 traces there is nothing to fit, and
promoting a change without a paired same-session control measures run-to-run
noise, not improvement.
```
````

````
FILE: .claude/skills/orch-rework/SKILL.md
```markdown
---
name: orch-rework
description: Replay the acceptance registry to detect functionality that used to work and now doesn't, attribute the break to the commit that caused it, and record or deny the rework link. Use for regression sweeps, "did we break something", or registry health checks.
---

# Rework

Rework means one thing: **an earlier task broke functionality, and later work
was needed to restore it.** No time window — a break found eight months later
is still that task's break.

Two separate questions. Keeping them separate removes most of the machinery.

## 1. Did something break?

Re-run the stored registry at HEAD, in a throwaway detached worktree — never in
the live tree:

```bash
git worktree add --detach .work/_sweep HEAD
# for each check in .orch/registry/<project>.yaml: run it in .work/_sweep
git worktree remove --force .work/_sweep
```

Only invariants belong here. A `delta == +k` check (`orch-task` §1), or one
that names its task's base commit, is meaningless at any commit but its own and
is never registered.

**Eval entries** (`kind: eval`, `orch-task` §8) replay the scoring, never the
run. Check the artifact's sha256 first — a changed artifact means the evidence
moved after the conclusion was drawn, which is a break in its own right — then
run the scoring command. An artifact absent from the checkout (too large to
commit) reports `artifact missing`, not a break, and so does an entry whose
`local:` inputs are absent (`inputs missing`): they were never in git, and a
fresh clone was never going to hold them.

**Skip retired checks.** A check is retired when a commit in HEAD's history
carries `Orch-Retires: <chk_id>` and that commit is still in effect — not
reverted by a commit that is itself in effect, so reverting an unship
un-retires its checks and reverting the revert retires them again:

```bash
in_effect() {   # in effect unless a commit that is itself in effect reverts it
  local r
  for r in $(git log --format=%H --grep="This reverts commit $1" HEAD); do
    in_effect "$r" && return 1
  done
  return 0
}
retired() {     # chk ids retired at HEAD, one per line
  git log --format='%H%x09%(trailers:key=Orch-Retires,valueonly,separator=%x2C)' HEAD \
  | while IFS=$'\t' read -r sha ids; do
      [ -n "$ids" ] && in_effect "$sha" && echo "$ids" | tr ',' '\n' | tr -d ' '
    done | sort -u
}
```

Without this, deleting a feature fails its checks, the sweep reads that as a
break, bisect names the deleting commit, and the task that did the unship
takes a rework penalty — the system punishes exactly the work that simplifies
it. Revert detection relies on git's default `This reverts commit <sha>`
message; a revert with a rewritten message is not seen, and the checks stay
retired.

Run `status: active` checks only. `quarantined`, `exempt` and **`superseded`**
are skipped — a superseded check asserted the wrong thing, so neither its pass
nor its failure means anything, and replaying it manufactures noise in both
directions.

A check that **passed at its own commit and fails now** is a confirmed break.
No model is involved in that judgment, and there is no proximity heuristic:
touching the same lines is not evidence that you broke something.

**Registry runs are sandboxed. This is not optional.** These commands were
drafted during task execution and are re-run unattended against historical
commits. Validate at registration time — no `sudo`, no package installs, no
network unless the packet declared it, no writes outside the worktree, and a
hard per-check timeout. A check that hangs is **quarantined**, not retried
forever. `orch-task.py register` is the only writer of new entries: it refuses
a command the guard blocks, registers only what a verify run (or, for an eval,
one replay at registration) saw hold, and proves each entry reads back exactly
as written. Status changes — `superseded`, `quarantined`, `exempt` — are still
edits by hand; keep values double-quoted.

## 2. Which task broke it?

Only once a break is confirmed:

```bash
git bisect start HEAD <last-known-good-commit>
git bisect run <the failing check>
```

The culprit commit's `Orch-Task:` trailer names the task.

**Bisect assumes old commits still build, and often they don't.** The ladder:

1. runnability precheck at each candidate — a commit that won't build is
   unrunnable, not guilty
2. `git bisect skip` those
3. **abandon if >40% skipped** — the answer would be noise
4. fallback: compare the failing check against candidate diffs by hand.
   Attribution confidence 0.6, **queued for confirmation, never auto-applied**
5. inconclusive → record the break with `attribution: none`. It still counts as
   a project quality signal; it just penalizes no specific task. Better than
   guessing.

Record `attribution_method` on every link (`bisect` | `fallback` | `manual` |
`none`) and weight the penalty by it. A fallback link never carries a bisected
link's weight.

Three outcomes:
- orchestrated commit → rework link against that task
- **a human's own commit → logged, never scored.** The system does not score
  your work.
- dependency or environment drift → attributed to the task that introduced it
  if one exists, otherwise logged and excluded from fitness

## 3. Record

`.orch/registry/<project>.yaml`:

Entries as `orch-task.py register` writes them (values quoted — shown bare here
for reading):

```yaml
checks:
  - id: chk_001
    task: tsk_...
    commit: a41f2c9
    cmd: "ffprobe -show_chapters out.m4b | grep -c CHAPTER"
    expect: ">=1"                 # a floor: it passed at 1 and at 394
    registered: 2026-08-28
    status: superseded            # active | quarantined | exempt | superseded
    superseded_by: chk_007
    superseded_why: "floor assertion; the real contract is one marker per chapter"
  - id: chk_007
    task: tsk_...
    commit: 9b31e4a
    cmd: "ffprobe -show_chapters out.m4b | grep -c CHAPTER"
    expect: "stdout == 12"
    waived: ckp_muxer_3e1829      # only if a resolved checkpoint set a finding aside
    registered: 2026-08-29
    status: active
  - id: chk_031
    kind: eval                    # code (the default) | eval — orch-task §8
    task: msr_p12-r8c             # a measurement's id: it runs outside a packet
    commit: 8d09db2
    cmd: "python3 scripts/score.py data/eval/results/p12-r8c.jsonl --net"
    expect: "stdout == +3"
    artifact: {path: data/eval/results/p12-r8c.jsonl, sha256: "3f1a…"}
    local: data/cache/p12-retrieved.json   # inputs git does not hold: replays on this disk only
    cost: {stages: "4", wall_h: "5.10", gpu_h: "5.02"}    # its stages, every attempt
    prereg: 5c1d0e2a9b3f          # the commit that first named msr_p12-r8c, before its data
    post_hoc: "rule written after G5's result"   # only when it was not; docs say so where they cite it
    conclusion: "routing nets +3 at matched abstention"
    noise: unmeasured             # or "7/40 answers differ, default serving"
    registered: 2026-09-29
    status: active
rework_links:
  - {id: rwk_001, breaker: tsk_..., broken_check: chk_001,
     method: bisect, confirmed: true, at: "..."}
```

**Supersession is explicit, never silent.** A check that was always too weak is
worse than no check: it sits in the registry as *passing evidence at the commit
where the behaviour was wrong*, and `/orch-rework` replays it as proof that
things were fine. When a packet replaces a check because the old one was wrong —
not because the code moved — it must mark the old entry `status: superseded`,
name its replacement in `superseded_by`, and say why. Rules:

- a `superseded` entry is **never run** in a sweep and can generate no rework
  link. It is informational: it records that a bad definition of success was
  once accepted, which is exactly the thing worth being able to grep for later.
- superseding is not the same as **retiring** (§1). A superseded check asserted
  the wrong thing and needs a replacement. A retired check asserted the right
  thing about behaviour that was deliberately removed; it needs no replacement,
  and it is recorded in git (`Orch-Retires:`), not here, so the record cannot
  drift from the commit that removed the code.
- **a check that still passes at HEAD is not superseded.** Supersession is for
  a check that asserted the wrong thing. A set that grew — two more named
  tests — did not make the old name-count wrong: register a check for the
  additions and leave the original `active` (`orch-task` §6).
- superseding is not the same as `exempt` (flaky) or `quarantined` (hangs).
  Those are checks that cannot be trusted to run. This one ran fine and asserted
  the wrong thing.
- a replacement must exist. `status: superseded` with no `superseded_by` is
  deleting evidence, and deleting evidence is not available.
- the supersession is a finding in its own right: it goes in the task summary's
  **attribution** field, because the origin of a weak check is always the packet.

Penalty: `min(1.0, 0.35 × confirmed_links)`, **capped at 3 links per task** —
one bad task is one incident, not five. Applied retroactively; a trace's score
is never frozen. Scores drifting downward as breaks surface is correct
behaviour, not a bug.

**Exempt matters more than it looks.** A flaky check generates a link every
sweep, forever, quietly poisoning one task's score. A check that fails
intermittently across ≥3 bisects should be flagged for exemption automatically.

## 4. Non-code work

Same principle — functionality broke, not "the output was touched again":

| signal | weight |
|---|---|
| the task's own judged acceptance no longer holds | 1.0 |
| a later block contradicts one this task wrote, and the later one wins | 0.8 |
| the user rejected or substantially rewrote the deliverable | 1.0 |
| a block this task wrote was superseded **as wrong** (not merely outdated) | 0.5 |

That last distinction is the whole row: superseding an `api-shape` block because
the API changed is not rework. Superseding it because it was recorded
incorrectly is.

## 5. Orchestrator sweep

§1–4 replay the registry to find breaks in *the code*. This is the mirror:
replay the **traces** to find recurring mistakes in the **packets**. It is the
only operation in this system that finds failure modes nobody thought to look
for, and it is the one that was previously done by hand, by accident, when a
session happened to run long enough for the same shape to recur twice.

Run it every ~10 completed tasks, and after any task whose defect reached the
user.

```bash
ls .orch/traces/*.json | wc -l     # under ~20 traces: say so and stop, there is
                                   # nothing to find and a pattern would be noise
```

Then dispatch `orch-auditor` with the traces, the open and resolved checkpoints,
and the project registry.

**It is a separate agent, and that is the load-bearing part of this section.**
The orchestrator wrote the packets, chose the acceptance checks, and filled in
its own `attribution`; asking it to audit those is asking the defendant to
summarize the evidence. Self-reported attribution is gameable in a way a re-run
exit code is not — an orchestrator that wanted to look good could simply
attribute less, and that would read as improvement.

What comes back, and what to do with it:

| | |
|---|---|
| **patterns** — ≥2 traces sharing a failure mode, with ids and quoted fields | present them; one instance is an incident, not a pattern |
| **proposed `process-failure` blocks** | dispatch `orch-archivist` to write them |
| **registry entries that encoded a bad definition of success** | supersede them per §3 |
| **findings about ORCH itself, not this project** | append to `.orch/upstream.md` (`ORCH.md` §5.5). The auditor is the only party that may mark one `proven` on recurrence alone — the orchestrator has a stake in that answer and may only self-certify with a reproduction |

Three rules keep it from becoming theatre:

- **Nothing the sweep produces is scored.** Same reason mandatory checkpoints
  and `orchestrator_error` are excluded from fitness (`ORCH.md` §4): score
  honest self-attribution and the cheapest way to raise the number is to
  attribute less.
- **A `null` `orchestrator_error` is not evidence of none.** On a task that took
  three attempts, tripped a spend checkpoint, or had a check superseded later,
  the `null` is itself the finding.
- **A sweep that finds nothing must report what it checked** — how many traces,
  which fields — or it is indistinguishable from a sweep that never ran. That
  failure is invisible by construction here, which is why it is stated.
```
````


### 5.5 Findings about ORCH itself

Everything else in §5 records what a task learned about *the project*. This is
the one route out of it. A defect in ORCH — a guard that blocks legitimate work,
a rule that cannot be followed, a documented shape that does not work, a test
that passes for the wrong reason — is learned inside one repo and is worth
nothing to the next one unless it travels.

`ORCH.md` is the portable artifact and it is never self-modified (§1.1: self-
modification needs ≥100 traces and a paired test, and it does not belong in a
drop-in). So the finding travels as **a proposal a human applies**, in a file
next to the project's memory, carried back when ORCH is next revised.

Two evidence tiers, and the difference is the whole design:

| | what it takes | what it owes |
|---|---|---|
| `observed` | one instance, described honestly | nothing further. Stating it **is** the job — a noticed issue left unstated is the failure this section exists to prevent |
| `proven` | a **reproduction** (commands anyone can run, with real output), or ≥2 independent instances found by the sweep | a proposed fix, the exact text it replaces, and **the test that would have caught it** |

**Who may write which.** The orchestrator may append `observed` entries during
any task, and may write `proven` **only when it carries a reproduction** —
exit codes are code-checked evidence, not self-report, and the same reasoning
that makes executable acceptance beat judged applies here. Promotion to `proven`
on *recurrence* is the auditor's call (`orch-rework` §5): recurrence is a
judgment about a pattern, and the orchestrator has a stake in that answer.

**A `proven` entry with no test stays `observed`.** A fix proposal with nothing
that fails before it and passes after is a suggestion, and Pattern A is exactly
what suggestions are worth. The two blockers found in this file's own review
were both caught by a test that did not exist yet, and one of them was a tier-1
test that had been passing for the wrong reason all along.

````
FILE: .orch/upstream.md
```markdown
# Upstream — findings about ORCH itself, not about this project
<!-- Carried back BY HAND to ORCH.md. Nothing here is ever applied
     automatically: ORCH.md is the portable source of truth and self-
     modification is off by design (ORCH.md §1.1).

     Written by the orchestrator (observed, or proven WITH a reproduction) and
     by orch-auditor (proven by recurrence). Project facts do NOT go here —
     those are memory blocks. This file is only for defects in the
     orchestration layer, which is the thing that travels between repos.

     Entry template below. Delete it when the first real entry lands. -->

## up_0000 — <one line: the claim, not the topic>
status: observed | proven
origin: guard | packet-rule | playbook | skill | agent | test | install | core
found: <ISO date> · <tsk_/trc_/ckp_ ids>

### What happened
<what was expected, what occurred. Facts only.>

### Reproduction
```
<commands anyone can run, and their real output. If there is none, write
 "none — recurrence only, N instances" and the status cannot be `proven`
 unless the auditor promoted it.>
```

### What ORCH.md says now
<§section, and the text quoted exactly as it stands>

### Proposed change            <!-- proven entries only; omit when observed -->
<the replacement text, written so it can be applied without re-deriving it>

### The test that would have caught it   <!-- proven entries only -->
<a command, what it prints before the fix, what it must print after. Without
 this the entry is not `proven`, however obvious the defect looks.>
```
````

---

## 6. First run

```bash
mkdir -p .work && echo ".work/" >> .gitignore
git add .orch .claude ORCH.md && git commit -m "ORCH: install orchestration layer"
```

Then, in order:

1. **Write 3–10 memory blocks for the project before the first task.** Conventions,
   the test command, the two things that always break. This is the single
   highest-return step and it takes ten minutes.
2. Run one task at L1 (`/orch-task`), approving each phase, to see the shape.
3. Move to L2 (default) once the acceptance checks are honest.
4. Stay at L2 for ~30 tasks before L3. The exit criterion worth holding to:
   **30 consecutive tasks with no unguarded irreversible action and no
   unresumable interruption.**
5. **Read `.orch/upstream.md` before you revise `ORCH.md`, and empty it when you
   do.** It is where this repo recorded what is wrong with the orchestration
   layer itself, and it is the only channel by which that reaches the portable
   file. An entry that is applied gets deleted; one that is rejected gets a line
   saying why, so it is not re-proposed from scratch next time.

## 7. Testing the install

Do this in a throwaway repo first, not a repo you care about.

```bash
mkdir /tmp/orch-test && cd /tmp/orch-test && git init -q .
cp /path/to/ORCH.md .
# then paste and run the installer block from §0
```

### Tier 1 — no model involved, run these first

These are deterministic and take about a minute. If any fails, stop.

```bash
# 1. all 28 files landed
ls .claude/agents/orch-*.md .claude/skills/orch-*/SKILL.md \
   .claude/hooks/orch-*.py .orch/config/*.yaml

# 1b. the scan helper parses the policy and carries no pattern in argv
python3 .claude/hooks/orch-scan.py 2>&1 | head -1        # want the usage line
echo '{"tool_name":"Bash","tool_input":{"command":"python3 .claude/hooks/orch-scan.py .work/tsk_x"}}' \
  | python3 .claude/hooks/orch-guard.py; echo "scan invocation exit=$? (want 0)"

# 2. the guard blocks what it must (want exit=2 on each)
for c in "git push --force" "rm -rf build" "kubectl delete pod x" "npm publish"; do
  echo "{\"tool_name\":\"Bash\",\"tool_input\":{\"command\":\"$c\"}}" \
    | python3 .claude/hooks/orch-guard.py >/dev/null 2>&1; echo "$c -> exit=$?"
done

# 3. the guard allows ordinary work (want exit=0)
echo '{"tool_name":"Bash","tool_input":{"command":"pytest -q"}}' \
  | python3 .claude/hooks/orch-guard.py; echo "exit=$?"

# 4. scope enforcement — BOTH yaml styles, and the fail-closed case.
#    A guard that understands only one style is disarmed by every packet
#    written in the other, and it is disarmed silently.
mkdir -p .orch/queue
printf 'task_id: tsk_b\nscope:\n  paths:\n    - "src/*"\n'        > .orch/queue/tsk_b.yaml
printf 'task_id: tsk_f\nscope: {paths: ["src/*"], network: false}\n' > .orch/queue/tsk_f.yaml
printf 'task_id: tsk_x\ngoal: "no scope at all"\n'                  > .orch/queue/tsk_x.yaml
for t in tsk_b tsk_f; do
  echo '{"tool_name":"Edit","tool_input":{"file_path":"src/a.py"}}'   | ORCH_TASK=$t python3 .claude/hooks/orch-guard.py 2>/dev/null; echo "$t in-scope exit=$? (want 0)"
  echo '{"tool_name":"Edit","tool_input":{"file_path":"other/b.py"}}' | ORCH_TASK=$t python3 .claude/hooks/orch-guard.py 2>/dev/null; echo "$t out      exit=$? (want 2)"
done
echo '{"tool_name":"Edit","tool_input":{"file_path":"src/a.py"}}' | ORCH_TASK=tsk_x python3 .claude/hooks/orch-guard.py 2>/dev/null; echo "unreadable scope exit=$? (want 2)"

# 5. thresholds cannot be loosened
printf 'governor:\n  confidence_floor: 0.1\n' > .orch/config/settings.yaml
python3 -c "
import importlib.util,os; os.environ['CLAUDE_PROJECT_DIR']='.'
s=importlib.util.spec_from_file_location('g','.claude/hooks/orch-guard.py')
g=importlib.util.module_from_spec(s); s.loader.exec_module(g)
t=g.load_thresholds(); assert t['confidence_floor']==0.6, t
print('loosening ignored:', t)"
git checkout -- .orch/config/settings.yaml 2>/dev/null || true
```

```bash
# 6. THE UPDATE PATH. Before v2 this was silently broken: the installer saw the
#    ORCH marker and skipped CLAUDE.md, so every core change ever shipped failed
#    to reach an existing install while the agents and skills moved forward. It
#    printed `skip`, which read as correct. It gets a test now.
printf '\n## my own notes\n' >> CLAUDE.md
sed -i 's/<!-- ORCH v[0-9]*/<!-- ORCH v0/' CLAUDE.md    # pretend an older install
#    ...now re-run the §0 installer block. It must print `update CLAUDE.md (v0 -> v5)`.
grep -c '<!-- ORCH v'    CLAUDE.md      # want 1 — replaced in place, not appended twice
grep -c '<!-- /ORCH -->' CLAUDE.md      # want 1
grep -c 'my own notes'   CLAUDE.md      # want 1 — your own text survived on both sides

# 7. drift detection. Hand-edit a template the way you are meant to, then ask.
sed -i 's/spend_fraction: 0.6/spend_fraction: 0.5/' .orch/config/settings.yaml
#    ...re-run the §0 block with `--check` appended after ORCH.md.
#    Want: exit 1, and one STALE line naming .orch/config/settings.yaml.
```

```bash
# 8. triggers must match COMMANDS, not mentions of them. Before this, any
#    command that merely named a dangerous one was blocked — a commit message,
#    a doc edit, a grep — which is how people learn to route around a guard.
for c in 'git commit -m "document the git push workflow"' \
         'grep -rn "rm -rf" scripts/' \
         'sed -i "s/chmod 777/chmod 644/" setup.sh'; do
  python3 -c "import json,sys;print(json.dumps({'tool_name':'Bash','tool_input':{'command':sys.argv[1]}}))" "$c" \
    | python3 .claude/hooks/orch-guard.py >/dev/null 2>&1; echo "allow? exit=$? (want 0)"
done
#    and the evasions must still stop (want 2 on each)
for c in 'bash -c "git push --force"' 'echo $(git push --force)' \
         'ls; git push --force' 'GIT PUSH --FORCE' 'git push --force "unbalanced'; do
  python3 -c "import json,sys;print(json.dumps({'tool_name':'Bash','tool_input':{'command':sys.argv[1]}}))" "$c" \
    | python3 .claude/hooks/orch-guard.py >/dev/null 2>&1; echo "block? exit=$? (want 2)"
done
```

```bash
# 9. policy root must not follow CLAUDE_PROJECT_DIR into a worktree. It does
#    in practice, and .orch/ is not there — the scan exited 2 on every task and
#    the guard read no policy. Tested with .orch/ and .claude/ absent from the
#    worktree, the gitignored layout where it first bit.
git add -A && git commit -qm base && base=$(git rev-parse HEAD)
git worktree add -q .work/tsk_w -b orch/tsk_w HEAD && rm -r .work/tsk_w/.orch .work/tsk_w/.claude
( cd .work/tsk_w && echo 'x = 1' > ok.py && git add ok.py && git commit -qm w )
CLAUDE_PROJECT_DIR=$PWD/.work/tsk_w python3 .claude/hooks/orch-scan.py .work/tsk_w $base
echo "drifted scan exit=$? (want 0)"
echo '{"tool_name":"Edit","tool_input":{"file_path":".orch/config/settings.yaml"}}' \
  | CLAUDE_PROJECT_DIR=$PWD/.work/tsk_w python3 .claude/hooks/orch-guard.py 2>/dev/null
echo "drifted guard config-write exit=$? (want 2)"

# 10. a non-UTF-8 diff is scanned, not crashed on. A crash used to exit 1,
#     the "findings" code.
( cd .work/tsk_w && printf 'a\xe2\x28b\n' > bin.dat && git add bin.dat && git commit -qm bin )
python3 .claude/hooks/orch-scan.py .work/tsk_w $base; echo "binary diff exit=$? (want 0)"
python3 .claude/hooks/orch-scan.py .work/tsk_w no-such-ref 2>/dev/null; echo "bad ref exit=$? (want 2)"
git worktree remove --force .work/tsk_w && git branch -qD orch/tsk_w
```

```bash
# 11. quoted heredoc bodies are data (up_0005) — and the strip is not an
#     evasion. A body fed to a shell, an unquoted heredoc, and an operator
#     hidden in quotes or a comment must all still block.
j() { python3 -c "import json,sys;print(json.dumps({'tool_name':'Bash','tool_input':{'command':sys.argv[1]}}))" "$1" \
      | python3 .claude/hooks/orch-guard.py >/dev/null 2>&1; echo "$2 exit=$? (want $3)"; }
B='blocked `rm -rf a/b` and git push --force'
j "cat > x.md <<'EOF'
$B
EOF"                                          "quoted -> file"     0
j "cat > x.md <<\"EOF\"
$B
EOF"                                          "dquoted -> file"    0
j "cat > x.md <<EOF
$B
EOF"                                          "unquoted"           2
j "cat <<'EOF' | bash
git push --force
EOF"                                          "quoted -> bash"     2
j "echo \"<<'Y'\"
git push --force
Y"                                            "op inside quotes"   2
j "ls # <<'Y'
git push --force
Y"                                            "op inside comment"  2
j "cat > x <<'A'
text
A
git push --force"                             "after the body"     2

# 12. prompt lint uses the guard's matcher (up_0006)
printf 'Build it.\n```bash\nmake\nrm -rf build\n```\nThen run `git push`.\n' > p.md
python3 .claude/hooks/orch-guard.py --lint p.md </dev/null >/dev/null; echo "lint hits exit=$? (want 1)"
printf 'Run `make test` and report.\n' > p.md
python3 .claude/hooks/orch-guard.py --lint p.md </dev/null >/dev/null; echo "lint clean exit=$? (want 0)"
rm p.md

# 13. content patterns: exact where it matters (up_0008, up_0011), manifests
#     (up_0003), test-path downgrade, packet expects and vendored (up_0004).
sc() { git add -A && git commit -qm "$1" && python3 .claude/hooks/orch-scan.py . HEAD^ $3 >/dev/null 2>&1; echo "$1 exit=$? (want $2)"; }
git add -A && git commit -qm pre >/dev/null
echo 'void performFetch();'                    > a.cpp; sc "performFetch("       0
echo 'm = RE.exec(text); n = /x/.exec(s)'      > a.js;  sc "regex .exec("        0
echo 'exec(code)'                              > b.py;  sc "python exec("        1
echo 'r = fetch(url)'                          > b.js;  sc "fetch(url)"          1
printf '[env]\nlib_deps = foo\n'      > platformio.ini; sc "platformio.ini"      1
mkdir -p tests && echo 'await fetch(`${base}/v1`)' > tests/api.test.js;  sc "loopback test" 0
echo 'await fetch("https://example.com/x")'  >> tests/api.test.js;       sc "external in test" 1
printf 'task_id: tsk_n\nscope: {paths: ["*"]}\nexpects: [network client code]\n' > .orch/queue/tsk_n.yaml
echo 'r = fetch(u)' > c.js;                        sc "expects network"      0 "--task tsk_n"
# the upstream tree is built as a bare object, never committed: if it came
# from history, git would diff the vendored copy as a rename with no added
# lines, and this test would pass on code with no vendoring support at all
b=$(echo 'password = "x"' | git hash-object -w --stdin)
t=$(printf '100644 blob %s	p.py
' $b | git mktree)
mkdir -p lib/v && echo 'password = "x"' > lib/v/p.py
printf 'task_id: tsk_v\nscope: {paths: ["*"]}\nvendored:\n  - {path: "lib/v", tree: "%s"}\n' $t > .orch/queue/tsk_v.yaml
sc "vendored identical" 0 "--task tsk_v"
echo 'password = "y"' > lib/v/q.py;                sc "vendored differs"     1 "--task tsk_v"
# slm up_0008: torch's Module.eval() is inference mode, not dynamic execution
echo 'model = AutoModel.from_pretrained(d).eval()' > m.py; sc "model.eval()"           0
echo 'x = eval(user_input)'                        > e.py; sc "eval(s)"                1
# slm up_0010: manifests are globs, so a new requirements-train.txt gates
printf 'torch==2.6.0\n' > requirements-train.txt;          sc "requirements-train.txt" 1
printf '# notes\n'      > requirements.md;                  sc "requirements.md"        0
```

```bash
# 14. scope from the PATH, with no ORCH_TASK — how a hook actually runs: it
#     inherits Claude Code's environment, never the agent's shell. Before this,
#     every subagent write went unchecked. Drained packets still enforce;
#     a worktree whose packet is missing fails closed.
mkdir -p .orch/queue/done
printf 'task_id: tsk_p\nscope: {paths: ["src/*"]}\n' > .orch/queue/tsk_p.yaml
printf 'task_id: tsk_d\nscope: {paths: ["src/*"]}\n' > .orch/queue/done/tsk_d.yaml
pw() { echo "{\"tool_name\":\"Write\",\"tool_input\":{\"file_path\":\"$PWD/.work/$1/$2\"}}" \
       | env -u ORCH_TASK python3 .claude/hooks/orch-guard.py 2>/dev/null; echo "$1/$2 exit=$? (want $3)"; }
pw tsk_p src/a.py   0
pw tsk_p other/b.py 2
pw tsk_d src/a.py   0
pw tsk_d other/b.py 2
pw tsk_none src/a.py 2
pw tsk_p .orch/config/settings.yaml 2
```

```bash
# 15. the packet lint. Each packet is a shape that reached the field and passed
#     every other check (session review, 2026-09-29). The lint shares the
#     guard's matcher and scope parser, never runs a command the guard blocks,
#     and leaves no worktree and no __pycache__ behind.
rm -rf .claude/hooks/__pycache__            # step 5 left one; the lint must not
git add -A && git commit -qm pre15
mkdir -p src tests && echo 'def total(): return 1' > src/m.py && echo 'from src.m import total' > tests/test_m.py
git add -A && git commit -qm lintbase && base=$(git rev-parse HEAD)
pk() { printf 'task_id: tsk_l\nrole: executor\ngoal: "%s"\nbase: %s\nscope: {paths: ["src/m.py"]}\nacceptance:\n  - %s\n' \
         "${2:-change total}" "$base" "$1" > p.yaml; }
lp() { python3 .claude/hooks/orch-lint.py packet p.yaml $2 > lint.out 2>&1; echo "$1 exit=$? (want $3)"; }
pk '{type: executable, cmd: "grep -c return src/m.py", expect: "stdout == 2"}';                lp clean         "" 0
pk '{type: executable, cmd: "git status --porcelain", expect: "stdout == M src/m.py"}';        lp git-status    "" 1
pk '{type: executable, cmd: "test -z \"$(git diff --name-only -- tests)\"", expect: "exit 0"}'; lp diff-no-base  "" 1
pk "{type: executable, cmd: \"test -z \\\"\$(git diff --name-only $base HEAD -- tests)\\\"\", expect: \"exit 0\"}"; lp diff-base "" 0
pk '{type: executable, cmd: "python3 /tmp/s/scratchpad/checks.py", expect: "exit 0"}';         lp scratchpad    "" 1
pk "{type: executable, cmd: \"python3 $PWD/src/m.py\", expect: \"exit 0\"}";                   lp main-checkout "" 1
pk '{type: executable, cmd: "grep -c return src/m.py", expect: ">=1"}';                        lp floor         "" 1
pk '{type: judged, criterion: "looks right"}';                                                lp judged-only   "" 1
pk '{type: executable, cmd: "rm -rf src", expect: "exit 0"}';                                  lp guarded --measure 1
echo "guarded check left src/m.py: $(ls src/m.py 2>/dev/null | wc -l) (want 1)"
pk '{type: executable, cmd: "grep -c return src/m.py", expect: "stdout == 2"}' 'call it from every `total(...)` site'
lp every-x "" 0; echo "out-of-scope site named: $(grep -c tests/test_m.py lint.out) (want 1)"
pk '{type: executable, cmd: "grep -c return src/m.py", expect: "stdout == 2"}';                lp measure --measure 0
echo "base count measured: $(grep -c 'account for +1' lint.out) (want 1)"
pk '{type: executable, cmd: "grep -q return src/m.py", expect: "exit 0"}';                     lp holds --measure 0
echo "holds-at-base flagged: $(grep -c 'cannot tell done from not started' lint.out) (want 1)"
pk '{type: executable, cmd: "grep -c return src/m.py", expect: "stdout == 1"}'; echo 'effort: low' >> p.yaml
lp effort-low "" 1
printf 'task_id: tsk_l\ngoal: "g"\nscope: {paths: ["src/m.py"]}\n' > p.yaml;                  lp unreadable    "" 2
echo "leftovers: $(git worktree list | grep -c _lint_) $(ls -d .claude/hooks/__pycache__ 2>/dev/null | wc -l) (want 0 0)"
rm -f p.yaml lint.out

# 16. the memory lint: one line per block, the index headline equal to the
#     block's own, a supersede that archives, a receipt that names what landed.
M=.orch/memory; cp $M/INDEX.md index.bak
blk() { printf -- '---\nid: %s\nheadline: "%s"\nsource: {task: tsk_m, artifact: "src/m.py:1"}\n---\nbody\n' "$1" "$2" > $M/blocks/$1.md; }
idx() { printf '%s · p · fact · %s\n' "$1" "$2" >> $M/INDEX.md; }
lm() { python3 .claude/hooks/orch-lint.py memory $2 > lint.out 2>&1; echo "$1 exit=$? (want $3)"; }
blk blk_a "total returns one"; idx blk_a "total returns one";  lm consistent "" 0
echo "count line: $(grep -c 'index: 1 lines · blocks/: 1 files' lint.out) (want 1)"
idx blk_a "total returns one";                                 lm duplicate      "" 1
cp index.bak $M/INDEX.md; idx blk_a "total returns two";       lm headline-drift "" 1
cp index.bak $M/INDEX.md; idx blk_a "total returns one"
blk blk_b "orphan";                                            lm unindexed      "" 1
mv $M/blocks/blk_b.md $M/blk_b.md; idx blk_b "orphan";         lm stray          "" 1
rm $M/blk_b.md; cp index.bak $M/INDEX.md; idx blk_a "total returns one"
blk blk_c "total returns two"; idx blk_c "total returns two"
echo "2026-10-02T00:00:00Z supersede blk_a -> blk_c total returns two" > $M/receipts/tsk_m.log
lm supersede-in-place "--task tsk_m" 1
mv $M/blocks/blk_a.md $M/archive/; cp index.bak $M/INDEX.md; idx blk_c "total returns two"
lm supersede-archived "--task tsk_m" 0
lm no-receipt "--task tsk_nope" 1
rm -f index.bak lint.out

# 17. the deep executor is the executor at another effort, and nothing else:
#     two copies of one contract drift, and then there are two contracts.
body() { sed '1,/^---$/{/^---$/!d}' "$1" | sed '1,/^---$/d'; }
diff <(body .claude/agents/orch-executor.md) <(body .claude/agents/orch-executor-deep.md) >/dev/null
echo "bodies identical exit=$? (want 0)"
grep -h '^effort:' .claude/agents/orch-executor.md .claude/agents/orch-executor-deep.md | tr '\n' ' '
echo "(want effort: medium effort: high)"
```

```bash
# 18. the lint, from the 2026-10-03 review: a base failure that is a wrong
#     command rather than a missing feature (up_0007); a prompt with no base
#     sha, an unfilled <base>, a guarded command, no worktree path; a cited
#     file the base does not hold as written.
git add -A && git commit -qm pre18 >/dev/null
mkdir -p src docs scripts && echo 'def total(): return 1' > src/m.py && echo '# plan' > docs/plan.md
printf 'import sys\nprint(len(open(sys.argv[1]).read().split()))\n' > scripts/count.py
git add -A && git commit -qm base18 && b18=$(git rev-parse HEAD)
pq() { printf 'task_id: tsk_q\nrole: executor\ngoal: "change total"\nbase: %s\nscope: {paths: ["src/*"]}\nacceptance:\n  - %s\n' \
         "$b18" "$1" > .orch/queue/tsk_q.yaml; }
lq() { python3 .claude/hooks/orch-lint.py packet .orch/queue/tsk_q.yaml $2 > lint.out 2>&1; echo "$1 exit=$? (want $3)"; }
pq '{type: executable, cmd: "python3 scripts/count.py runs/p11-pool-answers-answers.json", expect: "stdout == 3"}'
lq wrong-run-name --measure 0; echo "wrong command flagged: $(grep -c 'likely a wrong command' lint.out) (want 1)"
pq '{type: executable, cmd: "python3 src/new_tool.py", expect: "exit 0"}'
lq in-scope-new-file --measure 0; echo "missing in-scope file flagged: $(grep -c 'wrong command' lint.out) (want 0)"
pq '{type: executable, cmd: "python3 scripts/count.py --total docs/plan.md", expect: "exit 0"}'
lq not-a-file --measure 0; echo "usage-shaped miss not called a wrong path: $(grep -c 'is not found, and it is outside' lint.out) (want 0)"
pq '{type: executable, cmd: "git diff --name-only <base> HEAD", expect: "stdout == src/m.py"}'
lq placeholder "" 1
pq "{type: executable, cmd: \"git diff --name-only $b18 HEAD\", expect: \"stdout == src/m.py\"}"
W="$PWD/.work/tsk_q"
printf 'Work in %s. Read docs/plan.md.\n' "$W" > p18.md;                lq no-base-sha  "--prompt p18.md" 1
printf 'Work in %s from %s. Read docs/plan.md.\n' "$W" "$b18" > p18.md;   lq base-sha     "--prompt p18.md" 0
printf 'Work in %s from %s. Then `git push`.\n' "$W" "$b18" > p18.md;     lq guarded      "--prompt p18.md" 1
printf 'Work from %s.\n' "$b18" > p18.md;                                  lq no-path      "--prompt p18.md" 1
echo '## new section' >> docs/plan.md && git commit -qam "master moves on"
echo 'draft' > docs/gone.md                  # on disk, untracked: the worktree will not hold it
printf 'Work in %s from %s. Read docs/plan.md, docs/gone.md and nowhere.md.\n' "$W" "$b18" > p18.md
lq stale-doc "--prompt p18.md" 0
echo "changed since base, not at base: $(grep -c 'changed since base forked' lint.out) $(grep -c 'not in git at base' lint.out) (want 1 1)"
rm -f p18.md lint.out docs/gone.md .orch/queue/tsk_q.yaml

# 19. results docs cite the registry: a number is evidence only if a
#     registered command reproduces it.
mkdir -p .orch/registry
printf 'checks:\n  - id: chk_101\n    kind: "eval"\n    cmd: "echo +3"\n    expect: "stdout == +3"\n    status: "active"\n  - id: chk_102\n    cmd: "echo 1"\n    expect: "stdout == 1"\n    status: superseded   # a floor\n    superseded_by: chk_101\n    superseded_why: "a floor"\n' > .orch/registry/t.yaml
ld() { printf '%s\n' "$2" > r.md; python3 .claude/hooks/orch-lint.py doc r.md $4 > lint.out 2>&1; echo "$1 exit=$? (want $3)"; }
ld cited      'Routing nets +3 [chk_101] answers.' 0
ld mismatch   'Routing nets +4 [chk_101] answers.' 1
ld superseded 'The old count, 1 [chk_102].' 1
ld unknown    'See [chk_999].' 1
ld uncited    'Router −31, quotas +17, 19 of 20 controls.' 0
echo "uncited listed: $(grep -c '3 result-shaped' lint.out) (want 1)"
ld strict     'Router −31.' 1 --strict
ld context    'Run 2026-10-02, phase 13, v2, tasks 3-5.' 0
echo "no false hits: $(grep -c 'result-shaped number(s) on' lint.out) (want 0)"
rm -f r.md lint.out .orch/registry/t.yaml
```

```bash
# 20. the task tool: the prompt carries what the lint requires; verify re-runs
#     every check and sees what the agent's report does not; register reads
#     its own write back; trace writes every key and will not call an
#     unverified task done.
git add -A && git commit -qm pre20 >/dev/null
mkdir -p src tests && echo 'def total(): return 1' > src/t.py && git add src/t.py && git commit -qm base20
b20=$(git rev-parse HEAD); T=.claude/hooks/orch-task.py
cat > .orch/queue/tsk_t.yaml <<'EOF'
task_id: tsk_t
project: t
role: executor
playbook: code.bugfix@v1
goal: "total returns 2"
base: B20
approved: "shown"
scope: {paths: ["src/t.py", "tests/test_t.py"]}
acceptance:
  - {type: executable, cmd: "grep -c \"return 2\" src/t.py", expect: "stdout == 1"}
  - {type: executable, cmd: "python3 tests/test_t.py", expect: "exit 0"}
  - {type: executable, cmd: "grep -cE '^assert total\\(\\) == 2$' tests/test_t.py", expect: "stdout == 1"}
EOF
sed -i "s/B20/$b20/" .orch/queue/tsk_t.yaml
git worktree add -q -b orch/tsk_t .work/tsk_t $b20
python3 $T prompt tsk_t > out20 2>&1; echo "prompt lints clean exit=$? (want 0)"
echo "prompt has sha, path: $(grep -c "$b20" .work/_prompts/tsk_t.md) $(grep -c "$PWD/.work/tsk_t\." .work/_prompts/tsk_t.md) (want 1 1)"
( cd .work/tsk_t && echo 'def total(): return 2' > src/t.py \
  && printf 'import sys; sys.path.insert(0, ".")\nfrom src.t import total\nassert total() == 2\n' > tests/test_t.py \
  && echo stray > notes.txt && git add src/t.py tests/test_t.py && git commit -qm fix --trailer "Orch-Task: tsk_t" )
echo '{"status": "done", "confidence": 0.9, "evidence": []}' > res.json
python3 $T verify tsk_t --test "python3 tests/test_t.py" --result res.json > out20 2>&1; echo "bash write exit=$? (want 1)"
echo "out-of-scope write seen: $(grep -c 'not committed: notes.txt' out20) (want 1)"
rm .work/tsk_t/notes.txt
python3 $T verify tsk_t --test "python3 tests/test_t.py" --result res.json > out20 2>&1; echo "verify exit=$? (want 0)"
echo "tests failed at base: $(grep -c 'tests at base: exit 1 ' out20) (want 1)"
python3 $T register tsk_t > out20 2>&1; echo "register exit=$? (want 0)"
echo "entries: $(grep -c '^  - id: chk_' .orch/registry/t.yaml) (want 3)"
python3 $T register tsk_t > out20 2>&1; echo "no duplicates: $(grep -c 'already registered' out20) (want 3)"
python3 -B -c "
import importlib.util as u; s=u.spec_from_file_location('l','.claude/hooks/orch-lint.py'); l=u.module_from_spec(s); s.loader.exec_module(l)
print('regex survived:', [e['cmd'] for e in l.registry_entries(open('.orch/registry/t.yaml').read())][2] == r\"grep -cE '^assert total\(\) == 2$' tests/test_t.py\")"
echo "(want regex survived: True)"
python3 $T trace tsk_t --outcome done > out20 2>&1; echo "no --error exit=$? (want 1)"
python3 $T trace tsk_t --outcome done --error none > out20 2>&1; echo "trace exit=$? (want 0)"
python3 -c "
import json,glob; t=json.load(open(sorted(glob.glob('.orch/traces/trc_t*.json'))[-1]))
k={'trace_id','task_id','playbook','loop_mode','outcome','attempts','checkpoints','context_usage','cost','interruptions','rework','orchestrator_error','human_actions','verification'}
print('every key:', k <= set(t), t['verification']['verdict'])"
echo "(want every key: True done)"
( cd .work/tsk_t && echo 'def total(): return 3' > src/t.py && git commit -qam regress --trailer "Orch-Task: tsk_t" )
python3 $T verify tsk_t > out20 2>&1; echo "regressed exit=$? (want 1)"
python3 $T trace tsk_t --outcome done --error none > out20 2>&1; echo "done refused exit=$? (want 1)"
python3 $T register tsk_t > out20 2>&1; echo "register refused exit=$? (want 1)"
mkdir -p scripts && printf 'import sys\nprint(sum(1 for l in open(sys.argv[1]) if l.strip()))\n' > scripts/score.py
printf '# msr_x — pre-registration\n' > msr_x.md && git add msr_x.md && git commit -qm "prereg msr_x"
printf 'a\nb\nc\n' > results.jsonl
ev() { python3 $T register --eval --msr msr_x --project t --cmd "python3 scripts/score.py results.jsonl" \
         --expect "$2" --artifact results.jsonl > out20 2>&1; echo "$1 exit=$? (want $3)"; }
ev untracked-scorer "stdout == 3" 1
git add scripts/score.py && git commit -qm scorer
ev eval "stdout == 3" 0
ev "does not hold" "stdout == 4" 1
echo "artifact hashed: $(grep -c 'sha256' .orch/registry/t.yaml) (want 1)"
git worktree remove --force .work/tsk_t; git branch -qD orch/tsk_t
rm -rf out20 res.json results.jsonl .orch/queue/tsk_t.yaml .orch/traces/trc_t*.json .orch/registry/t.yaml .work/_verify .work/_prompts
```

```bash
# 21. the stage runner: a stage is its own job, its outcome is a file its own
#     shell writes, a chain cannot start early, and nothing waits in a job.
S=.claude/hooks/orch-stage.py
python3 $S run s1 -- 'echo one >> st.out' >/dev/null; echo "run exit=$? (want 0)"
python3 $S run s1 -- 'echo one >> st.out' >/dev/null; echo "re-run resumes, ran once: $(wc -l < st.out) (want 1)"
python3 $S run s2 --after s9 -- 'true' 2>/dev/null; echo "after a stage never run exit=$? (want 3)"
python3 $S run s2 --after s1 -- 'true' >/dev/null; echo "after a done stage exit=$? (want 0)"
python3 $S run s3 -- 'git push --force' 2>/dev/null; echo "guarded exit=$? (want 2)"
python3 $S run s4 -- 'exit 7' >/dev/null; echo "failing exit=$? (want 7)"
python3 $S status s1 s4 >/dev/null; echo "status with a failure exit=$? (want 1)"
python3 $S run s5 --lock gpu -- 'sleep 4' >/dev/null 2>&1 & r5=$!; sleep 1
python3 $S run s6 --lock gpu -- 'true' 2>/dev/null; echo "lock busy exit=$? (want 3)"
kill -9 $r5; sleep 0.5
python3 $S status s5 >/dev/null; echo "runner killed, job alive exit=$? (want 3)"
python3 $S run s6 --lock gpu -- 'true' 2>/dev/null; echo "the job keeps the lock exit=$? (want 3)"
sleep 4
python3 $S status s5 >/dev/null; echo "job finished unattended exit=$? (want 0)"
python3 $S run s7 -- 'sleep 30' >/dev/null 2>&1 & r7=$!; sleep 1; kill -9 $r7
kill -9 -"$(python3 -c "import json;print(json.load(open('.work/_stages/s7.json'))['pgid'])")"; sleep 0.5
python3 $S status s7 >/dev/null; echo "runner and job killed exit=$? (want 1)"
python3 $S status s7 | grep -c interrupted | sed 's/$/ (want 1)/'
rm -rf st.out .work/_stages
```

```bash
# 22. the blind judge's hook: reads inside .work/_blind/ only, for orch-judge
#     only, every refusal logged — the log line is the canary's evidence.
mkdir -p .work/_blind/msr_x && echo q > .work/_blind/msr_x/items.md && ln -sfn ../../../.orch .work/_blind/msr_x/up
bl() { python3 -c "import json,sys;print(json.dumps({'agent_type':sys.argv[1],'agent_id':'ag1','cwd':sys.argv[2],'tool_name':sys.argv[3],'tool_input':json.loads(sys.argv[4])}))" "$2" "$PWD" "$3" "$4" \
       | python3 .claude/hooks/orch-guard.py --blind 2>/dev/null; echo "$1 exit=$? (want $5)"; }
bl "blind read"     orch-judge Read "{\"file_path\":\"$PWD/.work/_blind/msr_x/items.md\"}" 0
bl canary           orch-judge Read "{\"file_path\":\"$PWD/.orch/state.json\"}" 2
bl "dot-dot"        orch-judge Read '{"file_path":".work/_blind/../_blind-denied.log"}' 2
bl symlink          orch-judge Read "{\"file_path\":\"$PWD/.work/_blind/msr_x/up/state.json\"}" 2
bl "grep, no path"  orch-judge Grep '{"pattern":"x"}' 2
bl "grep in blind"  orch-judge Grep "{\"pattern\":\"x\",\"path\":\"$PWD/.work/_blind/msr_x\"}" 0
bl "glob escape"    orch-judge Glob "{\"pattern\":\"../../**\",\"path\":\"$PWD/.work/_blind\"}" 2
bl bash             orch-judge Bash '{"command":"cat results.json"}' 2
bl "not the judge"  orch-executor Read "{\"file_path\":\"$PWD/.orch/state.json\"}" 0
echo '{"tool_name":"Read"' | python3 .claude/hooks/orch-guard.py --blind 2>/dev/null; echo "unreadable input exit=$? (want 0)"
echo "canary logged: $(grep -c 'ag1 Read .*/\.orch/state\.json' .work/_blind-denied.log) (want 1)"
rm -rf .work/_blind .work/_blind-denied.log
```

```bash
# 23. the blind check is WIRED where the harness runs it. Step 22 calls the hook
#     directly, so it passed while v4 wired it only in orch-judge's frontmatter
#     — where Claude Code never ran it, and the judge read its canary (tier 2,
#     2026-10-04). This replays .claude/settings.json the way the harness does:
#     every PreToolUse entry whose matcher matches the tool runs, and any exit 2
#     refuses the call.
mkdir -p .work/_blind/msr_x && echo q > .work/_blind/msr_x/items.md
hk() { python3 - "$@" <<'PY'
import json, os, re, subprocess, sys
name, agent, tool, ti, want = sys.argv[1:]
ev = {"tool_name": tool, "tool_input": json.loads(ti), "cwd": os.getcwd()}
if agent != "-":
    ev.update(agent_type=agent, agent_id="ag2")
env = dict(os.environ, CLAUDE_PROJECT_DIR=os.getcwd())
codes = [subprocess.run(h["command"], shell=True, input=json.dumps(ev), text=True,
                        env=env, capture_output=True).returncode
         for e in json.load(open(".claude/settings.json"))["hooks"]["PreToolUse"]
         if e.get("matcher", "") in ("", "*") or re.fullmatch(e["matcher"], tool)
         for h in e["hooks"]]
print(f"{name} exit={2 if 2 in codes else 0} (want {want})")
PY
}
hk "judge canary"     orch-judge Read "{\"file_path\":\"$PWD/.orch/state.json\"}" 2
hk "judge blind read" orch-judge Read "{\"file_path\":\"$PWD/.work/_blind/msr_x/items.md\"}" 0
hk "judge bash"       orch-judge Bash '{"command":"ls"}' 2
hk "main canary"      -          Read "{\"file_path\":\"$PWD/.orch/state.json\"}" 0
hk "main bash"        -          Bash '{"command":"ls"}' 0
echo "canary logged: $(cat .work/_blind-denied.log 2>/dev/null | grep -c 'ag2 Read .*/\.orch/state\.json') (want 1)"
echo "wired once: $(grep -c 'orch-guard.py\\" --blind' .claude/settings.json) $(grep -c '^hooks:' .claude/agents/orch-judge.md) (want 1 0)"
rm -rf .work/_blind .work/_blind-denied.log
echo "leftovers: $(git worktree list | grep -c '_lint_\|_base_') $(ls -d .claude/hooks/__pycache__ 2>/dev/null | wc -l) (want 0 0)"

# 23b. the upgrade path for that entry. Drop it, the way an install made before
#      it looks, then re-run the §0 block with `--check`: want exit 1 and a
#      STALE line `.claude/settings.json missing 1 hook entry`. Re-run it
#      without `--check` (prints `merge`), and step 23 holds again.
python3 -c "import json;p='.claude/settings.json';d=json.load(open(p));d['hooks']['PreToolUse']=[e for e in d['hooks']['PreToolUse'] if e.get('matcher')!='*'];open(p,'w').write(json.dumps(d,indent=2)+'\n')"
```

```bash
# 24. the guard reads commands, not text (slm up_0011). What never runs — a
#     code span in single quotes, text echoed or cat'ed into a file, plain text
#     in a heredoc — passes. What does run is still read, and the newline that
#     ends a command is a separator: before v5, `ls⏎bash -c "…"` was allowed.
j() { python3 -c "import json,sys;print(json.dumps({'tool_name':'Bash','tool_input':{'command':sys.argv[1]}}))" "$1" \
      | python3 .claude/hooks/orch-guard.py >/dev/null 2>&1; echo "$2 exit=$? (want $3)"; }
j "printf 'the user ran \`git push origin master\`\n' >> notes.md"  "code span, single quotes" 0
j 'echo "the agent wrote DROP TABLE in prose" > notes.md'          "SQL text echoed to a file" 0
j "cat > r.json <<'EOF'
{\"note\": \"never DROP TABLE here\"}
EOF
git add r.json"                                                      "result file through cat"  0
j "cat > n.md <<EOF
plain text naming git push --force
EOF"                                                                 "unquoted, text only"      0
j "cat > n.md <<EOF
\$(git push --force)
EOF"                                                                 "unquoted, substitution"   2
j "python3 - <<'PY'
print('DROP TABLE x')
PY"                                                                  "python may run SQL"       2
j 'echo "DROP TABLE x" | sqlite3 db'                                 "SQL piped to a client"    2
j "echo \"\$(sqlite3 db 'DROP TABLE x')\" > f"                       "substitution in echo"     2
j 'ls
bash -c "git push --force"'                                          "newline ends a command"   2
j 'ls # a note
bash -c "git push --force"'                                          "comment, then a wrapper"  2
j 'g\
it push --force'                                                     "line continuation"        2
j "find . -exec sh -c 'git push --force' \;"                         "sh -c mid-line"           2
j 'rg -t sh "rm -rf" .'                                              "sh as a file type"        0
j 'rm -fr build'                                                     "rm -fr"                   2
j "rm -rf $(mktemp -d)/scratch"                                       "rm -rf inside temp"       0
j "rm -rf $PWD/src"                                                   "rm -rf inside the repo"   2
j 'rm -rf /tmp'                                                      "rm -rf the temp root"     2
echo "block names the Write tool: $(python3 -c "import json;print(json.dumps({'tool_name':'Bash','tool_input':{'command':'python3 - <<\'PY\'\nprint(\'DROP TABLE x\')\nPY'}}))" \
  | python3 .claude/hooks/orch-guard.py 2>&1 >/dev/null | grep -c 'Write tool') (want 1)"
```

```bash
# 25. a waived checkpoint registers (slm up_0009). verify writes the
#     checkpoint itself, one finding per line; only a RESOLVED one that quotes
#     the user waives, exactly the findings it lists, and the waiver rides into
#     the registry entry. Before v5 nothing could register after a waiver.
git add -A && git commit -qm pre25 >/dev/null
mkdir -p src && echo 'def run(s): return s' > src/w.py && git add src/w.py && git commit -qm base25
b25=$(git rev-parse HEAD); T=.claude/hooks/orch-task.py
printf 'task_id: tsk_w5\nproject: t\nrole: executor\ngoal: "run evaluates"\nbase: %s\napproved: "shown"\nscope: {paths: ["src/w.py"]}\nacceptance:\n  - {type: executable, cmd: "grep -c eval src/w.py", expect: "stdout == 1"}\n' $b25 > .orch/queue/tsk_w5.yaml
git worktree add -q -b orch/tsk_w5 .work/tsk_w5 $b25
echo 'def run(s): return eval(s)' > .work/tsk_w5/src/w.py
python3 $T verify tsk_w5 --commit "run evaluates" > out25 2>&1; echo "scan finding exit=$? (want 1)"
echo "committed with the trailer: $(git -C .work/tsk_w5 log -1 --format=%B | grep -c '^Orch-Task: tsk_w5$') (want 1)"
C=$(ls .orch/checkpoints/ckp_w5_*.md); c=$(basename "$C" .md)
echo "checkpoint lists the finding: $(grep -c '^- scan: medium src/w.py: dynamic execution' "$C") (want 1)"
python3 $T register tsk_w5 > out25 2>&1; echo "register before a decision exit=$? (want 1)"
python3 $T verify tsk_w5 --waive $c > out25 2>&1; echo "an open checkpoint waives exit=$? (want 1)"
mv "$C" .orch/checkpoints/resolved/ && R=.orch/checkpoints/resolved/$c.md
sed -i 's/^<!-- First line.*/approve/' $R
python3 $T verify tsk_w5 --waive $c > out25 2>&1; echo "no user words exit=$? (want 1)"
sed -i 's/^approve$/false positive (user, 2026-10-05, "waive it")/' $R
python3 $T verify tsk_w5 --waive $c > out25 2>&1; echo "resolved waiver exit=$? (want 0)"
python3 $T register tsk_w5 > out25 2>&1; echo "register exit=$? (want 0)"
echo "the entry carries it: $(grep -c "waived: \"$c\"" .orch/registry/t.yaml) (want 1)"
( cd .work/tsk_w5 && printf 'import os\nos.system(cmd)\n' >> src/w.py && git commit -qam more --trailer "Orch-Task: tsk_w5" )
python3 $T verify tsk_w5 --waive $c > out25 2>&1; echo "a new finding is not waived exit=$? (want 1)"
echo "new one open, old one waived: $(grep -c '^checkpoint scan: .*os.system' out25) $(grep -c '^waived' out25) (want 1 1)"
git worktree remove --force .work/tsk_w5; git branch -qD orch/tsk_w5
rm -rf out25 .orch/queue/tsk_w5.yaml .orch/checkpoints/ckp_w5_* $R .orch/registry/t.yaml .work/_verify
```

```bash
# 26. the fast lane: verify --commit, then finish — register, trace, packet to
#     done/, worktree pruned, merge checked and printed but never run. And the
#     papercuts: no `approved:`, a reply with no JSON, --error given an origin.
git add -A && git commit -qm pre26 >/dev/null
echo 'def two(): return 1' > src/f.py && git add src/f.py && git commit -qm base26; b26=$(git rev-parse HEAD)
printf 'task_id: tsk_f6\nproject: t\nrole: executor\ngoal: "two returns 2"\nbase: %s\nscope: {paths: ["src/f.py"]}\nacceptance:\n  - {type: executable, cmd: "grep -c \\"return 2\\" src/f.py", expect: "stdout == 1"}\n' $b26 > .orch/queue/tsk_f6.yaml
git worktree add -q -b orch/tsk_f6 .work/tsk_f6 $b26
python3 $T prompt tsk_f6 > out26 2>&1; echo "no approved: exit=$? (want 1)"
echo 'approved: "shown"' >> .orch/queue/tsk_f6.yaml
python3 $T prompt tsk_f6 > out26 2>&1; echo "prompt exit=$? (want 0)"
echo 'def two(): return 2' > .work/tsk_f6/src/f.py
echo 'All done, see the diff.' > reply.txt
python3 $T verify tsk_f6 --commit "two returns 2" --result reply.txt > out26 2>&1; echo "reply with no JSON exit=$? (want 1)"
echo "says to resume the agent: $(grep -c 'Resume that agent' out26) (want 1)"
printf 'Done.\n{"status": "done", "confidence": 0.9, "evidence": [], "new_facts": []}\n' > reply.txt
python3 $T verify tsk_f6 --result reply.txt > out26 2>&1; echo "verify exit=$? (want 0)"
python3 $T finish tsk_f6 --error packet > out26 2>&1; echo "an origin as --error exit=$? (want 1)"
echo "nothing written: $(cat .orch/registry/t.yaml 2>/dev/null | grep -c tsk_f6) $(ls .orch/traces/ | grep -c f6) (want 0 0)"
python3 $T finish tsk_f6 --error none > out26 2>&1; echo "finish exit=$? (want 0)"
echo "registered, traced, moved, pruned: $(grep -c 'task: "tsk_f6"' .orch/registry/t.yaml) $(ls .orch/traces/ | grep -c f6) $(ls .orch/queue/done/tsk_f6.yaml | wc -l) $(git worktree list | grep -c tsk_f6) (want 1 1 1 0)"
echo "merge printed, not run: $(grep -c 'merges cleanly' out26) $(grep -c '^  git merge --no-edit orch/tsk_f6$' out26) $(git branch --merged | grep -c tsk_f6) (want 1 1 0)"
python3 -c "import json,glob; print('approval:', json.load(open(glob.glob('.orch/traces/trc_f6*.json')[0]))['approval'])"
echo "(want approval: shown)"
git branch -qD orch/tsk_f6; rm -rf out26 reply.txt .orch/queue/done/tsk_f6.yaml .orch/traces/trc_f6*.json .orch/registry/t.yaml .work/_verify .work/_prompts
```

```bash
# 27. measurement bookkeeping: an expect written to a file (no shell quoting;
#     indentation forgiven), inputs git does not hold labelled `local`, the
#     stages' time on the entry; a stage after a timed-out one prints the
#     command that resumes it; a pre-registration's parameters are checked
#     against the command — a flag left to the script's default included.
git add -A && git commit -qm pre27 >/dev/null
printf '# msr_y — pre-registration\n' > msr_y.md && git add msr_y.md && git commit -qm "prereg msr_y"
printf 'import sys\nfor l in open(sys.argv[1]): print("  " + l.strip())\n' > scripts/show.py
git add scripts/show.py && git commit -qm show
printf 'a\nb\n' > cache.json; printf 'a\nb\n' > want.txt; S=.claude/hooks/orch-stage.py
python3 $S run msr_y-fetch -- 'true' >/dev/null
python3 $S run msr_y-score --lock gpu -- 'sleep 1' >/dev/null
python3 $T register --eval --msr msr_y --project t --cmd "python3 scripts/show.py cache.json" --expect-file want.txt > out27 2>&1
echo "multi-line expect from a file exit=$? (want 0)"
echo "local input, stage cost: $(grep -c 'local: "cache.json"' .orch/registry/t.yaml) $(grep -c 'cost: {stages: "2"' .orch/registry/t.yaml) (want 1 1)"
python3 $S run msr_y-slow --timeout 1 -- 'sleep 5' >/dev/null 2>&1; echo "timeout exit=$? (want 124)"
python3 $S run msr_y-next --after msr_y-slow -- 'true' > out27 2>&1; echo "after a timeout exit=$? (want 3)"
echo "prints the resume: $(grep -c 'run msr_y-slow --timeout 1 -- ' out27) (want 1)"
python3 $S status 'msr_y*' > out27; echo "status by glob, totals: $(grep -c '^total: 3 stage(s)' out27) $(grep -c 'holding gpu' out27) (want 1 1)"
printf '# prereg\n```yaml\nparams:\n  - {stage: "msr_y-train*", flag: --dev-frac, value: 0.01}\n```\n' > prereg.md
python3 $S run msr_y-train1 --prereg prereg.md -- 'true --dev-frac 0.01' 2>/dev/null; echo "prereg not committed exit=$? (want 2)"
git add prereg.md && git commit -qm prereg
python3 $S run msr_y-train1 --prereg prereg.md -- 'python3 -c "" --epochs 3' 2>/dev/null; echo "flag left to its default exit=$? (want 2)"
python3 $S run msr_y-train1 --prereg prereg.md -- 'python3 -c "" --dev-frac 0.05' 2>/dev/null; echo "another value exit=$? (want 2)"
python3 $S run msr_y-train1 --prereg prereg.md -- 'python3 -c "" --dev-frac=0.010' >/dev/null 2>&1; echo "as registered exit=$? (want 0)"
python3 $S run msr_y-eval --prereg prereg.md -- 'true' 2>/dev/null; echo "no entry for the stage exit=$? (want 2)"
rm -rf out27 cache.json want.txt .work/_stages .orch/registry/t.yaml
```

```bash
# 28. the measurement helpers, on rows whose answers are known by
#     construction: paired and pooled, at matched abstention, run identity.
M=.claude/hooks/orch-measure.py
python3 -c "
import json
s = [0.9, 0.8, 0.7, 0.1, 0.6, 0.5, 0.95, 0.4, 0.05, 0.15]
row = lambda i, ok, ab: json.dumps({'id': f'q{i}', 'g': 'a' if i < 6 else 'b', 'ok': ok, 'abst': ab, 'ans': i < 8, 's': s[i]})
open('ctl.jsonl', 'w').write(''.join(row(i, i in (0, 1, 6), i in (8, 9)) + '\n' for i in range(10)))
open('arm.jsonl', 'w').write(''.join(row(i, i in (0, 1, 2, 3, 6, 7), False) + '\n' for i in range(10)))"
python3 $M paired  ctl.jsonl arm.jsonl --key id --outcome ok --where ans --value net; echo "(want +3)"
python3 $M paired  ctl.jsonl arm.jsonl --key id --outcome ok --where ans --group g --value net --in b; echo "(want +1)"
python3 $M paired  ctl.jsonl arm.jsonl --key id --outcome ok --where ans --value p; echo "(want 0.25)"
python3 $M matched ctl.jsonl arm.jsonl --key id --outcome ok --abstained abst --score s --answerable ans --value net; echo "(want +2)"
python3 $M matched ctl.jsonl arm.jsonl --key id --outcome ok --abstained abst --score s --answerable ans --value threshold; echo "(want 0.15)"
python3 $M identity ctl.jsonl arm.jsonl --key id --field ok --value differ; echo "(want 3)"
echo '["q0", "q1"]' > ids.json
python3 $M identity ctl.jsonl arm.jsonl --key id --field ok --ids ids.json --value differ; echo "(want 0)"
head -5 arm.jsonl > short.jsonl
python3 $M paired ctl.jsonl short.jsonl --key id --outcome ok 2>/dev/null; echo "runs that do not pair exit=$? (want 2)"
rm -f ctl.jsonl arm.jsonl ids.json short.jsonl
```

```bash
# 29. memory claims quote their sources. An archivist once wrote a workaround
#     that never happened and a recommendation nobody made, and the lint
#     passed both. Each sentence of a touched block cites
#     [src: <path>:<line> "<excerpt>"], and the excerpt must be at that line.
git add -A && git commit -qm pre29 >/dev/null
M2=.orch/memory; cp $M2/INDEX.md index.bak
printf 'def total():\n    return 1  # the one total\n' > src/c.py && git add src/c.py && git commit -qm c
cb() { printf -- '---\nid: blk_q\nheadline: "total returns one"\nsource: {task: tsk_c9, artifact: "src/c.py:2"}\n---\n%s\n' "$1" > $M2/blocks/blk_q.md; }
printf 'blk_q · p · fact · total returns one\n' >> $M2/INDEX.md
echo "2026-10-05T00:00:00Z new blk_q total returns one" > $M2/receipts/tsk_c9.log
lc() { python3 .claude/hooks/orch-lint.py memory --task tsk_c9 > lint.out 2>&1; echo "$1 exit=$? (want $2)"; }
cb 'total() returns 1 [src: src/c.py:2 "return 1  # the one total"].';  lc cited 0
cb 'total() returns 2 [src: src/c.py:2 "return 2  # the one total"].';  lc "excerpt not at the line" 1
cb 'total() returns 1 [src: src/c.py:2 "return 1  # the one total"]. The user edits the queue and pushes.'
lc "an invented sentence" 1; echo "it is named: $(grep -c 'The user edits the queue' lint.out) (want 1)"
cb 'total() returns 1 [src: src/c.py:2 "return 1"].';                  lc "excerpt too short" 1
rm $M2/receipts/tsk_c9.log; cb 'total() returns 1, says nobody.'
python3 .claude/hooks/orch-lint.py memory > lint.out 2>&1; echo "untouched and uncited, a warning exit=$? (want 0)"
echo "counted: $(grep -c '1 block(s) state claims with no \[src:\]' lint.out) (want 1)"
rm $M2/blocks/blk_q.md; cp index.bak $M2/INDEX.md; rm -f index.bak lint.out
echo "leftovers: $(git worktree list | grep -c '_lint_\|_base_\|tsk_') $(ls -d .claude/hooks/__pycache__ 2>/dev/null | wc -l) (want 0 0)"
```

```bash
# 30. reports are held to their contract where they leave the agent, and
#     recorded by code (slm up_0013, up_0012). A report with no result JSON is
#     refused — the handback denied, or the stop prevented — twice at most,
#     then let through and recorded as failing, so verify says incomplete.
#     One that passes lands in .work/_results/ with the agent type and id the
#     harness supplied, which no model writes.
git add -A && git commit -qm pre30 >/dev/null
R=.claude/hooks/orch-report.py
rp() { python3 -c "
import json, sys
aid, ag, ev, msg = sys.argv[1:5]
d = {'agent_type': ag, 'agent_id': aid, 'hook_event_name': ev}
if ev == 'PreToolUse':
    d.update(tool_name='SubagentHandback', tool_input={'message': msg})
else:
    d.update(last_assistant_message=msg)
print(json.dumps(d))" "$2" "$3" "$4" "$5" | python3 $R >/dev/null 2>&1; echo "$1 exit=$? (want $6)"; }
J='{"task_id": "tsk_r", "status": "done", "confidence": 0.9, "evidence": [], "new_facts": [], "open_questions": []}'
rp "prose handback"          e1 orch-executor      PreToolUse   "All done, tests pass." 2
rp "json handback"           e1 orch-executor      PreToolUse   "$J" 0
rp "stop after a handback"   e1 orch-executor      SubagentStop "" 0
echo "recorded by code: $(python3 -c "import json; r = json.load(open('.work/_results/tsk_r/e1.json')); print(r['agent_type'], r['via'], r['refusals'], r['json'])") (want orch-executor SubagentHandback 1 True)"
rp "no task_id"              e2 orch-executor-deep PreToolUse   '{"status": "done"}' 2
rp "prose again"             e2 orch-executor-deep PreToolUse   'still prose, tsk_r' 2
rp "refusals spent"          e2 orch-executor-deep PreToolUse   'prose once more, tsk_r' 0
echo "let through as failing: $(python3 -c "import json; r = json.load(open('.work/_results/tsk_r/e2.json')); print(r['ok'], r['json'])") (want False False)"
rp "stop, plain mode, prose" e3 orch-reviewer      SubagentStop "Looks right to me." 2
rp "stop, plain mode, json"  e3 orch-reviewer      SubagentStop "Checked. $J" 0
rp "a judge's labels"        j1 orch-judge         PreToolUse   '{"msr_id": "msr_t", "status": "done", "canary": "refused", "items": [{"id": "q1", "label": "yes"}, {"id": "q2", "label": "no"}]}' 0
printf '{"id": "q1", "label": "yes"}\n{"id": "q2", "label": "yes"}\n' > key30.jsonl
echo "controls scored from the record: $(python3 .claude/hooks/orch-measure.py identity .work/_results/msr_t/j1.json key30.jsonl --key id --field label --value differ) (want 1)"
rm -f key30.jsonl
rp "not an ORCH agent"       x1 Explore            PreToolUse   "prose" 0
echo 'not json' | python3 $R >/dev/null 2>&1; echo "unreadable input exit=$? (want 0)"
echo "refusals logged: $(grep -c ' report ' .work/_results/refusals.log) (want 4)"

# 31. the archivist: its report is refused while the memory lint fails on its
#     own ingest — one batch reported "complete" over 58 errors.
M3=.orch/memory; cp $M3/INDEX.md index.bak
printf 'def total():\n    return 1  # the one total\n' > src/c31.py && git add src/c31.py && git commit -qm c31
ab() { printf -- '---\nid: blk_a31\nheadline: "total returns one"\nsource: {task: tsk_a31, artifact: "src/c31.py:2"}\n---\n%s\n' "$1" > $M3/blocks/blk_a31.md; }
printf 'blk_a31 · p · fact · total returns one\n' >> $M3/INDEX.md
echo "2026-10-07T00:00:00Z new blk_a31 total returns one" > $M3/receipts/tsk_a31.log
ab 'total() returns 1, says nobody.'
rp "archivist, lint failing" a1 orch-archivist PreToolUse "ingest tsk_a31: complete" 2
ab 'total() returns 1 [src: src/c31.py:2 "return 1  # the one total"].'
rp "archivist, lint clean"   a1 orch-archivist PreToolUse "ingest tsk_a31: complete" 0
echo "its lint recorded: $(python3 -c "import json; r = json.load(open('.work/_results/tsk_a31/a1.json')); print(r['ok'], r['lint']['exit'])") (want True 0)"
rm $M3/blocks/blk_a31.md $M3/receipts/tsk_a31.log; cp index.bak $M3/INDEX.md; rm -f index.bak
rm -rf .work/_results

# 32. ...and WIRED: settings.json replayed as the harness runs it. A judge's
#     handback passes the blind check, which refused it in v5; its reads
#     outside the blind dir still do not. An executor's prose handback and a
#     prose stop are refused by the entries that run orch-report.py.
mkdir -p .work/_blind/msr_x && echo q > .work/_blind/msr_x/items.md
hk2() { python3 - "$@" <<'PY'
import json, os, re, subprocess, sys
name, aid, event, agent, tool, ti, want = sys.argv[1:]
ev = {"hook_event_name": event, "agent_type": agent, "agent_id": aid, "cwd": os.getcwd()}
if event == "PreToolUse":
    ev.update(tool_name=tool, tool_input=json.loads(ti))
else:
    ev.update(last_assistant_message=json.loads(ti)["message"])
env = dict(os.environ, CLAUDE_PROJECT_DIR=os.getcwd())
codes = [subprocess.run(h["command"], shell=True, input=json.dumps(ev), text=True, env=env,
                        capture_output=True).returncode
         for e in json.load(open(".claude/settings.json"))["hooks"].get(event, [])
         if event != "PreToolUse" or e.get("matcher", "") in ("", "*") or re.fullmatch(e["matcher"], tool)
         for h in e["hooks"]]
print(f"{name} exit={2 if 2 in codes else 0} (want {want})")
PY
}
hk2 "judge hands back"     w1 PreToolUse   orch-judge    SubagentHandback '{"message": "{\"msr_id\": \"msr_x\", \"status\": \"done\", \"canary\": \"refused\", \"items\": []}"}' 0
hk2 "judge reads a result" w1 PreToolUse   orch-judge    Read "{\"file_path\": \"$PWD/.orch/state.json\"}" 2
hk2 "executor, prose"      w2 PreToolUse   orch-executor SubagentHandback '{"message": "Done, see the diff."}' 2
hk2 "stop, prose"          w3 SubagentStop orch-debugger -                '{"message": "Found it at a.py:3."}' 2
echo "judge's report recorded: $(ls .work/_results/msr_x/ 2>/dev/null | grep -c json) (want 1)"
rm -rf .work/_blind .work/_blind-denied.log .work/_results

# 33. what verify reads: the report the hook recorded, labelled the agent's;
#     a --result file is accepted and labelled the orchestrator's. The prompt
#     carries the commit line and the report object, task id filled in. The
#     guard refuses writes into .work/_results/, by Write or by shell.
git add -A && git commit -qm pre33 >/dev/null
T=.claude/hooks/orch-task.py
j() { python3 -c "import json,sys;print(json.dumps({'tool_name':'Bash','tool_input':{'command':sys.argv[1]}}))" "$1" \
      | python3 .claude/hooks/orch-guard.py >/dev/null 2>&1; echo "$2 exit=$? (want $3)"; }
echo 'def three(): return 1' > src/v.py && git add src/v.py && git commit -qm base33; b33=$(git rev-parse HEAD)
printf 'task_id: tsk_v3\nproject: t\nrole: executor\ngoal: "three returns 3"\nbase: %s\napproved: "shown"\nscope: {paths: ["src/v.py"]}\nacceptance:\n  - {type: executable, cmd: "grep -c \\"return 3\\" src/v.py", expect: "stdout == 1"}\n' $b33 > .orch/queue/tsk_v3.yaml
git worktree add -q -b orch/tsk_v3 .work/tsk_v3 $b33
python3 $T prompt tsk_v3 >/dev/null 2>&1; echo "prompt exit=$? (want 0)"
echo "trailer and id in the prompt: $(grep -c 'trailer "Orch-Task: tsk_v3"' .work/_prompts/tsk_v3.md) $(grep -c '^{"task_id": "tsk_v3"' .work/_prompts/tsk_v3.md) (want 1 1)"
echo 'def three(): return 3' > .work/tsk_v3/src/v.py
rp "the agent hands back" e5 orch-executor PreToolUse '{"task_id": "tsk_v3", "status": "done", "confidence": 0.9, "evidence": [], "new_facts": [], "open_questions": []}' 0
python3 $T verify tsk_v3 --commit "three returns 3" > out33 2>&1; echo "verify exit=$? (want 0)"
echo "read from the agent: $(grep -c 'report: from orch-executor e5 via SubagentHandback' out33) (want 1)"
echo '{"status": "done", "confidence": 0.9, "evidence": []}' > mine.json
python3 $T verify tsk_v3 --result mine.json > out33 2>&1; echo "verify, a file of mine exit=$? (want 0)"
echo "labelled the orchestrator's: $(grep -c 'SUPPLIED BY THE ORCHESTRATOR' out33) $(python3 -c "import json; print(json.load(open('.work/_verify/tsk_v3.json'))['report']['source'])") (want 1 orchestrator)"
echo "{\"tool_name\":\"Write\",\"tool_input\":{\"file_path\":\"$PWD/.work/_results/tsk_v3/e5.json\"}}" \
  | python3 .claude/hooks/orch-guard.py 2>/dev/null; echo "Write into _results exit=$? (want 2)"
j 'cp mine.json .work/_results/tsk_v3/e5.json'   "cp into _results"      2
j 'cat > .work/_results/tsk_v3/e5.json <<EOF
{}
EOF'                                               "heredoc into _results" 2
j 'cat .work/_results/tsk_v3/e5.json'              "read _results"         0
j 'cp .work/_results/tsk_v3/e5.json /tmp/e5.json' "copy out of _results"  0
mkdir -p .orch/approvals
printf 'plan: docs/plan.md\nuser: "run them all"\n' > .orch/approvals/apr_p33.md
sed -i 's/^approved: "shown"/approved: "apr_p33"/' .orch/queue/tsk_v3.yaml
python3 $T prompt tsk_v3 >/dev/null 2>&1; echo "a plan approval record exit=$? (want 0)"
echo 'retires: [chk_001]' >> .orch/queue/tsk_v3.yaml
python3 $T prompt tsk_v3 > out33 2>&1; echo "a plan never covers retires: exit=$? (want 1)"
sed -i '/^retires:/d; s/^approved: "apr_p33"/approved: "shown"/' .orch/queue/tsk_v3.yaml

# 34. the merge gate (slm ixvec): a task branch merges only as a command of
#     its own, once verify said done at its tip and finish ran. The exact line
#     that merged a failed verdict in the field is refused before any of it runs.
j "python3 $T verify tsk_v3 | tail -2 && python3 $T finish tsk_v3 --error none | grep merge: ; git merge --no-edit orch/tsk_v3 2>&1 | tail -1" \
                                                 "the field's chained line" 2
j 'git merge --no-edit orch/tsk_v3'             "verified, not finished"   2
python3 $T finish tsk_v3 --error none > out34 2>&1; echo "finish exit=$? (want 0)"
echo "finish says whose report: $(grep -c 'report: supplied by the orchestrator' out34) (want 1)"
j 'git merge --no-edit orch/tsk_v3'             "finished"                 0
j "cd $PWD && git merge --no-edit orch/tsk_v3 | tail -1" "cd before, a pipe after" 0
j 'bash -c "git merge --no-edit orch/tsk_v3"'   "inside a wrapper"         2
j 'git merge --no-edit orch/tsk_none'           "never verified"           2
j 'git merge --abort'                           "not a task branch"        0
git branch -q orch/tsk_f34 HEAD && mkdir -p .work/_verify && cp .orch/queue/done/tsk_v3.yaml .orch/queue/done/tsk_f34.yaml
printf '{"verdict": "failed", "head": "%s"}\n' "$(git rev-parse HEAD)" > .work/_verify/tsk_f34.json
j 'git merge --no-edit orch/tsk_f34'            "verdict failed"           2
git branch -qD orch/tsk_v3 orch/tsk_f34
rm -rf out33 out34 mine.json .orch/approvals .orch/queue/done/tsk_v3.yaml .orch/queue/done/tsk_f34.yaml .orch/traces/trc_v3*.json .orch/registry/t.yaml .work/_verify .work/_prompts .work/_results

# 35. the lint, from the 2026-10-06 review (up_0014): a bare file name is
#     looked up by its path in git, `-answers.json` in prose is no file, a
#     placeholder counts only as a word of a command, "every X" only for code.
#     And (ixvec) a grep -c over two files can never equal a number; a ceiling
#     reads as `stdout <= N`; a test file run outside scope is named.
git add -A && git commit -qm pre35 >/dev/null
mkdir -p scripts && echo 'def run(): return 0' > scripts/tool35.py && printf 'def total(): return 1\n' > src/m.py
git add -A && git commit -qm base35 && b35=$(git rev-parse HEAD)
pz() { printf 'task_id: tsk_z\nrole: executor\ngoal: "change total"\nbase: %s\nscope: {paths: ["src/m.py"]}\nacceptance:\n  - %s\nnotes: "%s"\n' "$b35" "$1" "$2" > .orch/queue/tsk_z.yaml; }
lz() { python3 .claude/hooks/orch-lint.py packet .orch/queue/tsk_z.yaml $2 > lint.out 2>&1; echo "$1 exit=$? (want $3)"; }
W="$PWD/.work/tsk_z"; C='{type: executable, cmd: "grep -c return src/m.py", expect: "stdout == 1"}'
pz "$C" 'Edit m.py after reading tool35.py and scripts/tool35.py; it writes runs/p1-answers.json, and the -answers.json and -retrieved.json files are data.'
lz "names in prose" "" 0; echo "no false citation: $(grep -c 'not in git at base' lint.out) $(grep -c '^warn' lint.out) (want 0 0)"
echo x > scripts/scratch35.py; pz "$C" 'Read scratch35.py first.'
lz "an untracked file" "" 0; echo "it is named: $(grep -c 'scratch35.py, which is not in git at base' lint.out) (want 1)"
rm scripts/scratch35.py; pz "$C" ''
printf 'Work in %s from %s. Print `<out>: replaced <n> of <N> rows (base <base>, alt <alt>)`.\n' "$W" "$b35" > pz.md
lz "a format template" "--prompt pz.md" 0
printf 'Work in %s from %s. Then run `git diff --name-only <base> HEAD`.\n' "$W" "$b35" > pz.md
lz "a placeholder in a command" "--prompt pz.md" 1
pz "$C" 'Check them all, prints `pass`; each path builds its own `results`; call every `total(...)` site.'
lz "every-x" "" 0; echo "words skipped, code checked: $(grep -c '`pass` also\|`results` also' lint.out) $(grep -c '`total` also occurs' lint.out) (want 0 1)"
pz '{type: executable, cmd: "grep -cE ^def src/m.py scripts/tool35.py", expect: "stdout == 0"}' ''
lz "grep -c over two files" "" 1
pz '{type: executable, cmd: "cat src/m.py scripts/tool35.py | grep -cE ^def", expect: "stdout == 2"}' ''
lz "one stream" "--measure" 0
pz '{type: executable, cmd: "grep -n return src/m.py", expect: "stdout == 1"}' ''
lz "not a number at base" "--measure" 1
pz "{type: executable, cmd: \"git diff $b35 HEAD -- src/m.py | grep -c '^-[^-]'\", expect: \"stdout <= 2\"}" ''
lz "a ceiling" "" 0
pz '{type: executable, cmd: "python3 tests/test_x35.py", expect: "exit 0"}' ''
lz "a test run unchanged" "" 0
echo "named, not warned: $(grep -c 'runs unchanged (outside scope.paths): tests/test_x35.py' lint.out) $(grep -c '^warn' lint.out) (want 1 0)"
python3 -B -c "
import importlib.util as u; s = u.spec_from_file_location('l', '.claude/hooks/orch-lint.py'); l = u.module_from_spec(s); s.loader.exec_module(l)
print('ceiling:', l.holds('stdout <= 2', 0, '2\n'), l.holds('stdout <= 2', 0, '3'), l.holds('stdout <= 2', 0, 'a:1'))"
echo "(want ceiling: True False False)"
rm -f pz.md lint.out .orch/queue/tsk_z.yaml

# 36. memory claims: a number a sentence states must be in what it cites, and
#     a fact block that only re-cites one tracked doc is flagged as a copy.
M3=.orch/memory; cp $M3/INDEX.md index.bak
mkdir -p docs && printf '# results\nG4 nets +24 answers, p 0.011.\n' > docs/r36.md && git add docs/r36.md && git commit -qm r36
cm() { printf -- '---\nid: blk_n\ntype: %s\nheadline: "g4 nets answers"\nsource: {task: tsk_n6, artifact: "docs/r36.md:2"}\n---\n%s\n' "$1" "$2" > $M3/blocks/blk_n.md; }
printf 'blk_n · p · fact · g4 nets answers\n' >> $M3/INDEX.md
echo "2026-10-07T00:00:00Z new blk_n g4 nets answers" > $M3/receipts/tsk_n6.log
lmn() { python3 .claude/hooks/orch-lint.py memory --task tsk_n6 > lint.out 2>&1; echo "$1 exit=$? (want $2)"; }
cm procedure 'G4 nets +24 answers [src: docs/r36.md:2 "G4 nets +24 answers, p 0.011"].'; lmn "a number as cited" 0
cm procedure 'G4 nets +28 answers [src: docs/r36.md:2 "G4 nets +24 answers, p 0.011"].'; lmn "a number its source lacks" 1
echo "it is named: $(grep -c 'states +28' lint.out) (want 1)"
cm fact 'G4 nets +24 answers [src: docs/r36.md:2 "G4 nets +24 answers, p 0.011"].';      lmn "a fact copied from a doc" 0
echo "flagged as a copy: $(grep -c 'restates a tracked doc' lint.out) (want 1)"
rm $M3/blocks/blk_n.md $M3/receipts/tsk_n6.log; cp index.bak $M3/INDEX.md; rm -f index.bak lint.out

# 37. measurement, from the same review: a stage of a measurement no
#     committed file names is refused — the rule comes first — and so is one
#     that cannot finish under the job cap. Totals are labelled, this
#     session's beside the project's; --new says when nothing changed. A
#     conclusion whose data predates its pre-registration registers only post
#     hoc, and a results doc must say so where it cites it.
git add -A && git commit -qm pre37 >/dev/null
S=.claude/hooks/orch-stage.py
python3 $S run msr_z-score -- 'true' 2>/dev/null; echo "no pre-registration exit=$? (want 3)"
python3 $S run msr_z-fetch --no-prereg "downloads the model" -- 'true' >/dev/null; echo "a setup stage exit=$? (want 0)"
printf 'scores\n' > z-early.jsonl; sleep 1
printf '# msr_z — pre-registration\nnet > 0 at matched abstention\n' > msr_z.md && git add msr_z.md && git commit -qm "prereg msr_z"
python3 $S run msr_z-score -- 'true' >/dev/null; echo "pre-registered exit=$? (want 0)"
python3 $S run msr_z-long --timeout 9000 -- 'true' 2>/dev/null; echo "a timeout over the cap exit=$? (want 3)"
python3 $S run msr_z-gen --est 130 -- 'true' > out37 2>&1; echo "an estimate over the cap exit=$? (want 3)"
echo "with the arithmetic: $(grep -c 'expected 130 min (--est), and one job gets 115' out37) (want 1)"
python3 $S run msr_z-gen2 --like msr_z-score -- 'true' >/dev/null; echo "as long as a short stage exit=$? (want 0)"
python3 -c "import json; print('default timeout:', json.load(open('.work/_stages/msr_z-gen2.json'))['timeout'])"
echo "(want default timeout: 6900.0)"
python3 $S status 'msr_z*' > out37; echo "the total says ALL: $(grep -c '^total: 3 stage(s), .* — ALL of them' out37) (want 1)"
sleep 1; echo '{"source": "startup"}' | python3 $S status --brief >/dev/null
python3 $S run msr_z-after -- 'true' >/dev/null
python3 $S status 'msr_z*' > out37; echo "this session: $(grep -c '^this session: 1 stage(s)' out37) (want 1)"
echo '{"source": "compact"}' | python3 $S status --brief >/dev/null
python3 $S status 'msr_z*' > out37; echo "compaction keeps it: $(grep -c '^this session: 1 stage(s)' out37) (want 1)"
python3 $S status --new >/dev/null; python3 $S status --new > out37; echo "nothing new: $(grep -c '^nothing new since' out37) (want 1)"
python3 $S run msr_z-last -- 'true' >/dev/null; python3 $S status --new > out37
echo "one change: $(grep -c 'msr_z-last *unseen -> done' out37) $(wc -l < out37) (want 1 1)"
printf 'import sys\nprint(sum(1 for l in open(sys.argv[1]) if l.strip()))\n' > scripts/z37.py && git add scripts/z37.py && git commit -qm z37
rz() { python3 $T register --eval --msr msr_z --project t --cmd "python3 scripts/z37.py $2" $3 > out37 2>&1; echo "$1 exit=$? (want $4)"; }
rz "data older than the rule" z-early.jsonl "--capture" 1
rz "registered post hoc"      z-early.jsonl "--capture --post-hoc rule-written-late" 0
printf 'a\nb\n' > z-late.jsonl
rz "data after the rule"      z-late.jsonl "--capture --conclusion two-rows" 0
echo "captured, kept: $(grep -c 'expect: "stdout == 2"' .orch/registry/t.yaml) $(grep -c 'conclusion: "two-rows"' .orch/registry/t.yaml) $(grep -c 'post_hoc: "rule-written-late"' .orch/registry/t.yaml) (want 1 1 1)"
c=$(python3 -c "import re; print(re.findall(r'id: (chk_\d+)(?:(?!- id:).)*?post_hoc', open('.orch/registry/t.yaml').read(), re.S)[0])")
printf 'Early rows: 1 [%s].\n' "$c" > r37.md; python3 .claude/hooks/orch-lint.py doc r37.md > out37 2>&1
echo "post hoc, unsaid: $(grep -c 'registered post hoc' out37) (want 1)"
printf 'Early rows, post hoc: 1 [%s].\n' "$c" > r37.md; python3 .claude/hooks/orch-lint.py doc r37.md > out37 2>&1
echo "post hoc, said: $(grep -c 'registered post hoc' out37) (want 0)"
rm -rf out37 r37.md z-early.jsonl z-late.jsonl .work/_stages .orch/registry/t.yaml
echo "leftovers: $(git worktree list | grep -c '_lint_\|_base_\|tsk_') $(ls -d .claude/hooks/__pycache__ 2>/dev/null | wc -l) (want 0 0)"
```

Expected: the usage line and `exit=0` in step 1b (the scan's own invocation
carries no policy literal, so the guard lets it through), 4 blocks in step 2,
allow in step 3, `0 2 0 2 2` in step 4, step 5 printing `loosening ignored`,
`0 2 0 2` in steps 9–10, and every line of steps 11–37 printing its `want`
(305 `want` lines in all, the last `leftovers: 0 0`).
Before the fix, step 9's scan exits 2 (`no policy`) and step 10's exits **1** on
a `UnicodeDecodeError`, which is the findings code. Steps 15–16 were also run
against a real field repo's packets and memory: the lint flagged both
`git status` checks and the scratchpad script, listed the six files outside
qvec's scope that use `Retriever` (given the review's quoted "every
`Retriever(...)`" line — the prompt itself was not kept), and found an INDEX
headline the manual repair had missed: a block title still stating the old
behaviour.

Steps 18–22 are v4 (session review, 2026-10-03). Step 18's `wrong-run-name`
packet is `up_0007`'s shape: a script that exists, opening a file whose name a
wrong run name made impossible. v3 printed `fails at base` for it, as it does for
a missing feature. Step 21 kills runners with `kill -9` and sleeps a few
seconds; it takes about ten. Step 22 exercises the blind hook directly, and
step 23 runs it as `settings.json` wires it. Step 23 was added 2026-10-04,
after tier 2 found the v4 judge unrestricted: against a v4 install it prints
`judge canary exit=0`, `judge bash exit=0`, `canary logged: 0` and
`wired once: 0 1`, while step 22 passes in full. Whether Claude Code actually
runs the entry for `orch-judge`'s calls is still tier 2.

Steps 24–29, and the last four lines of step 13, are v5 (session review,
2026-10-05). Run against v4's tools, 50 of their lines fail and steps 1–23
still pass, so each test fails without the fix it covers:
- `model.eval()` and `requirements-train.txt` (step 13);
- every text-not-a-command case of step 24, and every bypass it closes:
  `newline ends a command`, `comment, then a wrapper`, `line continuation`,
  `sh -c mid-line`, `rm -fr` — each `exit=0` on v4;
- the whole waiver, fast-lane, bookkeeping, helper and claim-citation
  sequences (steps 25–29), which v4 has no command for.

Step 28's numbers are built into its fixture. The helper itself was checked
against the field session's own scorer on its real runs (§5.1).

Steps 30–37 are v6 (session review, 2026-10-06/07). Three older steps changed
with the behaviour they test: step 18's not-in-git file is now one on disk
that git does not track (a name found nowhere is prose, and no longer warns),
and steps 20 and 27 commit a pre-registration naming their measurement before
its data exists. Run against v5's tools, 59 lines fail — step 18's changed
line and lines in every one of steps 30–37 — and steps 1–29 otherwise pass.
Step 35's cases are the field's shapes, and the lint itself was first run on
that session's 18 real packets: 129 warnings and a false error before, 13
warnings after (9 of them only because master has moved on since). Step 32 replays `settings.json`, as step 23 does, because a hook is
enforcement only where it is wired.

Step 4 is the one to actually read. Scope enforcement fails **silently** when it
fails: a packet the guard cannot parse produces no error, no log line, and no
block — just an agent that can write anywhere. That is why the test asserts both
YAML styles and a packet with no scope, rather than the one shape a passing run
happens to use.

### Tier 2 — needs a Claude Code restart

**Restart the session.** Hooks, subagents, and skills register at session
start; before that, none of them exist.

| check | how | expect |
|---|---|---|
| SessionStart fires | start a session in the repo | an `## ORCH state` line with checkpoint/queue counts |
| skills registered | `/orch-` and look at the completions | four `orch-*` skills listed |
| subagents registered | ask "what agents are available?" | eight `orch-*` agents |
| CLAUDE.md core loaded | ask "what are the ORCH risk triggers?" | answers from the §2 stanza without reading a file |
| **guard actually blocks a live call** | ask it to `git push --force` on a throwaway branch | the tool call is refused with the ORCH checkpoint text — this is the one that proves oversight is code, not prompting |
| **blind judge is restricted** | `mkdir -p .work/_blind/msr_t && echo 'item 1: 2+2' > .work/_blind/msr_t/items.md`, then dispatch `orch-judge` on it with canary `<repo>/.orch/state.json` | its reply says `"canary": "refused"`, AND `.work/_blind-denied.log` gained a `Read …/.orch/state.json` line. The log line is the proof; the reply is testimony. No line = the `settings.json` `--blind` entry is not running (workspace trust? an install older than the entry — `--check`?). v4 as first shipped wired it in the judge's frontmatter and failed this row (2026-10-04, 2.1.289, `claude -p`); the `settings.json` entry passed it |
| stage status at session start | run `python3 .claude/hooks/orch-stage.py run t1 -- true`, then restart | a `## ORCH stages: 1 done` line |
| **an agent's report is recorded** | in auto mode, dispatch any `orch-executor` packet | `.work/_results/<task_id>/<agent_id>.json` exists with `"via": "SubagentHandback"`, and `verify` prints `report: from orch-executor …`. If the agent first sent prose, `.work/_results/refusals.log` has a line and the record says `"refusals": 1`. Not yet seen live when v6 shipped: that an agent resends after a refused handback — the hook's reason is in its stderr, as for any refused call |
| **plain-mode report** | the same, outside auto mode | the record says `"via": "SubagentStop"` |
| **a judge delivers** | the blind-judge row above | the canary line, AND `.work/_results/<msr_id>/<agent_id>.json` holding its labels. v5 failed the second half: SubagentHandback refused six of six times |
| **the archivist is held to its lint** | an ingest that leaves a claim uncited | its first report is refused with the lint's errors (`lint` lines in `refusals.log`) and it repairs before reporting |
| this session's total | restart, run a stage, `orch-stage.py status` | a `this session:` line counting that stage only |

### Tier 3 — one real task

The only test that exercises the whole loop. Use a small, genuine bug.

1. Write two memory blocks first (`/orch-memory`) — the project's test command
   and one thing that reliably breaks.
2. `/orch-task`, mode L1, so you approve each phase and can see the shape.
3. Watch for, specifically:
   - a worktree appears under `.work/` and the live tree is untouched
   - the commit carries an `Orch-Task:` trailer
   - the acceptance check is actually re-run by the orchestrator, not just
     claimed by the agent
   - the packet lint ran before the confirmation, and the counts in the packet
     match what `--measure` printed at `base`
   - a new test is shown failing at `base` (§5 step 4)
   - a passing check lands in `.orch/registry/<project>.yaml`
   - the prompt was rendered by `orch-task.py prompt`, and `verify`'s verdict —
     not the agent's status — is what the summary reports
   - `verify` prints `report: from orch-executor … via …`: the report was
     recorded by the hook, not saved by hand
   - a trace lands in `.orch/traces/` with every key, written by `orch-task.py
     trace`, and `orch-lint.py memory --task` exits 0 after the archivist —
     every sentence of each new block citing `[src: path:line "excerpt"]`
   - the packet carries `approved:`, the work was committed by `verify
     --commit`, and `finish` did the rest; the summary's merge line is the one
     it printed
4. Deliberately break it: give a packet a `scope.paths` that excludes the file
   the fix needs. The correct outcome is `escalate` naming the right file —
   **not** a quietly widened scope. If it widens scope, the guard is not wired.

### Known-good baseline

On a fresh Debian-ish box with Python 3.11+: tier 1 passes in full; the
installer is idempotent (re-running prints `same` for every file); it merges
rather than clobbers a pre-existing `CLAUDE.md`, `.claude/settings.json`, and
`.gitignore`; and a v1 install upgrades in place — the ORCH block is replaced by
version with the user's own text preserved above and below it, a hand-tightened
`confidence_floor` survives, and the shipped playbooks land beside the edited
ones as `*.new` rather than overwriting them.

---

## 8. Sharp edges

Learned from running the original, not theorized:

- **Always scope a registry replay to one project**, or it runs one project's
  checks against another's repo.
- **Unshipping looks exactly like breaking** to a registry replay: code changed,
  check fails, bisect names the commit. Only intent tells them apart, so intent
  is written down — declared in the packet, carried in an `Orch-Retires:`
  trailer, and undone by reverting the commit.
- **A registry entry outlives the check that was wrong.** A weak check that
  passed is evidence pointing the wrong way, and it stays that way until someone
  marks it `superseded`. Better checks added later do not cancel it out.
- **A packet scoped to a test file is gameable** unless a second check greps for
  the assertions that must survive.
- **A floor assertion is not a correctness check.** `>=1 chapter marker` passes
  on the one you wanted and on the 394 you did not. Assert the exact count, or
  admit in `notes` that you do not know it.
- **Absence needs a command.** "Writes nothing", "installs nothing", "leaves X
  alone" stated in prose is verified by nothing.
- **Capture agent writes before running acceptance checks.** Otherwise the
  checks' own byproducts read as scope creep, and the guard fires on every repo
  with an imperfect `.gitignore`.
- **A default budget that trips the spend trigger on every task** trains everyone
  to ignore the trigger. Set a real one per packet. Acceptance that invokes a
  model, a build or a GPU job is 3–5× a text task; budget it before dispatch,
  and never fix an expected overrun by raising `spend_fraction`.
- **`blocked` must be selectable by the loop**, or a task interrupted by a usage
  limit can never resume.
- **A trigger that fires on talking about a command, not running one, gets
  routed around.** `git commit -m "document the git push workflow"` is not a
  push. Match at command position — quoted multi-word arguments cannot be
  commands — and recurse into the two things that really do execute their
  argument, a shell wrapper and a command substitution. Keep SQL and
  pipe-to-shell on the raw string: neither is a single argv.
- **A noticed issue that is not written down did not happen.** ORCH learns
  about a project through memory blocks; it learns about *itself* only through
  `.orch/upstream.md` and a human carrying it back. Every defect in the first
  field report was stated out loud in the session that hit it, and reached this
  file only because a human wrote it down afterwards.
- **A scope the guard cannot parse is not a scope.** Silent disarmament is the
  worst failure this system has: no error, no log line, an agent writing
  anywhere, and an adversarial-scope test that passes for the wrong reason.
  Accept both YAML styles and block when a declared task has no readable scope.
- **A mandatory checkpoint must never lower a task's fitness.** If it does, the
  only lever left for raising the score is loosening a risk trigger — and the
  system will find it, and it will look like an improvement.
- **The orchestrator is the commonest defect source, so its output gets code
  checks too.** One field session: five packet defects across four tasks, none
  shipped, all of them the orchestrator's. A rule the orchestrator must remember
  is a rule it will eventually forget; `orch-lint.py` is the rules it kept
  forgetting.
- **By the time checks run, the tree is clean.** §5 commits first. A check on
  `git status` asserts nothing about the task; diff against `base`.
- **A count is measured at base, never remembered** — and never taken in a
  checkout holding other tasks' unmerged work. "22 test files" counted across
  unmerged branches was wrong in exactly that way.
- **A check in the scratchpad is a check nobody can run again.** Commit it.
- **A set that grew is not a check that was wrong.** Register the additions;
  supersede only what asserted the wrong thing.
- **Rewriting a memory block in place makes the index lie.** Counts still match;
  only a headline-to-block comparison sees it. Supersede = new block + archive.
- **An archivist's count is testimony.** Quote the lint's line, never a number
  the model produced (95 reported, 85 real).
- **Most decisions come from measurements, not code changes.** If the scorer
  is untracked and the noise unmeasured, the decision rests on nothing that can
  be replayed. A determinism check that finds 7/40 answers changing between
  identical runs re-prices every earlier conclusion at once.
- **Refused by the platform is not failed.** A classifier outage blocked every
  call, `pwd` included, and stalled an executor mid-draft. Resuming that same
  agent afterwards worked; a fresh one would have started over.
- **Do not stop a run to wait for a merge.** Stack dependent branches, and hand
  over one `!` line at the end. Every per-task merge stop is a round-trip that
  "run it without asking me" was meant to remove.
- **The orchestrator's quick analysis is outside every gate, and that is where
  the worst errors came from.** Twelve tasks went through packets with no
  serious defect; one inline script that re-implemented part of the scorer
  produced both wrong conclusions of the session. A number counts when tracked
  code prints it — and a doc cites the registry entry that does.
- **A check can fail at base for the wrong reason.** A wrong run name fails
  exactly like the missing feature it was meant to detect. Read *why* it failed:
  a file outside the scope that cannot exist is a wrong command.
- **A stacked branch cannot see master.** A prompt that cites a doc section added
  on master after the fork points the agent at nothing; neither the prompt nor
  the agent knows.
- **Never wait inside a background job.** The wait spends that job's time cap and
  the next stage dies midway; `pgrep -f` matches its own command line and never
  fires. One stage per job, its outcome a file, the next started from outside.
- **Typed bookkeeping drifts.** Registry entries appended through heredocs, and
  traces whose keys came and went, are a record nobody can sweep reliably.
  Generate both from the verification run.
- **Blindness asserted is testimony.** A judge that says it never opened the
  results cannot be checked. A hook that refused it — logged — can.
- **A hook is enforcement only where the harness runs it.** v4's blind hook
  passed every direct test and sat in the judge's frontmatter, which Claude
  Code never ran; the first live judge read its canary. Wire a hook where it has
  been seen to fire, and test the wiring — replay `settings.json` — not only
  the script.
- **The orchestrator's own findings need a route to memory.** Agents' facts flow
  through every task; a measurement's result, its cause and the gotchas met
  along the way stayed in docs until the archivist accepted them from the
  orchestrator too.
- **A pattern for a function name matches every method of that name.**
  `\beval\(` flagged torch's `model.eval()` and forced two mandatory stops in
  one ML session. Every repo that runs a model would hit it. Give such a pattern
  a non-identifier left edge, as `exec` and `fetch` already had.
- **A waiver that code cannot see does not exist.** The procedure said an
  approved checkpoint is "waived once", but the tools derived the finding
  afresh on every run and registered only a clean `done`. Two verified tasks
  merged with no registry entries. A decision the user makes has to be a record
  the tool reads.
- **An exact-name list misses the next name.** `requirements.txt` was gated,
  and `requirements-train.txt` (torch, transformers) went through. A gate on a
  family of files matches the family.
- **Text written to a file is not a command, and a newline ends one.** The
  guard blocked a checkpoint note, a result file and a scratch script. In
  each, the matched text was something being written, not run. The same
  review found the opposite hole: the lexer read a newline as a space, so
  `ls⏎bash -c "…"` hid its wrapper. Read what bash runs — no less, and no more.
- **Fourteen steps for two files was the typing, not the gates.** The gates
  are the worktree, the subagent, the re-run checks and the scan. Commit,
  register, trace, prune and the merge check are one command each now, at
  most.
- **A flag the command does not pass runs the script's default.** A
  pre-registered 1 % split met a 5 % default, and only a manual read caught
  it. Stages check pre-registered parameters, and an omitted flag counts as a
  mismatch.
- **An invented sentence looks exactly like a true one.** A lint that checks
  only structure passed a memory block describing a workaround that never
  happened. Require an excerpt per claim, and look for it where the claim
  says it is.
- **A replay that reads an untracked file is local.** The registry kept a
  sha256 and no file, so a fresh clone could replay nothing that depended on a
  cache. Say so on the entry, and treat a missing input as missing, not as a
  break.
- **The harness decides what a subagent's report is.** In auto mode it is the
  `SubagentHandback` message — "your full report" — and text after it is never
  delivered. An agent definition saying "end your reply with the JSON" competed
  with that and lost, 15 times in 19. Check a report where it leaves the agent,
  tell the agent why, and record what passes there.
- **A result file anyone can write is testimony.** Two result files the
  orchestrator wrote read exactly like the agents'. Record the report from the
  hook input, where the agent's identity comes from the harness, and label
  whatever the orchestrator supplies as its own.
- **A check that refuses all but reads refuses the report too.** The blind
  check was right about reads and wrong about the one call that delivers a
  judge's labels; six judges finished and none could deliver.
- **A gate the orchestrator must remember to read is not a gate.** A merge ran
  on a `failed` verdict because a pipe hid the exit code and a `;` ignored it.
  The guard reads the verify record before the merge runs.
- **A warning that is usually wrong teaches dispatching past an error.** 129
  lint warnings over 18 packets were nearly all bare file names; one real
  error went out with them. Resolve a name before warning about it, and warn
  only about what the agent will actually miss.
- **A total is not a session.** A project's 23.72 GPU hours went to the user
  as one session's, which was about 10. Print every total with its window.
- **Code cannot judge a pre-registration, but it can date one.** Whether a rule
  is good is judgment; whether it was committed before the data it decides on
  is a git log. Check what can be checked, and label the rest post hoc.
