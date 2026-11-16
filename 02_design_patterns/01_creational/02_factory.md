# Factory

Source: Hello Interview LLD → Design Patterns, and my teach-back on 17 Nov 2026 (not yet a clean one).

## What this note covers

- What a factory decides, and when it earns its place
- The gotchas interviewers poke at
- The registry trick for the open/closed challenge

## Review points

- **One job:** turn a key (`"sms"`) into the right concrete object. Callers get back the interface and never name a concrete class.
- **Payoff:** a new type is one new class plus one change in the factory. No caller changes.
- **Add one only when** the same creation decision is made in several places, or the type arrives as runtime data. Otherwise construct the object once and inject it.
- **The honest pitch:** a factory doesn't delete the `if/elif`, it gives it one home.
- Factories are polarizing. Hello Interview's advice: if you add one, check your interviewer for a grimace.

---

### The whole pattern

```python
from abc import ABC, abstractmethod
from typing import Callable

class Notification(ABC):
    @abstractmethod
    def send(self, to: str, message: str) -> str: ...

class EmailNotification(Notification):
    def send(self, to: str, message: str) -> str:
        return f"email to {to}: {message}"

class SmsNotification(Notification):
    def send(self, to: str, message: str) -> str:
        return f"sms to {to}: {message}"

def create_notification(kind: str) -> Notification:      # the if/elif lives here, once
    if kind == "email":
        return EmailNotification()
    if kind == "sms":
        return SmsNotification()
    raise ValueError(f"unknown notification type: {kind}")

class AlertService:
    def __init__(self, create: Callable[[str], Notification]):   # the factory is injected
        self._create = create

    def alert(self, kind: str, to: str, message: str) -> str:
        return self._create(kind).send(to, message)              # kind arrives as data

service = AlertService(create_notification)     # pass the function itself: no parentheses
service.alert("sms", "alice", "disk full")      # "sms to alice: disk full"
```

In Python a factory can be a plain function. No `NotificationFactory` class is needed until it has state.

### Gotchas

- **Counting types is the wrong test.** Three types created in one place: no factory. Two types created in five places: factory. Count creation sites, not classes.
- **Injection and a factory are not rivals.** Injection covers a choice known when the app is wired (`OrderService(EmailNotification())`). A factory covers a choice that arrives per request as data. When you need both, inject the factory, as `AlertService` does.
- **"It's still open/closed" is a trap.** The factory *is* edited for every new type. Say so, then add that it is the only thing edited and every caller stays closed.
- **Unknown key:** raise. Returning `None` moves the crash to a confusing place later.
- **A factory that just wraps one constructor** is ceremony. Delete it.

### Registry: when they push on open/closed

Classes are objects in Python, so the `if/elif` becomes a dict and a new type becomes a registration.

```python
class PushNotification(Notification):
    def send(self, to: str, message: str) -> str:
        return f"push to {to}: {message}"

class NotificationRegistry:
    def __init__(self) -> None:
        self._classes: dict[str, type[Notification]] = {}

    def register(self, kind: str, klass: type[Notification]) -> None:
        self._classes[kind] = klass

    def create(self, kind: str) -> Notification:
        if kind not in self._classes:
            raise ValueError(f"unknown notification type: {kind}")
        return self._classes[kind]()

registry = NotificationRegistry()
registry.register("push", PushNotification)              # no factory code changed
service = AlertService(registry.create)         # same AlertService, now given the registry's method
service.alert("push", "alice", "disk full")     # "push to alice: disk full"
```

Start with the plain `if/elif`. Reach for the registry only when asked, or when types are added often.

### Connections

- Builds on polymorphism and DIP: the factory is what lets callers hold only the abstraction.
- Factory vs Strategy: to fill in when I do Strategy.

---

## Review questions

Answer first, then open the answer to check.

1. Your factory still has an `if/elif` on type. What did you gain?

   <details><summary>Answer</summary>

   - The conditional exists once, in a place whose only job is creation, not at every creation site.
   - Callers depend only on the interface, so a new type changes the factory and nothing else.

   </details>

2. Parking lot: `Car`, `Bike` and `Truck` are created in exactly one place from user input. Do you add a `VehicleFactory`?

   <details><summary>Answer</summary>

   - No. One creation site means there is nothing to centralize, so the factory is only indirection.
   - Add it when a second place needs the same decision.

   </details>

3. "Just inject the dependency in the constructor." When does that replace a factory, and when can't it?

   <details><summary>Answer</summary>

   - It replaces it when the choice is known at wiring time: one notifier for the whole service.
   - It can't when the type arrives per request as data. Something must map `"sms"` to a class. That is the factory, and you inject *it*.

   </details>

4. "You edit the factory for every new type, so this violates open/closed." What do you say, and how do you avoid the edit in Python?

   <details><summary>Answer</summary>

   - Agree that it is edited, and that it is the only thing edited: callers stay closed.
   - To avoid it: a registry, `dict[str, type[Notification]]`, with `register(kind, klass)`. `create` never changes.

   </details>
