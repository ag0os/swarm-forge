# SwarmForge — Multi-Agent Coding Workflow

A reference describing how the SwarmForge swarm coordinates AI agents, intended
to be handed to an agent running on a similar harness so its ideas can be
applied elsewhere.

## What it is

A tmux-based orchestration layer that runs 4 AI agents as a disciplined
software-engineering pipeline. Each agent has a fixed role, works in its own git
worktree/branch, and hands off via message-passing. There is **no central
scheduler** — it is event-driven. A human seeds each feature; the rest of the
cycle runs autonomously.

## The pipeline

```
                          +---------+
                          |  HUMAN  |
                          +----+----+
              "next feature"   |   ^  "review this spec"
                               v   |  (explicit accept required)
   +-------------------------------------------+
   | SPECIFIER          (worktree: master)     |
   | - user intent -> Gherkin acceptance specs |
   | - parameterize varying fields             |
   | - commit, ask human to review             |
   +---------------+---------------------------+
                   |  handoff: branch + commit + changes
                   v
   +-------------------------------------------+
   | CODER              (worktree: coder)      |
   | - TDD: unit test -> code (3 rules of TDD) |
   | - build acceptance pipeline, run tests    |
   | - run Gherkin mutation, fix issues        |
   +---------------+---------------------------+
                   |  handoff (all tests green)
                   v
   +-------------------------------------------+
   | REFACTORER         (worktree: refactorer) |
   | - structure-preserving cleanup, NO behavior|
   | - coverage up, CRAP <= 6 (crap4go)        |
   | - verify via acceptance + unit tests      |
   +---------------+---------------------------+
                   |  handoff
                   v
   +-------------------------------------------+
   | ARCHITECT          (worktree: architect)  |
   | - design review, module boundaries        |
   | - merges ALL queued refactorer handoffs    |
   | - final gate: mutate4go -> dry4go ->       |
   |   Gherkin mutation (fix between each)      |
   +---------------+---------------------------+
                   | "job complete, merge"
                   v
            back to SPECIFIER -> merges, asks human for next feature
```

## Stages

| Stage | Agent | Owns | Output |
|---|---|---|---|
| 1. Specify | Specifier | Externally visible behavior | Gherkin acceptance specs, committed |
| — gate — | Human | Approval | Explicit accept before stage 2 |
| 2. Implement | Coder | Behavior slices | Production code + unit tests, all green |
| 3. Refactor | Refactorer | Internal structure | Cleaner code, same behavior |
| 4. Review/design | Architect | High-level design + final QA | Verified, merged feature |
| — loop — | Specifier | Merge + next feature | Asks human what's next |

## What the verifications check

The methodology layers complementary quality gates — each catches a different
failure mode:

- **Unit tests (TDD)** — written *before* code; correctness of each slice. Coder.
- **Acceptance tests** — generated from the Gherkin specs (parser -> generator ->
  executable tests); confirm externally visible behavior matches the spec. Coder.
- **Gherkin mutation** — mutates the *spec itself* to ensure the spec is precise
  and the tests actually pin it down. Run by coder and again by architect.
- **Mutation testing (`mutate4go`)** — mutates *production code*; surviving
  mutants = untested logic. "Kill all survivors." Architect (`--scan` mode to
  find oversized modules).
- **CRAP score (`crap4go`)** — `complexity^2 * (1-coverage)^3 + complexity` per
  function; flags complex+untested code. Threshold <= 6 (refactorer) /
  <= 4 (reviewer). Refactorer.
- **DRY (`dry4go`)** — duplication detection. Architect.

Key principle: **tests are not trusted until mutation-tested.** Coverage alone
is rejected as a quality signal.

## Tools used

| Tool | Purpose | Stage | Source |
|---|---|---|---|
| `mutate4go` | Code mutation testing | Architect (final) | github.com/unclebob/mutate4go |
| `dry4go` | Duplication detection | Architect (final) | github.com/unclebob/dry4go |
| `crap4go` | CRAP score reduction | Refactorer | github.com/unclebob/crap4go |
| Acceptance-Pipeline-Specification | Gherkin parser/generator/mutator | Specifier + Coder | github.com/unclebob/Acceptance-Pipeline-Specification |

All tools are Go-specific; the swarm as shipped targets a Go project. A Clojure
variant uses `mutate4clj` and similar.

## Coordination mechanics

- **Role prompt = standing instructions**, loaded once at startup; never
  changes. The current task arrives as an injected message.
- **Handoff protocol**: every message starts with `Review your rules.` and must
  include **branch name + commit hash + what changed**. Code travels via git;
  the message just points to it.
- **Message queueing**: messages arriving mid-job are saved as numbered files in
  `pending-messages/`; processed in sorted order after the current job; lower
  numeric prefix = higher priority (architect handoffs jump the queue).
- **Isolation**: each role in its own git worktree/branch — agents cannot
  clobber each other.
- **No-op suppression**: empty-diff handoffs are dropped, not forwarded.
- **Batching**: the architect merges *all* queued refactorer handoffs together
  rather than one at a time.

## Ideas worth porting to another harness

1. **Separation of concerns by role** — specify / implement / refactor / review
   as distinct agents with narrow, non-overlapping mandates and explicit
   "do NOT" lists (e.g. refactorer must not add behavior; specifier must not run
   mutation).
2. **Layered, ordered quality gates** — do not run one big "check"; run
   correctness -> mutation -> duplication -> complexity, fixing between each,
   because each gate exposes a different defect class.
3. **Mutation testing as the trust anchor** — treat passing tests / coverage as
   necessary-but-insufficient; only mutation-killed code is "verified."
4. **Specs as mutation-tested artifacts** — the spec is itself an object under
   test, not just prose.
5. **Structured handoffs** — a fixed message contract (rules reminder + branch +
   commit + diff summary) so a receiver always has enough context to act cold.
6. **Priority-ordered work queue** — durable, sortable pending-task files
   instead of interrupting in-flight work.
7. **A human gate at exactly one point** (spec acceptance) — autonomy
   everywhere else.
