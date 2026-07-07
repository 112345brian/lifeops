# Life-Ops

A personal operations layer that turns an auto-scheduling calendar (FlowSavvy) into a system that doesn't just *schedule* your life — it makes the judgment calls about **what** should happen and **how much**, across every domain at once.

It started as "should I go to the gym today?" and grew into a set of small scheduled agents that reason across **body, school, chores, food, relationships, and money** — together.

> This repo is a **sanitized template**. All personal identifiers (calendar IDs, ntfy topics, connector IDs, names, paths) are replaced with `<PLACEHOLDERS>`. Fill them in with your own to run it.

## The core idea

Four principles hold the whole thing together:

- **FlowSavvy is the engine, the agents are the judgment.** FlowSavvy is great at *packing* tasks into open time. It can't decide whether the gym should yield to a deadline, whether you slept badly, or whether you're overdue to see your partner. The agents make those calls and express them as FlowSavvy tasks; FlowSavvy then places them — around your real coursework, work hours, and sleep.
- **ntfy is the nervous system.** Your phone's signals come *in* over a private ntfy topic (gym check-ins, sleep, manual triggers); the system's alerts go *out* over a separate topic.
- **Completion is truth.** Nothing is assumed done until it's confirmed — a gym session via a phone geofence, a chore by you checking it off, a transaction by you approving it.
- **Cross-domain reasoning is the point.** A late show shortens tomorrow's gym; a deadline crunch yields the gym *and* warns your wallet; a date is both protected time *and* budgeted dollars. No single app does this because no app sees all of it.

## Architecture

```
   PHONE (MacroDroid)                 SCHEDULED AGENTS                 ENGINES
   ─────────────────                  ────────────────                ───────
   24GO app opens ──┐                 gym-planner        ┐
   sleep sensor  ───┼─► ntfy signal ─► gym-confirmer     ├─► FlowSavvy (schedules
   "catchup" text ──┘     topic        homework-watcher  │     everything together)
                                       meal-prep         │
   Canvas ───────────► (FlowSavvy      social-balance    ├─► YNAB (money: import,
   Google Calendars ──► native sync)   chore-cycler      │     categorize, approve,
                                       catchup-replan    │     earmark)
                                       ynab-categorizer  │
   agents ──► ntfy alerts topic ──►    spend-forecaster  ┘
   (to your phone)
```

## The agents

Each agent is a cron-scheduled prompt (one folder = one `SKILL.md`).

| Agent | When | What it does |
|---|---|---|
| **gym-planner** | daily AM | Decides gym intensity from sleep quality, recovery (late nights on social calendars), and deadline load; creates a FlowSavvy gym task (or skips). |
| **gym-confirmer** | daily 2 AM | Reads the ntfy geofence check-in; marks the gym task complete if you went, removes it if you didn't (keeps the weekly count honest). |
| **homework-planner** | daily AM | A *load-watcher* — FlowSavvy already schedules Canvas coursework natively, so this only flags at-risk deadlines, bumps task priority, and alerts. |
| **meal-prep-checkin** | Wed + Sat | When prep is due, creates `Groceries` + `Meal prep` tasks with the cook *blocked by* the shopping. |
| **social-balance** | daily PM | Protects a weekly partner-time and friend-time block; nudges (gently) only when overdue. Recency counts only *confirmed* time. |
| **chore-cycler** | daily | Gives chores **completion-relative** recurrence (see below). |
| **catchup-replan** | every 20 min | Idle until you text `catchup`; then re-prioritizes the whole week in FlowSavvy. The panic button. |
| **ynab-categorizer** | daily AM | YNAB autopilot: import bank-export files → categorize (confidence-gated) → auto-approve the confident ones → cover overspent categories from a buffer. |
| **spend-forecaster** | daily AM | Looks ahead at shows/parties/dates/hangouts, learns your real per-outing cost from history, earmarks budget toward them, and bluntly warns when fun money is short. |
| **birthday-prep** | *disabled* | Gift-prep lead time + day-of reminders. Shelved. |

## Key conventions

### `[cycle:Nd]` — completion-relative recurrence
FlowSavvy's native recurrence is **due-date-relative** (next occurrence is a fixed interval from the *due date*), which is wrong for chores — do laundry 4 days late and you don't want it due again in 3 days. So chores/appointments/etc. are **single** tasks with a tag like `[cycle:10d]` in their notes. The `chore-cycler` agent watches for completions and, when you finish one, creates the next occurrence due = **completion date + N days**. Tag *any* task with `[cycle:Nd]` and it auto-cycles.

### ntfy vocabulary
Signals you send (phone → system, on the signal topic):

| Message | Meaning |
|---|---|
| `gym` | gym check-in (auto-sent by MacroDroid when the gym app opens) |
| `fell-asleep` / `woke-up` | sleep tracking (auto-sent by a phone sleep automation) |
| `catchup` | re-plan my whole week now |

Alerts come back to you on a **separate** topic (subscribe to it in the ntfy app).

### Confidence-gated categorization (YNAB)
Transactions are categorized only above a confidence bar: HIGH (a repeat payee in your history) auto-assigns; MEDIUM gets a sanity double-check (a $0.50 charge won't be tagged "Groceries"); LOW is left for you. An uncategorized transaction always beats a wrong one.

### Action log — every mutation, in one place
Every agent appends one line to a shared, append-only log — `~/.claude/scheduled-tasks/action-log.jsonl` — immediately after any call that creates, updates, deletes, or completes a FlowSavvy item, a Google Calendar event, or a YNAB transaction/category. This is what makes "why does this task exist" or "what deleted that event" answerable later instead of guesswork.

Format (one JSON object per line, create the file/dir if missing):
```
{"ts":"<UTC ISO8601>","skill":"<agent-name>","action":"create_task|update_task|delete_item|complete_task|create_event|delete_event|ynab_categorize|ynab_approve|ynab_cover","id":"<item id, if any>","title":"<title or payee>","detail":"<one-line why>"}
```
Written via Bash, e.g.:
```
mkdir -p ~/.claude/scheduled-tasks && printf '%s\n' "{\"ts\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"skill\":\"chore-cycler\",\"action\":\"create_task\",\"id\":\"19746159\",\"title\":\"Car wash\",\"detail\":\"next occurrence after completion\"}" >> ~/.claude/scheduled-tasks/action-log.jsonl
```
To debug: `tail -n 200 ~/.claude/scheduled-tasks/action-log.jsonl | jq .` or `grep -i '<title>' ~/.claude/scheduled-tasks/action-log.jsonl`. If something exists that no line in this log explains, it wasn't created by one of these agents.

## External pieces & setup

To run this you need:

- **FlowSavvy** with its connector enabled, and the connector tool IDs (`<FLOWSAVVY_CONNECTOR>`), list IDs, and scheduling-hours IDs filled into each agent.
- **Google Calendar** connector (`<GCAL_CONNECTOR>`) and your calendar IDs (`<CALENDAR_ID>`, `<PRIMARY_CALENDAR_ID>`) for reading deadlines, social/late-night events, and partner time.
- **ntfy** (free) — pick two unguessable topic names for `<NTFY_SIGNAL_TOPIC>` and `<NTFY_ALERTS_TOPIC>`; subscribe to the alerts topic on your phone.
- **MacroDroid** (Android) on the phone to POST signals to the signal topic: an *Application Launched* trigger on your gym app → `gym`; sleep-sensor automations → `fell-asleep` / `woke-up`.
- **YNAB** with a personal access token saved to a local file **outside this repo** (never commit it). The autopilot reads it at runtime.
- A permission allow-list (in the harness `settings.json`) so the scheduled agents don't prompt on every run: the connector tools and `Bash(curl/cat/mkdir/mv)`.

## Money safety boundary

Everything money-related stays **inside YNAB**, which is a tracker, not a bank. The agents import, categorize, approve, and reallocate *budgeted* dollars between categories. They will **never** initiate a real bank transfer, send money, or move funds between actual accounts. Budget reallocation suggestions are arithmetic on your own spending — not investment advice.

## Repo layout

```
<agent-name>/SKILL.md     # one folder per agent; the SKILL.md is its prompt
spend-forecaster/config.json   # per-outing cost baselines + options
```

State files (`state.json`), tokens, and anything personal live **outside** version control.

---

*Built collaboratively, one feature at a time. The fun part was watching a gym-yes/no question turn into a thing that quietly runs a life.*
