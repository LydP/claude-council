# Council Summary

This skill prints an inline summary of a council session — either the whole session or a single round. Nothing is written to disk.

## Directory Layout (relative to project root)
- `sessions/<session_id>/session.json` — session metadata
- `sessions/<session_id>/round_001.json`, `round_002.json`, … — round data

---

## Step 1: Select Session

Use Glob to list `sessions/*/session.json`. For each file found, Read it and extract `session_id`, `topic`, `updated_at`, and `round_count`.

If no session files are found, tell the user: "No sessions found. Run `/council` to start a session." Then stop.

Present to the user as **plain text output** — do not use an interactive question widget:

```
=== Council Summary ===

Sessions:
  [1] "<topic>" — <round_count> rounds — last updated <updated_at>
  [2] "<topic>" — <round_count> rounds — last updated <updated_at>

Enter a number (or q to quit):
```

Wait for user input. If `q`, stop.

---

## Step 2: Select Scope

Using the `round_count` from the selected session's `session.json`, present the scope menu as **plain text output** — do not use an interactive question widget, which has an option limit that may be lower than `round_count` and will silently drop rounds.

```
Summarize:
  [a] Whole session
  [1] Round 1
  [2] Round 2
  ...
  [N] Round N

Enter a choice:
```

List only rounds that actually exist (Glob `sessions/<session_id>/round_*.json` to confirm). Wait for user input.

---

## Step 3a: Whole-Session Summary

Read `sessions/<session_id>/session.json` and extract `topic`, `updated_at`, `round_count`, `problem_brief` (all fields: `interview_answers`, `qa_log`).

Context-efficient round loading: Glob `sessions/<session_id>/round_*.json`. For each file, Read it and extract **only** `round_num`, `orchestrator_plan`, `synthesis`, `strategist`, `critic`, and `orchestrator_notes`. Discard `experts` and `pending_questions`.

Print the following inline (do not save to any file):

```
=== Session Summary: "<topic>" ===
Rounds: <round_count> | Last updated: <updated_at>

PROBLEM
  Desired outcome: <interview_answers.desired_outcome>
  Core tension:    <interview_answers.core_tension>
  Constraints:     <interview_answers.constraints>
```

If `qa_log` is non-empty, append:

```
CLARIFICATIONS (<count>)
  Round <N>: Q: <question> / A: <answer>
  [...]
```

Then append:

```
ROUND PROGRESSION
  Round 1 | Perspectives: <orchestrator_plan.perspectives as comma-separated list>
    Synthesis:  <first 2 sentences of synthesis>
    Approaches: <extract and list only the approach titles/names, comma-separated>
    Critic:     <first sentence of critic output>
    Notes:      <first sentence of orchestrator_notes>

  Round 2 | ...
  [repeat for each round]

FINAL APPROACHES (Round <N>)
<Full strategist output from the last round>

UNRESOLVED CHALLENGES
<Review orchestrator_notes across all rounds for challenges flagged as unresolved or persistent. List them. If none are explicitly flagged, pull the Critic's top challenge from the last round.>
```

---

## Step 3b: Single-Round Summary

Read `sessions/<session_id>/session.json` and extract `topic` only.

Compute the zero-padded filename: `round_<NNN>.json` (e.g., round 2 → `round_002.json`). Read `sessions/<session_id>/round_<NNN>.json` in full.

Print the following inline (do not save to any file):

```
=== Round <N> Summary: "<topic>" ===
Perspectives: <orchestrator_plan.perspectives as comma-separated list>

EXPERT HIGHLIGHTS
  <Perspective 1>: <2–3 sentence summary of that expert's output>
  <Perspective 2>: <2–3 sentence summary>
  [repeat for each perspective]

SYNTHESIS
<Full synthesizer output>

PROPOSED APPROACHES
<Full strategist output>

CRITIC'S CHALLENGE
<Full critic output>

ORCHESTRATOR NOTES
<Full orchestrator_notes>
```
