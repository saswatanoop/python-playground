# OOP Concepts

Source: Hello Interview LLD → In a Hurry → OOP Concepts. Learned 16 Nov 2026.

Snippets assume `from abc import ABC, abstractmethod`. `SpotSize` is an `Enum`, and `ParkingLot._find_spot(size)` returns a free spot of that size or `None`.

## Review points

- **Encapsulation = expose operations, not fields with guards.** A validated setter protects a *value*; domain methods (`deposit`, `withdraw`) protect the *rules*: why state changed and how it stays consistent with the rest of the object.
- Hygiene check interviewers run: no public mutable fields, no getters that return internal collections.
- Python can't make anything truly private. `_x` is a convention, `__x` is name mangling (avoids subclass clashes), and `@property` without a setter blocks only reassignment. Real enforcement is API design.
- **Abstraction:** callers depend on *what*, never *how*. Pitch the interface at what the caller needs to do: not `execute(dict)` (too generic), not `send_email()` on a general notifier (too specific).
- **Polymorphism:** one call, each object runs its own version, no branching on type. It replaces `if/elif` on a type. In Python it needs no parent class (duck typing, `Protocol`). Cost: harder to trace, so keep `if/elif` for a few stable types.
- **Inheritance:** shares implementation at the cost of tight coupling (fragile base class). The test: children *reuse* the parent unchanged → inherit. Children *replace* behaviour → compose (`Car(drivetrains)`). Default to interfaces plus composition.

| Concept | One-liner |
|---|---|
| Encapsulation | Hide state, expose operations |
| Abstraction | Interface where things vary; pitched at the caller's need |
| Polymorphism | Let objects handle themselves; no type checks |
| Inheritance | Compose behaviour; inherit only stable, shared implementation |

---

### Encapsulation

- **Crux:** the object owns its state and exposes **operations**, not fields. Every state change goes through a method that keeps the invariants intact.
- **Getters and setters are not encapsulation.** Even a validated `set_balance(x)` falls short of `deposit()` / `withdraw()`:
  - **Logic leaks to callers:** `acc.set_balance(acc.get_balance() + 100)` puts the business calculation outside the class, duplicated in every caller.
  - **No intent:** going from 500 to 1000 could be a deposit, a refund or a bug. The setter can't log the transaction type, keep the history consistent with the balance, or apply per-operation rules such as a withdrawal limit or a fee.
  - **Race:** get-then-set is a read-modify-write split across two calls, so two threads can lose an update. `deposit()` can lock the whole operation.
- **Don't leak mutable internals.** Best is not exposing the collection at all ("tell, don't ask": `available_count()`, `is_full()`). If callers need the data, return a copy, a `tuple`, or a `MappingProxyType`.

```python
class Account:
    def __init__(self):
        self._balance = 0.0
        self._history: list[tuple[str, float]] = []

    def deposit(self, amount: float) -> None:
        if amount <= 0: raise ValueError("amount must be positive")
        self._balance += amount
        self._history.append(("deposit", amount))   # intent recorded, history stays in sync

    def withdraw(self, amount: float) -> None:
        if amount <= 0: raise ValueError("amount must be positive")
        if amount > self._balance: raise ValueError("insufficient funds")
        self._balance -= amount
        self._history.append(("withdraw", amount))

    @property
    def balance(self) -> float:                      # read-only: no setter
        return self._balance

    @property
    def history(self) -> tuple[tuple[str, float], ...]:
        return tuple(self._history)                  # caller can't mutate the real list
```

**What `@property` and the prefixes actually do in Python:**

```python
acc = Account()
acc.balance = 500         # AttributeError: no setter        ✔ blocked
acc.history.append(...)   # AttributeError: tuple            ✔ blocked (returned a tuple)
acc._balance = 500        # works: `_` is only a convention  ✘
# self.__x is stored as _Account__x: still writable; mangling only avoids subclass name clashes
```

---

### Abstraction

- **Crux:** the caller never needs to know the implementation. It depends on *what* something does (a contract), never *how*.
- **Payoff:** swap implementations without touching callers. `OrderService` calls `process(amount)` and doesn't care whether it hits Stripe's API or a stored token.
  - Bad: `OrderService.checkout()` creates `StripeAPI()`, sets the key and calls `create_charge()`. Changing providers means editing `OrderService`.
  - Good: `OrderService(payment_method: PaymentMethod)` calls `payment_method.process(order.total)`.
- **Where:** only where there's variation or messy detail (several payment providers, vehicle types, storage back-ends), i.e. logic that feels tangled or requirements that suggest multiple approaches. Not everywhere.
- **Right level:** design the interface around what the **caller needs to do**.

  | Level | Example | Problem |
  |---|---|---|
  | Too generic | `execute(payload: dict) -> dict` | Callers still need provider-specific keys, so the implementation leaks. No type safety. |
  | Too specific | `Notifier.send_email(to, subject, body)` | Bakes one implementation into the contract, so SMS and push don't fit. |
  | Right | `Notifier.notify(recipient, message)`, `PaymentMethod.process(amount)` | Says what the caller wants; each implementation maps it to its own details (email adds a subject, SMS truncates). |

- **vs encapsulation:** encapsulation **protects state** (who can change it, and how). Abstraction **hides complexity** (what, not how).
- **vs DIP:** abstraction hides *how* behind a contract. DIP is about *who owns* the contract (business logic) and which way the dependency points. You can have an abstraction without DIP, e.g. a vendor SDK's own interface that your code bends to fit.

### Polymorphism

- **Crux:** the caller calls one method through an interface, and each object runs its own version. **The caller never branches on type.**
- **Trigger:** an `if/elif` or `match` on a type or kind enum. That's the sign to use polymorphism.
- **Follows from abstraction:** once the interface exists, which implementation runs depends on the concrete type passed in.
- **Python needs no parent class.** Any object with the method works:
  - **ABC (nominal):** classes must inherit it explicitly, and instantiation fails if an abstract method is missing. Use it when you own the hierarchy and want that check.
  - **`Protocol` (structural):** a class fits if it has the methods, no inheritance needed. Use it for third-party classes, or to keep implementations decoupled. Checked by type checkers, not at runtime.
- **Trade-off:** the flow is harder to trace, debug and onboard onto as implementations multiply, because you can't see which code runs from the call site. Companies differ: some prefer explicit branches per type, others the extensibility. With a few types that won't change, an explicit `if/elif` is fine. Be ready to argue both sides in the interview.
- **Responsibility stays where it belongs.** The vehicle *reports* what it needs, and the lot still *does* the parking. Polymorphism doesn't mean "the object does everything".

```python
class Vehicle(ABC):
    @abstractmethod
    def required_spot_size(self) -> SpotSize: ...

class Car(Vehicle):
    def required_spot_size(self) -> SpotSize: return SpotSize.REGULAR
class Truck(Vehicle):
    def required_spot_size(self) -> SpotSize: return SpotSize.LARGE

class ParkingLot:
    def park(self, vehicle: Vehicle) -> bool:
        spot = self._find_spot(vehicle.required_spot_size())   # no type check
        if spot is None: return False
        spot.occupy(vehicle)
        return True
# New vehicle type = new class. ParkingLot is untouched (OCP).
```

### Inheritance vs composition

- **Crux:** inheritance shares *implementation*, at the cost of tight coupling. A change in the parent can break every child: the **fragile base class** problem.
- **Use it only when:** the base is stable, and children **reuse the parent's behaviour unchanged**, adding to it rather than replacing it. `SavingsAccount` and `CheckingAccount` both reuse `BankAccount`'s deposit, withdraw and history.
- **The one test:** does the child reuse the parent's implementation, or replace it with something different? If it replaces, inheritance is the wrong tool (extending via `super()` is fine). This also keeps you on the right side of LSP.
- **Classic interview mistake:** using inheritance to model *behaviour differences*. `ElectricCar(Car)` overrides `start_engine()` with completely different logic and inherits engine concepts it doesn't have. When a hybrid arrives, neither `Car` nor `ElectricCar` is a clean parent.
- **Fix:** pull the varying behaviour out into its own abstraction and **compose** it.

```python
class Drivetrain(ABC):
    @abstractmethod
    def start(self) -> None: ...

class GasEngine(Drivetrain):
    def start(self) -> None: ...
class ElectricMotor(Drivetrain):
    def start(self) -> None: ...

class Car:
    def __init__(self, drivetrains: list[Drivetrain]):
        self._drivetrains = drivetrains
    def start(self) -> None:
        for d in self._drivetrains: d.start()

hybrid = Car([GasEngine(), ElectricMotor()])   # no new class needed
# Hydrogen car = new Drivetrain implementation. Car is untouched.
```

- **Default in interviews:** interfaces plus composition. You keep abstraction and polymorphism without forcing a parent-child relationship or shared state. Most LLD problems need no inheritance beyond implementing an interface.

---

### In the interview

Don't recite the terms; let them show in the design. Requirement cues:

| You hear or see | Do |
|---|---|
| "Support multiple payment methods / vehicle types" | Define an interface (abstraction) |
| Designing any class | Private fields, expose operations (encapsulation) |
| Yourself writing type checks | Interface plus implementations (polymorphism) |
| Behaviour that varies by type | Compose it; don't subclass it (inheritance) |

---

## Review questions

Answer first, then open the answer to check.

1. A teammate adds validation to `set_balance()` and says the class is now encapsulated. What do you say?

   <details><summary>Answer</summary>

   - It protects a value, not the rules. The caller still computes the new balance, so the logic leaks.
   - The setter can't tell a deposit from a refund or a bug, so there's no transaction log and no per-operation rules.
   - Get-then-set can lose updates under concurrency.
   - Expose operations (`deposit`, `withdraw`), not fields with guards.

   </details>

2. `ParkingLot.spots` returns `self._spots`. What breaks, and what are your options, best first?

   <details><summary>Answer</summary>

   - Callers can mutate the internal list and bypass every rule.
   - Best: don't expose it. Offer operations (`available_count()`, `is_full()`).
   - Otherwise: return a copy, a `tuple`, or a read-only view (`MappingProxyType` for a dict).

   </details>

3. What does `@property` without a setter prevent, and what doesn't it prevent? What do `_x` and `__x` do?

   <details><summary>Answer</summary>

   - Prevents: reassigning the public name (`obj.spots = ...` raises `AttributeError`).
   - Doesn't prevent: mutating a returned mutable object, or writing `obj._spots` directly.
   - `_x` is a convention only. `__x` is renamed to `_Class__x` to avoid subclass clashes; it is still writable.

   </details>

4. What's the difference between encapsulation and abstraction? And between abstraction and DIP?

   <details><summary>Answer</summary>

   - Encapsulation protects state (who can change it, and how). Abstraction hides complexity (callers see what, not how).
   - Abstraction hides how behind a contract. DIP decides who owns the contract (business logic) and which way the dependency points.

   </details>

5. Your `Notifier` has `send_email(to, subject, body)`, and now SMS and push are needed. Fix it. Then a teammate proposes `execute(payload: dict)`. Why is that wrong too?

   <details><summary>Answer</summary>

   - `send_email` is too specific: it bakes one implementation into the contract. Use `notify(recipient, message)`; each channel maps it to its own details.
   - `execute(dict)` is too generic: callers still need channel-specific keys, so the implementation leaks, and there's no type safety.
   - Pitch the interface at what the caller needs to do.

   </details>

6. `ParkingLot.park()` branches on `vehicle.type`. Refactor it. When would you keep the `if/elif`? Does the vehicle now park itself?

   <details><summary>Answer</summary>

   - Each `Vehicle` implements `required_spot_size()`, and the lot calls it with no type check. A new type is a new class, and the lot is untouched.
   - Keep the `if/elif` for a few stable types. Polymorphism makes the flow harder to trace.
   - No. The vehicle reports what it needs, and the lot still does the parking (SRP).

   </details>

7. ABC vs `Protocol`: what's the difference, and when do you use each?

   <details><summary>Answer</summary>

   - ABC is nominal: classes inherit explicitly, and instantiation fails if an abstract method is missing. Use it when you own the hierarchy and want that check.
   - `Protocol` is structural: a class fits if it has the methods. Checked by type checkers, not at runtime. Use it for third-party classes, or to keep implementations decoupled.

   </details>

8. `SavingsAccount(BankAccount)` is fine and `ElectricCar(Car)` isn't. What's the test? What breaks when a hybrid arrives, and what's the fix?

   <details><summary>Answer</summary>

   - Test: does the child reuse the parent's implementation unchanged (inherit), or override it to do something different (compose)?
   - `ElectricCar` replaces `start_engine()` and inherits engine concepts it lacks. A hybrid fits under neither `Car` nor `ElectricCar`.
   - Fix: a `Drivetrain` interface composed into `Car`. A hybrid is `Car([GasEngine(), ElectricMotor()])`, with no new class.

   </details>

9. What is the fragile base class problem, and why does it make composition the default?

   <details><summary>Answer</summary>

   - Children depend on the parent's implementation, so a change in the parent can silently break every child.
   - Composition depends only on an interface. Implementations change independently, and behaviour can be mixed (hybrid) without new subclasses.

   </details>
