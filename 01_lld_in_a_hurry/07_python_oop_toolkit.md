# Python OOP toolkit

Source: my older notes (`oop_concepts.py`, `class_and_access_modifiers.py`, `types_enums_dataclass.py`), merged 16 Nov 2026. Only the Python mechanics the Hello Interview notes don't cover. Access prefixes and `@property` are in [05_encapsulation_abstraction.md](05_encapsulation_abstraction.md); ABC vs `Protocol` and composition are in [06_polymorphism_inheritance.md](06_polymorphism_inheritance.md).

## What this note covers

- Instance, class and static methods
- Class vs instance attributes and the shadowing trap
- When a setter is fine, and why a child calls `super().__init__()`
- MRO and what `super()` really means
- Enum, abstract properties and dataclasses

## Review points

- **Method kinds:** an instance method (`self`) works on one object. `@classmethod` (`cls`) works on the class, mainly for alternate constructors. `@staticmethod` gets neither; it is a plain function kept in the class's namespace.
- **Class attribute trap:** `obj.x = v` never changes the class attribute; it creates an instance attribute that shadows it. A mutable class attribute (`tags = []`) is shared by every instance.
- **Setter (`@x.setter`):** fine for a plain value with a rule about that value alone (`display_name`). Wrong for state that changes for a reason (`balance` → operations).
- **`super().__init__()`:** a child `__init__` replaces the parent's, so call it or the parent's fields never exist. Children go through the parent's methods, not its private fields.
- **Abstract property:** `@property` over `@abstractmethod`. Forces every subclass to expose a value (`Vehicle.size`).
- **MRO:** the order Python searches classes for an attribute (C3 linearization). `super()` calls the *next class in the MRO*, which is not always the parent.
- **Enum:** a closed set of named constants for states and kinds. Compare with `is`. Members can carry methods.
- **dataclass:** generates `__init__`, `__repr__`, `__eq__`. Use `frozen=True` for value objects, `__post_init__` for validation, `field(default_factory=list)` for mutable defaults.
- **Type hints:** no runtime effect. They document the contract and feed the type checker. Write `list[int]`, `X | None`, `Callable[[int, int], int]`.
- **Coupling and cohesion:** aim for low coupling (few dependencies between classes) and high cohesion (one class, one purpose). The design principles are ways to get there.

---

### Instance, class and static methods

```python
class Ticket:
    def __init__(self, plate: str, hours: int):
        self.plate = plate
        self.hours = hours

    @classmethod
    def from_string(cls, raw: str) -> "Ticket":   # alternate constructor
        plate, hours = raw.split(":")
        return cls(plate, int(hours))             # cls, not Ticket: subclasses get their own type

    @staticmethod
    def is_valid_plate(plate: str) -> bool:       # needs neither self nor cls
        return plate.isalnum()

class VIPTicket(Ticket): ...

type(VIPTicket.from_string("KA01AB1234:3"))       # VIPTicket
```

### Class vs instance attributes

```python
class Config:
    timeout = 10
    tags: list[str] = []      # mutable class attribute: one list for everyone

a = Config()
b = Config()
a.timeout = 500               # new instance attribute on a; Config.timeout is still 10
Config.timeout = 999          # b.timeout -> 999, a.timeout -> 500 (shadowed)
a.tags.append("x")            # b.tags -> ["x"]
```

- Fix for shared mutable state: create it in `__init__` (`self.tags = []`).

### Setter and `super().__init__()`

```python
class User:
    def __init__(self, display_name: str):
        self.display_name = display_name           # runs the setter: validated from the start

    @property
    def display_name(self) -> str:
        return self._display_name

    @display_name.setter
    def display_name(self, value: str) -> None:
        value = value.strip()
        if not 3 <= len(value) <= 30:
            raise ValueError("display name must be 3-30 characters")
        self._display_name = value
```

- **A setter is fine when the field really is "set to a new value" and the rule is about that value alone.** A display name has no history, no intent to record, and no other state that depends on it.
- **`balance` is the opposite:** it changes for a reason (deposit, withdraw), so it gets operations, not a setter (see [05](05_encapsulation_abstraction.md)).
- Assign through the property in `__init__` too, or the constructor skips the validation.

```python
class BankAccount:
    def __init__(self, owner: str):
        self.owner = owner
        self._balance = 0.0
    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("amount must be positive")
        self._balance += amount
    @property
    def balance(self) -> float:
        return self._balance

class SavingsAccount(BankAccount):
    def __init__(self, owner: str, interest_rate: float):
        super().__init__(owner)                     # parent sets up owner and _balance
        self.interest_rate = interest_rate
    def add_interest(self) -> None:
        self.deposit(self.balance * self.interest_rate)    # reuses the parent's rules

acc = SavingsAccount("alice", 0.05)
acc.deposit(1000)
acc.add_interest()
acc.balance                                         # 1050.0
```

- A child `__init__` *replaces* the parent's. Skip `super().__init__()` and `_balance` never exists: `AttributeError` on first use.
- `add_interest` goes through `deposit()` and reads `balance`, never touching `_balance`, so the parent's rules still apply. This is "reuse, don't replace" (see [06](06_polymorphism_inheritance.md)).

### MRO (method resolution order)

```python
class A:
    def process(self):
        return "A"
class B(A):
    def process(self):
        return "B>" + super().process()
class C(A):
    def process(self):
        return "C>" + super().process()
class D(B, C): ...

[k.__name__ for k in D.mro()]   # ['D', 'B', 'C', 'A', 'object']
d = D()
d.process()                     # 'B>C>A': inside B, super() is C, not A
```

- Order: the class itself, then its bases left to right, each class once, and a parent only after all of its children.
- In LLD interviews, don't inherit implementation from several parents. Recognize the rule; you rarely need it.

### Enum

```python
from enum import Enum

class SpotSize(Enum):
    MOTORCYCLE = 1
    REGULAR = 2
    LARGE = 3

    def fits(self, needed: "SpotSize") -> bool:   # members can carry behaviour
        return self.value >= needed.value

SpotSize.LARGE.fits(SpotSize.REGULAR)   # True
SpotSize(2) is SpotSize["REGULAR"]      # True: lookup by value or by name returns the same member
```

- Use it for states, kinds and categories instead of strings: a typo fails loudly and the set is closed.
- `class Direction(str, Enum)` when members must also behave as strings (e.g. JSON).
- An `if/elif` over an enum that picks *behaviour* is the cue for polymorphism (see [06](06_polymorphism_inheritance.md)).

### Abstract property

```python
class Vehicle(ABC):
    @property
    @abstractmethod                                 # order matters: @property outermost
    def size(self) -> SpotSize: ...

class Car(Vehicle):
    @property
    def size(self) -> SpotSize:
        return SpotSize.REGULAR

class Bike(Vehicle): ...

Car().size     # SpotSize.REGULAR
Bike()         # TypeError: can't instantiate abstract class Bike without 'size'
```

- Use it when every subclass must *expose a value* (size, rate, capacity) rather than perform an action.
- `ABC` and `abstractmethod` come from `abc`; `SpotSize` is the enum above.

### dataclass

```python
from dataclasses import dataclass, field

@dataclass(frozen=True)
class Money:                       # value object: immutable, equal by value, hashable
    amount: int
    currency: str = "INR"

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("negative amount")

@dataclass
class Order:
    id: str
    items: list[str] = field(default_factory=list)   # `= []` raises ValueError when the class is defined

Money(5) == Money(5)               # True
```

- Use a dataclass for data holders (value objects, DTOs). An entity with rules and invariants is a normal class that exposes operations.
- `order=True` adds `<` and `>`, comparing fields in order.

---

## Review questions

Answer first, then open the answer to check.

1. `Ticket.from_string()` returns `Ticket(plate, hours)` instead of `cls(plate, hours)`. What breaks? When would you pick `@staticmethod` over `@classmethod`?

   <details><summary>Answer</summary>

   - `VIPTicket.from_string(...)` returns a plain `Ticket`, so the subclass loses its alternate constructor.
   - `@staticmethod` when the function needs neither the instance nor the class and only belongs there by topic (`is_valid_plate`).

   </details>

2. `Config` has `timeout = 10` and `tags = []`. After `a.timeout = 500`, `Config.timeout = 999` and `a.tags.append("x")`, what are `a.timeout`, `b.timeout` and `b.tags`, and why?

   <details><summary>Answer</summary>

   - `a.timeout` is 500: the assignment created an instance attribute that shadows the class one.
   - `b.timeout` is 999: `b` has no instance attribute, so it reads the class attribute.
   - `b.tags` is `["x"]`: `append` mutates the one list on the class; no assignment happened, so nothing was shadowed.

   </details>

3. `User.display_name` has a validated setter, and you argued against `Account.set_balance()`. Why is one fine and the other not? What goes wrong if `User.__init__` writes `self._display_name = name`?

   <details><summary>Answer</summary>

   - A display name is just a value: no history, no intent, no other state depending on it. A balance changes for a reason (deposit, withdraw) that must be recorded and ruled on, so it needs operations.
   - Writing `_display_name` directly in `__init__` skips the setter, so an invalid name can get in through the constructor.

   </details>

4. `SavingsAccount.__init__` forgets `super().__init__(owner)`. What happens? And why does `add_interest()` call `self.deposit()` instead of `self._balance += ...`?

   <details><summary>Answer</summary>

   - A child `__init__` replaces the parent's, so `owner` and `_balance` are never set. The first use raises `AttributeError`.
   - Going through `deposit()` keeps the parent's rules (validation, and any logging added later) in force. Writing the field directly couples the child to the parent's internals: the fragile base class problem.

   </details>

5. How do you force every `Vehicle` subclass to define `size`, and what happens if one doesn't?

   <details><summary>Answer</summary>

   - Stack `@property` over `@abstractmethod` on `size` in an ABC.
   - A subclass without it can't be instantiated: `TypeError` at construction, not a silent `None` later.

   </details>

6. `D(B, C)`, where `B` and `C` both extend `A` and each calls `super().process()`. What does `D().process()` return, and why isn't it `B>A`?

   <details><summary>Answer</summary>

   - `B>C>A`. The MRO is `D, B, C, A, object`.
   - `super()` means "next class in the MRO of the object's type", not "my parent". For a `D` instance, the class after `B` is `C`.

   </details>

7. `Money` is a frozen dataclass and `Account` is a normal class. Why each? Why can't `Order` declare `items: list[str] = []`?

   <details><summary>Answer</summary>

   - `Money` is a value object: no identity, equal by value, never changes, usable as a dict key. A frozen dataclass gives all of that for free.
   - `Account` has invariants and state changes that must go through operations (`deposit`, `withdraw`), so generated field access is the wrong shape.
   - A mutable default would be shared by every instance. dataclass refuses it with `ValueError`; use `field(default_factory=list)`.

   </details>

8. Why an `Enum` rather than string constants for `SpotSize`? When is an enum a sign that you need polymorphism?

   <details><summary>Answer</summary>

   - Closed set, typos fail at once, members compare with `is`, and they can carry methods (`fits`).
   - When code branches on the enum to choose *behaviour* in several places. Then each kind should be a class with its own method.

   </details>

9. Low coupling and high cohesion: which design principles serve each?

   <details><summary>Answer</summary>

   - High cohesion: SRP and Separation of Concerns (one purpose per class).
   - Low coupling: DIP (depend on an abstraction), Law of Demeter (don't know others' internals), ISP (depend only on what you use).

   </details>
