<img src="../assets/icons/playbook.svg" width="40" height="40" alt="">

[**← playbook index**](README.md) &nbsp;·&nbsp; chapter 06 of 08

# 06 — The Memory System

Sub-agents are stateless: every spawn is a blank slate that knows only its
template and its prompt. Without a memory mechanism, your QA agent re-discovers
the same fragile test every week and re-flags the same intentional exception in
every review. The memory system is the fix — a small, versioned notebook per
agent that turns individual sessions into institutional knowledge.

## The loop

Every agent template ends with the same standing instruction:

1. **At session start:** read your `MEMORY.md`. It contains the recurring
   issues to check, the known weak spots, and the exceptions already ruled on.
2. **Work.**
3. **Before finishing:** if you discovered something a future session would
   otherwise re-learn the hard way — a recurring pattern, a fragile test, a
   disproven suspicion — append it to the relevant section.

That is the whole protocol. Its value compounds: a six-month-old QA memory
reads like a map of exactly where this codebase bites.

## Directory convention

```text
.claude/agent-memory/
├── qa-engineer/
│   ├── MEMORY.md              ← the index — loaded every session
│   └── flaky-tests.md         ← topic file — loaded on demand
├── integration-agent/
│   ├── MEMORY.md
│   ├── events-and-queues.md
│   └── module-wiring-notes.md
├── backend-dev/
│   └── MEMORY.md
└── ...one directory per agent
```

One directory per agent (`.claude/agent-memory/<agent>/`), one `MEMORY.md`
index inside it. Agents read only their own directory — cross-agent knowledge
that everyone needs belongs in the constitution, not in memory.

## The index cap and the overflow pattern

**The index is capped at ~200 lines, and the cap is enforced: an index over
the cap is a maintenance blocker, not a style suggestion.** The index is
loaded into *every* session of that agent — every line is a token tax on all
future work, and an unbounded index eventually costs more context than it
saves. We know because we let one grow to 7× its self-declared cap before
enforcing this ([chapter 08, note 8](08-field-notes.md)).

The overflow pattern keeps depth without bloat:

- The **index** holds one line per entry — the claim and a pointer.
- **Topic files** in the same directory hold the detail (evidence, history,
  examples). The index lists them under a `## Topic Files` heading with a
  one-line description each; the agent reads a topic file only when its
  subject is relevant to the current task.
- When an index section outgrows its budget, move the detail to a topic file
  and leave the one-line summary behind.

## The four canonical sections

| Section | What goes in it |
| :--- | :--- |
| **Recurring Issues to Always Check** | Failure modes seen in ≥2 changes, phrased as checks ("verify controllers use the request-validation class, not inline validation"). The agent runs these every session. |
| **Known Weak Spots** | Areas of the codebase that historically need extra scrutiny — the module with the tangled event flow, the service where money math lives. |
| **Approved Exceptions** | Things that *look* wrong but were ruled intentional, with who ruled and when. Prevents re-flagging the same accepted pattern forever. |
| **Disproven Findings** | Suspicions that were investigated and found false, with the evidence. |

**Disproven findings deserve emphasis** because they are the section people
skip. A finding that took an hour to disprove will be re-raised by a fresh
session next month — same reasoning, same hour, same conclusion — unless the
disproof is written down. Recording "we suspected the lock was racy; traced
it; the enclosing transaction serializes it — do not re-flag" converts wasted
work into a permanent filter. It is also the honest counterpart to the pattern
library: the library records what *is* broken; disproven findings record what
*isn't*, so paranoia stays calibrated.

## What NOT to record

- **One-off issues** specific to a single change. They are resolved; memory is
  for what recurs.
- **Anything already in the constitution.** Memory supplements `CLAUDE.md`; a
  duplicated rule will drift from its source and eventually contradict it.
- **Secrets. Ever. Hard rule.** No credentials, tokens, API keys, connection
  strings, or real personal data — memory files are committed to the repo and
  travel with every clone. Enforce this with CI-style scanning and review
  rather than trusting prose — pasted diagnostic snippets are exactly how a
  credential slips into a memory file unnoticed, and nothing flags it until a
  human happens to read that file. Treat memory files as public the moment
  they are written, and treat any credential that appears in one as
  compromised.

## Version-control policy

Memory is **committed with the work it came from** — it is part of the
deliverable, not a local scratchpad. An uncommitted memory update is knowledge
that exists on one machine until it doesn't. The orchestrator includes
`.claude/agent-memory/` changes at the pipeline's commit checkpoints; for
standalone memory updates, use the convention:

```text
docs(qa-memory): record flaky scheduler test and disproven lock-race finding
```

(`docs(<agent>-memory): ...` — greppable, and keeps memory history separable
from code history.)

## The consolidation chore

Append-only notebooks rot: duplicates accumulate, entries go stale, the index
creeps toward its cap. Schedule a periodic **memory-consolidation chore** — a
cheap-model agent (`{{MODEL_FAST}}`; this is checklist work, not judgment
work) that, per agent directory:

1. Merges duplicate or overlapping entries.
2. Prunes entries invalidated by code that has since changed (verify before
   deleting).
3. Moves oversized detail into topic files and re-checks the index against the
   ~200-line cap.
4. Commits the result with the memory commit convention.

Monthly is a good default; after any heavy multi-PR stretch, sooner.

## Starting points

- [templates/MEMORY.md.template](../templates/MEMORY.md.template) — a blank
  index with the four canonical sections and the cap noted in the header.
- [examples/laravel-react/memory-example.md](../examples/laravel-react/memory-example.md)
  — what a lived-in index looks like after a few months of real reviews.

Seed each agent's memory with an empty template on day one. An agent that
finds a memory file — even an empty one — learns the habit of maintaining it;
an agent that finds nothing writes nothing.

---

<div align="center">

[← 05 — Phantom Checks](05-phantom-checks.md) &nbsp;·&nbsp; [Playbook index](README.md) &nbsp;·&nbsp; [07 — Standing Routines →](07-standing-routines.md)

</div>
