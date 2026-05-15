# Council Deliverable Generator

This skill generates the session's real-world deliverable — the artifact the council was convened to produce — using the outputs of a selected round as the authoritative basis.

## Directory Layout (relative to project root)
- `sessions/<session_id>/session.json` — session metadata including `deliverable` spec
- `sessions/<session_id>/round_001.json`, `round_002.json`, … — round data
- `sessions/<session_id>/deliverables/round_<NNN>/` — deliverable artifacts organized by round; base filename comes from `session.deliverable.output_path`

---

## Step 1: Select Session

Use Glob to list `sessions/*/session.json`. For each file found, Read it and extract `session_id`, `topic`, `round_count`, `updated_at`, and `deliverable` (all fields).

If no session files are found, tell the user: "No sessions found. Run `/council` to start a session." Then stop.

Present as **plain text output** — do not use an interactive question widget:

```
=== Council Deliverable Generator ===

Sessions:
  [1] "<topic>" — <round_count> rounds — Deliverable: <deliverable.description> [produced <N>× — latest round <from_round>]
  [2] "<topic>" — <round_count> rounds — Deliverable: <deliverable.description>

Enter a number (or q to quit):
```

Compute the produced annotation from `deliverable.history` (or `deliverable.produced_at` for legacy sessions):
- `deliverable.history` non-empty: show `[produced <N>× — latest round <from_round>]`
- `deliverable.history` absent/empty but `deliverable.produced_at` non-null (legacy): show `[produced — legacy]`
- Otherwise: omit the annotation entirely

Wait for user input. If `q`, stop.

If the selected session has `round_count` of 0, tell the user: "This session has no completed rounds — nothing to build from." Then stop.

---

## Step 2: Select Round

Using the selected session, Glob `sessions/<session_id>/round_*.json` to get the actual list of completed round files, sorted by name.

Present as **plain text output** — do not use an interactive question widget:

```
Generate deliverable from:
  [1] Round 1
  [2] Round 2
  ...
  [N] Round N  (latest)

Enter a round number:
```

Mark the last round as `(latest)`. Wait for user input.

---

## Step 3: Compute Versioned Path and Check for Existing Version

Compute the **versioned output path** for the selected round:
- Extract the directory portion and filename from `session.deliverable.output_path`. Example: from `sessions/<id>/deliverables/tailor-resume.md`, take directory `sessions/<id>/deliverables/` and filename `tailor-resume.md`.
- Construct: `sessions/<id>/deliverables/round_<NNN>/<filename>` where `<NNN>` is the selected round number zero-padded to three digits (e.g., round 4 → `round_004`).

Attempt to Read the versioned path. If the file already exists, ask:

```
A deliverable from round <N> already exists at: <versioned_path>
Overwrite it? [y/n]
```

Wait for user input. If `n`, stop. If the file does not exist, proceed without asking.

---

## Step 4: Load Context

**Selected round (full load):** Read `sessions/<session_id>/round_<NNN>.json` in full. This is the primary source. Extract all fields: `orchestrator_plan`, `experts`, `synthesis`, `strategist`, `critic`, `orchestrator_notes`.

**Prior rounds (summary load):** For each round before the selected round, Read it and extract only `round_num`, `synthesis`, `orchestrator_notes`. Discard `experts`, `strategist`, `critic`, `pending_questions`. These provide developmental context — what was established, revised, and resolved across rounds.

**Session brief:** Read `sessions/<session_id>/session.json` and extract `topic`, `problem_brief` (all fields: `interview_answers`, `qa_log`).

---

## Step 5: Generate Deliverable

Generate the deliverable artifact specified by `session.deliverable.description` and `session.deliverable.format`.

### Synthesis instructions

Use the selected round as the authoritative state of the design. The prior rounds' orchestrator_notes and synthesis inform what was settled earlier vs. what changed in the selected round — use them to avoid regressing to earlier positions.

The deliverable is the **real-world output the council was convened to produce** — not a report about the deliberation. Do not describe what the council said; build the thing they designed.

### Source hierarchy

1. **Selected round's `strategist`** — concrete approaches to implement. These are the primary implementation blueprint.
2. **Selected round's `synthesis`** — integrative framework; governs architecture and philosophy.
3. **Selected round's `orchestrator_notes`** — flags what is resolved vs. still open. Treat "resolved" items as implementation decisions; treat "still open" items as known limitations to note.
4. **Selected round's `critics`** — unresolved challenges that survived. Acknowledge in the artifact where they affect implementation.
5. **Selected round's `experts`** — detailed supporting material for any approach.
6. **Problem brief `interview_answers`** — constraints, success criteria, and the core tensions. These override any synthesis that contradicts them.
7. **`qa_log`** — user-clarified constraints from prior rounds. These are binding decisions, not suggestions.

### Format

Generate the deliverable in `session.deliverable.format`. The artifact should be self-contained and ready to use. Do not include meta-commentary about the deliberation process inside the artifact itself.

If the deliverable format is a Claude Code slash command (`.md` file):
- Write it as a complete, production-ready command file
- Include all prompting logic, input handling, and output instructions
- The command must work standalone — a reader who did not participate in the council session should be able to run it

If the deliverable format is something else (a document, a plan, a design spec, etc.), produce the cleanest version that reflects the selected round's synthesis.

---

## Step 6: Write Output and Update State

Write the generated artifact to the **versioned path** computed in Step 3 (`sessions/<id>/deliverables/round_<NNN>/<filename>`). Create intermediate directories if needed.

Then Read `sessions/<session_id>/session.json`. If `deliverable.history` does not exist, initialize it as an empty array — and if the legacy `deliverable.produced_at` field is non-null, seed history with `[{"produced_at": "<produced_at value>", "from_round": null}]` first. Append `{"produced_at": "<today's ISO date>", "from_round": <selected round number>, "file_path": "<versioned_path>"}` to `deliverable.history`. Write it back.

Tell the user:

```
Deliverable written to: <versioned_path>
Based on: Round <N> of "<topic>"
Session state updated: sessions/<session_id>/session.json

Open limitations (from orchestrator notes):
<list any items the orchestrator_notes flagged as still open or unresolved>
```

If there are no open limitations, omit that section.
