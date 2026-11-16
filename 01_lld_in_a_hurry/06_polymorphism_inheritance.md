# Polymorphism and inheritance

Source: Hello Interview LLD → In a Hurry → OOP Concepts (part 2 of 2). Learned 16 Nov 2026. Previous: [05_encapsulation_abstraction.md](05_encapsulation_abstraction.md).

Snippets assume `from abc import ABC, abstractmethod`. `SpotSize` is an `Enum`, and `ParkingLot._find_spot(size)` returns a free spot of that size or `None`.

## What this note covers

- Polymorphism in place of type checks, and what it costs
- ABC vs `Protocol`
- Inheritance vs composition: the reuse-or-replace test
- Requirement cues for all four OOP concepts

## Review points

- **Polymorphism:** one call, each object runs its own version, no branching on type. It replaces `if/elif` on a type. In Python it needs no parent class (duck typing, `Protocol`). Cost: harder to trace, so keep `if/elif` for a few stable types.
- **Inheritance:** shares implementation at the cost of tight coupling (fragile base class). The test: children *reuse* the parent unchanged → inherit. Children *replace* behaviour → compose (`Car(drivetrains)`). Default to interfaces plus composition.

| Concept | One-liner |
|---|---|
| Polymorphism | Let objects handle themselves; no type checks |
| Inheritance | Compose behaviour; inherit only stable, shared implementation |

---

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

### In the interview (all four OOP concepts)

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

1. `ParkingLot.park()` branches on `vehicle.type`. Refactor it. When would you keep the `if/elif`? Does the vehicle now park itself?

   <details><summary>Answer</summary>

   - Each `Vehicle` implements `required_spot_size()`, and the lot calls it with no type check. A new type is a new class, and the lot is untouched.
   - Keep the `if/elif` for a few stable types. Polymorphism makes the flow harder to trace.
   - No. The vehicle reports what it needs, and the lot still does the parking (SRP).

   </details>

2. ABC vs `Protocol`: what's the difference, and when do you use each?

   <details><summary>Answer</summary>

   - ABC is nominal: classes inherit explicitly, and instantiation fails if an abstract method is missing. Use it when you own the hierarchy and want that check.
   - `Protocol` is structural: a class fits if it has the methods. Checked by type checkers, not at runtime. Use it for third-party classes, or to keep implementations decoupled.

   </details>

3. `SavingsAccount(BankAccount)` is fine and `ElectricCar(Car)` isn't. What's the test? What breaks when a hybrid arrives, and what's the fix?

   <details><summary>Answer</summary>

   - Test: does the child reuse the parent's implementation unchanged (inherit), or override it to do something different (compose)?
   - `ElectricCar` replaces `start_engine()` and inherits engine concepts it lacks. A hybrid fits under neither `Car` nor `ElectricCar`.
   - Fix: a `Drivetrain` interface composed into `Car`. A hybrid is `Car([GasEngine(), ElectricMotor()])`, with no new class.

   </details>

4. What is the fragile base class problem, and why does it make composition the default?

   <details><summary>Answer</summary>

   - Children depend on the parent's implementation, so a change in the parent can silently break every child.
   - Composition depends only on an interface. Implementations change independently, and behaviour can be mixed (hybrid) without new subclasses.

   </details>
