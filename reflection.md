# PawPal+ Project Reflection

## 1. System Design

### a. Initial design

My initial design had four main classes: `Owner`, `Pet`, `Task`, and `Scheduler`. The idea was to model real life as closely as possible without over-complicating things.

- **Owner** holds the owner's name and their daily time budget (`available_minutes`), plus a list of pets.
- **Pet** holds a pet's profile (name, species, age, breed, medical notes) and a list of tasks.
- **Task** represents one care activity. From the start I knew each task needed a duration, a priority level, and an optional preferred time so the scheduler could order them.
- **Scheduler** was responsible for building the daily plan given the owner's constraints.

I did not plan a `ScheduledTask` class at first — I originally thought the scheduler could just return a list of tasks directly.

### b. Design changes

Two changes happened during implementation:

1. **Added `ScheduledTask`** — When I started writing `build_daily_schedule`, I realized I needed to store more than just the task itself: I needed the calculated `start_time`, `end_time`, pet reference, and a reason string. Creating a separate `ScheduledTask` dataclass made the output much cleaner and easier to display in the Streamlit table.

2. **Stored the `Pet` object in `ScheduledTask` instead of just the pet's name** — Early on I stored `pet_name: str`. During UI wiring I needed to display the pet's name alongside the task. Storing the full `Pet` object made that access straightforward without any extra lookup.

3. **Added `clear_schedule()` as an explicit step** — The first version of `build_daily_schedule` did not clear previous results, which caused tasks to double up when the button was pressed more than once. Adding an explicit `clear_schedule()` call at the start of `build_daily_schedule` fixed this.

---

## 2. Scheduling Logic and Tradeoffs

### a. Constraints and priorities

The scheduler respects two hard constraints and one soft hint:

- **Hard constraint — time budget:** A task is only added to the schedule if `used_minutes + task.duration_minutes <= owner.available_minutes`. Tasks that would exceed the budget are silently skipped.
- **Hard constraint — task completion:** Only incomplete tasks (`task.completed == False`) are considered. Completed tasks are excluded by `collect_tasks()`.
- **Soft hint — preferred time:** The preferred time is used for sorting, not enforcement. A task with `preferred_time="09:00"` will appear before one set to `10:00`, but the scheduler does not guarantee it actually starts at that time — it starts as soon as the previous task ends.

Priority is the primary sort key. Within the same priority level, earlier preferred times break the tie.

### b. Tradeoffs

**Greedy ordering, no rescheduling:** Once the sorted list is built, the scheduler walks it in order and adds each task as long as it fits. It never backtracks to try a different combination that might fit more tasks. This means a single long high-priority task can push out several shorter lower-priority ones even if rearranging them would have fit everything. For a basic pet-care app, this tradeoff is acceptable — the owner gets the most important tasks first, which is the main goal.

**Exact-time conflict detection:** `conflicts_with()` returns `True` only when two tasks share the exact same `preferred_time` string. It does not check whether time ranges overlap. For example, a 60-minute task at `08:00` and a 10-minute task at `08:30` would not be flagged even though they overlap in reality. This keeps the conflict logic simple, which was the right call for this scope.

---

## 3. AI Collaboration

### a. How I used AI

I used two AI tools at different points:

- **GitHub Copilot** was open in the editor while I wrote code. It was most useful for filling in repetitive boilerplate — for example, completing the list comprehensions inside `remove_task` and `remove_pet`, and generating the `__str__`-style output in `explain_schedule`. I accepted those suggestions when they matched exactly what I had in mind.

- **Claude** was useful for talking through design decisions before writing code. I used it to think through whether `ScheduledTask` should be its own class and whether `Scheduler` should own the `scheduled_tasks` list or return a fresh one each time.

I kept AI tools more separate from the tests — I wrote those manually after the logic was working, so I could be sure I understood what each test was actually checking.

### b. Judgment and verification

Two cases where I did not accept a suggestion as-is:

1. **Rejected a more complex conflict check.** An AI suggestion replaced the simple `preferred_time == other.preferred_time` check with a full time-range overlap calculation using `datetime` objects. The logic was correct, but it made `conflicts_with` significantly longer and introduced a dependency on string parsing that could fail on unexpected input. I kept the simpler exact-match version because it was enough for this project's requirements and much easier to test.

2. **Rejected a suggestion to add a `priority` attribute directly on `Pet`.** The idea was that a pet's overall priority would influence scheduling. I decided against it because task-level priorities already capture this — a pet's high-priority feeding task will naturally rank above another pet's low-priority grooming task without needing a separate pet-level weight. Adding it would have made the sort logic more complex without a clear benefit.

For verification, I ran the affected functions manually with small examples in a Python shell before writing the corresponding tests. If the output matched my expectation, I committed the implementation.

---

## 4. Testing and Verification

### a. What I tested

| Test | Behavior covered |
|------|-----------------|
| `test_mark_complete_changes_status` | `mark_complete()` sets `completed` to `True` |
| `test_add_task_to_pet` | `pet.add_task()` appends correctly and is retrievable by index |
| `test_sort_by_time_returns_chronological_order` | `sort_by_time` puts 08:00 before 08:30 before 09:00 regardless of insertion order |
| `test_daily_task_completion_creates_next_recurring_task` | Completing a `daily` task marks it done and creates a new task with `due_date + 1 day`; the pet ends up with 2 tasks |
| `test_conflict_detection_flags_duplicate_times` | Two tasks at `08:30` produce exactly one conflict string containing that time |
| `test_pet_with_no_tasks_returns_empty_list` | `collect_tasks()` returns `[]` when no tasks exist |

I focused on the behaviors most likely to break quietly — sorting order, recurring task creation, and conflict detection — because those are harder to spot visually in the UI.

### b. Confidence and next steps

I am confident the core behaviors work correctly because each test directly checks a specific outcome rather than just checking that nothing crashes. The recurring task test, for example, checks both the completion status of the original task and the exact `due_date` of the new one.

If I had more time, I would add tests for:
- The `available_minutes` budget cutoff — verifying that a task which would exceed the budget is not added to `scheduled_tasks`
- `filter_tasks` with combined pet name and completion filters
- `create_next_recurring_task` for `weekly` frequency

---

## 5. Reflection

### a. What went well

Keeping `pawpal_system.py` completely independent of Streamlit was the best structural decision I made. Because all the logic is in plain Python dataclasses with no UI imports, I could test it directly with pytest and reason about it without needing to run the browser app. Connecting it to `app.py` afterward was straightforward.

### b. What I would improve

The scheduling algorithm skips tasks that exceed the budget but gives the user no feedback about which tasks were left out or why. I would add a `skipped_tasks` list to `build_daily_schedule` so the UI can show a "Not scheduled today" section with the reason (e.g., "not enough time remaining"). I would also make the conflict check time-range aware, since the current version can miss real overlaps.

### c. Key takeaway

The most valuable thing I learned is that AI tools work best as a sounding board during design, not as a replacement for thinking through the design yourself. When I accepted an AI suggestion without reading it carefully, I sometimes had to undo it later because it didn't match my data model. When I described the problem clearly and then evaluated the suggestion against my own understanding, the collaboration went much faster and the result was code I actually understood.
