# ORCH.md — portable agent-orchestration layer for Claude Code

**One file. Drop it in any repo. It installs itself.**

This is a port of the `orch` system (SQLite + Python CLI + headless `claude -p`
subagents) onto Claude Code's own primitives: subagents, skills, hooks, and
plain files. Same architecture, no Python package, no database, no daemon.

## 0. Install

Deterministic — copy `ORCH.md` into the repo root and run this. It extracts the
26 `FILE:` blocks in §5 verbatim; no model is in the loop, so nothing can be
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
assert len(blocks) == 26, f"expected 26 FILE blocks, found {len(blocks)}"

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
    for h in ("orch-guard.py", "orch-scan.py", "orch-lint.py", "orch-task.py", "orch-stage.py"):
        (pathlib.Path(".claude/hooks") / h).chmod(0o755)
    pathlib.Path(".work").mkdir(exist_ok=True)
    # git does not track empty dirs; .gitkeep keeps the layout intact on clone
    for d in ("memory/blocks", "memory/archive", "memory/receipts", "cards", "queue", "registry",
              "checkpoints", "traces"):
        q = pathlib.Path(".orch") / d
        q.mkdir(parents=True, exist_ok=True)
        (q / ".gitkeep").touch()
    gi = pathlib.Path(".gitignore")
    if ".work/" not in (gi.read_text() if gi.exists() else ""):
        gi.write_text((gi.read_text().rstrip("\n") + "\n" if gi.exists() else "") + ".work/\n")

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
byte-exact where the model is not, and reading this file costs **~46k tokens of
context** where the script costs none.

**Then restart Claude Code.** Hooks, subagents, and skills are registered at
session start; until you restart, the guard is not enforcing anything.

Verify (see §7 for the full test procedure):

```bash
ls .claude/agents/orch-* .claude/skills/orch-*/SKILL.md && \
echo '{"tool_name":"Bash","tool_input":{"command":"git push --force"}}' \
  | python3 .claude/hooks/orch-guard.py; echo "exit=$? (want 2)"
```

To uninstall: `rm -rf .claude/agents/orch-* .claude/skills/orch-* .claude/hooks/orch-*.py .orch`, delete everything from `<!-- ORCH` through `<!-- /ORCH -->` in `CLAUDE.md`, and remove the ORCH hook entries from `.claude/settings.json` (two `PreToolUse`, both running `orch-guard.py`; two `SessionStart`).

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
| what | §2 core (45 lines) + every skill's and agent's `description` line + the SessionStart memory headlines | `.claude/skills/orch-*` bodies | `.claude/agents/orch-*` bodies (subagent-only), block bodies, traces |
| cost | ~770 + ~760 = **~1.5k tokens**, plus headlines | ~0.8k–5k when invoked (`orch-task` is the large one) | 0 |

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
skill had loaded and nothing in the core said a quick script was a risk.

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
  through, at the cost of one Python start per call.
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
<!-- ORCH v4 — orchestration layer. Source of truth: ORCH.md. -->
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
declared scope — at the tool boundary, during your own manual work too. If it
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
  hooks/orch-task.py          # prompt, verify, register, trace: the task bookkeeping, generated
  hooks/orch-stage.py         # long jobs as resumable stages, one background job each
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
  traces/trc_*.json           # scored history
  state.json                  # loop state, counters
.work/                        # gitignored: worktrees, and the tools' scratch state
  <task_id>/                  # one worktree per task
  _prompts/ _verify/          # rendered prompts, verify records (orch-task.py)
  _stages/                    # stage status, exit codes, logs (orch-stage.py)
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
needs_decision · failed · escalate. Acceptance checks are re-run by the
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

Note for orchestrators: this guard matches its own policy. An inline grep for
the sensitive.yaml content_patterns carries the force-push pattern in argv and
is blocked, read-only or not. Use .claude/hooks/orch-scan.py, which loads the
patterns from the YAML at runtime. Do not add a name-based exemption here.
"""
import json
import os
import re
import shlex
import sys
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
    (r"\brm\s+-[a-zA-Z]*r[a-zA-Z]*f\b", "recursive force delete"),
    (r"\b(npm|pnpm|yarn)\s+publish\b", "package publish"),
    (r"\b(twine|poetry)\s+(upload|publish)\b", "package publish"),
    (r"\bdocker\s+push\b", "image push"),
    (r"\b(kubectl|helm)\s+(apply|delete|upgrade|rollout)\b", "cluster mutation"),
    (r"\bterraform\s+(apply|destroy)\b", "infrastructure change"),
    (r"\bgh\s+(pr|release|issue)\s+(create|merge|edit|close)\b", "GitHub write"),
    (r"\bchmod\s+777\b", "world-writable chmod"),
]

# CONTENT is matched against the raw string, quoted arguments included, because
# these are not a single argv and cannot be: SQL reaches a client as a quoted
# payload, and pipe-to-shell IS the pipeline — split it into commands and the
# pipe that makes it dangerous is gone.
# The cost is that `grep "DROP TABLE" .` checkpoints. Erring toward a stop on
# destructive SQL is the right side to be wrong on; erring toward one on every
# commit message that says "git push" is not.
CONTENT = [
    (r"\b(DROP|TRUNCATE)\s+TABLE\b", "destructive SQL"),
    (r"\bDELETE\s+FROM\b(?!.*\bWHERE\b)", "unbounded DELETE"),
    (r"\bcurl\b[^|]*\|\s*(ba)?sh\b", "pipe-to-shell"),
]

SEPARATORS = {";", "&", "&&", "||", "|", "(", ")", "{", "}", "\n"}
# things that really do execute their quoted argument
WRAPPERS = {"bash", "sh", "zsh", "dash", "ksh", "env", "eval", "exec", "sudo",
            "doas", "nohup", "timeout", "xargs", "ssh", "nice", "setsid", "command"}
SUBST = re.compile(r"\$\(([^()]*)\)|`([^`]*)`")
HEREDOC = re.compile(r"<<(-?)[ \t]*(?:'([^'\n]+)'|\"([^\"\n]+)\"|\\(\w+)|([A-Za-z_]\w*))")


def runs_shell(header):
    """Does this heredoc's header line hand its body to a shell? Conservative:
    unparseable, or any wrapper word anywhere, counts as yes."""
    try:
        lex = shlex.shlex(header, posix=True, punctuation_chars=True)
        lex.whitespace_split = True
        return any(t.rsplit("/", 1)[-1] in WRAPPERS for t in lex)
    except ValueError:
        return True


def strip_quoted_heredocs(cmd):
    """Drop the bodies of quoted heredocs (<<'X', <<"X", <<\\X) that are not fed
    to a shell. Bash expands nothing in them, so a body is data: markdown that
    shows `rm -rf a/b` in a code span is not a command substitution, and the
    guard blocking it blocked the orchestrator writing its own checkpoint.
    Unquoted heredocs keep their bodies — bash does expand $(...) there. A
    body piped to a shell (`cat <<'X' | bash`) is kept: it runs.

    Operators are found by a quote- and comment-aware scan, never a regex over
    the raw string — otherwise `echo "<<'Y'"` would hide the next line."""
    out, pending, q, i, n, head = [], [], None, 0, len(cmd), 0
    while i < n:
        c = cmd[i]
        if q:
            out.append(c)
            if c == q:
                q = None
            elif c == "\\" and q == '"' and i + 1 < n:
                out.append(cmd[i + 1])
                i += 1
            i += 1
            continue
        if c == "\\" and i + 1 < n:
            out.append(cmd[i:i + 2])
            i += 2
            continue
        if c in "'\"":
            q = c
        elif c == "#" and (i == 0 or cmd[i - 1] in " \t\n;&|()"):
            j = cmd.find("\n", i)
            j = n if j < 0 else j
            out.append(cmd[i:j])
            i = j
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
            executes = runs_shell(cmd[head:i])
            out.append("\n")
            i += 1
            for m in pending:
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
                quoted = m.group(5) is None
                out.append(body if (not quoted or executes) else "")
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
    than masked: a shell wrapper (`bash -c "..."`) and a command substitution
    (`$(...)`, backticks). Quoted heredoc bodies are data and are removed
    first — see strip_quoted_heredocs().
    """
    if depth > 3:
        return
    cmd = strip_quoted_heredocs(cmd)
    for m in SUBST.finditer(cmd):
        yield from commands(m.group(1) or m.group(2) or "", depth + 1)
    try:
        lex = shlex.shlex(cmd, posix=True, punctuation_chars=True)
        lex.whitespace_split = True
        toks = list(lex)
    except ValueError:
        yield cmd              # unparseable (unbalanced quote): match raw, never open
        return
    seg = []
    for t in toks + [";"]:
        if t in SEPARATORS:
            if seg:
                wrapper = seg[0].rsplit("/", 1)[-1] in WRAPPERS
                out = []
                for tok in seg:
                    if any(c in tok for c in " \t\n"):
                        if wrapper:
                            yield from commands(tok, depth + 1)
                        out.append("\x00")     # an argument, not a command
                    else:
                        out.append(tok)
                yield " ".join(out)
            seg = []
        else:
            seg.append(t)

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
    for pat, why in CONTENT:
        if re.search(pat, cmd, re.I):
            return why
    for seg in commands(cmd):
        for pat, why in COMMANDS:
            if re.search(pat, seg, re.I):
                return why
    return None


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
    any error denies: a broken check wedges one judge, never the session."""
    try:
        ev = json.load(sys.stdin)
    except Exception:
        return 0                          # unattributable: allowed; the canary detects it
    if ev.get("agent_type") != JUDGE:
        return 0
    tool, raw = ev.get("tool_name", "?"), ""
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
        "checkpoint to .orch/checkpoints/ and ask the user to approve or reject.\n")
    sys.exit(2)


def main():
    try:
        ev = json.load(sys.stdin)
    except Exception:
        sys.exit(0)                      # fail open: never wedge on bad input
    tool = ev.get("tool_name", "")
    ti = ev.get("tool_input", {}) or {}

    if tool == "Bash":
        why = bash_trigger(ti.get("command", ""))
        if why:
            block(f"{why} — irreversible or externally visible")

    if tool in ("Write", "Edit", "NotebookEdit"):
        fp = os.path.normpath(ti.get("file_path", "") or ".")
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
has ever run.

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
              for p in paths if Path(p).name in manifests]
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
        like a wrong command. A command the guard would block is never run.
        --prompt checks a dispatch prompt against the packet: guarded
        commands, unfilled placeholders, the base sha and worktree path, and
        files it cites that the agent's checkout will not hold as written.
memory  INDEX.md against blocks/: one line per block, no orphans, no strays,
        each headline equal to its block's own, archived blocks out of the
        index. --task also checks that every block the task's receipt names
        landed. The summary line is the count an archivist's report quotes.
doc     a results doc against the registry: every `[chk_…]` citation names an
        active entry, and the number right before it is that entry's
        registered value. Result-shaped numbers on lines that cite nothing
        are listed (errors with --strict) — a number nobody can replay.

exit 0  clean (warnings may print)
exit 1  findings
exit 2  could not read its input. Never reported as clean.
"""
import importlib.util
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
EXPECT = re.compile(r"exit\s+\d+|stdout\s*==.*|stdout\s+matches\s+/.*/|delta\s*==\s*[+-]?\d+", re.S)
TESTISH = re.compile(r"(^|/)(tests?|__tests__)/|(^|/)test_[^/]*$|_test\.[^/]+$|\.test\.[^/]+$")
ABS = re.compile(r"(?<![\w.~$:/-])(~?/[^\s'\"|;&()<>`]+)")
SYSTEM = ("/usr/", "/bin/", "/sbin/", "/lib", "/opt/", "/etc/", "/dev/", "/proc/", "/sys/")
QUANT = re.compile(r"\b(every|all|each)\b[^.\n`]{0,40}?`([^`\n]+)`", re.I)
PLACEHOLDER = re.compile(r"(?<![<\w])<(base|sha|base_sha|task_id|tsk_id|worktree)>")
CITED = re.compile(r"(?<![\w./~-])((?:[\w.-]+/)*[\w-][\w.-]*\.(?:py|md|ya?ml|jsonl?|toml|txt|cfg|ini|sh"
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
            s = re.match(r"[A-Za-z_][\w.]*", m.group(2).strip())
            sym = s.group(0).rstrip(".").rsplit(".", 1)[-1] if s else ""
            if len(sym) < 3 or sym in seen:
                continue
            seen.add(sym)
            cp = git("grep", "-lwF", sym, base, "--", ".", ":(exclude)*.md",
                     ":(exclude).claude", ":(exclude).orch")    # call sites, not prose
            files = [x.split(":", 1)[1] for x in cp.stdout.splitlines() if ":" in x]
            out = [f for f in files if not G.in_scope(f, scope)]
            if out:
                more = f" (+{len(out) - 5} more)" if len(out) > 5 else ""
                report("warn", f"\"{m.group(0)}\" — `{sym}` also occurs in {len(out)} "
                       f"file(s) outside scope.paths: {', '.join(out[:5])}{more}. Name the "
                       "ones in scope, or widen it; as written it cannot be followed")


def cited_files(texts, scope, base_sha):
    """Files a prompt, goal or notes cites that the agent's checkout will not
    hold as written. A worktree holds what git tracks at `base` and nothing
    else, so a file only on master — a doc section added after a stacked
    branch forked — is invisible to the agent that was told to read it."""
    head, seen = git("rev-parse", "HEAD").stdout.strip(), set()
    for src, txt in texts:
        for m in CITED.finditer(txt):
            rel, line = m.group(1), m.group(2)
            if rel in seen or rel.startswith((".work/", ".orch/")) or G.in_scope(rel, scope):
                continue                   # in scope: the task's to write, or to create
            seen.add(rel)
            if git("cat-file", "-e", f"{base_sha}:{rel}").returncode != 0:
                at_head = git("cat-file", "-e", f"HEAD:{rel}").returncode == 0
                report("warn", f"{src} cites {rel}, which is not in git at base {base_sha[:12]} — "
                       + ("it is at HEAD, and a stacked base does not have master's newer files"
                          if at_head else "the agent's worktree holds tracked files only"))
            elif base_sha != head and git("diff", "--quiet", f"{base_sha}...HEAD", "--", rel).returncode == 1:
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


def holds(expect, code, out):
    e = expect.strip()
    m = re.fullmatch(r"exit\s+(\d+)", e)
    if m:
        return code == int(m.group(1))
    m = re.fullmatch(r"stdout\s*==\s*(.*)", e, re.S)
    if m:
        return out.strip() == m.group(1).strip()
    m = re.fullmatch(r"stdout\s+matches\s+/(.*)/", e, re.S)
    if m:
        return re.search(m.group(1), out) is not None
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
        if d and num:
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
                   "stdout matches /re/ · delta == +k")
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
        for m in sorted(set(PLACEHOLDER.findall(ptxt))):
            report("error", f"prompt {pth}: unfilled <{m}> — the agent cannot run a placeholder")
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
    for b, p in sorted(files.items()):
        t = p.read_text(errors="replace")
        if not re.search(r"(?im)^[ \t*_#-]*sources?[ \t*_]*(:|$)", t):
            content(b, f"{b}: no source — no source, no block")
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
orchestrator's: staging the commit, the mutation check, and every judgment a
finding asks for.

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
    python3 .claude/hooks/orch-task.py verify   <tsk_id> [--test "<cmd>"] [--result <file>] [--timeout S]
    python3 .claude/hooks/orch-task.py register <tsk_id>
    python3 .claude/hooks/orch-task.py register --eval --msr <msr_id> --cmd "<cmd>" --expect "stdout == V"
                                       [--artifact <path>] [--noise "<text>"] [--project <name>]
    python3 .claude/hooks/orch-task.py trace    <tsk_id> --outcome <o> --error none|<kind> [--what ..] [...]

prompt    renders the dispatch prompt from the packet to .work/_prompts/<id>.md
          — absolute worktree path, base sha, goal, every check, scope,
          forbidden, the text of its context blocks — and lints it with the
          packet. Append task-specific prose if you must; then lint again.
verify    orch-task §5 steps 3-6, after the commit: every executable check
          re-run in the worktree (a delta one at base too), the task's changed
          tests run at base (want: fail), writes outside scope.paths whether
          committed or not, the Orch-Task trailer, the sensitive scan, and the
          agent's result read for its confidence and the blocks its evidence
          cites. Writes .work/_verify/<id>.json for register and trace.
register  appends a verified task's passing, registrable checks — or one eval
          entry, replayed once now — to .orch/registry/<project>.yaml under a
          fresh id, reads the file back with the parser every other tool uses,
          and restores it if any entry did not survive the round trip.
trace     writes .orch/traces/trc_<id>.json with every schema key present and
          every enum checked; `verification` is copied from the verify record,
          and a `done` trace without a clean one is refused.

exit 0 ok · 1 findings (verify) or refused (register, trace) · 2 could not run
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
KINDS = ("weak_acceptance", "wrong_scope", "bad_reference", "unsound_plan", "budget")


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
            "acceptance": acc, "refs": listval(txt, "context_refs"),
            "forbidden": listval(txt, "forbidden"), "network": bool(net and net.group(1) == "true")}


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


def cmd_prompt(tid):
    pk = packet(tid)
    wt, sha = ROOT / ".work" / tid, rev(pk["base"])
    if not sha:
        die(f"base {pk['base']} does not resolve to a commit")
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


def last_json(txt):
    """The result object an agent ends its reply with."""
    dec = json.JSONDecoder()
    for i in [m.start() for m in re.finditer(r"\{", txt)][::-1]:
        try:
            obj, _ = dec.raw_decode(txt[i:])
            if isinstance(obj, dict) and "status" in obj:
                return obj
        except ValueError:
            continue
    return None


def cmd_verify(tid, test, result, timeout):
    pk = packet(tid)
    wt = ROOT / ".work" / tid
    if not (wt / ".git").exists():
        die(f"no worktree at {wt}")
    head, base0 = rev("HEAD", wt), rev(pk["base"])
    if not base0:
        die(f"base {pk['base']} does not resolve to a commit")
    findings, base = [], base0             # findings: (fail | checkpoint | fix, message)
    if git("merge-base", "--is-ancestor", base, head).returncode != 0:
        base = git("merge-base", base, head).stdout.strip()
        print(f"note: base {base0[:12]} is not an ancestor of the task's HEAD (did HEAD move?); "
              f"using the fork point {base[:12]}")
    if head == base and pk["role"] == "executor":
        die("nothing is committed on top of base — commit the agent's work first (§5 step 2)")
    vf = ROOT / ".work/_verify" / f"{tid}.json"
    prior = set(json.loads(vf.read_text()).get("byproducts", [])) if vf.exists() else set()
    rec = {"task_id": tid, "at": now(), "base": base, "head": head, "checks": [],
           "tests_at_base": None, "changed": [], "scope_violations": [], "uncommitted": [],
           "byproducts": sorted(prior), "scan": None, "result": None, "mutation": None}

    # what changed: committed, and anything left in the tree. A Bash write is
    # invisible to the guard (up_0006); it is not invisible here.
    rec["changed"] = [x for x in git("diff", "--name-only", base, head).stdout.splitlines() if x]
    for f in rec["changed"]:
        if not G.in_scope(f, pk["scope"]):
            rec["scope_violations"].append(f)
            findings.append(("checkpoint", f"committed outside scope.paths: {f}"))
    for f in dirty(wt):
        if f in prior:                     # an earlier verify's checks made it, not the agent
            continue
        rec["uncommitted"].append(f)
        if G.in_scope(f, pk["scope"]):
            findings.append(("fix", f"in scope but not committed: {f} — commit it, or say why not"))
        else:
            rec["scope_violations"].append(f)
            findings.append(("checkpoint", f"written outside scope.paths and not committed: {f} — "
                             "a Bash write the guard never saw (up_0006)"))
    trailers = git("log", "--format=%(trailers:key=Orch-Task,valueonly)", f"{base}..{head}").stdout
    if head != base and tid not in trailers.split():
        findings.append(("fix", f"no `Orch-Task: {tid}` trailer on {base[:7]}..{head[:7]} — bisect "
                         "cannot name this task"))

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
            findings.append(("fail", f"#{n}: the guard blocks this check ({why}) — not run"))
            continue
        r = L.run(cmd, wt, timeout)
        if r is None:
            findings.append(("fail", f"#{n}: timed out after {timeout:g}s"))
            continue
        c["exit"], out, err = r[0], r[1], r[2]
        c["stdout"] = out.strip()[:300]
        d = re.fullmatch(r"delta\s*==\s*([+-]?\d+)", expect.strip())
        if d:
            rb = L.at_base(cmd, base, timeout)
            if not isinstance(rb, tuple):
                findings.append(("fail", f"#{n}: delta needs the count at base, which could not be "
                                 f"measured ({rb or 'timed out'})"))
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
            findings.append(("fail", f"#{n}: does not hold — exit {c['exit']}, stdout "
                             f"\"{c['stdout'][:60]}\", expect {expect}" + (f"\n         {hint}" if hint else "")))

    if test:
        tests = [f for f in rec["changed"] if L.TESTISH.search(f)]
        rec["tests_at_base"] = {"cmd": test, "tests": tests, "exit": None}
        if G.bash_trigger(test):
            die(f"the guard blocks the test command ({G.bash_trigger(test)})")
        if not tests:
            findings.append(("fail", "--test given, but the task changed no test file: nothing can "
                             "show the change is detected"))
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
                findings.append(("fail", "the task's tests timed out at base — not measured"))
            elif r[0] == 0:
                findings.append(("fail", "the task's tests PASS at base: they cannot detect the change, "
                                 "and acceptance resting on them is a floor in disguise"))

    rec["byproducts"] = sorted(prior | (set(dirty(wt)) - set(rec["uncommitted"])))
    sc = subprocess.run([sys.executable, str(Path(__file__).resolve().with_name("orch-scan.py")),
                         str(wt), base, "--task", tid], capture_output=True, text=True, errors="replace")
    rec["scan"] = {"exit": sc.returncode, "lines": [x for x in sc.stdout.splitlines() if x.strip()][:20]}
    if sc.returncode == 1:
        findings.append(("checkpoint", "the sensitive scan found something above low — checkpoint "
                         "before anything registers"))
    elif sc.returncode != 0:
        findings.append(("fail", f"the scan could not look (exit {sc.returncode}): "
                         f"{sc.stderr.strip()[:120]}"))

    if result:
        try:
            obj = last_json(Path(result).read_text())
        except OSError as e:
            die(str(e))
        if obj is None:
            die(f"{result}: no result JSON with a `status` key")
        cites = set(re.findall(r"blk_[\w.-]+", json.dumps(obj.get("evidence", []))))
        floor = G.load_thresholds()["confidence_floor"]
        conf = obj.get("confidence")
        rec["result"] = {"status": obj.get("status"), "confidence": conf,
                         "blocked_on": obj.get("blocked_on"), "refs_cited": sorted(cites & set(pk["refs"])),
                         "refs_foreign": sorted(cites - set(pk["refs"])),
                         "open_questions": len(obj.get("open_questions") or [])}
        if obj.get("status") == "done" and isinstance(conf, (int, float)) and conf < floor:
            findings.append(("checkpoint", f"the agent says done at confidence {conf} < {floor}"))

    fails = [m for k, m in findings if k == "fail"]
    rec["verdict"] = ("failed" if fails else "needs_decision" if any(k == "checkpoint" for k, _ in findings)
                      else "incomplete" if findings else "done")
    rec["findings"] = [{"kind": k, "msg": m} for k, m in findings]
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
    r = rec["result"]
    if r:
        print(f"  agent: {r['status']}, confidence {r['confidence']}, cites "
              f"{', '.join(r['refs_cited']) or 'no given block'} of {', '.join(pk['refs']) or 'none given'}"
              + (f" · {r['open_questions']} open question(s): read them, a flagged check is one"
                 if r["open_questions"] else ""))
    for k, m in findings:
        print(f"{k:10} {m}")
    nxt = {"done": "register, then trace", "failed": "trace --outcome failed; keep the worktree",
           "needs_decision": "write the checkpoint before anything registers",
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
    new = [{"id": i, "task": tid, "commit": rec["head"][:12], "cmd": c["cmd"], "expect": c["expect"],
            "registered": date.today().isoformat(), "status": "active"}
           for i, c in zip(next_ids(have, len(todo)), todo)]
    if new:
        append(ROOT / ".orch/registry" / f"{pk['project']}.yaml", new)
    for e in new:
        print(f"  registered {e['id']}: {e['cmd'][:70]}  ({e['expect']})")
    print(f"{len(new)} registered in .orch/registry/{pk['project']}.yaml")
    return 0


def cmd_register_eval(o):
    msr, cmd, expect = o.get("--msr", ""), o.get("--cmd", ""), o.get("--expect", "").strip()
    if not re.fullmatch(r"msr_[\w.-]+", msr):
        die("--msr <msr_id> names the measurement (orch-task §8)", 1)
    if not re.fullmatch(r"stdout\s*==\s*.+", expect, re.S):
        die("an eval entry registers the value its conclusion cites: --expect \"stdout == <value>\"", 1)
    why = G.bash_trigger(cmd)
    if why or L.PLACEHOLDER.search(cmd):
        die(f"refused: {why or 'an unfilled placeholder'}", 1)
    try:
        toks = shlex.split(cmd)
    except ValueError:
        toks = cmd.split()
    for t in toks:                          # the scorer is tracked, committed code (§8)
        p = ROOT / t
        if (t == o.get("--artifact") or not p.is_file() or ROOT.resolve() not in p.resolve().parents
                or p.suffix.lower() not in SCRIPTS):
            continue
        rel = str(p.resolve().relative_to(ROOT.resolve()))
        if git("ls-files", "--error-unmatch", "--", rel).returncode != 0:
            die(f"{rel} is not tracked — a scorer that decides something is committed code (§8)", 1)
        if git("diff", "--quiet", "HEAD", "--", rel).returncode != 0:
            die(f"{rel} has uncommitted changes — the registered commit would not be what ran", 1)
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
    e.update({"noise": o.get("--noise", "unmeasured"), "registered": date.today().isoformat(),
              "status": "active"})
    append(ROOT / ".orch/registry" / f"{project}.yaml", [e])
    print(f"registered {e['id']} (eval, {msr}): {cmd[:70]}  ({expect})")
    return 0


# ── trace

def cmd_trace(tid, o):
    pk = packet(tid)
    outcome, err = o.get("--outcome"), o.get("--error")
    if outcome not in OUTCOMES:
        die(f"--outcome is one of {', '.join(OUTCOMES)}", 1)
    if err is None:
        die("--error is required: `none` is a claim that the packet was right, and the key is never blank", 1)
    if err == "none":
        oe = None
    elif err in KINDS and o.get("--what"):
        oe = {"kind": err, "what": o["--what"], "better": o.get("--better"),
              "superseded_check": o.get("--superseded-check")}
    else:
        die(f"--error is none, or one of {', '.join(KINDS)} with --what (and --better)", 1)
    mode = (o.get("--mode") or "L2").split(":")
    if not all(re.fullmatch(r"L[0-5]", m) for m in mode) or len(mode) > 2:
        die("--mode L2, or proposed:used as L2:L3", 1)
    vf = ROOT / ".work/_verify" / f"{tid}.json"
    rec = json.loads(vf.read_text()) if vf.exists() else None
    executable = any(e.get("type", "executable") != "judged" for e in pk["acceptance"])
    if outcome == "done" and executable and (rec is None or rec.get("verdict") != "done"):
        die("a done trace needs a clean verify record — the outcome is what code verified, "
            "not what the agent said", 1)
    if o.get("--mutation") and rec:
        rec["mutation"] = o["--mutation"]
    ck = {"mandatory": 0, "discretionary": 0}
    for f in (ROOT / ".orch/checkpoints").glob("*.md"):
        t = f.read_text(errors="replace")
        if re.search(rf"^task:\s*{re.escape(tid)}\s*$", t, re.M):
            k = "mandatory" if re.search(r"^class:\s*mandatory", t, re.M) else "discretionary"
            ck[k] += 1
    if o.get("--checkpoints"):
        m, d = (int(x) for x in o["--checkpoints"].split(","))
        ck = {"mandatory": m, "discretionary": d}
    ints = []
    for x in o.get("--interruption", []):
        on, _, how = x.partition(":")
        if on not in ("platform", "usage") or how not in ("same-agent", "redispatched", "pending"):
            die("--interruption platform|usage:same-agent|redispatched|pending", 1)
        ints.append({"on": on, "at": now(), "resumed": how})
    num = lambda k: int(o[k]) if o.get(k) else None
    trace = {
        "trace_id": None, "task_id": tid, "playbook": pk["playbook"] or None,
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
                                    "uncommitted", "mutation", "findings")}
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


def main():
    a = sys.argv[1:]
    if not a or a[0] not in ("prompt", "verify", "register", "trace"):
        die("usage: orch-task.py prompt|verify|register|trace <tsk_id> [...], or register --eval — "
            "see the docstring")
    o, pos, i = {}, [], 1
    multi = ("--interruption", "--human")
    while i < len(a):
        if a[i] in ("--eval",):
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
        return cmd_verify(pos[0], o.get("--test"), o.get("--result"), float(o.get("--timeout", 300)))
    if a[0] == "register":
        return cmd_register(pos[0])
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
                                             [--timeout <seconds>] [--rerun] -- <command>
    python3 .claude/hooks/orch-stage.py status [<stage>...] [--brief]

run     runs <command> in the foreground — launch it as ONE background job —
        logging to .work/_stages/<stage>.log. Refuses (exit 3) while a stage
        named in --after is not done, while the same stage is still running,
        or while another stage holds --lock (one GPU, one holder). A stage
        already done with the same command is skipped, so re-running a whole
        chain after an interruption resumes it; --rerun forces it. A command
        the guard blocks is refused (exit 2). Exits with the command's code,
        124 on --timeout.
status  every stage (or the named ones): done · failed · running ·
        interrupted (the runner died and the job left no exit file) ·
        orphaned (the runner died, the job is still running) · timeout · killed.
        exit 0 all done · 1 any failed/interrupted/timeout/killed · 3 any running
        or never run · 2 could not read. --brief prints one line, for SessionStart.
"""
import fcntl
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


def _guard():
    p = Path(__file__).resolve().with_name("orch-guard.py")
    sys.dont_write_bytecode = True
    spec = importlib.util.spec_from_file_location("orch_guard", p)
    g = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(g)
    return g


try:
    G = _guard()
except Exception as e:
    die(f"cannot load orch-guard.py: {e}")
DIR = G.ROOT / ".work/_stages"
NAME = re.compile(r"[A-Za-z0-9][\w.-]*")
FAILED = ("failed", "interrupted", "timeout", "killed")


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


def run(stage, cmd, after, lock, timeout, rerun):
    if not NAME.fullmatch(stage):
        die(f"stage name {stage!r}: letters, digits, . _ -")
    why = G.bash_trigger(cmd)
    if why:
        die(f"the guard blocks this command ({why}) — a stage is not a way around it")
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
        ds = state(load(dep))
        if ds != "done":
            die(f"{stage} runs after {dep}, which is {ds}. Start {stage} when {dep} is done — "
                "never wait for it inside a job", 3)
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
    st = {"stage": stage, "cmd": cmd, "status": "running", "pid": os.getpid(), "pgid": p.pid,
          "cwd": os.getcwd(), "after": after, "lock": lock, "log": str(log.relative_to(G.ROOT)),
          "started": now(), "ended": None, "exit": None, "elapsed_s": None,
          "attempt": ((st or {}).get("attempt") or 0) + 1}
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
    st.update({"status": end, "exit": rc, "ended": now(), "elapsed_s": round(time.monotonic() - t0)})
    save(st)
    print(f"{stage}: {end} (exit {rc}, {st['elapsed_s']}s) — log {st['log']}")
    return rc


def status(names, brief):
    if not DIR.is_dir():
        return 0 if brief else (print("no stages") or 0)
    names = names or sorted(f.stem for f in DIR.glob("*.json"))
    rows = [(n, load(n)) for n in names]
    states = [(n, st, state(st)) for n, st in rows]
    if brief:
        done = sum(1 for _, _, s in states if s == "done")
        other = [f"{n} {s}" for n, _, s in states if s != "done"]
        if states:
            print(f"## ORCH stages: {done} done" + ("".join(f" · {x}" for x in other)))
        return 0
    for n, st, s in states:
        extra = ""
        if st and s == "running":
            extra = f" · started {st['started']}"
        elif st and st.get("ended"):
            extra = f" · exit {st.get('exit')} · {st.get('elapsed_s')}s"
        print(f"{n:24} {s:12}{extra}" + (f" · log {st['log']}" if st else ""))
    ss = [s for _, _, s in states]
    if any(s in FAILED for s in ss):
        return 1
    return 3 if any(s in ("running", "orphaned", "never run") for s in ss) else 0


def main():
    a = sys.argv[1:]
    if not a or a[0] not in ("run", "status"):
        die("usage: orch-stage.py run <stage> [--after s,..] [--lock name] [--timeout s] [--rerun] -- <cmd>"
            " | status [<stage>...] [--brief]")
    if a[0] == "status":
        return status([x for x in a[1:] if x != "--brief"], "--brief" in a)
    if "--" not in a:
        die("run needs `-- <command>`")
    k = a.index("--")
    opts, cmd = a[1:k], a[k + 1:]
    if not opts or not cmd:
        die("run <stage> [options] -- <command>")
    stage, after, lock, timeout, rerun, i = opts[0], [], None, None, False, 1
    while i < len(opts):
        if opts[i] == "--rerun":
            rerun, i = True, i + 1
        elif opts[i] in ("--after", "--lock", "--timeout") and i + 1 < len(opts):
            v = opts[i + 1]
            if opts[i] == "--after":
                after += [x for x in v.split(",") if x]
            elif opts[i] == "--lock":
                lock = v
            else:
                timeout = float(v)
            i += 2
        else:
            die(f"unknown option {opts[i]}")
    return run(stage, cmd[0] if len(cmd) == 1 else shlex.join(cmd), after, lock, timeout, rerun)


if __name__ == "__main__":
    sys.exit(main())
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
  - {severity: medium, why: dynamic execution,          pattern: "\\beval\\(|(?-i:(?<![.\\w])exec\\()|os\\.system\\(|shell=True"}
  - {severity: medium, why: network client code,        pattern: "socket\\.socket\\(|requests\\.(get|post|put|delete)\\(|urllib\\.request|(?-i:(?<![A-Za-z0-9_])fetch\\()"}
  - {severity: high,   why: destructive SQL,            pattern: "DROP TABLE|TRUNCATE TABLE|DELETE FROM .* WHERE 1"}
  - {severity: high,   why: irreversible operation,     pattern: "git push --force|rm -rf /|chmod 777"}

# exec/fetch are case-exact and need a non-identifier left edge: `.exec(` is
# RegExp matching and `performFetch(` is a method name; neither is the risk.

# Any added line in these files is the `new_dependency` trigger, separately
# from severity. Embedded/C++ build manifests are dependency surfaces too —
# platformio.ini `lib_deps` was a real one that passed this list silently.
dependency_manifests: [requirements.txt, pyproject.toml, package.json, go.mod, Cargo.toml, Gemfile, pom.xml, platformio.ini, library.json, CMakeLists.txt, conanfile.txt, conanfile.py, vcpkg.json, idf_component.yml]

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

End your reply with ONLY this JSON object, no fence and no prose after it:

{"status": "done|blocked|needs_decision|failed|escalate",
 "blocked_on": null,
 "summary": "one or two sentences, file:line for code claims",
 "evidence": [{"cite": "file:line", "reason": "..."}, {"check": "cmd", "exit": 0}],
 "confidence": 0.0,
 "new_facts": [{"type": "fact|failure", "headline": "<=15 words"}],
 "open_questions": []}

`confidence` is your honest posterior that this passes review. Below 0.6 forces
a human checkpoint — that is the system working, so do not inflate it.
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

End your reply with ONLY this JSON object, no fence and no prose after it:

{"status": "done|blocked|needs_decision|failed|escalate",
 "blocked_on": null,
 "summary": "one or two sentences, file:line for code claims",
 "evidence": [{"cite": "file:line", "reason": "..."}, {"check": "cmd", "exit": 0}],
 "confidence": 0.0,
 "new_facts": [{"type": "fact|failure", "headline": "<=15 words"}],
 "open_questions": []}

`confidence` is your honest posterior that this passes review. Below 0.6 forces
a human checkpoint — that is the system working, so do not inflate it.
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

End your reply with ONLY the result JSON (same schema as orch-executor). Put
the repro command in `evidence` as a `check` entry with its real exit code.
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

End with ONLY the result JSON. `status: done` means it passed; anything else
must carry the specific reason in `summary`.
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
quotes its `index: … · blocks/: …` line verbatim; never state a number you did
not read from it (a report once said 95 blocks for a store of 85). That command
is the only thing you run with Bash.

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

Input: `.orch/traces/*.json`, `.orch/checkpoints/*.md`, and
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
and Glob. It protects you only while it runs, so:

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

End your reply with ONLY this JSON object, no fence and no prose after it:

{"status": "done|blocked",
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
4. **Run it** — §2–§6 below. Never execute a packet inline in the main session.
   The bookkeeping is generated, never typed: `orch-task.py prompt` renders the
   dispatch prompt, `verify` re-runs the checks, `register` and `trace` write
   the registry entry and the trace from what `verify` saw.
5. **Stop at any checkpoint** and present the decision. Otherwise do not
   interrupt.
6. **Summarize**, always in this shape:

   > **what changed** (1–2 sentences, `file:line`) · **acceptance** (each check
   > + real exit code, as `orch-task.py verify` printed it) · **checkpoints** hit
   > and how resolved · **memory**
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
#   dispatch it; save the agent's reply to .work/_prompts/<id>.result; stage and commit (§5)
python3 .claude/hooks/orch-task.py verify <id> --test "<test cmd>" --result .work/_prompts/<id>.result
python3 .claude/hooks/orch-task.py register <id>
python3 .claude/hooks/orch-task.py trace <id> --outcome done --error none
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
checkpoint is a file in `.orch/checkpoints/`; resolve it before the task
advances.

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
expects: []                      # sensitive.yaml `why`s that ARE this task's subject,
                                 # e.g. [network client code]; medium only (§5)
vendored: []                     # - {path: "lib/x", tree: "<sha>[:subdir]"} — verbatim
                                 # upstream, verified by tree hash at scan time
retires: []                      # chk_ ids whose behaviour this task deliberately
                                 # removes (unship) — becomes an Orch-Retires trailer
tools: [read, grep, edit, "bash:test"]
forbidden: [write:outside_scope, network, "git:push"]
budget: {steps: 25, wall_s: 600}
effort: medium                   # executor phases only: high -> orch-executor-deep (§4)
notes: ""                        # required when budget deviates, or a floor check is justified
urgency: 0.5
impact: 0.5
```

`expect:` takes one of three forms, and the choice is not cosmetic —
`exit N` accepts anything the command tolerates, so a check whose real contract
is a value must say the value:

| form | means |
|---|---|
| `exit 0` / `exit N` | the command's exit status, and nothing about its output |
| `stdout == <value>` | stdout, stripped, equals this exactly — the form a count check takes |
| `stdout matches /re/` | stdout matches the regex; use only when the exact string genuinely varies |
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
placeholders, judged-only executor acceptance and a test-only scope with no
survival grep, and warns on "every X" outside scope, on acceptance that already
holds at `base`, and on a check that fails at `base` because a file outside the
scope is missing — a wrong command, not a missing feature. What it
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
  construction. Anything you add by hand goes in with the Write or Edit tool,
  never a shell heredoc (every prompt's forbidden list names guarded commands,
  and an unquoted heredoc carrying it is blocked as though it ran them); then
  lint it again:

  ```bash
  python3 .claude/hooks/orch-lint.py packet .orch/queue/<task_id>.yaml --prompt .work/_prompts/<task_id>.md
  ```

  It errors on a command the guard blocks (the guard's own matcher, over
  fenced blocks and inline code), an unfilled `<base>`, a prompt that omits the
  base sha when a check names it, or the worktree's absolute path. It warns on
  a cited file the worktree will not hold as you read it: not in git at
  `base`, or changed on HEAD since `base` forked — a stacked branch's prompt
  once pointed at a doc section that existed only on master.

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

Save the agent's final reply to `.work/_prompts/<task_id>.result` — `verify`
reads its confidence and the block ids its evidence cites from there.

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
2. **Commit** with an `Orch-Task:` trailer. Stage the **files** step 1
   captured, by explicit path — never `git add -A`, and never a directory.
   `--force` only for a path step 1 reported as ignored-but-intended (a
   vendored tree whose own `.gitignore` hides real upstream files); applied to
   a directory it overrides every nested `.gitignore` and commits build output.
   Then look at what is staged before committing:
   ```bash
   git -C .work/<id> add -- <file> <file> ...
   git -C .work/<id> diff --cached --stat        # no build output, no caches
   git -C .work/<id> commit -m "<summary>" -m "Orch-Task: <task_id>"
   ```
   If the packet has `retires:`, add both trailers with `--trailer` (git ≥
   2.32), never by hand. Git parses trailers from the last paragraph only, and
   an indented line there is read as a *continuation* of the line above: the
   retirement vanishes and the `Orch-Task:` value is corrupted, silently.
   ```bash
   git -C .work/<id> commit -m "<summary>" \
     --trailer "Orch-Task: <task_id>" --trailer "Orch-Retires: chk_012, chk_019"
   ```
Steps 3–6 are one command, run after the commit:

```bash
python3 .claude/hooks/orch-task.py verify <task_id> --test "<test command>" --result .work/_prompts/<task_id>.result
```

It prints each check with its real exit code, then a `verdict` — `done`,
`failed`, `needs_decision` (write the checkpoint now) or `incomplete` (fix the
bookkeeping it names and verify again) — and writes the record that `register`
and `trace` read. A failing check that hit a not-found error at the task's
HEAD is flagged as a probable wrong check: say so in **attribution**. What it
cannot do stays yours — the mutation check below, the post-check items it
leaves to judgment, and what any finding means.

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
   evidence, verified against the given set (`verify --result` does). Discard
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

   It gates on content patterns above `low` and on dependency manifests. Path
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
  was genuinely right, and the key is never blank. It is the machine-readable
  half of the summary's `attribution`, it is never scored, and its schema is in
  `ORCH.md` §5.4 (`orch-loop`, *Traces*). The tool writes every key every
  time, copies `verification` from the verify record, and refuses a `done`
  that no clean verify backs — hand-typed traces drifted, and a trace missing
  a key is the one failure this system cannot detect later
- prune the worktree (`git worktree remove`); the work lives on the branch
- **merging is the user's**, unless they granted it — and even a grant can be
  refused by the harness: auto mode's permission classifier has denied
  `git merge` to an orchestrator the user had told to merge. So never stop a
  run to wait for a merge (stack dependent work instead, §3). End the run with
  **one line** covering every finished branch, parents first, each checked to
  merge cleanly (`git merge-tree --write-tree HEAD orch/<task_id>` exits 0;
  git ≥ 2.38, and it touches no branch or file):

  > `! git merge --no-edit orch/tsk_a && git merge --no-edit orch/tsk_b`

  Holding a grant, try the merge once; if it is refused, that line is the
  fallback. Never retry a refused merge, and never route around the refusal

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

`.orch/checkpoints/ckp_<id>.md`:

```markdown
# ckp_<id> — <trigger name>
class: mandatory | discretionary
task: tsk_...
raised: <ISO>
## Trigger
<which rule fired, with the evidence that fired it>
## What the agent did
<summary + diffstat>
## To approve
<for a blocked command the user asked for: the exact command, as one `! <cmd>`
 line for them to run. Their running it IS the approval. No path turns a chat
 "yes" into the model running a guarded command, and that is deliberate: an
 approval the model can act on is an approval the model can imagine.>
## Decision
- approve → the trigger is waived ONCE; task requeues from its saved phase
- reject  → task fails; nothing registers; worktree kept
- **false positive** → the trigger was wrong, not the work. Record the waiver as
  above AND append a `proven` entry to `.orch/upstream.md` (`ORCH.md` §5.5): you
  have the reproduction already — it is the command that was blocked and the
  exit code it got. A guard that blocks legitimate work, left unreported,
  teaches the next orchestrator to route around guards
```

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
   what threshold — and commit it before the run.
2. **The harness is tracked code.** Scorers, checkers and run scripts live in
   the repo and get there through a task, with a test, like any other code. A
   scratchpad script that decides an outcome is code that escaped review.
3. **Quick analysis is measurement too.** A breakdown, a sample, a comparison
   run to answer a question: call the scorer's own entry points, never
   re-implement a piece of its logic inline — a partial copy is a second scorer
   nobody tested. A number that comes from an inline script anyway is a draft:
   write "inline, unverified" beside it, keep it out of docs, decide nothing on
   it. If the question recurs, the analysis belongs in the scorer: a task.
4. **Freeze the raw output** — per-item results, committed. If the file is too
   large to commit, keep it at a stable path; `register` records its sha256.
5. **Register the conclusion** — the scoring command over the frozen output,
   `expect` the exact number the conclusion cites:

   ```bash
   python3 .claude/hooks/orch-task.py register --eval --msr <msr_id> \
     --cmd "python3 scripts/score.py data/results/p14.jsonl --net" --expect "stdout == +3" \
     --artifact data/results/p14.jsonl --noise "7/40 answers differ"
   ```

   It refuses a scorer that is untracked or has uncommitted changes, runs the
   command once and refuses if it does not hold, and writes a `kind: eval`
   entry (`orch-rework` §3). A sweep replays the scoring, never the run.
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
again: void them, or say so in the results doc. Mix in controls whose answers
are known, unmarked, so the judge's reliability is measured (19 of 20 in the
session that asked for this), not assumed.
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
<= 60-word summary (L1).

Full body (L2) below — only if the summary genuinely cannot carry it.
```

Non-negotiable:
- **No source, no block.**
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
   "scope_violations": [], "uncommitted": [], "scan_exit": 0,
   "mutation": "deleted the flush at muxer.py:412 -> test failed", "findings": []}}
```

`verification` is what `orch-task.py verify` observed, copied from its record —
not the orchestrator's account of it. It is `null` only for a phase with no
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
commit) reports `artifact missing`, not a break.

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
    registered: 2026-08-29
    status: active
  - id: chk_031
    kind: eval                    # code (the default) | eval — orch-task §8
    task: msr_p12-r8c             # a measurement's id: it runs outside a packet
    commit: 8d09db2
    cmd: "python3 scripts/score.py data/eval/results/p12-r8c.jsonl --net"
    expect: "stdout == +3"
    artifact: {path: data/eval/results/p12-r8c.jsonl, sha256: "3f1a…"}
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
# 1. all 26 files landed
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
#    ...now re-run the §0 installer block. It must print `update CLAUDE.md (v0 -> v4)`.
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
printf 'Work in %s from %s. Read docs/plan.md and docs/gone.md.\n' "$W" "$b18" > p18.md
lq stale-doc "--prompt p18.md" 0
echo "changed since base, not at base: $(grep -c 'changed since base forked' lint.out) $(grep -c 'not in git at base' lint.out) (want 1 1)"
rm -f p18.md lint.out .orch/queue/tsk_q.yaml

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

Expected: the usage line and `exit=0` in step 1b (the scan's own invocation
carries no policy literal, so the guard lets it through), 4 blocks in step 2,
allow in step 3, `0 2 0 2 2` in step 4, step 5 printing `loosening ignored`,
`0 2 0 2` in steps 9–10, and every line of steps 11–23 printing its `want`
(146 `want` lines in all, the last `leftovers: 0 0`).
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
   - a trace lands in `.orch/traces/` with every key, written by `orch-task.py
     trace`, and `orch-lint.py memory --task` exits 0 after the archivist
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
