# Claude Council

An agentic deliberation system for multi-round, multi-perspective analysis of any problem, question, or challenge. Invoked via the `/council` skill in Claude Code, it orchestrates a team of specialized AI agents to iteratively develop, synthesize, and stress-test frameworks for addressing complex problems.

## What it does

Claude Council simulates an expert advisory team with four distinct roles:

| Agent | Role |
|---|---|
| **Expert** | Brings analytical depth from one specific perspective or lens |
| **Synthesizer** | Integrates expert analyses across perspectives, surfaces tensions |
| **Strategist** | Designs concrete, actionable approaches from the synthesis |
| **Critic** | Adversarially challenges assumptions, logic, and approach weaknesses |

The orchestrator (Claude itself) conducts an upfront interview to understand the problem, plans each round, selects relevant perspectives, directs focus angles, reflects on round quality, and detects convergence risk — intentionally introducing contrarian perspectives when the deliberation is settling too easily.

Between rounds, agents can surface clarifying questions. The orchestrator filters these and presents only the most critical ones; the user's answers update the problem brief for all subsequent rounds.

## Usage

From within this project directory in Claude Code:

```
/council
```

You'll be presented with a menu to either resume an existing session or start a new one.

**New sessions** begin with a brief interview (3–5 questions) to establish a problem brief before any analysis starts. After the interview, the orchestrator proposes a concrete deliverable artifact and confirms it with you.

**Per-round checkpoint options:**
- `c` — continue to next round
- `r` — redirect (add a focus instruction for the next round)
- `i` — add information (inject a context note into the problem brief; file paths on separate lines are read and inlined)
- `q` — answer questions surfaced by agents (shown only when pending questions exist)
- `d` — produce deliverable (shown only when a deliverable is defined for the session)
- `s` — stop and generate the final report

When resuming an existing session, you are also offered the `[i]` option before the first round begins — useful for feeding back results from a prior deliverable or attaching experiment output.

**Companion commands:**

| Command | What it does |
|---|---|
| `/council-report` | Generates a structured markdown report from any session with completed rounds |
| `/council-summary` | Prints an inline summary of a session or a single round (no files written) |
| `/council-deliverable` | Standalone deliverable producer — picks a session and writes the artifact to disk |

## Project structure

```
claude_council/
├── .claude/commands/
│   ├── council.md               # Main deliberation skill
│   ├── council-deliverable.md   # Standalone deliverable producer
│   ├── council-report.md        # Report generator for existing sessions
│   └── council-summary.md       # Inline session/round summary viewer
├── prompts/
│   ├── expert.md            # Perspective expert role prompt
│   ├── synthesizer.md       # Synthesizer role prompt
│   ├── strategist.md        # Approach strategist role prompt
│   ├── critic.md            # Adversarial critic role prompt
│   └── orchestrator_reflect.md  # Orchestrator reflection guidance
├── sessions/
│   └── YYYYMMDD_HHMMSS/     # One directory per session
│       ├── session.json     # Metadata: topic, timestamps, round count, problem brief
│       ├── round_001.json   # Full round output
│       └── round_NNN.json
└── reports/
    └── <session_id>_report.md   # Generated at session end
```

## Session lifecycle

### 1. Initialization
A session directory is created with a timestamp ID. `session.json` records the topic and a `problem_brief` structure that holds interview answers and any Q&A gathered during rounds.

### 2. Interview
The orchestrator asks 3–5 questions tailored to the topic (desired outcome, constraints, prior attempts, success criteria, core tension). Answers are stored in `session.json` and passed to all agents as context.

### 2.5. Deliverable definition
After the interview, the orchestrator analyzes the problem brief and proposes a concrete file artifact — a slash command, script, spec document, config file, or similar. After confirmation, the deliverable description and output path are stored in `session.json`. Say "none" to skip; a deliverable can also be defined or produced later via `[d]` at any checkpoint or by running `/council-deliverable`.

### 3. Round loop
Each round:
1. Orchestrator selects 2–4 perspectives and focus angles per perspective
2. One Expert agent spawned per perspective
3. Synthesizer integrates all expert outputs
4. Strategist proposes 3–5 actionable approaches
5. Critic challenges the synthesis and approaches
6. Orchestrator reflects, identifies priorities, and collects any agent questions worth surfacing
7. Round saved to `round_NNN.json`; user checkpoint presented

### 4. Agent questions and user context (optional)
If agents raised clarifying questions the orchestrator deems worth surfacing, they appear at the checkpoint. Choosing `[q]` lets the user answer them; answers are appended to `problem_brief.qa_log` and passed to all agents in subsequent rounds.

Choosing `[i]` at any checkpoint injects additional context — a note, new constraints, or real-world feedback from using a deliverable. File paths included on separate lines are read and inlined verbatim, so you can attach command output, experiment results, or any file the council should see.

### 5. Report generation
When the user stops, all round files are read and compiled into a structured markdown report covering:
- Problem definition (interview answers + Q&A log)
- Executive summary
- Round-by-round breakdown
- Synthesis evolution narrative
- Final proposed approaches
- Unresolved questions
- Orchestrator conclusions

## Key mechanisms

**Interview phase** — Before any agents are spawned, the orchestrator conducts a brief consultant-style interview. This grounds all analysis in the user's actual constraints, goals, and context rather than generic exploration of the topic.

**Agent-surfaced questions** — Any agent can flag clarifying questions when a missing piece of information would materially change their analysis. The orchestrator filters these and surfaces only the most valuable ones at the checkpoint. Answers are stored in the problem brief and injected into all future agent contexts.

**Divergence detection** — If the Critic's challenges are superficial for two or more consecutive rounds, the orchestrator flags `divergence_needed` and assigns a contrarian angle to one perspective in the next round. This breaks unexamined consensus before the deliberation settles.

**Context-efficient loading** — During live rounds, only metadata and summary fields from prior rounds are loaded per agent. Full round files are only read at report generation. This keeps each agent's context focused and avoids runaway token usage.

## Deliverable system

Each session can have one defined deliverable — a concrete file artifact Claude writes based on the deliberation. Examples: a `.claude/commands/` slash command, a shell script, a structured spec, a configuration file.

**Definition** — At the end of the interview, the orchestrator proposes an artifact and path, and confirms with you. This is stored in `session.json` as `deliverable.description` and `deliverable.output_path`. You can skip it, or define one later.

**Production** — Select `[d]` at any round checkpoint, or run `/council-deliverable` as a standalone command. The artifact is built from the most recent round's synthesis and strategist output, plus the full problem brief (interview answers + all Q&A). The result is written to the specified path.

**Status tracking** — Each production appends an entry to `deliverable.history` in `session.json`, recording the timestamp and the round it was based on. The session resume header shows the full count and most recent round (e.g., `produced 3× — latest from round 5`). The final report lists the complete history. You can re-produce the deliverable at any point; you'll be asked to confirm before overwriting.

## Requirements

- Claude Code
- No external libraries or configuration needed beyond this directory
