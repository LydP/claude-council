# Claude Council — Agentic Deliberation Team

This skill runs a multi-agent deliberation session. You (Claude) act as the Orchestrator, spawning specialized subagents to iteratively develop, synthesize, challenge, and refine frameworks for addressing any problem, question, or challenge the user brings.

## Directory Layout (relative to project root)
- `sessions/<session_id>/` — one directory per session
  - `session.json` — session metadata (topic, timestamps, round count, problem brief)
  - `round_001.json`, `round_002.json`, … — one file per completed round
  - `reports/report.md` — final deliberation report (written at session end)
  - `deliverables/` — generated artifacts from deliverable production
- `prompts/` — agent role instructions (read at runtime)
  - `expert.md`, `synthesizer.md`, `strategist.md`, `critic.md`, `orchestrator_reflect.md`

Round files are zero-padded to three digits so they sort lexicographically in round order.

---

## Step 1: Session Setup

Use Glob to list `sessions/*/session.json`. For each file found, Read it and extract `session_id`, `topic`, `updated_at`, and `round_count`.

Present to the user:
```
=== Claude Council ===

Existing sessions:
  [1] "<topic>" — <round_count> rounds — last updated <updated_at>
  [2] "<topic>" — <round_count> rounds — last updated <updated_at>
  [N] Start a new session

Enter a number:
```

Wait for user input.

**If resuming**: Read `sessions/<session_id>/session.json` only. Display the topic, round count, the existing problem brief (interview answers + any Q&A log entries), and the deliverable status:

```
Topic: <topic>
Rounds completed: <round_count>
Deliverable: <deliverable.description> → <latest history[].file_path if history is non-empty, otherwise deliverable.output_path>  [<history status>]
  (or "No deliverable defined" if deliverable is null)
```

Compute `<history status>` from `deliverable.history`:
- `deliverable.history` non-empty: `produced <N>× — latest from round <from_round> on <produced_at>`
- Otherwise: `not yet produced`

Ask: "Continue from round <round_count+1>?" and wait for confirmation. Do not read any round files yet. Skip Step 1.5 — the interview was already completed.

After the user confirms, offer a pre-round context prompt:

```
Add context before starting round <round_count+1>?
  [i] Add information or attach files   [n] Begin round
```

If `[i]`: process using the file-attachment logic described in the `[i]` handler (Step 2h) — ask for text and/or file paths, store any file paths by reference (do not inline contents), append to `problem_brief.qa_log`, then begin the round.

If `[n]`: proceed directly to Step 2.

**If new**: Ask: "What problem, question, or challenge would you like to work through?" Wait for user input. Create the session directory and write an initial `sessions/<session_id>/session.json`:

```json
{
  "session_id": "<descriptive-slug_YYYYMMDD>",
  "topic": "<user's topic>",
  "created_at": "<ISO timestamp>",
  "updated_at": "<ISO timestamp>",
  "round_count": 0,
  "problem_brief": {
    "raw_topic": "<user's topic>",
    "interview_answers": {},
    "qa_log": []
  },
  "deliverable": null,
  "final_report": null
}
```

Derive the session ID from the topic: extract 3–5 key nouns or verbs, lowercase them, strip punctuation, and join with hyphens. Append `_YYYYMMDD` (today's date) for uniqueness. Examples: "How should I approach hiring for my startup?" → `hiring-startup-approach_20260514`; "Build a content strategy for Q3" → `content-strategy-q3_20260514`. Keep the slug under 40 characters. If the slug would collide with an existing session directory, append `_2`, `_3`, etc.

---

## Step 1.5: Interview Phase (new sessions only)

Before entering the round loop, conduct a brief consultant-style interview to deepen your understanding of the problem. You reason directly — no Agent spawn.

Generate 3–5 targeted questions tailored specifically to the topic the user provided. Cover these areas (adapt the phrasing naturally to the topic — do not ask questions that are obviously inapplicable):

- **Desired outcome**: What would a successful resolution look like? What changes?
- **Constraints**: What can't be changed? (Time, budget, people, technology, ethics, institutional limits, non-negotiables)
- **Prior attempts**: What has already been tried or seriously considered? Why wasn't it sufficient?
- **Success criteria**: How will you know this worked? What are the concrete indicators?
- **Core tension**: What makes this hard? What is the fundamental trade-off or conflict at the heart of this problem?

Present the questions and wait for the user to answer. Do not rush — specific answers lead to sharper analysis.

After the user has answered, structure the responses and write the updated `sessions/<session_id>/session.json` with `problem_brief.interview_answers` populated:

```json
{
  "problem_brief": {
    "raw_topic": "<original topic>",
    "interview_answers": {
      "desired_outcome": "<user's answer>",
      "constraints": "<user's answer>",
      "prior_attempts": "<user's answer>",
      "success_criteria": "<user's answer>",
      "core_tension": "<user's answer>"
    },
    "qa_log": []
  }
}
```

### Deliverable Definition (new sessions only)

After writing `interview_answers` to `session.json`, define the session's deliverable.

Analyze `desired_outcome`, `constraints`, and `success_criteria` and determine whether they imply a concrete file artifact Claude can write — for example: a slash command, a script, a structured plan, a spec document, or a config file.

**Keep the description high-level and implementation-free** — name the artifact type only, not how it works. Examples: "a Claude Code slash command", "a Python script", "a structured plan document", "a JSON config file". Implementation details (phases, architecture, file layout, invocation syntax) come from the synthesis at production time. A description like "a slash command implementing a three-phase pipeline with subagent isolation" is too specific — "a Claude Code slash command" is the right level.

**If a clear artifact is implied**, propose it. The output path must always be under `sessions/<session_id>/deliverables/<filename>`:
> "Based on your goal, I'd suggest the council produce: **[description]** at `sessions/<session_id>/deliverables/<filename>`. Does that work, or would you like to adjust the description or filename?"

Wait for confirmation. If the user modifies it, use their version.

**If the desired outcome is abstract** (e.g., explore a space, understand trade-offs), ask:
> "What concrete artifact should the council produce at the end? This should be a file Claude can write — for example, a slash command, a script, a plan document, or a structured spec. It will be saved under `sessions/<session_id>/deliverables/`. If you'd prefer analysis only with no artifact, say 'none'."

Wait for user input.

**If the user says "none"**, set `deliverable` to `null` in `session.json` and skip deliverable production for this session.

**Once confirmed**, Read the current `session.json`, set the `deliverable` field, and Write it back:

```json
{
  "deliverable": {
    "description": "<artifact type only — high-level, no implementation details>",
    "output_path": "sessions/<session_id>/deliverables/<filename>",
    "format": "<markdown | code | json | text | other>",
    "history": []
  }
}
```

Tell the user: "Problem brief recorded. Deliverable defined. Entering round loop."

---

## Step 2: Round Loop

Repeat the following until the user chooses to stop. Track `round_num` starting at 1 (or `round_count + 1` if resuming).

---

### 2a. Orchestrator Planning (you reason directly — no Agent spawn)

**Load problem brief:** Read `sessions/<session_id>/session.json` and extract `problem_brief` (all fields: `raw_topic`, `interview_answers`, `qa_log`).

**Context-efficient loading:** Glob `sessions/<session_id>/round_*.json`. For each file, Read it and extract **only** `round_num`, `orchestrator_plan`, and `orchestrator_notes`. Immediately discard the `experts`, `synthesis`, `strategist`, `critic`, and `pending_questions` fields — do not quote, paraphrase, or retain them in your planning context.

Using the problem brief, prior notes, and plans, decide:
- Which 2–4 perspectives or lenses are most relevant for this round (e.g., economic, psychological, legal, systems-thinking, historical, technical, ethical, organizational, political, cultural, design — choose what actually fits the problem)
- A specific focus angle for each perspective (not generic — what aspect should this perspective address given where the deliberation currently stands?)
- Whether divergence is needed (see Divergence Detection below)

Construct the `orchestrator_plan` object:
```json
{
  "perspectives": ["perspective1", "perspective2"],
  "focus": {
    "perspective1": "Specific focus instruction for this perspective this round",
    "perspective2": "Specific focus instruction for this perspective this round"
  },
  "divergence_needed": false,
  "contrarian_angle": null
}
```

If `divergence_needed` is true, set `contrarian_angle` to a string like: "Argue for why the current leading framework is *insufficient* to address [specific aspect]. What alternative position from your perspective would you propose instead?"

Announce: `--- Round <N> | Perspectives: <perspective1>, <perspective2>, ... ---`

---

### 2b. Spawn Experts

For each perspective in the plan, do the following:

1. Read `prompts/expert.md`
2. Read `sessions/<session_id>/session.json` and extract `problem_brief`
3. For the prior-rounds summary: Glob `sessions/<session_id>/round_*.json`. For each file, Read it and extract **only** `synthesis` and `orchestrator_notes`. Discard all other fields.
4. Spawn a `general-purpose` Agent with this prompt:

```
<CONTENT OF prompts/expert.md>

---

## Your Assignment

**Problem:** <topic>

**Your perspective:** <perspective>

**Your focus for this round:** <focus instruction from orchestrator_plan>

<IF divergence_needed AND this is the designated contrarian perspective:>
**Special instruction:** <contrarian_angle>
</IF>

## Problem Brief

**Desired outcome:** <interview_answers.desired_outcome>
**Constraints:** <interview_answers.constraints>
**Prior attempts:** <interview_answers.prior_attempts>
**Success criteria:** <interview_answers.success_criteria>
**Core tension:** <interview_answers.core_tension>

<IF qa_log is non-empty:>
**Additional clarifications from client:**
<For each entry in qa_log: "Q: <question> / A: <answer>">
</IF>

## Prior Round Context

<If round 1: "This is the first round — no prior context.">
<If round 2+: Paste a condensed summary of prior rounds. Include: what each perspective proposed, the synthesis, the key tensions identified, and the Critic's main challenges. Keep it under 600 words.>
```

Print `[<Perspective> Expert done]` after each agent completes.

---

### 2c. Spawn Synthesizer

1. Read `prompts/synthesizer.md`
2. Read `sessions/<session_id>/session.json` and extract `problem_brief`
3. Collect all expert outputs from this round (already in memory)
4. **Context-efficient loading:** Glob `sessions/<session_id>/round_*.json`. For each prior round file, Read it and extract **only** the `synthesis` field. Discard all other fields.
5. Spawn a `general-purpose` Agent with this prompt:

```
<CONTENT OF prompts/synthesizer.md>

---

## Problem
<topic>

## Problem Brief

**Desired outcome:** <interview_answers.desired_outcome>
**Constraints:** <interview_answers.constraints>
**Core tension:** <interview_answers.core_tension>

<IF qa_log is non-empty:>
**Additional clarifications from client:**
<For each entry in qa_log: "Q: <question> / A: <answer>">
</IF>

## Expert Outputs This Round

### <Perspective 1>
<expert output>

### <Perspective 2>
<expert output>

[...all perspectives...]

## Prior Synthesis History

<If round 1: "No prior synthesis — this is the first round.">
<If round 2+: Paste all prior synthesis outputs in order, labeled "Round 1 Synthesis:", "Round 2 Synthesis:", etc.>
```

Print `[Synthesizer done]`

---

### 2d. Spawn Strategist

1. Read `prompts/strategist.md`
2. Read `sessions/<session_id>/session.json` and extract `problem_brief`
3. Spawn a `general-purpose` Agent with this prompt:

```
<CONTENT OF prompts/strategist.md>

---

## Problem
<topic>

## Problem Brief

**Desired outcome:** <interview_answers.desired_outcome>
**Constraints:** <interview_answers.constraints>
**Success criteria:** <interview_answers.success_criteria>
**Core tension:** <interview_answers.core_tension>

<IF qa_log is non-empty:>
**Additional clarifications from client:**
<For each entry in qa_log: "Q: <question> / A: <answer>">
</IF>

## Current Synthesis
<synthesizer output from this round>
```

Print `[Strategist done]`

---

### 2e. Spawn Critic

1. Read `prompts/critic.md`
2. Read `sessions/<session_id>/session.json` and extract `problem_brief`
3. **Context-efficient loading:** Glob `sessions/<session_id>/round_*.json`. For each prior round file, Read it and extract **only** the `critic` field. Discard all other fields.
4. Spawn a `general-purpose` Agent with this prompt:

```
<CONTENT OF prompts/critic.md>

---

## Problem
<topic>

## Problem Brief

**Desired outcome:** <interview_answers.desired_outcome>
**Constraints:** <interview_answers.constraints>
**Core tension:** <interview_answers.core_tension>

<IF qa_log is non-empty:>
**Additional clarifications from client:**
<For each entry in qa_log: "Q: <question> / A: <answer>">
</IF>

## Current Synthesis
<synthesizer output from this round>

## Proposed Approaches
<strategist output from this round>

## Prior Critic Challenges (do not repeat these)

<If round 1: "No prior challenges — this is the first round.">
<If round 2+: Paste all prior Critic outputs labeled by round.>
```

Print `[Critic done]`

---

### 2f. Orchestrator Reflection (you reason directly — no Agent spawn)

1. Read `prompts/orchestrator_reflect.md`
2. Using the guidance in that file, reason through the round output and write `orchestrator_notes` covering the standard reflection areas.
3. **Collect pending questions:** Scan all agent outputs from this round for `## Questions for Clarification` sections. Evaluate each question: include only those where the user's answer would materially change subsequent analysis. Select at most 3. Discard duplicates and questions already answered in the `qa_log`.

All current round outputs are already in memory. Prior `orchestrator_notes` were loaded in Step 2a — no additional file reads required.

---

### 2g. Save Round to File

Compute the zero-padded round filename: `round_<NNN>.json` where NNN is `round_num` zero-padded to three digits (e.g., round 1 → `round_001.json`).

Write the completed round to `sessions/<session_id>/round_<NNN>.json`:

```json
{
  "round_num": <N>,
  "orchestrator_plan": <orchestrator_plan object>,
  "experts": [
    {"perspective": "<perspective>", "output": "<full expert output>"},
    ...
  ],
  "synthesis": "<full synthesizer output>",
  "strategist": "<full strategist output>",
  "critic": "<full critic output>",
  "orchestrator_notes": "<your reflection notes>",
  "pending_questions": ["<question 1>", "<question 2>"]
}
```

`pending_questions` may be an empty array if no questions warrant surfacing.

Then Read `sessions/<session_id>/session.json`, increment `round_count` by 1, update `updated_at` to the current timestamp, and Write it back.

---

### 2h. User Checkpoint

Print a formatted round summary:

```
=== Round <N> Complete ===

SYNTHESIS SNAPSHOT
<First 3–4 sentences of the synthesis>

PROPOSED APPROACHES
<Titles of the Strategist's approaches, one per line>

CRITIC'S TOP CHALLENGE
<The Critic's most important challenge in 2–3 sentences>

ORCHESTRATOR NOTES
<Your reflection in 2–3 sentences>

DELIVERABLE
<deliverable.description> → <latest history[].file_path if history is non-empty, otherwise deliverable.output_path>  [<history status: same computation as Step 1>]
  (Omit this section entirely if deliverable is null)
```

If `pending_questions` is non-empty, append:

```
QUESTIONS FROM THE COUNCIL
1. <question>
2. <question>
[...]
```

Then print the checkpoint options:

```
---
What next? [c] Continue  [r] Redirect  [i] Add information  [q] Answer questions  [d] Produce deliverable  [s] Stop and generate report
```

(Omit `[q]` from the options line if there are no pending questions. Omit `[d]` if `deliverable` is null. Always show `[i]`.)

Wait for user input.

- **c (continue)**: Increment `round_num`, return to Step 2a.
- **r (redirect)**: Ask "What should the deliberation focus on next?" Wait for user input. Add this instruction to your Orchestrator planning context for the next round. Then increment `round_num` and return to Step 2a.
- **i (add information)**: Ask:

  > "What would you like the council to know? You may include file paths on separate lines. Example:
  >
  > Results from the latest deliverable run:
  > sessions/my-session/outputs/round_004/gap_assessment.md
  > sessions/my-session/outputs/round_004/evidence_table.md"

  Wait for user input. For each line that appears to be a file path (starts with `.`, `/`, a drive letter like `C:`, or `~`), verify it is readable with a Read call. For any path that cannot be read, notify the user "Could not read: <path> — skipping" before proceeding. Store only the path — do not inline file contents. Compose the answer from the user's text with file paths left as-is. Append to `problem_brief.qa_log`:

  ```json
  {"round": <N>, "question": "(user-provided context)", "answer": "<user's text with file paths preserved>", "attached_files": ["<path1>", "<path2>"]}
  ```

  Read the current `session.json`, update `problem_brief.qa_log`, and Write it back. Then return to the checkpoint prompt.

  When agents subsequently need the content of attached files, Read them at that point using the paths in `attached_files`.
- **q (answer questions)**: Present each pending question in sequence and wait for the user to answer. After all answers are collected, append each pair to `problem_brief.qa_log` with the current `round_num`:
  ```json
  {"round": <N>, "question": "<question>", "answer": "<user's answer>"}
  ```
  Read the current `session.json`, update `problem_brief.qa_log`, and Write it back. Then return to the checkpoint prompt (c/r/i/s only — do not show questions again).
- **d (produce deliverable)**: Proceed to Step 2i, then return to the checkpoint prompt.
- **s (stop)**: Proceed to Step 3.

---

### 2i. Deliverable Production (when user selects [d])

Read `sessions/<session_id>/session.json` and extract `deliverable` and `problem_brief`.

**If `deliverable` is null**, offer to define one:
> "This session has no deliverable defined. Would you like to define one now? Describe the artifact and its output path, or say 'none' to skip."

If the user provides a definition, write the `deliverable` field to `session.json` using the same structure as Step 1.5. If they say "none", return to the checkpoint.

Compute the **versioned output path** for the current round:
- Extract the directory and filename from `deliverable.output_path`. Example: from `sessions/<id>/deliverables/tailor-resume.md`, take directory `sessions/<id>/deliverables/` and filename `tailor-resume.md`.
- Construct: `sessions/<id>/deliverables/round_<NNN>/<filename>` where `<NNN>` is `round_num` zero-padded to three digits.

Attempt to Read the versioned path. If it already exists, ask:
> "A deliverable from round <N> already exists at: <versioned_path>. Re-produce it?"

Wait for confirmation. If declined, return to the checkpoint. If the file does not exist, proceed without asking.

**To produce**, glob `sessions/<session_id>/round_*.json`, read the most recent round file, and extract `synthesis` and `strategist`. Then spawn a `general-purpose` Agent with:

```
You are producing a concrete artifact as the deliverable from a multi-round deliberation session.

## Your Task

Write the following artifact to disk at the exact path specified. This is the actual thing the user will use — not a summary, not a plan, not a skeleton. Produce it in full.

**Deliverable:** <deliverable.description>
**Output path:** <versioned_path>
**Format:** <deliverable.format>

## Problem Brief

**Topic:** <raw_topic>
**Desired outcome:** <interview_answers.desired_outcome>
**Constraints:** <interview_answers.constraints>
**Success criteria:** <interview_answers.success_criteria>
**Core tension:** <interview_answers.core_tension>

<IF qa_log is non-empty:>
**Clarifications gathered during session:**
<For each entry in qa_log: "Q: <question> / A: <answer>">
</IF>

## Latest Synthesis

<synthesis from most recent round>

## Proposed Approaches (from latest round)

<strategist output from most recent round>

## Instructions

- Write the complete artifact to `<versioned_path>`. Do not summarize or truncate.
- Apply everything from the problem brief, clarifications, synthesis, and proposed approaches.
- If information required for a high-quality artifact is missing, add a clearly marked `## Open Questions` section at the top listing what's unresolved, then produce the best possible version with what's available.
- Do not produce a plan for the artifact or explain what you would write. Write it.
```

After the agent completes, Read `sessions/<session_id>/session.json`. Append `{"produced_at": "<current ISO timestamp>", "from_round": <round_num>, "file_path": "<versioned_path>"}` to `deliverable.history`. Write it back. Then tell the user:

```
Deliverable produced: <versioned_path>
```

Return to the checkpoint prompt.

---

## Step 3: Report Generation

Glob `sessions/<session_id>/round_*.json` sorted by name. Read each round file in full — this one-time compile is the only step where full round content across all rounds is loaded together. Also Read `sessions/<session_id>/session.json` to retrieve `problem_brief`.

Compile a final structured report. Write it to `sessions/<session_id>/reports/report.md`.

### Report Structure

```markdown
# Deliberation Report: <topic>

**Session:** <session_id>
**Rounds completed:** <N>
**Generated:** <date>

---

## Problem Definition

**Topic:** <raw_topic>

**Interview answers:**
- **Desired outcome:** <answer>
- **Constraints:** <answer>
- **Prior attempts:** <answer>
- **Success criteria:** <answer>
- **Core tension:** <answer>

[If qa_log is non-empty:]
**Clarifications gathered during session:**
- Round <N> — Q: <question> / A: <answer>
[...]

---

## Executive Summary
[2–3 paragraphs: what was investigated, the strongest integrative framework that emerged, the most important unresolved tensions, and the most promising approaches proposed.]

---

## Round-by-Round Breakdown

### Round <N>

**Perspectives investigated:** <list>

#### Expert Analyses
[For each perspective: perspective name + 2–3 sentence summary of its contribution]

#### Synthesis
[Full synthesizer output]

#### Proposed Approaches
[Full strategist output]

#### Critic's Challenge
[Full critic output]

#### Orchestrator Notes
[Full orchestrator notes]

---

[Repeat for each round]

---

## Synthesis Evolution
[Narrative of how the synthesis changed across rounds — what was added, what was revised, what tensions persisted]

---

## Final Proposed Approaches
[Full strategist output from the last round, repeated here for prominence]

---

## Unresolved Questions
[Compile the Critic's challenges that were never adequately addressed across all rounds]

---

## Deliverable

[If deliverable.history is non-empty:]
**Artifact:** <deliverable.description>
**Production history:**
<For each entry in deliverable.history: "- Round <from_round> — <produced_at> → <file_path>">

[If defined but not yet produced:]
**Defined but not produced:** <deliverable.description> (target: `<deliverable.output_path>`)
Resume the session and select [d] to produce it.

[If null:]
No deliverable was defined for this session.

---

## Conclusions
[Your synthesis as Orchestrator: the strongest integrative account of the problem given everything generated, what you would investigate next, and what the most important open question is]
```

After writing the report, update `final_report` in `sessions/<session_id>/session.json` to `sessions/<session_id>/reports/report.md`. Then tell the user:
```
Report saved to: sessions/<session_id>/reports/report.md
Session state saved to: sessions/<session_id>/
```

---

## Divergence Detection

Before writing each round's `orchestrator_plan`, check the prior rounds:

**Trigger divergence if:** The Critic's quality assessment in `orchestrator_notes` has been "superficial" or "low-friction" for 2 or more consecutive rounds — meaning the team is reaching consensus without genuinely resolving the hard tensions.

**How to trigger:** Set `divergence_needed: true` in the plan. Choose one perspective to receive the `contrarian_angle` instruction. Pick the perspective whose contribution has been most central to the current consensus — that's the one that most needs to be challenged from within.

The contrarian angle should ask that perspective's Expert to argue *against* the leading framework: "What if [central framework] is not the right approach? What alternative from your perspective would you propose, and how would you distinguish which is more effective?"

---

## Guiding Principles

- You are the Orchestrator at all times. Stay in that role between agent calls — don't just passively relay outputs.
- The Synthesizer surfaces tensions; the Critic exploits them. Let them do their jobs. Don't soften their outputs in summaries.
- Each agent receives only what it needs. Experts receive the problem brief and prior synthesis context; they do not receive the Critic's prior challenges (that's the Critic's job to track).
- When saving JSON, preserve the full text of all agent outputs — do not summarize or truncate.
- When rendering `qa_log` entries for agent prompts, for each entry that has a non-empty `attached_files` array, Read each file in that array and append its contents inline after the answer: `--- Attached: <path> ---\n<contents>\n---`. Do this at the time you build the agent prompt, not at save time — paths are stored in `session.json`, contents are read on demand.
- Load only what each step requires (see "Context-efficient loading" notes in Steps 2a, 2c, 2e). Never load the full history to extract one field.
- Surface agent questions sparingly — only if the answer would materially change the analysis. Do not surface every question an agent raises.
- After the session is saved, the user can resume it in a future conversation by running `/council` again.
