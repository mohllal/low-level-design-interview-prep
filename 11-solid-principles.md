# SOLID principles

Five principles for evaluating and improving object-oriented designs. The value is not in following them mechanically but in having precise vocabulary for design critiques: "this class violates SRP because it changes for two different reasons" is more actionable than "this class is too big."

All examples use one domain: a notification service that builds messages, routes them to channels, and audits delivery.

## Single responsibility principle

A class should have only one reason to change. "Reason to change" means a specific stakeholder or business concern: the rule about message formatting, the rule about which channel to use, the rule about retry logic. A class that changes for two different reasons is two classes sharing one name.

### Violation

```python
class NotificationService:
    def send(self, user: User, event: str, payload: dict) -> None:
        # formats the message
        subject = f"[{event.upper()}] notification"
        body = "\n".join(f"{k}: {v}" for k, v in payload.items())
        # routes to the channel
        if user.preferred_channel == "email":
            smtp = smtplib.SMTP("mail.example.com")
            smtp.sendmail("noreply@example.com", user.email, f"{subject}\n{body}")
        elif user.preferred_channel == "slack":
            requests.post(SLACK_WEBHOOK, json={"text": body})
        # audits delivery
        with open("delivery.log", "a") as f:
            f.write(f"{datetime.now()}: sent {event} to {user.email}\n")
```

Three concerns: message formatting, channel routing, and audit logging. A change to the log format touches the same class as a change to the Slack integration.

### Fix

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass
class Message:
    subject: str
    body: str


class MessageFormatter:
    def format(self, event: str, payload: dict) -> Message:
        subject = f"[{event.upper()}] notification"
        body = "\n".join(f"{k}: {v}" for k, v in payload.items())
        return Message(subject, body)


class NotificationChannel(Protocol):
    def send(self, user: User, message: Message) -> None: ...


class DeliveryLog(Protocol):
    def record(self, user: User, event: str) -> None: ...


class NotificationService:
    def __init__(
        self,
        formatter: MessageFormatter,
        channels: dict[str, NotificationChannel],
        log: DeliveryLog,
    ) -> None:
        self._formatter = formatter
        self._channels = channels
        self._log = log

    def send(self, user: User, event: str, payload: dict) -> None:
        message = self._formatter.format(event, payload)
        self._channels[user.preferred_channel].send(user, message)
        self._log.record(user, event)
```

Each class changes for one kind of reason. `MessageFormatter` changes when the message structure changes. `NotificationService` changes when the orchestration logic changes.

### Failure mode

Taken too far, SRP produces a class per method: `UserEmailSubjectFormatter`, `UserEmailBodyFormatter`, `UserEmailSender`. Classes become microscopic and the system becomes impossible to navigate. "One reason to change" is one coherent concern, not one line of code.

## Open/closed principle

A class should be open for extension and closed for modification. Adding new behavior should require adding new code, not editing code that already works. The mechanism is an abstraction that existing code depends on, with new behavior added as new implementations of that abstraction.

### Violation

```python
class NotificationRouter:
    def route(self, user: User, message: Message) -> None:
        if user.preferred_channel == "email":
            EmailChannel().send(user, message)
        elif user.preferred_channel == "slack":
            SlackChannel().send(user, message)
        elif user.preferred_channel == "sms":
            SmsChannel().send(user, message)
```

Adding a push notification channel requires editing `NotificationRouter`. Every modification risks breaking the existing channels.

### Fix

```python
class NotificationChannel(Protocol):
    def send(self, user: User, message: Message) -> None: ...


class NotificationRouter:
    def __init__(self, channels: dict[str, NotificationChannel]) -> None:
        self._channels = channels

    def route(self, user: User, message: Message) -> None:
        channel = self._channels.get(user.preferred_channel)
        if channel is None:
            raise ValueError(f"unknown channel: {user.preferred_channel!r}")
        channel.send(user, message)
```

Adding a push notification channel is a new `PushChannel` class and a new entry in the registry. `NotificationRouter` is not touched.

### Failure mode

OCP applied speculatively produces extension points for variation that never occurs. A router with a channel protocol is warranted when multiple channels exist or are required. A `MessageSorter` with a `SortingStrategy` protocol when there is one sort order and no requirement to change it is an interface with nothing to interface against.

## Liskov substitution principle

A subtype must be substitutable for its supertype without callers knowing or caring. If `S` is a subtype of `T`, any property provable about objects of type `T` must hold for objects of type `S`.

Three substitution failures recur:

- Preconditions strengthened: the subtype requires more from callers than the parent promised to require.
- Postconditions weakened: the subtype guarantees less than the parent promised to guarantee.
- Invariants violated: the subtype breaks a property the parent always maintained.

### Violation

```python
class NotificationChannel(Protocol):
    def send(self, user: User, message: Message) -> None: ...


class ThrottledChannel:
    def send(self, user: User, message: Message) -> None:
        if self._is_rate_limited(user):
            return  # silently drops the message
        self._inner.send(user, message)
```

`ThrottledChannel` claims to implement `NotificationChannel`, but its contract is weaker: `send` may silently do nothing. Any caller that depends on delivery — an audit log, a retry mechanism, an SLA check — breaks when `ThrottledChannel` is substituted.

### Fix

```python
class RateLimitExceeded(Exception): ...


class ThrottledChannel:
    def send(self, user: User, message: Message) -> None:
        if self._is_rate_limited(user):
            raise RateLimitExceeded(f"rate limit for {user.user_id!r}")
        self._inner.send(user, message)
```

The channel fails explicitly. Callers can catch `RateLimitExceeded` and decide whether to retry, queue, or skip — but they are informed rather than silently misled. The original contract holds: `send` either delivers or raises.

### Failure mode

Forcing substitutability when the types are genuinely different. A `ReadOnlyLog` that records but never sends is not a broken `NotificationChannel`; it is a different kind of thing and should not be a subtype. The fix is a narrower type with its own interface, not a subtype that weakens the parent's postconditions.

Commonly probed: `Square` extends `Rectangle`. `Rectangle.set_width(w)` changes only width while `Square.set_width(w)` must set both dimensions to maintain the square invariant. Code that does `r.set_width(5); assert r.height == old_height` breaks when a `Square` is substituted. The geometry "is-a" relationship is real, but the behavioral contract is not: `Square` is not substitutable for `Rectangle` under this interface.

## Interface segregation principle

No class should be forced to depend on methods it does not use. A fat interface that bundles unrelated capabilities forces implementors to stub methods they do not have, and forces callers to depend on capabilities they do not need.

### Violation

```python
class NotificationChannel(Protocol):
    def send(self, user: User, message: Message) -> None: ...
    def retry(self, message: Message, attempts: int) -> None: ...
    def schedule(self, message: Message, at: datetime) -> None: ...
    def archive(self, message: Message) -> None: ...
```

An `EmailChannel` that cannot schedule messages must implement `schedule` anyway, probably raising `NotImplementedError`. A caller that only needs `send` is still coupled to the full interface — any change to `retry` or `archive` requires recompiling or retesting the caller.

### Fix

```python
class NotificationChannel(Protocol):
    def send(self, user: User, message: Message) -> None: ...

class RetryableChannel(Protocol):
    def retry(self, message: Message, attempts: int) -> None: ...

class SchedulableChannel(Protocol):
    def schedule(self, message: Message, at: datetime) -> None: ...
```

Each capability is its own protocol. `EmailChannel` implements `NotificationChannel` and `RetryableChannel`. `SlackChannel` implements only `NotificationChannel`. A scheduler accepts only `SchedulableChannel`. No implementor is forced to stub a capability it lacks.

### Failure mode

Interface-per-method fragmentation. A `Sender`, `Receiver`, `Connector`, and `Initializer` each with one method produces a proliferation of interfaces that are impossible to navigate and provide no real abstraction. ISP is a constraint on fat interfaces, not a direction to atomize every behavior.

## Dependency inversion principle

High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details and details should depend on abstractions.

In practice: the class that orchestrates logic should depend on a protocol or abstract class, not on a concrete implementation. The concrete implementation is supplied from outside, usually through the constructor.

### Violation

```python
class NotificationService:
    def __init__(self) -> None:
        self._channel = EmailChannel()      # hardcoded low-level detail
        self._log = FileDeliveryLog()       # hardcoded low-level detail
        self._formatter = MessageFormatter()
```

`NotificationService` is bound to `EmailChannel` and `FileDeliveryLog`. Testing it requires a live SMTP server and a writable filesystem. Switching to Slack requires editing the constructor.

### Fix

```python
class NotificationService:
    def __init__(
        self,
        formatter: MessageFormatter,
        channel: NotificationChannel,
        log: DeliveryLog,
    ) -> None:
        self._formatter = formatter
        self._channel = channel
        self._log = log
```

Dependencies are declared as abstractions and injected through the constructor. `NotificationService` never names a concrete class.

### Dependency injection

Constructor injection (above) is the most common form. The object is always valid after `__init__` completes, the dependencies are visible at the call site, and the requirements are explicit.

Two alternatives:

- **Property injection**: the dependency is set via an attribute after construction. Leaves the object in an invalid state between construction and injection .. it is useful only when circular dependencies make constructor injection impossible.
- **Method injection**: the dependency is passed directly to the method that uses it. Appropriate when the dependency changes per call rather than per object — `send(user, message, channel=override)` — not as a substitute for constructor injection when the dependency is fixed for the object's lifetime.

### Designing for testability

DIP makes testing possible without mocking frameworks. A `FakeChannel` that records messages, an `InMemoryDeliveryLog` that stores records in a list, and a `FakeUserRepository` that returns seeded data cover all scenarios:

```python
class FakeChannel:
    def __init__(self) -> None:
        self.sent: list[tuple[User, Message]] = []

    def send(self, user: User, message: Message) -> None:
        self.sent.append((user, message))


class InMemoryDeliveryLog:
    def __init__(self) -> None:
        self.records: list[tuple[User, str]] = []

    def record(self, user: User, event: str) -> None:
        self.records.append((user, event))


def test_notification_sent_on_overdue_task():
    channel = FakeChannel()
    log = InMemoryDeliveryLog()
    service = NotificationService(MessageFormatter(), channel, log)

    service.send(user=alice, event="task_overdue", payload={"task_id": "t1"})

    assert len(channel.sent) == 1
    assert channel.sent[0][0] == alice
    assert len(log.records) == 1
```

No monkeypatching, no mock framework, no network. The test is fast, deterministic, and reads like a specification of the behavior.

### Failure mode

DI containers and factory overkill. Constructor injection scales to every problem at interview scope. A container that wires dependencies automatically, a service locator, or a three-layer abstract factory is infrastructure for a scale the problem does not have. The principle is "depend on abstractions"; the mechanism is constructors.

## How the principles interact

The principles reinforce each other and each violation tends to cascade:

- SRP makes ISP easier: a class with one responsibility rarely needs to implement an interface with ten methods.
- DIP enables OCP: a class that depends on abstractions can be extended by adding new implementations of those abstractions without modification.
- LSP is a prerequisite for OCP: if subtypes do not substitute cleanly, the extension point cannot be used without knowing which concrete subtype is in play, defeating the abstraction.
- SRP violations create DIP violations: a class that mixes orchestration with infrastructure has no seam at which to inject the infrastructure dependency.

Commonly probed follow-up: which principle is most commonly violated? SRP and DIP in practice. SRP because classes accumulate responsibilities gradually ("just add it here, it is related"); DIP because instantiating a dependency directly is the path of least resistance and its cost only becomes visible at test time.
