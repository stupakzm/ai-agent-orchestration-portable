# ORCH.md — portable agent-orchestration layer for Claude Code

**One file. Drop it in any repo. It installs itself.**

This is a port of the `orch` system (SQLite + Python CLI + headless `claude -p`
subagents) onto Claude Code's own primitives: subagents, skills, hooks, and
plain files. Same architecture, no Python package, no database, no daemon.

## 0. Install

Deterministic — copy `ORCH.md` into the repo root and run this. It extracts the
21 `FILE:` blocks in §5 verbatim; no model is in the loop, so nothing can be
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
assert len(blocks) == 21, f"expected 21 FILE blocks, found {len(blocks)}"

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
    for h in ("orch-guard.py", "orch-scan.py"):
        (pathlib.Path(".claude/hooks") / h).chmod(0o755)
    pathlib.Path(".work").mkdir(exist_ok=True)
    # git does not track empty dirs; .gitkeep keeps the layout intact on clone
    for d in ("memory/blocks", "cards", "queue", "registry", "checkpoints", "traces"):
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
byte-exact where the model is not, and reading this file costs **~24k tokens of
context** where the script costs none.

**Then restart Claude Code.** Hooks, subagents, and skills are registered at
session start; until you restart, the guard is not enforcing anything.

Verify (see §7 for the full test procedure):

```bash
ls .claude/agents/orch-* .claude/skills/orch-*/SKILL.md && \
echo '{"tool_name":"Bash","tool_input":{"command":"git push --force"}}' \
  | python3 .claude/hooks/orch-guard.py; echo "exit=$? (want 2)"
```

To uninstall: `rm -rf .claude/agents/orch-* .claude/skills/orch-* .claude/hooks/orch-*.py .orch`, delete everything from `<!-- ORCH` through `<!-- /ORCH -->` in `CLAUDE.md`, and remove the two hook entries from `.claude/settings.json`.

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
| what | §2 core (37 lines) + every skill's and agent's `description` line + the SessionStart memory headlines | `.claude/skills/orch-*` bodies | `.claude/agents/orch-*` bodies (subagent-only), block bodies, traces |
| cost | ~577 + ~628 = **~1.2k tokens**, plus headlines | 1.0k–2.7k when invoked | 0 |

Two things that table gets asked about:

- **The `description:` lines are not free.** Every skill and agent description
  sits in the always-on listing whether or not it is ever invoked — 628 tokens
  across ten of them, more than the core itself. Keep them one sentence longer
  than feels necessary (they are what makes routing fire) and no longer.
- **It is a cached prefix read, not fresh input.** `CLAUDE.md` renders ahead of
  the conversation, so in a warm session those ~1.2k tokens are served from
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
it took the core from 94 lines to 37 with nothing lost.

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
  for everything else.
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
<!-- ORCH v2 — orchestration layer. Source of truth: ORCH.md. -->
## Orchestration (ORCH)

This repo runs orchestrated work: tasks are contracts, subagents execute them in
sandboxed worktrees, and results are checked by code before they count.

**You are the orchestrator.** The user states what they want in plain English.
You classify it and route it — they should never have to name a skill, write a
packet, or ask for orchestration. Route every request that changes code:

| the request is | route | why |
|---|---|---|
| a question, or a read-only look at the code | answer inline | nothing to guard |
| a one-line, single-file, obvious edit (typo, rename, bump a constant) | do it inline, say you did it inline | a worktree costs more than the change |
| **anything else that changes code** — a feature, a bug, a refactor, multi-file work, anything needing a test | **invoke the `orch-task` skill and follow it** | this is what the system is for |
| you cannot tell which | ask, in one line | never improvise a chain silently |

`orch-task` carries the whole procedure: retrieval, drafting and linting the
packet, the single confirmation, dispatch, code-verified acceptance, the
registry, and the required summary shape. **Load it — never hand-roll the
procedure from memory, and never execute a packet inline in this session.** A
task runs in a worktree under a subagent, or it does not run.

**Guards are code.** `.claude/hooks/orch-guard.py` blocks irreversible
operations, secret paths, edits to `.orch/config/`, and writes outside a task's
declared scope — at the tool boundary, during your own manual work too. If it
blocks you, do not route around it: surface it to the user. If the block was
*wrong*, that is a defect in ORCH itself and it belongs in `.orch/upstream.md`
(`ORCH.md` §5.5) — noticing an issue and not writing it down is the failure that
costs the most, because the next repo starts from the same `ORCH.md`.

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
    orch-executor.md          # writes code, sonnet
    orch-debugger.md          # reproduces + isolates, sonnet
    orch-reviewer.md          # verifies against acceptance, sonnet
    orch-librarian.md         # selects memory blocks, haiku
    orch-archivist.md         # the only writer of memory, haiku
    orch-auditor.md           # audits the ORCHESTRATOR's own packets, sonnet
  skills/
    orch-task/SKILL.md        # run one task end-to-end
    orch-memory/SKILL.md      # block CRUD, retrieval, compaction
    orch-loop/SKILL.md        # selection, loop modes, checkpoints
    orch-rework/SKILL.md      # registry replay + break attribution
  hooks/orch-guard.py         # the risk triggers, as code
  hooks/orch-scan.py          # sensitive-content post-check; reads policy at runtime
  settings.json               # hook wiring (merge, don't clobber)
.orch/
  config/settings.yaml        # thresholds — tightening only
  config/sensitive.yaml       # security-gate policy — never self-modified
  playbooks/code.bugfix.yaml  # the shipped playbook
  upstream.md                 # findings about ORCH itself — carried back by hand
  memory/INDEX.md             # L0 headline index
  memory/blocks/blk_*.md      # L1 summary + L2 body
  cards/<project>.md          # project cards
  queue/tsk_*.yaml            # task packets
  registry/<project>.yaml     # acceptance registry
  checkpoints/ckp_*.md        # open decisions
  traces/trc_*.json           # scored history
  state.json                  # loop state, counters
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
orchestrator; a `done` that fails its check becomes `failed`. `context_usage`
is **measured** — which block ids actually appear in the evidence — never
self-reported, because it feeds retrieval tuning and a self-report is gameable.

**The packet is the likelier defect, and nothing else looks at it.** Guards
constrain the subagent's writes, acceptance verifies its output, checkpoints
fire on its behaviour and spend — every mechanism assumes *the agent* may be
wrong. A wrong packet fails silently instead: each check passes, and the defect
is registered as evidence that things are fine. Verification cannot catch a
defect encoded in the definition of success. Three mechanisms point the other
way — mandatory `attribution` in every task summary (§2), a nullable-but-present
`orchestrator_error` on every trace, and the orchestrator sweep in
`/orch-rework` §5, which is a separate agent because self-assessment is the weak
link this is meant to remove.

**Memory has depth.** L0 headline (index) → L1 summary → L2 body → L3 the
original artifact (subagent only, never the orchestrator). Auto-attach rule:
for every selected block, also pull the head of its `supersedes` chain — the
single highest-value rule in the retrieval path, because it prevents acting on
a fact that has since been replaced.

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

ROOT = Path(os.environ.get("CLAUDE_PROJECT_DIR", ".")).resolve()

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
    (`$(...)`, backticks).
    """
    if depth > 3:
        return
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


def active_scope():
    """scope.paths of the task this worktree is running.

    None  — no task active; the guard enforces only the global triggers.
    []    — a task IS active but its scope could not be read. That blocks
            writes. An unreadable contract is not an unlimited one, and this
            is the failure mode that silently disarms scope enforcement.

    Both YAML styles are accepted, because both get written in practice and a
    guard that only understands one of them is off half the time:

        scope: {paths: ["src/a.py"], network: false}   # flow — the §5.4 skeleton
        scope:                                          # block — §7's test
          paths:
            - "src/a.py"
    """
    tid = os.environ.get("ORCH_TASK")
    if not tid:
        return None
    p = ROOT / f".orch/queue/{tid}.yaml"
    if not p.exists():
        return []                          # a task was declared, no packet exists
    txt = p.read_text()
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
        cmd = ti.get("command", "")
        for pat, why in CONTENT:
            if re.search(pat, cmd, re.I):
                block(f"{why} — irreversible or externally visible")
        for seg in commands(cmd):
            for pat, why in COMMANDS:
                if re.search(pat, seg, re.I):
                    block(f"{why} — irreversible or externally visible")

    if tool in ("Write", "Edit", "NotebookEdit"):
        fp = ti.get("file_path", "")
        rel = os.path.relpath(fp, ROOT) if os.path.isabs(fp) else fp
        if SECRET_PATH.search(rel):
            block(f"write to a secrets path ({rel})")
        if rel.startswith(".orch/config/"):
            block("edit of .orch/config — the trigger list and security policy "
                  "are human-owned and never self-modified")
        scope = active_scope()
        if scope is not None:
            if not scope:
                block(f"ORCH_TASK={os.environ.get('ORCH_TASK')} is set, but no "
                      "readable scope.paths was found in its packet. Fix the "
                      "packet — a scope the guard cannot read is not a scope.")
            from fnmatch import fnmatch
            if not any(fnmatch(rel, g) or rel.startswith(g.rstrip("/*") + "/")
                       for g in scope):
                block(f"write outside scope.paths ({rel}); declared: {', '.join(scope)}")
    sys.exit(0)


if __name__ == "__main__":
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
      }
    ]
  }
}
```
````

If `.claude/settings.json` already exists, merge the `hooks` keys — do not
overwrite the file.

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

    python3 .claude/hooks/orch-scan.py <worktree> [<base-ref>]   # base: HEAD^

exit 0  clean, or low severity only
exit 1  findings above `low` -> mandatory checkpoint before anything registers
exit 2  usage or git error. It never reports clean when it could not look.
"""
import os
import re
import subprocess
import sys
from fnmatch import fnmatch
from pathlib import Path

RANK = {"low": 0, "medium": 1, "high": 2}
ROOT = Path(os.environ.get("CLAUDE_PROJECT_DIR", ".")).resolve()
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
    pats, manifests, globs, section = [], [], [], None
    for line in p.read_text().splitlines():
        if re.match(r"^\w[\w_]*\s*:", line):
            section = line.split(":")[0]
        m = ENTRY.match(line)
        if m:
            raw = m.group(3).replace("\\\\", "\\")     # YAML double-quoted
            try:
                pats.append((m.group(1), m.group(2), re.compile(raw, re.I)))
            except re.error as e:
                die(f"unparseable pattern {raw!r}: {e}")
            continue
        m = re.match(r"\s*dependency_manifests\s*:\s*\[(.+)\]", line)
        if m:
            manifests = [x.strip().strip("\"'") for x in m.group(1).split(",")]
        elif section == "path_globs":
            m = re.match(r'\s*-\s*"(.+)"\s*$', line)
            if m:
                globs.append(m.group(1))
    if not pats:
        die("policy parsed to zero patterns — refusing to pass")
    return pats, manifests, globs


def added(worktree, base):
    """(path, text) for every ADDED line. Added lines only: an existing
    credential the task merely moved past is not this task's finding."""
    cp = subprocess.run(["git", "-C", worktree, "diff", "--unified=0", base, "HEAD"],
                        capture_output=True, text=True)
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
    if len(sys.argv) < 2:
        die("usage: orch-scan.py <worktree> [<base-ref>]")
    wt, base = sys.argv[1], sys.argv[2] if len(sys.argv) > 2 else "HEAD^"
    pats, manifests, globs = policy()
    lines = added(wt, base)
    found = []
    for path, text in lines:
        for sev, why, rx in pats:
            if rx.search(text):
                found.append((sev, f"{path}: {why} — {text.strip()[:110]}"))
    paths = sorted({p for p, _ in lines if p})
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
  - {severity: medium, why: dynamic execution,          pattern: "\\beval\\(|\\bexec\\(|os\\.system\\(|shell=True"}
  - {severity: medium, why: network client code,        pattern: "socket\\.socket\\(|requests\\.(get|post|put|delete)\\(|urllib\\.request|fetch\\("}
  - {severity: high,   why: destructive SQL,            pattern: "DROP TABLE|TRUNCATE TABLE|DELETE FROM .* WHERE 1"}
  - {severity: high,   why: irreversible operation,     pattern: "git push --force|rm -rf /|chmod 777"}

# Any added line in these files is the `new_dependency` trigger, separately
# from severity.
dependency_manifests: [requirements.txt, pyproject.toml, package.json, go.mod, Cargo.toml, Gemfile, pom.xml]
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

**Minimality.** Apply the smallest change that satisfies the goal. No drive-by
refactors, no reformatting, no "while I was here". Every changed line you
cannot justify against the goal is scope creep and will be flagged.

**Evidence.** Cite `file:line` for every claim about code. Report only commands
you actually ran and their real exit codes. Never weaken, skip, or delete a
test to make acceptance pass — if a test blocks you, that is a finding.

**Memory.** If a context block informed your work, cite its `blk_` id in an
evidence entry. If you had to read a file you were not pointed at, say so in
`open_questions` — that is a retrieval miss and it is useful.

End your reply with ONLY this JSON object, no fence and no prose after it:

{"status": "done|blocked|needs_decision|failed|escalate",
 "summary": "one or two sentences, file:line for code claims",
 "evidence": [{"cite": "file:line", "reason": "..."}, {"check": "cmd", "exit": 0}],
 "confidence": 0.0,
 "new_facts": [{"type": "fact|failure", "headline": "<=15 words"}],
 "open_questions": []}

`confidence` is your honest posterior that this passes review. Below 0.6 forces
a human checkpoint — that is the system working, so do not inflate it.
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
4. **Never select a superseded block** unless the query is explicitly historical
   ("why did we used to…"). If a block's header says it was superseded, return
   the successor instead.
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
description: The only writer of project memory. Turns a completed task's proposed facts into new/updated/superseded blocks, and runs compaction. Use after a task finishes, and when the block count exceeds its cap.
tools: Read, Grep, Glob, Edit, Write
model: haiku
---

You are the ONLY thing that writes memory. Result packets *propose*; you decide.

For each proposed fact:

1. Grep `.orch/memory/INDEX.md` for near-duplicates.
2. Decide one of: **new** · **update** (same claim, better detail) ·
   **supersede** (the old block is now wrong or outdated) · **discard**
   (duplicate, or too specific to ever match again).
3. **Contradiction rule — immutable.** If it contradicts an existing block with
   `confidence >= 0.8`, you may NOT overwrite. Write a `decision` block holding
   both claims and their sources, and flag it for escalation. Silently resolving
   a contradiction is how a wrong fact propagates into every future task.
4. **No source, no block.** Every block needs `source:` naming the task id and
   the artifact (`file:line`, a command, or a URL). A fact with no provenance
   cannot be checked later, so it is not written. No exceptions.
5. Headline ≤ 15 words, summary ≤ 60 words. If you cannot say it that short,
   the block is really two blocks.
6. **`process-failure` blocks are accepted from the orchestrator, not only from
   agents.** They record how a task was *specified* wrongly — a weak acceptance
   check, a plan whose resource cost was never reasoned about, a reference file
   that was itself broken. Same rules as any block: no source, no block, and the
   source is a task id plus the trace or checkpoint that shows it. Write them
   `decay_class: permanent`; the constraint outlives the code.

Write the block file, then append its line to `INDEX.md`. Newest first.

**Compaction** (when a project passes `block_cap`, or on request):
- merge near-duplicate clusters, unioning their sources
- a fact independently confirmed ≥3× is promoted to `convention`
- `decay_class: fast` unread 30d → confidence × 0.8; below 0.3 → move to
  `.orch/memory/archive/`
- `cite_count / read_count < 0.15` after ≥10 reads → the headline is
  misleadingly broad. Rewrite it or archive the block.
- rebuild `INDEX.md` from the surviving block files

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
   `acceptance`, tight `scope.paths`, `context_refs`, `budget`. Infer these; do
   not interrogate the user for them. Lint your own draft against §1 before
   anyone sees it.
3. **Show it in ~8 lines and take one confirmation.** Scope and acceptance are
   where a task goes wrong, and a wrong one wastes the whole run. Include the
   mode line: `L2 — reversible (worktree), 3 files, 2 prior runs here`. After
   that single confirm, run the whole task without further questions.
4. **Run it** — §2–§6 below. Never execute a packet inline in the main session.
5. **Stop at any checkpoint** and present the decision. Otherwise do not
   interrupt.
6. **Summarize**, always in this shape:

   > **what changed** (1–2 sentences, `file:line`) · **acceptance** (each check
   > + real exit code) · **checkpoints** hit and how resolved · **memory**
   > blocks cited and written · **attribution** · **branch** to merge, or where
   > the worktree is if it did not finish.

   Report the verified outcome, never the agent's claim. A `done` whose check
   failed is `failed`.

Multi-phase work (feature, refactor) runs its playbook's phases in order, each
its own packet and subagent. Report once at the end, not per phase.

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

If given a `tsk_` id, read `.orch/queue/<id>.yaml`. Otherwise create one from
the playbook phase and **have the user edit it before queueing**. The four
fields that decide whether this works are `goal`, `acceptance`, `scope.paths`,
and `context_refs`.

```yaml
task_id: tsk_<ulid>
project: <name>
playbook: code.bugfix@v1
phase: fix
role: executor
goal: "one sentence, testable"
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

Rules that are not negotiable:

- **Executable beats judged.** If the task admits an executable check, it must
  have one. "Judged" acceptance alone is how a task avoids being scored.
- **Count checks assert an exact number, never a floor.** `>=1`, `>0`, `-gt 0`
  and "at least" are rejected in an `expect:` unless the packet states in
  `notes` why the exact count is genuinely unknowable. A floor proves the
  feature *exists*; correctness needs it to be *exactly right*, and those are
  different assertions. If you cannot predict the exact count, you do not yet
  understand the behaviour well enough to accept it.
- **Negative space needs its own check.** "Writes nothing to `output/`", "does
  not install anything", "leaves `config.yaml` untouched" are assertions and
  need commands, not prose. Absence is never verified by a check that only
  looks at what is present.
- **A packet whose scope is a test file needs a second check** that greps for
  the assertions that must survive. Otherwise "make the test pass" is gameable
  by deleting the test.
- **Scope tight.** Every path the agent may write, and no others. The guard
  hook blocks writes outside it.
- **Budget realistic, and per-packet when the work is heavy.** A default budget
  that trips the spend trigger on every real task teaches everyone to ignore the
  trigger. If any acceptance check invokes a model, a build, a GPU job, or a
  network fetch, the playbook default is wrong for this packet: set `budget`
  explicitly and justify it in `notes`, or mark the phase `heavy: true` and let
  the playbook's `heavy_budget_multiplier` apply. Never raise `spend_fraction`
  to make a real overrun stop reporting.

The first three are a **lint you run on your own draft before showing it to the
user.** They exist because a bad packet fails silently: every check passes and
the defect ships as registered evidence. Verification cannot catch a defect that
is encoded in the definition of success.

## 2. Context

Dispatch `orch-librarian` with the goal. Put the returned ids in
`context_refs`, and their headline+summary in the subagent's prompt. Ids the
librarian auto-attached go in too.

## 3. Sandbox

```bash
git worktree add -b orch/<task_id> .work/<task_id> HEAD
```

If the phase is `heavy: true`, multiply the playbook `default_budget` by its
`heavy_budget_multiplier` **now**, before dispatch — a budget known in advance
to be too small produces a spend checkpoint that carries no information, and
that is how the trigger gets trained out of everyone.

Pre-check before dispatch — any hit is a **mandatory checkpoint, before any
agent runs and before any cost**:
- a `scope.paths` entry matches a `path_globs` pattern in `sensitive.yaml`
- `attempts >= max_attempts` for this task
- the task is `deferred_until` a future time

## 4. Dispatch

Spawn the subagent named by the phase's `role` (`orch-executor`,
`orch-debugger`, `orch-reviewer`) with:

- the packet's goal, acceptance, scope, forbidden
- the retrieved blocks as `[blk_id] (type) headline / summary / body`
- `Work only inside .work/<task_id>. It is a throwaway worktree.`

Set `ORCH_TASK=<task_id>` in the environment for bash calls so the guard hook
can enforce scope.

## 5. Verify — this is the part that matters

The subagent's claimed status is an input, not the answer.

1. **Capture agent writes first.** `git -C .work/<id> status --porcelain`,
   BEFORE running acceptance checks — otherwise the checks' own byproducts
   (`__pycache__`, `.pytest_cache`, build output, lockfiles) look like agent
   writes and false-positive as scope creep. Filter generated artifacts out.
2. **Commit** with an `Orch-Task:` trailer. Stage explicit paths with `--force`,
   never `git add -A`:
   ```bash
   git -C .work/<id> commit -m "<summary>" -m "Orch-Task: <task_id>"
   ```
3. **Run every executable check** inside the worktree. A `done` whose check
   fails is `failed`. No discussion.
4. **Measure `context_usage`** — which `blk_` ids actually appear in the
   evidence, verified against the given set. Discard anything the model wrote
   in those fields; a self-reported retrieval score is gameable and it feeds
   tuning.
5. **Post-check** — any hit downgrades the result to `needs_decision` and writes
   a checkpoint **before** anything is registered:
   - writes outside `scope.paths`
   - `status: done` with `confidence < 0.6`
   - a dependency manifest in the diff (`new_dependency`)
   - added diff lines matching `sensitive.yaml` above `low`
   - steps or wall time over `spend_fraction` of budget
   - an irreversible or externally-visible operation

   The last three of those are yours to judge; the sensitive-content, manifest
   and path-glob checks are one command:

   ```bash
   python3 .claude/hooks/orch-scan.py .work/<id> <sha-at-worktree-creation>
   ```

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
- append each passing executable check to `.orch/registry/<project>.yaml` with
  its commit sha
- **if a check here replaces an existing one because the old one was wrong**,
  mark the old entry `status: superseded`, `superseded_by: <new id>`, and give
  the reason. Silently outnumbering a weak check with better ones leaves it in
  the registry as passing evidence at the commit where the behaviour was wrong
  (`/orch-rework` §3)
- dispatch `orch-archivist` with `new_facts`
- write `.orch/traces/trc_<id>.json`, **including `orchestrator_error`** — the
  key is always present, `null` only if the packet was genuinely right. It is
  the machine-readable half of the summary's `attribution`, it is never scored,
  and its schema is in `ORCH.md` §5.4 (`orch-loop`, *Traces*). A trace written
  without the key is the one failure this system cannot detect later
- prune the worktree (`git worktree remove`); the work lives on the branch
- tell the user: `git merge orch/<task_id>` — **you do not merge**

On anything else, keep the worktree for inspection and say where it is.

**On every outcome, `done` included:** if the task's `attribution` named the
*system* as an origin — the guard blocked something legitimate, a rule here
could not be followed as written, a documented shape did not work — append an
entry to `.orch/upstream.md` (`ORCH.md` §5.5). `observed` if you saw it once,
`proven` if you can reproduce it, and a `proven` entry owes a proposed fix and
the test that would have caught it. This is the only path by which a defect in
the orchestration layer leaves the repo it was found in.

A task interrupted by a usage limit ends **blocked**, not failed. Everything is
preserved on the branch and in the worktree; rerunning resumes in place.

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

Two sources, not one. Agents propose blocks about **the project**. The
orchestrator proposes `process-failure` blocks about **how the work was
specified** — from a task's `attribution` field, or from an orchestrator sweep
(`/orch-rework` §5). A lesson like "a packet that plans an external run must
state the resource ceiling it assumes" has nowhere else to live, and it is
retrieved on the next task that plans one.

```markdown
FILE: .orch/memory/blocks/blk_<id>.md
---
id: blk_<id>
project: <name>
type: fact | decision | failure | convention | procedure | api-shape | entity | process-failure
headline: "<= 15 words, the claim itself"
links: {parent: blk_..., supersedes: [blk_...], related: [blk_...]}
source: {task: tsk_..., artifact: "src/muxer.py:412"}
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

When a project exceeds `block_cap` (default 300), or the index reads noisy,
dispatch `orch-archivist` to compact. It merges duplicates, promotes 3×-confirmed
facts to `convention`, decays unread fast blocks, archives blocks with high
reads and low cites, and rebuilds the index.

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
(did its dependency finish? did its checkpoint resolve?) — leaving `blocked`
out silently defeats the whole resume design. `needs_decision` is NOT eligible;
it is waiting on a human.

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

Every terminal outcome writes `.orch/traces/trc_<id>.json`:

```json
{"trace_id": "trc_...", "task_id": "tsk_...", "playbook": "code.bugfix@v1",
 "loop_mode": {"proposed": "L2", "used": "L2", "overridden": false},
 "outcome": "done", "attempts": 1,
 "checkpoints": {"mandatory": 1, "discretionary": 0},
 "context_usage": {"refs_given": [], "refs_cited": [], "l3_escapes": 0},
 "cost": {"steps": 11, "wall_s": 143},
 "rework": {"links": [], "penalty": 0.0},
 "orchestrator_error": null,
 "human_actions": [{"cmd": "approve", "at": "...", "modified": false}]}
```

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
forever.

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
# 1. all 21 files landed
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
#    ...now re-run the §0 installer block. It must print `update CLAUDE.md (v0 -> v2)`.
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

Expected: the usage line and `exit=0` in step 1b (the scan's own invocation
carries no policy literal, so the guard lets it through), 4 blocks in step 2,
allow in step 3, `0 2 0 2 2` in step 4, and step 5 printing `loosening ignored`.

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
| subagents registered | ask "what agents are available?" | six `orch-*` agents |
| CLAUDE.md core loaded | ask "what are the ORCH risk triggers?" | answers from the §2 stanza without reading a file |
| **guard actually blocks a live call** | ask it to `git push --force` on a throwaway branch | the tool call is refused with the ORCH checkpoint text — this is the one that proves oversight is code, not prompting |

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
   - a passing check lands in `.orch/registry/<project>.yaml`
   - a trace lands in `.orch/traces/`
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
