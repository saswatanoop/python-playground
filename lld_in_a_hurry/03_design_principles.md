# Design Principles

Source: Hello Interview LLD → In a Hurry → Design Principles. Learned 10 Nov 2026.

## Review points

- Two groups: **general** (KISS, DRY, YAGNI, SoC, Law of Demeter) and **object-oriented** (SOLID; OOD gets more interview attention). The three general ones to remember: KISS, DRY, YAGNI.
- Interviewers want you to *apply* the principles and explain your calls; reciting acronyms earns nothing.
- **KISS:** simplest thing that works. Add complexity only when simplicity breaks.
- **DRY:** same *logic* goes in one place. Code that merely *looks* similar can stay duplicated.
- **YAGNI:** think ahead, don't build ahead.
- **SoC:** each part owns one responsibility and doesn't know the others' internals.
- **Law of Demeter:** talk only to your immediate friends. Don't reach through a chain of different objects.
- **SRP:** one reason to change. **OCP:** extend by adding, not editing. **LSP:** a child keeps every parent promise; it can add, never remove. **ISP:** small interfaces; no stub methods. **DIP:** business logic owns the interface, and implementations conform to it (through injection).
- **SOLID comes from Java-era deep hierarchies.** Outside Java and C#, overusing it is going out of fashion: modern code favours composition over hierarchies and functions over interfaces. Apply SOLID only when the problem calls for it, or you break KISS.

**What each one buys you** (the article's cheat sheet):

| Principle | Buys you |
|---|---|
| KISS | Start simple, add complexity only when needed |
| DRY | Less duplication, easier maintenance |
| YAGNI | Build for today, not hypothetical futures |
| SoC | Independent testing and changes |
| Law of Demeter | Less coupling, hidden internal structure |
| SRP | Classes focused on one responsibility |
| OCP | Future requirements without modifying existing code |
| LSP | No brittle hierarchies that break at runtime |
| ISP | Clean, focused interfaces |
| DIP | Business logic decoupled from implementation details |

- Senior signal: name the tension (DRY vs KISS, YAGNI vs extensibility, OCP/DIP vs KISS) and say which way you lean *and why*.
- ⚠ **My weak spot (missed twice):** *when not to abstract*. With one or two stable cases, keep it simple and name the seam. Ask whether variation is coming before you build an interface. Test this first in a review.

---

## General principles

### KISS: Keep It Simple, Stupid
- **Crux:** pick the simplest solution that works. If a conditional does the job, don't write a strategy pattern.
- **Most-violated principle in LLD interviews.** Over-engineering usually comes from showing off patterns (factories, builders, decorators), and interviewers notice it.
- **When to add complexity:**
  - a class grows into a god class (about 500 lines, many responsibilities)
  - adding one variant (e.g. a payment method) means editing several places

### DRY: Don't Repeat Yourself
- **Crux:** logic that is *conceptually the same* lives in one place, so a fix or a rule change happens once.
- **Don't overdo it.** If code is textually similar but means different things, sharing it creates *artificial coupling*: a change made for one breaks the other.
  - Example: `ParkingTicket` computes `hours * rate` and `LibraryLoan` computes `days * fine`. Keep them separate; they will evolve differently.

```python
# DRY: same rule, one place
def is_valid_email(email: str) -> bool:
    return "@" in email and "." in email.split("@")[-1]

class User:
    def __init__(self, email: str):
        if not is_valid_email(email): raise ValueError("bad email")
        self.email = email

class Newsletter:
    def subscribe(self, email: str):
        if not is_valid_email(email): raise ValueError("bad email")

# NOT DRY-worthy: looks alike, means different things. Keep separate.
class ParkingTicket:
    def fee(self) -> float: return self.hours * self.hourly_rate     # may add peak pricing
class LibraryLoan:
    def fine(self) -> float: return self.days_late * self.fine_per_day  # may add a cap
```
- **DRY vs KISS:** sometimes duplicating is simpler than adding an abstraction. A line to use in an interview: *"I'll keep this in `User` for now. If it shows up 3–4 times, I'll pull it into a shared validator."*

### YAGNI: You Aren't Gonna Need It
- **Crux:** think ahead, but don't build ahead. Design with extension in mind and implement only what the requirements ask for.
- **Why:** guesses about the future are usually wrong. You end up maintaining dead code, and the real requirement still doesn't fit.
- **In interviews:** no valet parking or EV charging unless the requirements ask for it. Save extensions for when the interviewer asks "how would you extend this?", then *talk* through them.
- **Example, a parking lot with one pricing rule:**
  - Don't build a `PricingStrategy` interface on a guess that rules will change.
  - Ask the interviewer: "Will there be multiple pricing rules?"
    - If yes, the strategy is justified.
    - If no, keep the fee calculation in a single `calculate_fee()` method. That is the one seam where a strategy can slot in later.

```python
class ParkingLot:
    HOURLY_RATE = 20

    def calculate_fee(self, ticket: Ticket) -> float:   # the seam: only place pricing lives
        return ticket.hours() * self.HOURLY_RATE

# Pricing inside ParkingLot is a 2nd responsibility (SRP), accepted on purpose (KISS):
# one line of logic doesn't earn its own class yet. Say that out loud in the interview.

# Later, only if a 2nd rule appears:
#   calculate_fee() -> self.pricing.calculate(ticket)   (PricingStrategy injected)
```

### Separation of Concerns (SoC)
- **Problem:** if one class mixes UI, input, rules and storage, every change risks breaking unrelated things, and you can't test any one part alone.
- **Crux:** split the code by responsibility (UI / business logic / data access). Each part talks to the others through a small interface and never uses their internals.
- **Example: TicTacToe.**
  - Bad: `play()` prints the board, reads `input()` and checks for a win, all in one method. Win logic can't be tested without stdin, and moving to a GUI means rewriting the game loop.
  - Good: `TicTacToe(board, display, input_handler)`. Each part changes on its own:
    - console → GUI: only `InputHandler` changes
    - new look: only `Display` changes
    - new win rule: only `Board` changes

```python
# Bad: display + input + rules in one method
class TicTacToe:
    def play(self):
        while True:
            for row in self.board: print(row)          # display
            r, c = int(input()), int(input())          # input
            self.board[r][c] = "X"
            if self.board[0][0] == self.board[1][1] == self.board[2][2]:  # rules
                print("Winner!"); break

# Good: each concern in its own class, the game just coordinates
class TicTacToe:
    def __init__(self, board: Board, display: Display, input_handler: InputHandler):
        self.board, self.display, self.input_handler = board, display, input_handler

    def play(self):
        while not self.board.is_game_over():           # win OR full board (draw)
            self.display.render(self.board)
            self.board.make_move(self.input_handler.get_next_move())
        self.display.show_result(self.board.get_winner())   # None = draw
```
- **SoC vs SRP:** SoC is the broad idea and applies at any level (layers, modules, classes). SRP is SoC applied to one class: *one reason to change*.

### Law of Demeter (principle of least knowledge)
- **Problem:** `garage.get_floor(2).get_spot(14).get_vehicle().license_plate` makes the caller depend on the internal structure of Garage, Floor, Spot and Vehicle. If any of them reorganizes, the caller breaks.
- **Crux:** a method talks only to its immediate friends: itself, its fields, its parameters, and objects it creates. It doesn't reach through them to strangers.
- **Fix:**
  - Give the top object a method that does the navigation for the caller: `garage.get_plate_at(floor, spot)`. Stricter form: each object delegates one step down (`floor.get_plate_at(spot)` → `spot.get_plate()`).
  - Or return exactly the data the caller needs, not an object graph it has to dig through.
- **Not a violation: fluent chaining.** In `query.where(...).order_by(...).limit(10)` and `builder.set_name().build()`, every call returns the *same type*, so no internal structure leaks. The rule is about crossing *different* object types.

```python
# Bad: caller knows Garage -> Floor -> Spot -> Vehicle
plate = garage.get_floor(2).get_spot(14).get_vehicle().license_plate

# Good: each object talks only to its direct friend and delegates one step down
class Spot:
    def get_plate(self) -> str | None:
        return self.vehicle.license_plate if self.vehicle else None   # vehicle is Spot's own field

class Floor:
    def get_plate_at(self, spot: int) -> str | None:
        return self.spots[spot].get_plate()

class Garage:
    def get_plate_at(self, floor: int, spot: int) -> str | None:
        return self.floors[floor].get_plate_at(spot)

plate = garage.get_plate_at(2, 14)

# Fine: fluent chaining, every call returns the same Query type
query.where("age > 30").order_by("name").limit(10)
```

---

## Object-oriented principles: SOLID

Code: [../SOLID/solid_in_python.py](../SOLID/solid_in_python.py). Composition over inheritance is covered in [04_oop_concepts.md](04_oop_concepts.md); it's not on this page.

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

1. When does DRY hurt? Give an example of similar code that should stay duplicated.

   <details><summary>Answer</summary>

   - When code only *looks* alike but means different things. Sharing it creates artificial coupling: a change made for one breaks the other.
   - Example: `ParkingTicket` (`hours * rate`) and `LibraryLoan` (`days * fine`) will evolve differently, so keep them separate.
   - Also when the abstraction is more complex than two copies (DRY vs KISS). Extract when it shows up 3–4 times.

   </details>

2. The interviewer sees you adding a factory and a strategy for a problem with one variant. Which principles are you breaking, and what should trigger adding them?

   <details><summary>Answer</summary>

   - KISS and YAGNI.
   - Triggers: a real second or third variant, requirements that say variants will keep growing, or adding a variant means editing several places.

   </details>

3. Parking lot with one pricing rule: interface now or not? What do you ask, and where do you leave the seam? The interviewer then says pricing inside `ParkingLot` violates SRP: defend it, and say what would change your mind.

   <details><summary>Answer</summary>

   - Not now. Ask: "Will there be multiple pricing rules?" If yes, a `PricingStrategy` is justified.
   - If no, keep pricing in one `calculate_fee()` method. That is the seam.
   - Defence: yes, it's a second responsibility, accepted on purpose. One line of logic doesn't earn its own class (KISS).
   - Change my mind: a second pricing rule appears. Then `calculate_fee()` delegates to an injected `PricingStrategy`.

   </details>

4. What concrete signals tell you to split a class or add a pattern, and what tells you to stop splitting?

   <details><summary>Answer</summary>

   - Split: the class is becoming a god class (about 500 lines, many responsibilities), adding one variant means editing several places, or a part can't be tested alone.
   - Stop: one class does the job without getting messy, the new class would hold a line or two, or there are only one or two stable cases and no stated variation.

   </details>

5. TicTacToe does display, input and win-checking in one `play()`. Name two concrete problems and the split that fixes them. Is that split SoC or SRP, and what is the difference?

   <details><summary>Answer</summary>

   - Problems: win logic can't be tested without stdin, and moving to a GUI means rewriting the game loop.
   - Split: `Board` (rules), `Display`, `InputHandler`; `TicTacToe` only coordinates.
   - Both. SoC is the broad idea at any level (layers, modules, classes). SRP is SoC applied to one class: one reason to change.

   </details>

6. Is `query.where().order_by().limit()` a Law of Demeter violation? Is `order.get_customer().get_address().get_zip()`? Why, and how do you fix the second?

   <details><summary>Answer</summary>

   - First: no. Every call returns the same type, so no internal structure leaks.
   - Second: yes. The caller depends on the internals of `Order`, `Customer` and `Address`, and breaks if any of them reorganizes.
   - Fix: `order.get_customer_zip()` does the navigation (strict form: each object delegates one step down), or return just the data the caller needs.

   </details>

7. Why is "the child has all the parent's methods" the wrong definition of LSP? Explain with Penguin or Square, and name two red flags.

   <details><summary>Answer</summary>

   - LSP is about keeping the parent's promises, not having its methods. In Python the child inherits every method anyway.
   - `Penguin` has `fly()`, but it raises, so `for b in birds: b.fly()` crashes.
   - `Square`: `set_width(5); set_height(10)` should give area 50, and gives 100.
   - Red flags: a child raises for a method the parent provides; callers need `isinstance` special cases.

   </details>

8. `Penguin.fly()` raises and `Robot.eat()` does nothing. One is an LSP violation and the other an ISP violation: which is which, why, and how do the fixes compare?

   <details><summary>Answer</summary>

   - `Penguin` is LSP: seen from the *caller*. Code written against `Bird` breaks when handed a `Penguin`.
   - `Robot` is ISP: seen from the *implementer*. The `Worker` interface is too big and forces stub methods.
   - The fixes have the same shape: split the contract so each class promises only what it can do (`FlyingBird`; `Workable` / `Feedable` / `Restable`).

   </details>

9. The interviewer says your 2-branch `if/elif` violates OCP. What do you answer?

   <details><summary>Answer</summary>

   "Yes, deliberately. There are only 2 stable types today, so the `if/elif` is simpler. When a 3rd comes, or if you tell me more are coming, I'll extract a `PaymentMethod` interface."

   </details>

10. After the OCP fix, `PaymentProcessor` has no `if/elif`. Where did the choice of class go, and what does adding crypto now cost?

    <details><summary>Answer</summary>

    - To the edge: one dict or factory that maps the request string to a class. The choice didn't vanish; it now lives in one place.
    - Adding crypto: one new `CryptoPayment` class plus one line in that mapping. `PaymentProcessor` is untouched.

    </details>

11. `NotificationService` creates `EmailSender()` itself. Which principle does that break, what exactly gets "inverted" in the fix, and how does it help testing? How is DIP different from dependency injection?

    <details><summary>Answer</summary>

    - DIP. Fix: take `sender: MessageSender` in the constructor.
    - Inverted: who defines the contract. Business logic defines the interface it needs, and implementations conform to it.
    - Testing: pass a `FakeSender` and assert on what it recorded; no real email is sent.
    - DIP is the principle. Injection is the technique that achieves it.

    </details>

12. In Python a test fake needs no ABC. With one real implementation, what do you still do, and what do you skip?

    <details><summary>Answer</summary>

    - Still do: inject the dependency through the constructor. That alone gives you swapping and test fakes.
    - Skip: the ABC. With one implementation and no stated variation it's YAGNI. Add it when a second implementation arrives.

    </details>

13. Which SOLID principles pull against KISS and YAGNI, why does the article warn against over-applying SOLID, and how do you balance them?

    <details><summary>Answer</summary>

    - OCP and DIP, because they add abstractions.
    - Warning: SOLID comes from Java-era deep hierarchies and interface-heavy design. Outside Java and C#, overusing it is going out of fashion, and forcing it breaks KISS.
    - Balance: start simple, name the seam, add the abstraction when real variation arrives, and say the trade-off out loud.

    </details>
