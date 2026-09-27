DependencyIQ — Design Document
1. Architecture
DependencylQ is a front-end web application built with React (via Vite) that models a project
as a directed acyclic graph (DAG) of tasks and dependencies, and computes schedule impact
deterministically.
High-level structure:
• src/scheduler. s —Scheduling engine (core logic layer). Pure functions with no IJI
dependencies:
0 D8-based cycle detection over the dependency graph; rejects invalid
schedules before they're computed.
¯ propagates earliest start/end dates through the graph using
memoized recursion, so a task's date is only ever calculated once even if multiple downstream
tasks depend on it (prevents double-counting delays across converging paths). Also derives
each task's "blocked" state and the project's critical path (the longest chain ending at the task
with the latest finish date).
¯ clones the task list, applies a hypothetical delay to
one task, recomputes the schedule, and diffs the result against the original to report which
downstream tasks shift and by how much. This powers the "what if" scenario feature without
mutating real project state.
• UI layer I state management. Holds the task list in React state and renders
three coordinated views:
Task board — Kanban-style columns (Blocked I To do / Done) driven by the scheduler's
computed blocked-state, not manual status fields.
o Dependency graph — an SVG rendering where tasks are laid out in columns by graph depth
(longest path from a root task) and edges are drawn between prerequisites and dependents;
the critical path and blocked tasks are visually highlighted.
o Assistant panel — a rule-based "explain" function that reads the scheduler's output (which
prerequisite is driving a task's start date, whether it's blocked) and turns it into a natural-
language sentence, plus a IJI for the what-if simulator.
Separation Of concerns: the scheduling engine is intentionally isolated from both the IJI and
the "Al" explanation layer, so the authoritative dates are always deterministic and testable,
and the explanation/assistant layer is a read-only interpreter on top Of them — it never
influences the computed schedule.
2. Data Model
The current model is in-memory (React state); it's structured so it maps directly onto a
relational schema ([EJEE], if/when a backend is added.
Task
Field
id
done
Type
Description
string Unique identifier
string Task name
number Duration in days
IDS Of tasks that must complete first
boolean Whether the task is marked complete
Derived (computed by the scheduler, not stored):
Field
earliestEnd
blocked
criticalPath
Description
Day offsets computed from the dependency chain
True if any prerequisite is not
Ordered list of task IDs forming the longest chain to project
completion
Dependencies are stored as a reference list on each task rather than a
separate join table, since the prototype runs in memory — a production version would
normalize this into its own table as described in the
original proposal, with a uniqueness/cycle constraint enforced at write time.
3. Known Limitations
• NO persistence layer. All data lives in browser memory and resets on page reload; there is no
database yet (the proposal's PostgreSQL schema — projects, tasks, dependencies,
status_history, audit_logs — has not been implemented).
• NO authentication or multi-user support. Anyone with the page can view and edit the single
in-memory project; there's no concept Of separate projects, users, or permissions yet
• The AI assistant is rule-based, not areal LLM. Explanations and what-if results are
generated from the scheduler's own output using fixed templates, not a live language model
call — this keeps the demo fully offline but doesn't yet demonstrate real natural-language
understanding Of free-form questions.
• No drag-and-drop or inline editing. Tasks can be created and marked done/removed, but
durations and dependencies can't be edited after creation without deleting and recreating
the task.
• Scheduling assumes a single project and no resource constraints. It computes date
propagation from dependencies only; it doesn't account for shared resources, working
calendars/weekends, or parallel-task capacity limits.
• Graph layout is basic. Nodes are placed by dependency depth only; large or densely
connected graphs (dozens+ of tasks) will need a more sophisticated layout algorithm to stay
readable.
• No automated test suite yet. The scheduling engine's correctness (cycle detection, no
double-counted delays, critical-path accuracy) has been manually verified but doesn't yet
have unit tests, as proposed
