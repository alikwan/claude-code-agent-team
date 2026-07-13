# FAQ

## Does this only work with Claude Code?

The **agent-file format** is Claude Code's (`.claude/agents/` Markdown with
YAML frontmatter), so the templates install as-is only there. The **method** —
single git owner, judge/fixer separation, verdict contracts, memory, the
pattern library — transfers to any environment that supports sub-agents or
multiple coordinated sessions. Porting the format is mechanical; porting the
discipline is the value.

## Does this work with stacks other than Laravel?

Yes — that's the design. The core files are framework-agnostic with a
[registered placeholder table](placeholders.md); Laravel + React is just the
worked example (and the original production system ran a different frontend
entirely — the agents didn't change). A Django, Rails, or Go adaptation is a
find-replace plus stack-specific wording in two agents. If you do one,
[contribute it](../CONTRIBUTING.md) as `examples/<your-stack>/`.

## How much does a pipeline run cost?

More than a single session, less than the bug it prevents. Rough shape: a
full 6-stage feature run makes 6–10 sub-agent calls on a strong model plus
2–3 on a cheap one; expect a multiple of roughly 3–5× the tokens of "just ask
one session to do it." Three levers control cost: the **hotfix lane** (3
stages, one builder) for small changes, **model tiering** (docs/security/
migration reviewers run on the cheap tier), and **conditional reviewers**
(spawned only when the diff triggers them). If token cost dominates your
thinking, your changes are probably small enough for the hotfix lane — or for
no pipeline at all, which is a legitimate answer.

## When should I *not* use the full pipeline?

- Throwaway code, prototypes, and one-off scripts — the overhead protects
  nothing.
- Changes that meet **all** the [hotfix criteria](../playbook/01-pipeline.md#the-hotfix-lane)
  — use the 3-stage lane.
- Pure explorations/spikes — run a normal session, then pipeline the real
  implementation.

The pipeline earns its cost on long-lived codebases where silent damage
compounds. That's the honest boundary.

## Can I start with fewer than 9 agents?

Yes — three: **orchestrator + one builder + qa-engineer**. That preserves the
two properties that matter most (plan-before-code, independent review). Add
`integration-agent` when your system grows wiring surface, the conditional
reviewers when you have a database and an attack surface, `documentation-agent`
when you have users. The growth path is in
[playbook/02](../playbook/02-roles.md#scaling-the-team-down-and-up).

## Why can't the team-lead write files? It would be faster

Until it isn't. An orchestrator that can "quickly fix" things stops
delegating, accumulates context from doing everyone's job, and degrades into
the single overloaded session this whole design escapes. The tool restriction
is the discipline — remove it and the pipeline dissolves within a week of
convenience decisions.

## Why is QA forbidden from fixing what it finds?

Because a judge with a pen approves their own unreviewed work. QA's read-only
tool list is what makes `QA APPROVED ✅` mean something. The write-capable
verification agent (`integration-agent`) exists separately, doesn't judge, and
its non-trivial changes go back to a scoped QA re-check. The split is the
methodology's core safety property — see the
[three load-bearing ideas](../playbook/00-overview.md#the-three-load-bearing-ideas).

## Aren't QA and integration-agent redundant?

They check different failure classes at different altitudes: QA judges the
**diff** (is this code correct?), integration walks the **system** (is this
feature connected — navigation, permissions, listeners, schedules?). One is a
read-only judge, the other a write-capable connector. In v1.1 their checklists
were explicitly de-duplicated: each pattern has exactly one owner.

## Why targeted tests only? Running everything is safer

Running everything *somewhere* is non-negotiable — the question is where. On
large suites, a full in-pipeline run per QA cycle makes review so slow that
pressure builds to skip it. The production answer: QA runs targeted tests
in-pipeline; **CI runs the full suite on every PR** as the backstop. If you
have no CI, set `{{FULL_SUITE_POLICY}}: full-suite-allowed`. The invariant is
documented with the [placeholder registry](placeholders.md).

## Why is there no deploy/release agent?

We tried automating past Gate 2. Field data was unambiguous: the release step
is exactly where backgrounded agents stall, and a stalled release is worse
than a manual one ([field note 2](../playbook/08-field-notes.md)). The
human-operated release is the documented design. The same evidence bar
rejected a dependency-update agent (it's the hotfix lane + CI) and a second
PR-review agent (that's CI's job — see the
[CI templates](../templates/ci/)).

## What's the deal with Arabic/RTL all over the templates?

The source system is Arabic-first, and the templates keep
`{{PRIMARY_LANGUAGE}}`-first as a first-class concern because localization
discipline is where AI-generated code fails quietly (English strings leaking
into user-facing surfaces). If your product language is English, the rule
costs you nothing; if it isn't, this may be the only agent playbook that
treats your case as the default rather than a footnote. The docs themselves
are bilingual for the same reason.

## How do agents get smarter over time?

The [memory system](../playbook/06-memory-system.md): each agent reads its
`MEMORY.md` at session start and appends after work — recurring issues, weak
spots, approved exceptions, and *disproven* findings. Memory is
version-controlled and committed alongside code. Governance is enforced
(index cap, topic-file overflow, absolutely no secrets).

## Something in the repo looks like private data. What do I do?

**Don't open a public issue.** Follow [SECURITY.md](../SECURITY.md) — private
vulnerability reporting, highest-priority handling.
