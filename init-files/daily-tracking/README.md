# Daily Task Tracker for Claude Code

A lightweight, conversational task and time tracker that runs inside [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Just tell Claude what you're working on, and it handles the rest — logging tasks, tracking interruptions, and giving you end-of-day summaries.

## How it works

Place the `CLAUDE.md` file in a folder (e.g., `daily-tracking/`). Claude will create daily log files, track your task switches, and build up productivity insights over time.

You talk to Claude in natural language. No commands to memorise.

## Quick start

1. Copy the `CLAUDE.md` file into a folder in your project
2. Create a `tasks/` subfolder (for optional task documents)
3. Start telling Claude what you're working on

## Usage examples

### Starting and switching tasks

```
You: I'm working on fixing the login bug
Claude: On **Fix login bug** (09:15). No previous task interrupted.

You: I need to switch to reviewing Sarah's PR
Claude: On **Review Sarah's PR** (09:42).
       Interrupted: Fix login bug (09:15–09:42)

You: Done with the review, what was I doing before?
Claude: Review completed (10:10). Before that you were working on
       **Fix login bug** (interrupted at 09:42). Want to pick it back up?
```

### Deadlines and pending tasks

```
You: Add a pending task: prepare slides for Friday's demo. Deadline is Friday.
Claude: Added. **Prepare slides for Friday's demo** — deadline 2026-03-14.

You: What should I work on next?
Claude: Your open items:
       Deadline Friday:
       1. Prepare slides for Friday's demo
       Pending:
       2. Fix login bug (interrupted earlier)
       3. Update API docs (day 3, starving)
```

### Monitoring tasks (waiting on others)

```
You: I submitted the PR for the bug fix. Done for now.
Claude: Do you want to mark this as completed or monitoring?
       Since you're waiting on PR review, monitoring might be better.

You: Yeah, monitoring
Claude: **Fix login bug** moved to monitoring — check PR status tomorrow.
```

### Workstreams

Tag tasks with a workstream to see where your time goes:

```
You: Working on the API refactor [backend]
Claude: On **API refactor** (14:00), workstream: Backend.

You: Switching to writing docs, workstream: documentation
Claude: On **Writing docs** (15:30), workstream: Documentation.
```

You can also use square brackets: `"fixing tests [backend]"` or explicit labels: `"workstream: backend"`. Claude fuzzy-matches against your known workstreams, so typos are handled.

### Breaks

```
You: Taking a break
Claude: On **Break** (10:30).

You: Back, picking up the login fix
Claude: Break done (10:52) — 22 min, on target.
       Back on **Fix login bug** (10:52).
```

The tracker monitors your break patterns against your target (default: 20 min per hour).

### End of day

```
You: Wrapping up for the day
Claude: Day closed at 17:30.

       Time breakdown:
       | Category   | Time    | %   |
       |------------|---------|-----|
       | Focus time | 4h 12m  | 53% |
       | Meetings   | 2h 30m  | 31% |
       | Breaks     | 1h 18m  | 16% |

       Carry over to tomorrow:
       1. Fix login bug (monitoring — check PR)
       2. Update API docs (day 3, starving)
```

### Start of day

```
You: Starting my day
Claude: Good morning. Carry-over from yesterday:
       Monitoring: Fix login bug — has the PR been approved?
       Pending: Update API docs (day 3)
       What do you want to pick up?
```

### Task documents

For tasks that need detailed tracking:

```
You: Create a document for the API refactor
Claude: Created tasks/API refactor.md with Overview, Progress,
       Learnings, and Status sections.
```

When you complete a task that has a document, Claude asks if you have any learnings to record. You can request weekly or monthly learnings summaries.

## Features

- **Single task focus** — only one task in progress at a time, others are interrupted
- **Automatic interruption tracking** — switching tasks logs what got interrupted
- **Monitoring status** — for tasks waiting on external input (PR reviews, replies, builds)
- **Deadlines** — optional deadlines with reminders when they're approaching
- **Workstreams** — optional categorisation for time-by-workstream breakdowns
- **Frequent tasks** — shortcuts for recurring activities (email, Slack, breaks)
- **Task documents** — detailed markdown files for complex tasks with learnings tracking
- **Productivity coaching** — nudges based on your patterns (starving tasks, rapid switching, break targets)
- **End-of-day summaries** — time breakdown by category and workstream
- **Start-of-day reminders** — carry-overs, monitoring checks, deadline alerts

## Customisation

The `CLAUDE.md` file has a **Custom Rules** section at the bottom where you can:

- Define known workstreams and aliases
- Add frequent tasks beyond the defaults
- Adjust any behaviour to your preferences

## Folder structure

```
daily-tracking/
├── CLAUDE.md              # Instructions for Claude (the tracker)
├── README.md              # This file
├── user-profile.md        # Productivity patterns (auto-generated)
├── tasks/                 # Task documents (created on request)
│   ├── Fix login bug.md
│   └── API refactor.md
├── 2026-03-09.md          # Daily logs
├── 2026-03-10.md
└── 2026-03-11.md
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- That's it. No dependencies, no setup, no database.
