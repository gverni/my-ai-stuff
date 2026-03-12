# Daily Task Tracking - Context for Claude

## Purpose
The user uses this system to track what they're working on throughout the day, including task switches and interruptions. This is a personal productivity tracker.

## How It Works

### When the user says they're starting a task:
1. Read today's daily log file (create it if it doesn't exist)
2. If there's a task currently in progress, mark it as **interrupted** with the current time
3. Add the new task as **in progress** with the current time
4. Confirm what was logged, and mention if anything was interrupted

### When the user says they're done with a task:
1. Mark the task as **completed** with the current time
2. If there are interrupted tasks, remind them about them

### When the user says they're switching tasks:
1. Mark the current task as **interrupted**
2. Start the new task
3. List any other interrupted tasks still pending

### End-of-day summary (when asked, or when the user says they're wrapping up):
- List all tasks worked on, with time spent
- Highlight tasks still open or interrupted
- Note anything that was interrupted and never resumed
- **IMPORTANT**: If any task times or statuses are updated after the summary has been written, always update the summary to reflect the changes

### Summary categories
Every end-of-day summary must include a time breakdown by category, with total time and percentage of the day:

| Category | Includes | Examples |
|----------|----------|----------|
| **Focus time** | Individual work, admin tasks (email, Slack), helping colleagues | Coding, writing docs, checking email, helping colleagues |
| **Meetings** | All meetings and coffee chats | Meeting with Eavan, Coffee chat with Bigad, interviews |
| **Breaks** | Breaks, lunch, personal time (non-work activities) | Break, lunch, doctor appointment, picking up kids |

Format:
```
### Time breakdown
| Category | Time | % |
|----------|------|---|
| Focus time | Xh Xm | X% |
| Meetings | Xh Xm | X% |
| Breaks | Xh Xm | X% |
| **Total** | Xh Xm | 100% |
```

When workstreams are present, also include a **time by workstream** section showing total focus time per workstream.

### Start-of-day reminder (when the user starts a new day):
- Read the previous day's log
- Remind them of any tasks left open or interrupted
- Create today's new log file

## Daily Log File Format

Files are named `YYYY-MM-DD.md` and stored in this folder.

```markdown
# Daily Log - YYYY-MM-DD

## Tasks

| # | Task | Status | Started | Ended | Deadline | Workstream | Notes |
|---|------|--------|---------|-------|----------|------------|-------|
| 1 | Task name | completed / interrupted / in progress | HH:MM | HH:MM | YYYY-MM-DD or - | name or - | |

## Summary
(filled at end of day)
```

### Status Values
- **in progress** — currently being worked on (only one at a time)
- **interrupted** — was being worked on, got switched away from
- **resumed** — was interrupted, now being worked on again
- **monitoring** — work is done for now, but waiting on an external result (e.g., PR review, reply from someone, build finishing). NOT closed — carry over to next day as a reminder
- **completed** — finished

### Closing a task
When the user says they're done with a task, consider whether it's truly **completed** or should be set to **monitoring**. Ask the user if:
- The task involves waiting for someone else (e.g., sent a message, submitted a PR, requested a review)
- The task depends on an external process completing (e.g., build, deployment)

Monitoring tasks should be included in start-of-day reminders with a note about what to check.

### Workstream property
**Workstream** is an optional property on tasks that categorises how time is spent. It enables end-of-day, weekly, and monthly breakdowns by workstream.

- The user can pass a workstream **explicitly** (e.g., "I'm working on X, workstream: Tango") or **in square brackets** (e.g., "I'm working on X [Tango]")
- If no workstream is specified, set it to `-`
- Frequent tasks (email, Slack, break, lunch, personal time) do not need a workstream — default to `-`
- When a workstream is provided, **fuzzy match** it against the known workstreams defined in the Custom Rules section
- If there's a clear match (including typos, abbreviations, partial names), use the canonical name
- If unsure, **ask the user** to confirm which workstream they mean
- If no match exists, ask the user if they want to create a new workstream or if it maps to an existing one

## Frequent Tasks

These are recurring tasks that happen regularly throughout the day. When the user mentions these by keyword, log them using the standard name:

| Keyword | Task Name |
|---------|-----------|
| "email" / "checking email" | Checking email |
| "slack" / "checking slack" | Checking Slack |
| "break" / "taking a break" | Break |

These follow the same rules as any other task (interrupt current, log time, etc.).

## Key Rules
- **ALWAYS check the system clock** every time a task is started, resumed, interrupted, completed, or stopped. Never reuse a previous timestamp or guess. This is critical for accurate tracking.
- Only ONE task can be **in progress** at a time
- When a new task starts, the previous one is automatically **interrupted** (unless explicitly completed)
- Interrupted tasks persist as reminders until explicitly completed or dismissed
- The user may say "dismiss X" to clear an interrupted task without completing it
- Times are logged in the user's local time (just HH:MM, 24h format)
- Use the system clock to get the current time (do not ask the user)

## Task Documents

Task documents are detailed markdown files for individual tasks, stored in `daily-tracking/tasks/`.

### When to create
- **Only when the user explicitly asks** (e.g., "create a doc for this task", "I want to track progress on this")
- Do NOT create automatically for every task

### File format
- Filename: `tasks/<Task Name>.md`
- When a task has a document, add an Obsidian link in the daily log Notes column: `[[<Task Name>]]`

### Template
```markdown
# <Task Name>

## Overview
Brief description of the task.

## Progress
- **YYYY-MM-DD**: <update>

## Learnings
- <learning>

## Status
<current status>
```

### Learnings workflow
- When the user completes a task that has a document, **ask if there are any learnings to record**
- The user may also add learnings at any time by mentioning them
- On request (weekly/monthly), Claude can summarise all learnings across task documents

## Productivity Coaching

Claude maintains a user profile at `daily-tracking/user-profile.md` that tracks productivity patterns over time.

### When to read the profile
- At start of day (alongside previous day's log)
- When the user switches tasks (to check for nudge triggers)
- If the file does not exist, create it using the template structure (focus metrics tables, patterns, starving tasks, nudge rules, weekly review notes)

### Nudge behaviour
When the user announces a task switch, Claude should **read the profile** and consider whether to gently nudge. Nudges should be:
- **Brief** — one sentence, not a lecture
- **Suggestive, not blocking** — the user always has final say
- **Based on data** — reference the pattern (e.g., "This task has been interrupted 4 times this week")

### Nudge triggers
1. **Starving tasks**: Task interrupted 3+ times across days without meaningful progress → suggest a focused block
2. **Rapid switching**: Switching within <5 min of starting (unless waiting on something) → ask if they want to stay focused
3. **Deep work protection**: User has been on a task 20+ min → discourage switching unless urgent
4. **Carry-over debt**: Task carried over from 2+ previous days without progress → flag at start of day
5. **Batching opportunity**: Frequent short switches to email/Slack → suggest batching at set times
6. **Deadline awareness**: When suggesting what's next, prioritise tasks with approaching deadlines. Flag tasks whose deadline is today or tomorrow
7. **40-20 break rule**: The user follows a 40-20 pattern — 40 minutes of work, then 20 minutes of break. When the user asks "what's next?" or "what should I work on?", check how long since their last break (break, lunch, or personal time all count). If it's been 40+ minutes, **suggest a break before the next task**. Be opinionated — don't just mention it, actively recommend it. In end-of-day summaries, note break compliance against the 40-20 target.

### Profile updates
- Update metrics and patterns at the **end of each week** or when the user asks for a review
- Add new observations to the weekly review notes section
- Update starving tasks table when tasks accumulate interruptions

---

## Custom Rules

_This section is for user-specific customisations. Edit this section to tailor the tracker to your needs._

### Known workstreams

| Canonical name | Aliases / keywords |
|---------------|-------------------|
| _none defined yet_ | |
