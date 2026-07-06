---
name: chore-cycler
description: Completion-aware chore recurrence: when a tagged chore is completed, schedules the next one for completion date + its interval.
---

You give The user's chores COMPLETION-RELATIVE recurrence (FlowSavvy's native recurrence is due-date-relative, which is wrong for chores). When he completes a tagged chore, you create the next occurrence due = his completion date + the chore's interval. Use the FlowSavvy connector tools mcp__<FLOWSAVVY_CONNECTOR>__* (list_items, create_task, recalculate).

TIMEZONE: America/Los_Angeles.

HOW CHORES ARE TAGGED: each chore task has a tag in its notes of the form [cycle:Nd] where N is the interval in days (e.g. [cycle:10d] = repeat 10 days after completion). Only tasks with this tag are managed here.

STATE FILE: ~/.claude/scheduled-tasks/chore-cycler/state.json — {"lastRunUtc":"<iso>","processedIds":[]}. Create with lastRunUtc = 24h ago and empty processedIds if missing.

STEPS
1. Load state.
2. Find newly-completed chores: list_items itemType=task completed=true modifiedAfter=<state.lastRunUtc>. Keep only items whose notes contain "[cycle:" AND whose id is NOT in state.processedIds.
3. For each such completed chore:
   a. Parse N from the [cycle:Nd] tag in its notes.
   b. Completion date = the date of its lastModified (Pacific).
   c. nextDue = completion date + N days. Keep the same time-of-day as the completed task's original dueDateTime if present, else 20:00 (use 17:00 for car wash / oil change).
   d. canBeStartedAt = nextDue minus min(3, N-1) days (the lead window each cycle).
   e. create_task a NEW single (non-recurring) task: same title, same durationMinutes and minLengthMinutes, same listId, same priority, same schedulingHoursId (if the completed task had a non-default one), isAutoIgnored=false, notes carrying the SAME [cycle:Nd] tag, dueDateTime=nextDue, canBeStartedAt as computed. Do NOT set recurrenceRule.
   f. Add the completed task's id to state.processedIds.
4. If you created any tasks, call recalculate.
5. Save state: lastRunUtc = now (UTC), processedIds updated (keep only the most recent ~300 ids to bound size).

NOTES
- Never create a duplicate: the processedIds list ensures each completion spawns exactly one next occurrence.
- If a completed chore somehow has no [cycle:Nd] tag, skip it (it's a normal task, not a managed chore).
- This is how new chores get added too: any FlowSavvy task whose notes contain a [cycle:Nd] tag will automatically cycle once completed.