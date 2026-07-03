### ZYPEH skills

> Grandmaster grade 25 years of craftmanship. Bald tier UNIX hacker style skill.

This is some skills written by zypeh that is super opinionated because he eats spicy food in Kuala Lumpur. And he is also dreaming of working as a forward deployment engineer since he was 5. Anyway,

##### You may install the skill you want via
    # chmod +x install
    # install ./spec-format

##### Done, it will be added to `~/.claude/skills` which is readable by opencode and claude code. The rest you need to figure it out yourself.

## What is


### spec-format
  - It defines schemas in markdown + xml tag + YAML because AI trains via it. And I added some xml tag to add some tight harness to LLM to achieve great power like Kurapika.
  - In short these are the examples

```markdown
<challenge id="C1" summary="brief one-line">
  The concern, ambiguity, or missing case raised against the spec.
</challenge>

<response to="C1" status="resolved | open">
  The resolution. status="open" means unresolved and likely a
  high-fidelity question needing a prototype (handoff, not a block).
</response>
```

##### It is good to have voice of challenge when in spec writing so that we can pretend that democracy is a virtue. I choose these words because it is mostly trained by LLM house. I slapped those id for program to search, **Claude** particularly likes these code name like airplane number "HM37", "R2", "W1-b", use it for the plot really frfr.


```markdown
<non_goal id="NG1">what this spec deliberately does NOT cover</non_goal>
<decision id="D1">a design decision made and its rationale</decision>
<assumption id="A1">something assumed true until proven otherwise</assumption>
<open_question id="Q1">a question still to be answered (not a challenge)</open_question>
```

##### and these are the context for you to do strip off the unused context and leaving the most important context for LLM to execute in a session. This is good for handsoff me personally believe.


