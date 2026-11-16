# SOLID

Source: Hello Interview LLD → In a Hurry → Design Principles (part 2 of 2). Learned 10 Nov 2026. Previous: [03_general_principles.md](03_general_principles.md).

## What this note covers

- The five SOLID principles, each with a bad and a good version
- When *not* to apply OCP and DIP
- How an LSP violation differs from an ISP violation
- Why over-applying SOLID breaks KISS

## Review points

- **SRP:** one reason to change. **OCP:** extend by adding, not editing. **LSP:** a child keeps every parent promise; it can add, never remove. **ISP:** small interfaces; no stub methods. **DIP:** business logic owns the interface, and implementations conform to it (through injection).
- **SOLID comes from Java-era deep hierarchies.** Outside Java and C#, overusing it is going out of fashion: modern code favours composition over hierarchies and functions over interfaces. Apply SOLID only when the problem calls for it, or you break KISS.

**What each one buys you** (the article's cheat sheet):

| Principle | Buys you |
|---|---|
| SRP | Classes focused on one responsibility |
| OCP | Future requirements without modifying existing code |
| LSP | No brittle hierarchies that break at runtime |
| ISP | Clean, focused interfaces |
| DIP | Business logic decoupled from implementation details |

- Senior signal: name the tension (OCP/DIP vs KISS and YAGNI) and say which way you lean *and why*.
- ⚠ **My weak spot (missed twice):** *when not to abstract*. See the "When *not* to" bullets under OCP and DIP, and test those first.

---

Code: [04_solid_example.py](04_solid_example.py), a notification service that uses all five. Composition over inheritance is in [06_polymorphism_inheritance.md](06_polymorphism_inheritance.md).

**Caveat from the article:** SOLID comes from Java/C#-style deep inheritance and interface-heavy design. Overusing it breaks KISS. Use a principle when the problem needs it, and refer to it briefly when explaining a trade-off. Never recite the list. SOLID manages change and coupling; KISS and YAGNI cap how much abstraction you add. Interviewers look for that balance.

Snippets below assume `from abc import ABC, abstractmethod` and `from typing import Protocol`.

### S: Single Responsibility
- **Crux:** a class has one reason to change, meaning one responsibility.
- **Example:** an `Order` that calculates its total, saves itself to the DB and emails the customer has 3 reasons to change:
  - pricing rules (business)
  - storage (schema)
  - notifications (messaging)

  Split it into `Order`, `OrderRepository` and `OrderNotifier`.

```python
# Bad: 3 reasons to change in one class
class Order:
    def total(self) -> float: ...
    def save(self) -> None: ...          # DB code
    def email_customer(self) -> None: ... # SMTP code

# Good: one reason each
class Order:
    def total(self) -> float: ...
class OrderRepository:
    def save(self, order: Order) -> None: ...
class OrderNotifier:
    def send_confirmation(self, order: Order) -> None: ...

# Caller (e.g. a checkout handler) uses each for its one job:
repository.save(order)
notifier.send_confirmation(order)
```
- **Link:** SRP is SoC applied to one class.

### O: Open/Closed
- **Crux:** open for extension, closed for modification. Add new behaviour by adding a new class, not by editing working code.
- **How:** an interface or abstract class, plus one implementation per variant (`PaymentMethod` → `CreditCardPayment`, `CryptoPayment`).
- **Why:** every edit to working code risks breaking it. New code in a new class can't break the old code.
- **Textbook violation (the article's example):** `PaymentProcessor.process(type, amount)` with `if credit / elif paypal`, where adding crypto means editing the method. The fix: `PaymentProcessor.process(method: PaymentMethod, amount)`, and crypto becomes a new `CryptoPayment` class.

```python
# Bad: adding crypto = editing working code
class PaymentProcessor:
    def process(self, payment_type: str, amount: float):
        if payment_type == "credit": ...
        elif payment_type == "paypal": ...

# Good: adding crypto = adding a class, nothing else edited
class PaymentMethod(ABC):
    @abstractmethod
    def process(self, amount: float) -> None: ...

class CreditCardPayment(PaymentMethod):
    def process(self, amount: float) -> None: ...
class PayPalPayment(PaymentMethod):
    def process(self, amount: float) -> None: ...
class CryptoPayment(PaymentMethod):            # new class; PaymentProcessor untouched
    def process(self, amount: float) -> None: ...

class PaymentProcessor:                        # depends on the abstraction (DIP)
    def process(self, method: PaymentMethod, amount: float):
        method.process(amount)

# Picking the class from a request string ("credit" -> CreditCardPayment) is a
# separate job: one dict or factory at the edge. Adding crypto = new class + one line there.
```

- **When *not* to:** with 2 stable cases, an `if/elif` is fine (KISS/YAGNI). Refactor to a strategy when a 3rd case arrives, or when the requirements say cases will keep growing.
  - Interview answer to "this violates OCP": *"Yes, deliberately. There are only 2 types today. When a 3rd comes, I'll extract a `PaymentMethod` interface."*

### L: Liskov Substitution
- **Crux:** a child must work anywhere the parent is expected. It can **add** behaviour but must not **remove or break** any of the parent's behaviour.
- **It's about behaviour, not methods.** Python gives the child every method anyway. What matters is keeping the parent's promises:
  - no surprise exceptions
  - no stricter input rules
  - no weaker results
- **Red flags:**
  - a child raises an exception for a method the parent provides
  - callers need special-case code such as `if isinstance(bird, Penguin)`
- **Examples:**
  - `Penguin(Bird)` whose `fly()` raises an exception: every `for b in birds: b.fly()` breaks.
  - `Square(Rectangle)`: `set_width(5); set_height(10)` should give an area of 50, but Square gives 100.
- **Fix:** don't put behaviour in the base class that not every child has. Split it: `Bird(eat)` → `FlyingBird(Bird, fly)`, so `Sparrow(FlyingBird)` and `Penguin(Bird)`. Or use composition.

```python
# Bad: Penguin breaks the parent's promise
class Bird:
    def fly(self) -> None: ...
class Penguin(Bird):
    def fly(self) -> None:
        raise NotImplementedError   # `for b in birds: b.fly()` now crashes

# Good: only birds that fly promise fly()
class Bird(ABC):
    @abstractmethod
    def eat(self) -> None: ...
class FlyingBird(Bird):
    @abstractmethod
    def fly(self) -> None: ...

class Sparrow(FlyingBird):
    def eat(self) -> None: ...
    def fly(self) -> None: ...
class Penguin(Bird):
    def eat(self) -> None: ...
```
- **In interviews:** decide carefully which methods go on the base class and which go on subclasses.

### I: Interface Segregation
- **Crux:** a class should need every method of the interfaces it implements. If it has to stub out methods it doesn't use, the interface is too big. Split it into smaller interfaces and implement only the ones you need.
- **Smell:** empty methods, or methods that raise, just to satisfy an interface.
- **Example:** `Worker(work, eat, sleep)` forces `Robot` to fake `eat` and `sleep`. Split it into `Workable`, `Feedable` and `Restable`:
  - `Human` has `work`, `eat` and `sleep`, so it fits all three.
  - `Robot` has only `work`, so it fits only `Workable`.

  In Python, use small `Protocol`s (classes match by having the methods, no inheritance needed) or ABCs. An interface only matters where some code *asks* for it, so the example shows that caller.

```python
# Bad: Robot forced to fake methods it doesn't need
class Worker(ABC):
    @abstractmethod
    def work(self): ...
    @abstractmethod
    def eat(self): ...
    @abstractmethod
    def sleep(self): ...

class Robot(Worker):
    def work(self): ...
    def eat(self): pass      # smell
    def sleep(self): pass    # smell

# Good: small interfaces, implement only what applies
class Workable(Protocol):
    def work(self) -> None: ...
class Feedable(Protocol):
    def eat(self) -> None: ...
class Restable(Protocol):
    def sleep(self) -> None: ...

class Human:                 # fits Workable, Feedable, Restable
    def work(self) -> None: ...
    def eat(self) -> None: ...
    def sleep(self) -> None: ...
class Robot:                 # fits Workable only, no fake methods
    def work(self) -> None: ...

# Who uses the interfaces: each caller asks only for what it needs
def run_shift(workers: list[Workable]) -> None:
    for w in workers: w.work()

def serve_lunch(eaters: list[Feedable]) -> None:
    for e in eaters: e.eat()

run_shift([Human(), Robot()])   # both fit
serve_lunch([Human()])          # serve_lunch([Robot()]): mypy/IDE flags it; at runtime it crashes on .eat()
```

### D: Dependency Inversion
- **Crux:** don't depend on a concrete implementation. High-level logic and low-level details both depend on an **abstraction**.
- **What gets "inverted":** who defines the contract.
  - Normally, business logic bends to whatever API the implementation offers.
  - With DIP, business logic defines the interface it *needs* (`MessageSender.send(message)`), and implementations conform to it. The implementation adapts to the business, not the other way round.
- **Example:** `NotificationService` doing `self.sender = EmailSender()` is tightly coupled. Take a `sender: MessageSender` in the constructor instead. Then you can:
  - swap in SMS without touching the service
  - pass a fake sender in tests

```python
# Bad: high-level service creates the low-level detail itself
class NotificationService:
    def __init__(self):
        self.sender = EmailSender()     # can't swap, can't test without real email

# Good: service defines what it needs; implementations conform
class MessageSender(ABC):
    @abstractmethod
    def send(self, message: str) -> None: ...

class EmailSender(MessageSender):
    def send(self, message: str) -> None: ...
class SmsSender(MessageSender):
    def send(self, message: str) -> None: ...

class NotificationService:
    def __init__(self, sender: MessageSender):   # dependency injection
        self.sender = sender
    def notify(self, message: str) -> None:
        self.sender.send(message)

# Who picks the implementation: whoever creates the service (app startup / tests)
NotificationService(EmailSender()).notify("Order shipped")
NotificationService(SmsSender()).notify("Order shipped")   # swap, service unchanged

class FakeSender(MessageSender):                           # test double
    def __init__(self): self.sent: list[str] = []
    def send(self, message: str) -> None: self.sent.append(message)

fake = FakeSender()
NotificationService(fake).notify("hi")
assert fake.sent == ["hi"]                                 # no real email sent
```
- **DIP vs dependency injection:**
  - DIP is the *principle*.
  - Injection (passing dependencies in through the constructor) is the *technique* that achieves it.
- **When *not* to:** if there's one implementation and no stated variation, an interface is just YAGNI. Still inject the dependency through the constructor: in Python a test fake needs no ABC, only the same method.

---

## Review questions

Answer first, then open the answer to check.

1. Why is "the child has all the parent's methods" the wrong definition of LSP? Explain with Penguin or Square, and name two red flags.

   <details><summary>Answer</summary>

   - LSP is about keeping the parent's promises, not having its methods. In Python the child inherits every method anyway.
   - `Penguin` has `fly()`, but it raises, so `for b in birds: b.fly()` crashes.
   - `Square`: `set_width(5); set_height(10)` should give area 50, and gives 100.
   - Red flags: a child raises for a method the parent provides; callers need `isinstance` special cases.

   </details>

2. `Penguin.fly()` raises and `Robot.eat()` does nothing. One is an LSP violation and the other an ISP violation: which is which, why, and how do the fixes compare?

   <details><summary>Answer</summary>

   - `Penguin` is LSP: seen from the *caller*. Code written against `Bird` breaks when handed a `Penguin`.
   - `Robot` is ISP: seen from the *implementer*. The `Worker` interface is too big and forces stub methods.
   - The fixes have the same shape: split the contract so each class promises only what it can do (`FlyingBird`; `Workable` / `Feedable` / `Restable`).

   </details>

3. The interviewer says your 2-branch `if/elif` violates OCP. What do you answer?

   <details><summary>Answer</summary>

   "Yes, deliberately. There are only 2 stable types today, so the `if/elif` is simpler. When a 3rd comes, or if you tell me more are coming, I'll extract a `PaymentMethod` interface."

   </details>

4. After the OCP fix, `PaymentProcessor` has no `if/elif`. Where did the choice of class go, and what does adding crypto now cost?

    <details><summary>Answer</summary>

    - To the edge: one dict or factory that maps the request string to a class. The choice didn't vanish; it now lives in one place.
    - Adding crypto: one new `CryptoPayment` class plus one line in that mapping. `PaymentProcessor` is untouched.

    </details>

5. `NotificationService` creates `EmailSender()` itself. Which principle does that break, what exactly gets "inverted" in the fix, and how does it help testing? How is DIP different from dependency injection?

    <details><summary>Answer</summary>

    - DIP. Fix: take `sender: MessageSender` in the constructor.
    - Inverted: who defines the contract. Business logic defines the interface it needs, and implementations conform to it.
    - Testing: pass a `FakeSender` and assert on what it recorded; no real email is sent.
    - DIP is the principle. Injection is the technique that achieves it.

    </details>

6. In Python a test fake needs no ABC. With one real implementation, what do you still do, and what do you skip?

    <details><summary>Answer</summary>

    - Still do: inject the dependency through the constructor. That alone gives you swapping and test fakes.
    - Skip: the ABC. With one implementation and no stated variation it's YAGNI. Add it when a second implementation arrives.

    </details>

7. Which SOLID principles pull against KISS and YAGNI, why does the article warn against over-applying SOLID, and how do you balance them?

    <details><summary>Answer</summary>

    - OCP and DIP, because they add abstractions.
    - Warning: SOLID comes from Java-era deep hierarchies and interface-heavy design. Outside Java and C#, overusing it is going out of fashion, and forcing it breaks KISS.
    - Balance: start simple, name the seam, add the abstraction when real variation arrives, and say the trade-off out loud.

    </details>
