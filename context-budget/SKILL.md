---
name: spec-driven-development
description: Use on any potentially complex project, which can lead to expansive specification and documentation.
license: MIT
metadata:
  audience: maintainers
  workflow: planning
---

# Context Budgeting

Excessive information and context lead to unnecessary token spending and all the costs associated, and increase the risk of model hallucination. So, this skill imposes a rigid policy on context size.

# 1 Agent Instruction File Cap

**Agent instruction files** — `AGENTS.md`, `CLAUDE.md` or equivalent — must not exceed **10 KB** each, measured with `wc -c`.

**They are a decision and action register, not an archive.** Record the minimum strictly necessary to act: which decisions are already made, which actions must be performed, and the mechanical commands that enforce them. **Lead with the imperative** — what to do, what not to do, what to re-derive — rather than the reasoning behind it. Detail, measurements and rationale belong in the specification that earned them and must not be duplicated here; a fact recorded in both will go stale in one of them. Prefer one-line rules over paragraphs, and a pointer over a restatement.

**When over the cap**, move the content into the owning specification (or the one being written) and leave a pointer, then cut in this order: worked rationale, history, restated specification content, and any background a reader can obtain by running a command.

**It is also the reload target after a session compression.** Section 0.3 makes the context a cache; `AGENTS.md` is where the working state is rebuilt from once a compression pass has run. Write it at GATE 1 (section 5) and keep it inside the 10 KB cap — a register that overflows cannot be reloaded, which defeats the compression it exists to serve.

# 2 Session Context Budget

**Baseline.** `B` is the size of the session context when work begins, recorded once, in bytes. Prefer the occupancy the harness reports. Failing that, accumulate the agent's own output and tool results into a scratch file outside the repository and measure it with `wc -c`. **When neither source can state it, `B` is 1 KB.** Record `B` and its source in `spec.md`; never re-derive it.

**Increment.** One loop is one author prompt followed by the agent's thinking and generation. `Δ = max(0.20 × B, 4,000 tokens)` (4,000 tokens ≈ 16 KB). A loop may retain at most `Δ` of new content.

**Growth is linear, never compounding.** The ceiling after `n` loops is `B + n·Δ`. An increment is measured against `B`, never against the total reached so far; `B·1.2ⁿ` is not the rule.

**Compress after every two loops**, before the next author prompt: distil the pair to its decisions, open questions and identifiers, and drop what is already written to disk. A pair may retain up to `2·Δ` before the pass runs, so the linear ceiling `B + n·Δ` still holds.

**At GATE 1 (section 5), compress the whole session to `max(B, Δ)`, write `AGENTS.md`, then continue as the author directs.**

**Durable state lives on disk, never in context.** `spec.md`, `plan.md` and `AGENTS.md` are the memory; context is a cache rebuilt from them. A fact needed after a compression that is not in a file is a defect.


