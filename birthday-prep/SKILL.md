---
name: birthday-prep
description: Daily: reads a birthdays list and creates FlowSavvy gift-prep tasks ahead of time plus day-of reminders.
---

You keep The user ahead of the birthdays that matter to him. Each day you check his birthdays list and, at the right lead time, create FlowSavvy tasks so he never scrambles for a gift or forgets a day. Use the FlowSavvy connector tools mcp__<FLOWSAVVY_CONNECTOR>__* (create_task, list_items, recalculate). List: personal=6784.

TIMEZONE: America/Los_Angeles. Establish today's date.

BIRTHDAYS CONFIG: C:\Users\user\.claude\scheduled-tasks\birthday-prep\birthdays.json — a JSON object with a "people" array; each entry is {name, date:"MM-DD", giftLeadDays (default 14), needsGift (bool, default true)}. If the file is missing or "people" is empty, do nothing.

STATE FILE: C:\Users\user\.claude\scheduled-tasks\birthday-prep\state.json — {"created":["<name>-<year>-gift", "<name>-<year>-dayof", ...]}. Used to avoid creating the same task twice. Create empty if missing.

STEPS
1. Load birthdays.json and state.json. If no people, stop.
2. For each person, compute this year's birthday date (use next occurrence if it already passed this year — i.e., roll to next year so leads compute correctly).
3. GIFT TASK: if needsGift and today is on/after (birthday − giftLeadDays) and (birthday) is still in the future and "<name>-<year>-gift" not in state.created:
   - create_task title "Gift for <name> (birthday <MM-DD>)", listId 6784, durationMinutes 60, dueDateTime = birthday minus 3 days at 19:00, canBeStartedAt = today, priority normal, isAutoIgnored=false, notes "Birthday gift prep."
   - add "<name>-<year>-gift" to state.created.
4. DAY-OF REMINDER: if today == birthday and "<name>-<year>-dayof" not in state.created:
   - create_task title "🎂 <name>'s birthday", listId 6784, durationMinutes 10, dueDateTime = today 09:00, canBeStartedAt = today 00:00, priority high, isAutoIgnored=false, notes "Wish them happy birthday."
   - add "<name>-<year>-dayof" to state.created.
5. If you created anything, recalculate. Save state.json (prune entries older than ~2 years).