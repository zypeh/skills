---
name: triage
description: Use when a spec-format artifact has passed review (status in_review or buildable) and you need to decide how much process a build warrants before implementing. Dispatch after the roast interview converges, or standalone on an existing reviewed spec. Draft specs go back to roast, not here.
---

# triage

A one-shot classifier. Reads a reviewed spec-format artifact and judges which
implementation tier (0-3) the work belongs to. Emits a structured verdict the
caller presents to the user for confirmation.

## When to use

After `roast` converges and the spec is written to `specs/<name>.md`, or
standalone on any existing reviewed spec, to decide how much process a build
warrants before writing code.

## Precondition — the spec must be past review

triage classifies a *reviewed* spec. Read the frontmatter `status` first,
before anything else:

| status       | meaning                                                                                     | action                                                                          |
|--------------|---------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| `draft`      | challenges still being raised; not converged                                                | STOP. Return to `roast` and finish the interview. Do not classify.              |
| `in_review`  | every challenge has a response; some are `status="open"` (high-fidelity prototype handoffs) | Classify. Carry the open ids forward as `pending`.                              |
| `buildable`  | every challenge resolved, none open                                                         | Classify.                                                                        |

Also bounce to `roast` any spec whose contract is violated — a `<challenge>`
with no matching `<response>` — regardless of its declared status. An open
*response* is NOT a bounce reason: it is a parked prototype handoff resolved
during implementation, not interviewing.

## Signals

| signal                  | reading                                                                                  |
|-------------------------|------------------------------------------------------------------------------------------|
| context-size risk       | implementation blows the ~120k dumb zone -> tier 2 floor (spec needed as handoff)        |
| decision reversibility  | hard-to-reverse (architecture, data model) -> tier 2 floor (review checkpoints)          |
| open high-fidelity Qs   | each `<response status="open">` is a prototype handoff; many -> tier 2-3                  |
| work decomposition      | build too large to hold in one session; must split into independent slices -> tier 3 (a lone prototype-handoff subagent does NOT count) |
| ambiguity               | more unresolved assumptions/open questions -> higher tier                                 |
| frontend vs backend     | frontend leans tier 1 with prototype handoffs; backend/multi-component -> tier 2-3        |

**Combining signals:** the tier is the MAXIMUM any single signal implies, never
the average — signals escalate, they do not cancel. Context-size risk and
hard-to-reverse decisions are hard floors: either one sets tier 2 minimum on its
own. Do not deflate below a floor; do not inflate above the max.

**Raw file count is a hint, not a driver.** The real size questions are
context-size risk (does the build fit one session?) and work decomposition — not
the file tally. A one-file change to a hard-to-reverse data model is tier 2; six
trivial CRUD files are not tier 3. Judge the work, not the count.

## Tiers

| tier | name       | when                                                                       | flow                                                                                                      |
|------|------------|----------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| 0    | just-do-it | trivial and self-contained, no ambiguity, no design decisions              | implement directly, no spec                                                                                |
| 1    | minimal    | most tasks incl. most frontend; clear path, manageable context            | spec inline; implement in-session after the interview                                                     |
| 2    | spec       | context nears the dumb zone, or a hard-to-reverse decision — but the build still fits one fresh session | spec artifact is the handoff; implement in a fresh session loading it; prototype handoffs for open Qs      |
| 3    | full       | the build itself is too big for one session; must be decomposed into independent slices dispatched with review between | spec -> slice -> dispatch -> review. **Not yet wired — recommend tier 2 as interim.**                      |

**Tier 2 vs 3 — the deciding question:** can the whole build fit in a single
fresh session? If yes — one sequential slice, even a large one — it is tier 2.
If the components are too many to hold in one context and the work must be
split into independently-built slices, it is tier 3. Spawning a prototype
subagent for an open high-fidelity question is NOT decomposition and never, by
itself, lifts work to tier 3.

**Borderline calls (the confidence gate).** When two adjacent tiers are a
genuine toss-up — you cannot call it high-confidence — do NOT silently pick.
Set `tier` to the safer default (for a 2/3 toss-up, tier 2 — tier 3 folds to a
tier-2 interim anyway), `confidence="low"`, and `fork` to the other candidate.
In the rationale, state the ONE deciding question and how each answer resolves
it, so the user decides at confirmation. This gate fires only on true
boundaries; clear-cut specs get one pass and cost nothing extra.

**Boundary anchors (2 vs 3) — the fact that tips each:**

- **Tier 2** — a Redis-backed rate limiter: ~5 files, one hard-to-reverse Lua
  primitive, but one person builds it end-to-end in a single session. The floor
  sets 2; fitting one context keeps it at 2.
- **Tier 3** — multi-tenant billing: metering + pricing service + invoicing +
  dashboard + Stripe + reconciliation — too many independent surfaces to hold
  in one session, so it must split into separately-built slices. Decomposition
  tips it to 3.

Tier 3 is recognized but not implemented. If the work is tier 3, say so, emit
the `tier3_note`, and recommend tier 2 as the interim path until the
slice/dispatch/review chain is built.

## Verdict

Emit a spec-format fragment — XML-style tags in markdown, the same
model-agnostic construct `spec-format` uses, so it appends onto the spec and
parses identically across models. Scalars are attributes; prose is the body;
open questions are cited by their spec id, not re-listed.

```xml
<triage tier="[0-3]" confidence="[high|medium|low]" pending="[open Q ids, or -]" fork="[other candidate tier if borderline, else -]">
[one paragraph: why this tier, naming the signals that drove it and the floor
or max that set it]

<flow>[concrete next steps for this tier]</flow>

<risk id="R1">[what could go wrong at this tier]</risk>
<risk id="R2">[...]</risk>

<tier3_note>[only if tier=3: full chain not yet wired; tier 2 is the interim fallback]</tier3_note>
</triage>
```

`risk` is a spec-format extension tag (a new `<tag id="...">` per the
spec-format extension rule). `pending` cites the spec's open `<response>` ids —
the high-fidelity questions carried into implementation. `fork` is `-`
unless the call is a genuine boundary toss-up (see Borderline calls), where it
names the other candidate tier so the user resolves it at confirmation.

## Worked example

Input — a reviewed spec with one high-fidelity question still open:

```markdown
---
spec: bulk-csv-import
status: in_review
---

Upload a CSV, validate rows, stream them through the existing IngestPipeline,
report per-row failures.

<decision id="D1">Reuse IngestPipeline, not a second write path — one
validation surface, one row-schema source of truth.</decision>

<challenge id="C2" summary="large files blow memory">
  A 500MB upload can't be buffered in memory.
</challenge>
<response to="C2" status="resolved">
  Stream-parse; never hold more than one 1k-row batch at once.
</response>

<challenge id="C3" summary="failure-report UX">
  How should the per-row failure report look and feel?
</challenge>
<response to="C3" status="open">
  High-fidelity. Needs a prototype of the report screen to answer.
</response>
```

Verdict:

```xml
<triage tier="2" confidence="high" pending="C3">
Five files across a backend streaming path plus one UI surface. C2's streaming
constraint is a hard-to-reverse floor that sets tier 2 on its own, independent
of the modest file count — signals take the max, not the average. Not tier 3:
single vertical slice, no parallel dispatch.

<flow>Spec is the handoff; implement in a fresh session loading it. Prototype
the C3 report screen as a trigger-1 handoff during implementation, then wire it
in. Test-first on row-validation and stream-batching, where behavior is
concrete.</flow>

<risk id="R1">An off-by-one in batch flushing silently drops rows — cover it
with a test before building on it.</risk>
<risk id="R2">Building the report collector before the C3 prototype answers its
shape risks rework.</risk>
</triage>
```

The example is calibrated to three failure points: an open response is *carried*
(`pending="C3"`), not bounced; the tier is set by the streaming floor even
though the file count reads tier 1 (max, not average); and it names why it
stops at tier 2 (single slice, no dispatch).

## Rules

- Read the whole spec before judging. Do not skim.
- Bounce `draft` and contract-violated specs to `roast`. Never classify them.
- Take the max signal; respect the hard floors; neither inflate nor deflate.
- Be honest about confidence. Mixed signals -> medium, and say why.
- Open high-fidelity responses are expected at any tier — cite them in
  `pending`, do not treat them as blockers.
- Tier 3: identify, flag as not-yet-wired, recommend tier 2 interim.
- Adjacent-tier toss-up: emit the safer tier with `fork` set and
  `confidence="low"`; never silently pick (see Borderline calls).

## Model-agnostic operation

A rubric, not a tool chain. It runs wherever it is dispatched:

- **In-session:** the primary agent applies the rubric after the interview.
- **Subagent dispatch:** if context is large, the caller dispatches a subagent
  (via the dispatch tool in your environment) with this rubric as the prompt
  and the spec path as input; the subagent returns the `<triage>` verdict.

The XML-style-tags-in-markdown verdict parses identically across models — no
reliance on any one model's training dyad. No environment-specific tool names
are hardcoded.
