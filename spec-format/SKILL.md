---
name: spec-format
description: Reference schema for structured spec/PRD documents. Defines a challenge/response validated contract (every challenge needs a resolved response before a spec is buildable) plus non_goal, decision, assumption, open_question blocks using XML-style tags in markdown. Use when writing or validating a spec/PRD, or when another spec-* skill references the format.
---

# spec-format

A model-agnostic schema for specs/PRDs: markdown body + YAML frontmatter + XML-style semantic tags. One validated contract; everything else is structured-but-unvalidated periphery. Designed so GLM, DeepSeek, and Claude all parse it identically via plain text constructs (no reliance on a model being trained on a specific dyad).

## Container

A `.md` file with YAML frontmatter:

```yaml
spec: <short-kebab-name>
status: drafting | grilled | buildable
```

`buildable` is the gate: set only when every challenge has a matching resolved response (zero open).

## Validated core — the ONE contract

```xml
<challenge id="C1" summary="brief one-line">
  The concern, ambiguity, or missing case raised against the spec.
</challenge>

<response to="C1" status="resolved | open">
  The resolution. status="open" means unresolved and likely a
  high-fidelity question needing a prototype (handoff, not a block).
</response>
```

**Contract rule:** every `challenge id=X` requires exactly one `response to=X`. A spec is `buildable` only when no response has `status="open"`.

## Recognized periphery (structured, no contract)

```xml
<non_goal id="NG1">what this spec deliberately does NOT cover</non_goal>
<decision id="D1">a design decision made and its rationale</decision>
<assumption id="A1">something assumed true until proven otherwise</assumption>
<open_question id="Q1">a question still to be answered (not a challenge)</open_question>
```

No pairing rule. Use them to preserve design work (`decision`), bound scope (`non_goal`), and surface pending items (`assumption`, `open_question`).

## Extension

A new block is just `<newtag id="...">...</newtag>`. Document it here when adopted. Add a contract rule ONLY if the new tag requires pairing (like challenge/response). No schema versioning, no restructuring.

## How to apply

1. Author the spec as one markdown file using the container + tags above.
2. Inline `<challenge>`/`<response>` pairs as they arise during grilling; raise `status="open"` for high-fidelity questions that need a prototype.
3. Before declaring `buildable`, run the contract check: every challenge has a matching response, none open.
4. Do NOT clear context before emitting a PRD in this format — the in-context decisions are the input.
