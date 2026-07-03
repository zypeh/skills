---
name: roast
description: Interview a vague problem into a buildable spec. Use when the plan is fuzzy and needs stress-testing before code.
---

# roast

A two-phase interview that scouts the question dependency tree upfront, then
walks it in rounds — batching up to 4 uncorrelated questions per round to
cover maximum surface per turn, while never batching questions that depend on
each other. Questions the codebase can answer are answered by reading the
codebase, not by asking the user. Decisions are captured inline as spec-format
challenge/response pairs. When the interview converges, the crystallized spec
is written to a file.

## When to use

Invoke at the start of a change when the plan is still fuzzy and you want to
stress-test it before any code exists. If the plan is already clear, skip
directly to implementation and start `triage` skill.

## Phase 1 — scout the tree

Before asking the user anything, map the question dependency tree:

1. Enumerate the questions the spec needs answered. Think broad first —
   cover the full surface of the problem, not just the first branch you see.
2. Build the dependency DAG as a compact node list — one line per question,
   each naming the questions it depends on:

   ```
   #    question                     deps    fid   status
   q1   route + URL                  -       lo    open
   q3   auth required?               q1      lo    open
   q7   nav entry placement          q1      lo    open
   q2   settings screen feel         q1      hi    open
   q5   data source                  -       hi    open
   ```

   `deps` is a comma-separated list of ids (`-` = none); an id in another node's
   `deps` is an edge A -> B meaning B cannot be asked until A resolves. Keep it
   acyclic with no dangling `deps`. Deliberately **not** JSON or JSONL — a flat
   list is cheap to mutate (a question is one line, a dependency is one field)
   and cheap to re-read, which matters because this artifact gets scrapped and
   rebuilt as the interview churns toward convergence. Yet the `deps` column is
   machine-parseable, so the answerable set (Phase 2) is computed
   deterministically from it — this is the one artifact, serving both the agent
   and the user (step 5), so there is nothing to keep in sync.

3. Mark each node's fidelity:
   - **low-fidelity** — answerable without a prototype (interviewable).
   - **high-fidelity** — needs a prototype or visual to answer (trigger 1,
     parked as `status="open"`, not asked in the interview).
4. Estimate total question count. If the tree is large enough to risk the
   dumb zone (trigger 2), recommend breaking scope into chunks BEFORE
   interviewing — see "Scope management."
5. Share that same node list with the user (one-time, compact). It is both the
   agent's substrate and the user's map — the `fid` column marks the
   high-fidelity leaves and the `deps` column shows the shape, so no second
   artifact is needed. It reads directly in a terminal. If a visual helps the
   user steer, sketch it as an indented tree, but the list stays the source of
   truth.

The scout phase costs one turn. It pays back across every subsequent round:
the next batch is pre-queued from the tree, so round-trip latency drops and
no round is wasted on a question whose dependency is still unresolved.

## Phase 2 — round-based interview

Walk the tree in rounds. Each round:

1. **Compute the answerable set** — the questions still `open` whose `deps` are
   all `done` (or empty). Compute this deterministically from the node list (a
   reachability / topological pass over the `deps` column), not by reasoning it
   out each round — this guarantees no question is surfaced before its
   dependencies resolve. See "Model-agnostic operation" for how to run it.
2. **Select the batch** — from the answerable set, pick up to 4 questions that are
   maximally uncorrelated and distant from each other in the tree (different
   subtrees, covering the most surface). This extracts the most direction from
   the user per round.
   - If the answerable set has only 1 question, ask 1. Do not pad.
   - If the answerable set has 7 independent questions, pick the 4 most distant and
     queue the rest for the next round.
   - Never exceed 4. Never ask two questions that depend on each other in the
     same round.
3. **Ask the batch** — present all selected questions together, each with a
   recommended answer (a concrete default the user can accept or override).
4. **Resolve** — the user answers all, some, or says "skip — I don't know
   yet" to any. Capture each resolved decision inline as a spec-format
   `<challenge>`/`<response>` pair (see the `spec-format` skill).
   Hard-to-reverse decisions become `<decision>` blocks. Scope boundaries
   become `<non_goal>` blocks. Things assumed true become `<assumption>`
   blocks.
5. **Advance** — flip each resolved node's `status` to `done` (a one-word line
   edit). That unlocks its dependents, which join the next round's answerable
   set. Recompute the answerable set and repeat.

### Rules that hold across all rounds

- **Dependency order is sacred.** A question is never asked before its
  dependencies are resolved. The tree enforces this — the answerable set only
  contains questions whose deps are done.
- **Batching is for uncorrelated questions only.** Correlated or dependent
  questions stay one-at-a-time. The selection step picks distant nodes
  precisely to avoid correlation.
- **Codebase-answerable questions** — read the code, do not ask the user.
  Remove these from the tree entirely; answer them yourself and capture as
  resolved challenge/response pairs.
- **Stop when you stop learning.** When all challenges have resolved
  responses, no high-fidelity questions remain open, and scope is bounded by
  non-goals — the interview is done. Do not over-plan into low-fidelity
  detail.

## Low-fidelity vs high-fidelity questions

| Type          | Definition                                      | Example                                        | Interviewable? |
| ------------- | ----------------------------------------------- | ---------------------------------------------- | -------------- |
| Low-fidelity  | Answerable without a prototype or visual        | "What URL should this route live on?"          | Yes            |
| High-fidelity | Needs a zoomed-in prototype or visual to answer | "How should this UI feel when we're using it?" | No             |

Only low-fidelity questions belong in the interview. High-fidelity questions
are handled by trigger 1 (below).

## The three handoff triggers

Three conditions force a context boundary. Recognize and handle each
explicitly.

### Trigger 1 — high-fidelity question (prototype handoff)

High-fidelity questions are identified during the scout phase (phase 1) as
leaves marked high-fidelity. They are never asked in the interview. When the
answerable set reaches one:

1. Park it immediately as a `<response status="open">` in the spec-format
   artifact — this is an open high-fidelity question, not a block.
2. Spawn a prototype subagent (via the Task tool) to build just enough to
   see or feel the answer. The prototype scope should be minimal — one screen,
   one interaction, one layout.
3. Carry the answer back to the interview.
4. Update the parked response from `status="open"` to `status="resolved"`
   with the answer.
5. Resume interviewing. The resolved answer may unlock dependents in the tree.

Pattern: interview → prototype → interview. Do not grind on high-fidelity
questions in the interview session. The scout tree makes these visible
upfront, so you can batch-prototype several in parallel if multiple
high-fidelity leaves are on the answerable set at once.

### Trigger 2 — context window limit (the dumb zone)

Most state-of-the-art models degrade past ~120k tokens — attention
relationships strain and decisions worsen. Monitor context size during the
interview.

When context is nearing the limit:

1. Cut the interview. Do not keep going into the dumb zone.
2. Emit a **partial** spec-format artifact to `specs/<short-kebab-name>.md`
   with everything resolved so far. Open high-fidelity questions stay
   `status="open"`.
3. Resume the interview in a fresh session, loading the partial spec as input.
4. **Do not classify yet** — the `triage` skill runs only when the interview
   has converged, not mid-trigger.

Never clear context before emitting a spec — the in-context decisions are the
input. Clearing throws them away.

### Trigger 3 — phase boundary

Each phase is a fresh context. The artifact is the handoff:

```
roast (interactive, this skill)
  -> spec artifact (specs/<name>.md, spec-format)
    -> triage (classification, fresh context)
      -> confirm (user approves tier)
        -> implementation (fresh context, later)
```

The spec artifact survives every boundary. It is the single source of truth
that each phase loads.

## Active, not passive

The interview is a conversation, not a one-sided extraction. The agent asks
batches of questions, but the user steers:

- Keep the scope on track. If a round drifts into low-fidelity detail,
  redirect to the next uncorrelated branch.
- If the agent asks a question already answered, point it back to the tree.
- Stop the interview when you stop learning — do not let it over-plan.
- The scout tree is a map, not a contract. The user can collapse branches
  ("already decided — skip those"), add branches ("you missed this concern"),
  or reorder ("answer this subtree first, it's blocking everything else").

Conversely, do not be too active: answering every low-fidelity detail when you
should be writing code is over-interviewing. Find the middle ground.

## Crystallization (at convergence)

When the interview converges:

1. Write the full spec to `specs/<short-kebab-name>.md` using the `spec-format`
   contract. Every `<challenge>` has a matching `<response>`. Set
   `status: buildable` in frontmatter if no responses are `status="open"`. Set
   `status: in_review` if any remain open.
2. Do not clear context before writing — the in-context decisions are the
   input.
3. Load the `triage` skill and apply its rubric to classify the implementation
   tier.
4. If context is large (trigger 2 fired during the interview), dispatch
   `triage` as a subagent via the Task tool with the spec path as input — this
   gives fresh-context isolation. Otherwise, apply the rubric in-session.
5. Present the triage verdict to the user for confirmation (tier, rationale,
   recommended flow, risks). Let the user confirm, drop to a lower tier, or
   escalate.
6. On confirmation, proceed to implementation per the confirmed tier's flow.

## Scope management

The scout phase (phase 1) estimates total question count upfront. If the tree
is large enough to risk the dumb zone (trigger 2), break scope BEFORE
interviewing:

1. Start with the large scope.
2. Break it into smaller, independently interviewable chunks — each its own
   subtree with a manageable question count.
3. Interview each chunk in its own session.
4. Each chunk gets its own spec artifact.

Large scopes blow both trigger 1 (hidden high-fidelity questions — the scout
tree surfaces them, but too many means too much prototype work) and trigger 2
(context limits). The scout estimate lets you catch this before wasting a
round.

## Model-agnostic operation

This skill uses environment-neutral language. Both opencode and Claude Code
provide the primitives it needs:

- **Subagent dispatch:** "spawn a prototype subagent" or "dispatch triage as a
  subagent" — use the Task tool available in your environment.
- **User confirmation:** "present the verdict to the user" — use the question
  or AskUserQuestion tool available in your environment.
- **Codebase reading:** "read the code" — use the read, grep, or glob tools
  available in your environment.
- **Deterministic computation:** "compute the answerable set from the node list"
  — run a small script or code-execution step available in your environment
  (reachability over the `deps` column); do not compute it by hand.
- **File writing:** "write the spec to specs/<name>.md" — use the write or
  edit tools available in your environment.

No environment-specific tool names are hardcoded. Interpret each instruction
using the tools you have.
