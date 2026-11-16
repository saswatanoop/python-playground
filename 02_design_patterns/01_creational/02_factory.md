# Factory

Source: Hello Interview LLD → Design Patterns, and my teach-back on 17 Nov 2026 (not yet a clean one).

## What this note covers

- What a factory decides, and when it earns its place
- The pushbacks an interviewer makes, and what to say back
- The registry trick for the open/closed challenge

## Review points

- **One job:** turn a key (`"sms"`) into the right concrete object. Callers get back the interface and never name a concrete class.
- **Payoff:** a new type is one new class plus one change in the factory. No caller changes.
- **Add one only when** the same creation decision is made in several places, or the type arrives as runtime data. Otherwise construct the object once and inject it.
- **The honest pitch:** a factory doesn't delete the `if/elif`, it gives it one home.
- **Open/closed pushback:** concede that the factory is edited, point out nothing else is, and offer a registry.

---

## The discussion

### "I need to send a notification. Why can't I just write `SmsNotification()`?"

You can. If one place creates it, do exactly that and close this note.

The trouble starts when the type arrives as data and more than one place has to deal with it:

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

def send_alert(kind: str, to: str, message: str) -> str:
    if kind == "email":
        notification = EmailNotification()
    elif kind == "sms":
        notification = SmsNotification()
    else:
        raise ValueError(f"unknown notification type: {kind}")
    return notification.send(to, message)

# send_report() and send_invoice() repeat the same six lines
```

Then someone asks for push notifications, and you go hunting for every copy of that `if/elif`. Miss one and you find out in production.

### "So what does a factory do about it?"

It gives that decision one home. That's the entire pattern.

```python
def create_notification(kind: str) -> Notification:
    if kind == "email":
        return EmailNotification()
    if kind == "sms":
        return SmsNotification()
    raise ValueError(f"unknown notification type: {kind}")

def send_alert(kind: str, to: str, message: str) -> str:
    notification = create_notification(kind)
    return notification.send(to, message)

send_alert("sms", "alice", "disk full")     # "sms to alice: disk full"
```

Push arrives: one new class, one new branch in `create_notification`. `send_alert`, `send_report` and `send_invoice` don't change. They no longer even know that `SmsNotification` exists.

Notice it's a plain function. In Python you don't need a `NotificationFactory` class until the factory has state of its own.

### "Hold on. The `if/elif` is still there. What did I gain?"

Fair, and an interviewer will say exactly this. A factory doesn't delete the conditional. It moves it from *every* creation site to *one* place whose only job is creation. Say it that plainly; don't oversell it.

### "So whenever I have three or more types, I add a factory?"

That was my first instinct, and it's the wrong test. Count **creation sites**, not classes:

- Three types, created in one place: no factory. It would only be indirection.
- Two types, created in five places: factory.

The other trigger is the type arriving as **runtime data**: a string in a request, a value in a config file.

Factories are polarizing. Hello Interview's advice: if you add one, check your interviewer for a grimace.

### "Why not just inject the notification through the constructor?"

Good default, and usually the right one. But the two aren't rivals; they answer different questions.

- **The choice is known when the app is wired:** `OrderService(EmailNotification())`. Inject it. No factory.
- **The choice arrives per request, as data:** something still has to turn `"sms"` into an object. That something is the factory, and you inject *the factory*.

```python
class AlertService:
    def __init__(self, create: Callable[[str], Notification]):
        self._create = create                     # the factory, injected

    def alert(self, kind: str, to: str, message: str) -> str:
        notification = self._create(kind)         # kind arrives as data
        return notification.send(to, message)

service = AlertService(create_notification)      # no parentheses: hand over the function itself
service.alert("sms", "alice", "disk full")       # "sms to alice: disk full"
```

### Interviewer: "You edit the factory for every new type. That breaks open/closed."

Don't answer "it's still closed for modification". I tried that; it isn't, and they know it.

Concede, then narrow: yes, the factory is edited, and it is the *only* thing edited. Every caller stays closed.

If they keep pushing, classes are objects in Python, so the `if/elif` can become a dict, and a new type becomes a registration:

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
registry.register("push", PushNotification)      # no factory code changed

service = AlertService(registry.create)          # same AlertService, different factory
service.alert("push", "alice", "disk full")      # "push to alice: disk full"
```

`AlertService` didn't change between the two examples. It only ever asked for "something that takes a string and returns a `Notification`".

### Gotchas

- **Don't lead with the registry.** Start with the plain `if/elif`; bring the registry out only when asked, or when types are added often.
- **Unknown key: raise.** Returning `None` moves the crash somewhere confusing.
- **A factory around a single constructor** is ceremony. Delete it.
- **A factory picks the class, nothing more.** Once it starts validating, logging and sending, it has become a service.

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

   - Concede that it is edited, and that it is the only thing edited: callers stay closed.
   - To avoid it: a registry, `dict[str, type[Notification]]`, with `register(kind, klass)`. `create` never changes.

   </details>
