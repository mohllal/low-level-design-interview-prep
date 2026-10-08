# General design heuristics

Eight short rules that guide modeling decisions in an LLD round. Each is a heuristic, not a law: it describes what works in the common case and should be set aside when it does not.

All examples use one domain: a task management system with tasks, projects, tags, assignees, and notification and export channels.

## DRY — Don't Repeat Yourself

Knowledge, not code, is what must not be duplicated. Every piece of knowledge should have one authoritative location in the system. When the same decision is encoded in two places, one will be updated and the other will not.

### Violation

```python
class Task:
    def is_overdue(self) -> bool:
        return self.due_date is not None and self.due_date < date.today()

class Project:
    def overdue_tasks(self) -> list[Task]:
        today = date.today()
        return [t for t in self.tasks if t.due_date is not None and t.due_date < today]
```

The overdue rule is encoded twice. Changing it — adding a grace period, switching to `datetime.now()`, adding timezone handling — requires finding both sites.

### Fix

```python
class Project:
    def overdue_tasks(self) -> list[Task]:
        return [t for t in self.tasks if t.is_overdue()]
```

The rule lives in one place. `Project` delegates to it.

### What DRY does not mean

DRY is about duplication of knowledge, not duplication of text. Two methods that look alike but encode different rules are not a DRY violation. Collapsing them creates accidental coupling.

The test: if the rule changes, should both call sites change together? If yes, the rule is one piece of knowledge and belongs in one place. If the changes could diverge independently, they are two pieces of knowledge that happen to look alike today.

Premature DRY is a common failure mode: extracting an abstraction to prevent imagined repetition builds infrastructure for hypothetical variation. The connection to YAGNI below.

## KISS — Keep It Simple

A design should be as simple as the requirements demand. The right question is not "what might this need to do someday" but "what does this need to do now".

### Violation

```python
import heapq

class TaskPriorityQueue:
    """Min-heap ordered by deadline and priority score."""

    def __init__(self) -> None:
        self._heap: list[tuple[float, Task]] = []

    def push(self, task: Task) -> None:
        heapq.heappush(self._heap, (self._score(task), task))

    def pop(self) -> Task:
        return heapq.heappop(self._heap)[1]

    def _score(self, task: Task) -> float: ...
```

If the operation is "give me the highest-priority task from a list of dozens", a `min()` call suffices unless profiling shows the scan is a bottleneck. A heap adds invariants to maintain and a custom sort key to explain.

### Fix

```python
def next_task(tasks: list[Task]) -> Task | None:
    return min(tasks, key=lambda t: t.priority.value, default=None)
```

Introduce the heap if profiling shows the linear scan is a bottleneck.

### What KISS does not mean

KISS is not an argument for unclear names, long methods that do many things, or skipping the abstractions that keep code navigable. The meaningful comparison is always "simple relative to what?" A design with clear responsibilities and well-named classes is simple. A single class with every method and no structure is not.

## YAGNI — You Ain't Gonna Need It

Features without a current requirement have no place in the design. Speculative code carries the same maintenance cost as code that is used, and has the added cost of being wrong when the feature finally arrives — the guess about how it would be needed was usually off.

### Violation

```python
@dataclass
class Task:
    title: str
    due_date: date | None = None
    locale: str = "en"        # added "in case we go international"
    timezone: str = "UTC"     # added "in case we go international"
```

If no requirement calls for localization, the fields are dead weight: they must be passed, defaulted, documented, and eventually deleted or updated correctly when the real requirement arrives with a shape nobody guessed.

YAGNI covers building features, not designing extension points. A `Notifier` protocol that makes the design open for new channels costs almost nothing. A whole localization layer for a feature nobody requested costs a great deal.

The interaction with premature DRY: extracting an abstraction to avoid imagined repetition is a YAGNI violation. The abstraction solves a problem that does not yet exist.

## Composition over inheritance

Prefer building behavior by composing objects rather than extending classes. Inheritance is strong coupling: the subclass depends on every implementation detail of the parent, and the hierarchy is hard to change once established ([note 05](05-class-relationships.md)).

### Violation

```python
class Task: ...
class UrgentTask(Task): ...            # adds priority handling
class RecurringTask(Task): ...         # adds recurrence schedule
class UrgentRecurringTask(Task): ...   # required to combine both
```

Every new combination of behaviors requires a new subclass. The hierarchy grows combinatorially.

### Fix

```python
from dataclasses import dataclass, field
from enum import Enum


class Priority(Enum):
    LOW = 1
    NORMAL = 2
    HIGH = 3


@dataclass
class Recurrence:
    interval_days: int


@dataclass
class Task:
    title: str
    priority: Priority = Priority.NORMAL
    recurrence: Recurrence | None = None
    notifier: Notifier | None = None
```

Behaviors that vary — priority, recurrence, notification — are fields or injected collaborators, not subclasses. Any combination is expressible without adding a class.

Inheritance is appropriate when the relationship is genuinely "is-a" and the subtype can substitute for the parent without surprises. The LSP section of [note 11](11-solid-principles.md) is the precise statement of when that holds.

## Program to an interface

Depend on the abstract description of what something does, not on how it does it. This decouples the caller from the implementation and makes variation points explicit.

### Violation

```python
class ProjectExporter:
    def __init__(self) -> None:
        self._writer = CsvWriter()   # tied to one format forever

    def export(self, project: Project) -> str:
        return self._writer.write(project.tasks)
```

Adding JSON or Markdown export requires modifying `ProjectExporter`. The format is a detail that should be outside its concern.

### Fix

```python
from typing import Protocol


class TaskFormatter(Protocol):
    def format(self, tasks: list[Task]) -> str: ...


class ProjectExporter:
    def __init__(self, formatter: TaskFormatter) -> None:
        self._formatter = formatter

    def export(self, project: Project) -> str:
        return self._formatter.format(project.tasks)
```

`ProjectExporter` depends on the protocol. `CsvFormatter`, `JsonFormatter`, and `MarkdownFormatter` each implement it independently. Adding a new format is a new class, not a change to the exporter.

## Encapsulate what varies

Identify which parts of a design change for different reasons or in response to different requirements, and put those parts behind a boundary. The stable core goes on one side and the varying part goes on the other, behind a protocol.

In the task domain, the variation points are:

- How tasks are notified (email, Slack, SMS, push)
- How projects are exported (CSV, JSON, Markdown)
- How tasks are prioritized (by deadline, by score, by assignee load)

Each is a candidate for its own protocol. The stable core — project structure, task lifecycle, tag assignment — does not change when a new notification channel is added.

This is the design intent behind the open/closed principle and the strategy pattern. The three examples above are each a strategy in waiting: one protocol, multiple implementations, the caller depending only on the protocol.

## Tell Don't Ask

Tell objects what to do and do not ask them for their state and act on it from outside. When an object's data must be consulted before taking action, that action often belongs inside the object.

### Violation

```python
def archive_overdue(project: Project) -> None:
    today = date.today()
    for task in project.tasks:
        if task.due_date is not None and task.due_date < today and not task.archived:
            task.archived = True
            task.archived_at = today
```

The caller reaches into `task`, checks its internals, and mutates it. The decision about what archiving means and when it applies is outside `Task`.

### Fix

```python
class Task:
    def archive_if_overdue(self) -> bool:
        if self.is_overdue() and not self.archived:
            self.archived = True
            self.archived_at = date.today()
            return True
        return False


def archive_overdue(project: Project) -> None:
    for task in project.tasks:
        task.archive_if_overdue()
```

The decision and the mutation live in `Task`. The caller sends a message and lets the object decide.

Tell Don't Ask is closely related to the Law of Demeter ([note 08](08-coupling-and-cohesion.md)): both are symptoms of logic that has drifted outside the class that owns the data it operates on. A Law of Demeter violation is usually a Tell Don't Ask violation seen from the navigation chain side.

## Principle of least astonishment

A method should do exactly what its name says, nothing more and nothing less. Callers build a mental model from names and a method that violates that model introduces a hidden contract that only shows up at runtime.

Common violations in a task management system:

- `task_manager.get_project(name)` that creates the project if it does not exist, when the name implies a pure lookup.
- `project.tasks` that sorts the list in place on access, mutating state through a getter.
- `project.add_task(task)` that silently deduplicates without documenting it, so callers cannot tell whether two identical tasks were added or one.
- `TaskSync.__init__()` that makes a network call to fetch remote tasks during construction.
- `task.delete()` that moves the task to an archive rather than removing it.

The fix is: rename to match the actual behavior (`get_or_create_project`, `sort_tasks`, `add_or_ignore`), split into two methods each with an honest name, or remove the hidden behavior. The cost of a surprising method is proportional to how long it takes to discover the surprise — usually during debugging at the worst possible time.

## The over-engineering trap

Every heuristic above can be misapplied. The over-engineering trap is invoking a principle beyond the point where it improves the design.

Signs of over-engineering at LLD interview scope:

- Three levels of abstraction for a problem with one known implementation.
- A `TaskFactory` that constructs tasks whose construction has no variation.
- A `Notifier` protocol with a single `SlackNotifier` and no second implementation in sight or in the requirements.
- A `TaskRepository` interface backed by an in-memory dict, with no persistence requirement stated.
- DRY applied to code that looks alike but changes for different reasons, coupling two pieces of logic that should stay independent.
- `TaskFormatter` hierarchy where the only format is CSV and no other format has been requested.

The test is: what does this abstraction enable? If the answer is "it would make adding X easier" and X is not in the requirements, the abstraction is speculative. The cost is paid now; the benefit never arrives.

The right level of abstraction is the one where variation exists in the requirements. An extension point earns its place when a second implementation can be named from what is actually required.
