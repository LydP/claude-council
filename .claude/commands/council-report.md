# Council Report Generator

This skill generates a deliberation report for any existing council session without requiring you to run a new round.

## Directory Layout (relative to project root)
- `sessions/<session_id>/` — one directory per session
  - `session.json` — session metadata (topic, timestamps, round count, problem brief, final_report path)
  - `round_001.json`, `round_002.json`, … — one file per completed round
  - `reports/report.md` — final deliberation report

---

## Step 1: Select Session

Use Glob to list `sessions/*/session.json`. For each file found, Read it and extract `session_id`, `topic`, `updated_at`, `round_count`, and `final_report`.

If no session files are found, tell the user: "No sessions found. Run `/council` to start a session." Then stop.

Present to the user:

```
=== Council Report Generator ===

Sessions:
  [1] "<topic>" — <round_count> rounds — last updated <updated_at> [report exists]
  [2] "<topic>" — <round_count> rounds — last updated <updated_at>

Enter a number (or q to quit):
```

Omit `[report exists]` if `final_report` is null. Wait for user input. If the user enters `q`, stop.

If the selected session has `round_count` of 0, tell the user: "This session has no completed rounds — nothing to report on." Then stop.

---

## Step 2: Confirm Regeneration (if applicable)

If the selected session has a non-null `final_report` value, ask:

```
A report already exists at: <final_report path>
Regenerate? [y/n]
```

Wait for user input. If `n`, stop.

---

## Step 3: Generate Report

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

## Conclusions
[Your synthesis as Orchestrator: the strongest integrative account of the problem given everything generated, what you would investigate next, and what the most important open question is]
```

After writing the report, Read `sessions/<session_id>/session.json`, set `final_report` to `sessions/<session_id>/reports/report.md`, and Write it back. Then tell the user:

```
Report saved to: sessions/<session_id>/reports/report.md
Session state saved to: sessions/<session_id>/
```
