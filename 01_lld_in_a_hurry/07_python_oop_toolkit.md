# Python OOP toolkit

Source: my older notes (`oop_concepts.py`, `class_and_access_modifiers.py`, `types_enums_dataclass.py`), merged 16 Nov 2026. Only the Python mechanics the Hello Interview notes don't cover. Access prefixes and `@property` are in [05_encapsulation_abstraction.md](05_encapsulation_abstraction.md); ABC vs `Protocol` and composition are in [06_polymorphism_inheritance.md](06_polymorphism_inheritance.md).

## Review points

- **Method kinds:** an instance method (`self`) works on one object. `@classmethod` (`cls`) works on the class, mainly for alternate constructors. `@staticmethod` gets neither; it is a plain function kept in the class's namespace.
- **Class attribute trap:** `obj.x = v` never changes the class attribute; it creates an instance attribute that shadows it. A mutable class attribute (`tags = []`) is shared by every instance.
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
        self.plate, self.hours = plate, hours

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

a, b = Config(), Config()
a.timeout = 500               # new instance attribute on a; Config.timeout is still 10
Config.timeout = 999          # b.timeout -> 999, a.timeout -> 500 (shadowed)
a.tags.append("x")            # b.tags -> ["x"]
```

- Fix for shared mutable state: create it in `__init__` (`self.tags = []`).

### MRO (method resolution order)

```python
class A:
    def process(self): return "A"
class B(A):
    def process(self): return "B>" + super().process()
class C(A):
    def process(self): return "C>" + super().process()
class D(B, C): ...

[k.__name__ for k in D.mro()]   # ['D', 'B', 'C', 'A', 'object']
D().process()                   # 'B>C>A': inside B, super() is C, not A
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

### dataclass

```python
from dataclasses import dataclass, field

@dataclass(frozen=True)
class Money:                       # value object: immutable, equal by value, hashable
    amount: int
    currency: str = "INR"

    def __post_init__(self):
        if self.amount < 0: raise ValueError("negative amount")

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

3. `D(B, C)`, where `B` and `C` both extend `A` and each calls `super().process()`. What does `D().process()` return, and why isn't it `B>A`?

   <details><summary>Answer</summary>

   - `B>C>A`. The MRO is `D, B, C, A, object`.
   - `super()` means "next class in the MRO of the object's type", not "my parent". For a `D` instance, the class after `B` is `C`.

   </details>

4. `Money` is a frozen dataclass and `Account` is a normal class. Why each? Why can't `Order` declare `items: list[str] = []`?

   <details><summary>Answer</summary>

   - `Money` is a value object: no identity, equal by value, never changes, usable as a dict key. A frozen dataclass gives all of that for free.
   - `Account` has invariants and state changes that must go through operations (`deposit`, `withdraw`), so generated field access is the wrong shape.
   - A mutable default would be shared by every instance. dataclass refuses it with `ValueError`; use `field(default_factory=list)`.

   </details>

5. Why an `Enum` rather than string constants for `SpotSize`? When is an enum a sign that you need polymorphism?

   <details><summary>Answer</summary>

   - Closed set, typos fail at once, members compare with `is`, and they can carry methods (`fits`).
   - When code branches on the enum to choose *behaviour* in several places. Then each kind should be a class with its own method.

   </details>

6. Low coupling and high cohesion: which design principles serve each?

   <details><summary>Answer</summary>

   - High cohesion: SRP and Separation of Concerns (one purpose per class).
   - Low coupling: DIP (depend on an abstraction), Law of Demeter (don't know others' internals), ISP (depend only on what you use).

   </details>
