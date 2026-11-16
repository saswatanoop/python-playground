# Python objects and containers

Source: drafted by Claude on 16 Nov 2026 to fill gaps in my Python toolkit for LLD. Not from a teach-back yet: teach it back, then reword.

## What this note covers

- Making objects equal by value, hashable and sortable (`__eq__`, `__hash__`, `__lt__`, `__repr__`)
- Domain exceptions: when to raise and when to return
- The containers LLD problems need: `deque`, `heapq`, `defaultdict`, `OrderedDict`
- Functions as strategies (`Callable`)
- Recognize-only: `Generic[T]`, `Self`, `uuid4`, `datetime`, `itertools.count`

## Review points

- **Default `==` is identity.** Defining `__eq__` makes the class unhashable unless you also define `__hash__`. Equal objects must have equal hashes, and the hash must use only fields that never change.
- **Value object or entity?** A value object (`Position`, `Money`) is equal by value: use `@dataclass(frozen=True)`. An entity (`ParkingSpot`, `User`) is equal by identity: leave `==` alone and keep it in a dict keyed by its id.
- **Ordering:** `sorted(key=...)` needs no dunder. `heapq` needs `<`: use `@dataclass(order=True)` or push `(priority, seq, item)` tuples.
- **Exceptions:** raise when a rule is broken and the caller must not ignore it. Return `None` or `False` when the outcome is normal and the caller will branch on it. One base exception per domain.
- **Containers:** `deque` for queues (O(1) at both ends), `heapq` for "smallest first", `defaultdict` for grouping, `OrderedDict` for LRU.
- **A one-method strategy can be a function.** Use a class when the strategy has state, config or several methods.

---

### Equality and hashing

```python
from dataclasses import dataclass

class Pos:
    def __init__(self, row: int, col: int):
        self.row = row
        self.col = col

Pos(1, 2) == Pos(1, 2)             # False: default == compares identity

@dataclass(frozen=True)
class Position:                     # value object: equal by value, immutable, hashable
    row: int
    col: int

board = {Position(1, 2): "X"}
board[Position(1, 2)]               # "X": a different instance finds the same key
```

The same thing by hand, when a dataclass doesn't fit:

```python
class Cell:
    def __init__(self, row: int, col: int):
        self._row = row
        self._col = col

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Cell):
            return NotImplemented
        return (self._row, self._col) == (other._row, other._col)

    def __hash__(self) -> int:
        return hash((self._row, self._col))        # same fields as __eq__

    def __repr__(self) -> str:
        return f"Cell({self._row}, {self._col})"   # what you see in logs and failed asserts

{Cell(1, 2)} == {Cell(1, 2)}        # True
```

- **`__eq__` without `__hash__`:** Python sets `__hash__` to `None`, so `set.add(obj)` raises `TypeError: unhashable type`. A non-frozen `@dataclass` has the same effect.
- **Hash only immutable fields.** Change a hashed field after the object is in a set or dict and lookups stop finding it.
- **Entities keep identity equality.** Spot 14 is the same spot whether free or occupied. Don't define `__eq__`; store entities as `dict[str, ParkingSpot]` keyed by id.

### Ordering and heaps

```python
import heapq
from dataclasses import dataclass, field

@dataclass(order=True)
class Job:
    priority: int
    name: str = field(compare=False)     # left out of <, ==

heap: list[Job] = []
heapq.heappush(heap, Job(2, "email"))
heapq.heappush(heap, Job(1, "payment"))
heapq.heappop(heap).name                 # "payment": smallest priority first
```

- `heapq` is a min-heap on a plain list, O(log n) per push and pop. For "largest first", negate the priority.
- **FIFO among equal priorities:** add a sequence number, `(priority, seq, item)`, with `seq` from `itertools.count()`. Without it, a tie falls through to comparing the items, which raises `TypeError` if they have no `<`.
- For a one-off sort, `sorted(jobs, key=lambda j: j.priority)` needs no `__lt__`.

### Exceptions

```python
class ParkingError(Exception): ...                    # one base per domain

class InvalidTicketError(ParkingError):
    def __init__(self, ticket_id: str):
        super().__init__(f"unknown ticket {ticket_id}")
        self.ticket_id = ticket_id

class Lot:
    def __init__(self):
        self._tickets: dict[str, int] = {}            # ticket id -> spot number

    def find_spot(self) -> int | None:
        return None                                    # "no spot free" is a normal outcome

    def exit(self, ticket_id: str) -> int:
        if ticket_id not in self._tickets:
            raise InvalidTicketError(ticket_id)        # broken rule: must not pass silently
        return self._tickets.pop(ticket_id)

lot = Lot()
try:
    lot.exit("T-9")
except ParkingError as e:                              # every domain error, nothing else
    message = str(e)                                   # "unknown ticket T-9"
```

- **Raise** when the caller broke a precondition or the method can't do what its name says. **Return** `None`, `False` or a result object when the outcome is expected.
- A base class lets callers catch the whole domain with one `except`, without swallowing real bugs. Never write a bare `except:` or catch `Exception` to hide errors.
- Validate in the constructor and in public methods, so an object can never exist in an invalid state.
- Built-ins still fit plain argument errors: `ValueError` for a bad value, `KeyError` for a missing key.

### Containers

| Need | Use |
|---|---|
| FIFO queue, recent history | `deque` (`deque(maxlen=n)` keeps the last n) |
| Always take the smallest or earliest | `heapq` |
| Group items by key | `defaultdict(list)` |
| LRU, or order that you rearrange | `OrderedDict` |
| Membership, uniqueness | `set` |

```python
from collections import deque, defaultdict, OrderedDict

queue: deque[str] = deque(["a", "b"])
queue.popleft()                              # "a" in O(1); list.pop(0) is O(n)

by_floor: defaultdict[int, list[str]] = defaultdict(list)
by_floor[2].append("KA01")                   # no "if key not in" check

class LRUCache:
    def __init__(self, capacity: int):
        self._capacity = capacity
        self._data: OrderedDict[str, int] = OrderedDict()

    def get(self, key: str) -> int | None:
        if key not in self._data:
            return None
        self._data.move_to_end(key)          # now the most recently used
        return self._data[key]

    def put(self, key: str, value: int) -> None:
        self._data[key] = value
        self._data.move_to_end(key)
        if len(self._data) > self._capacity:
            self._data.popitem(last=False)   # evict the least recently used

cache = LRUCache(2)
cache.put("a", 1)
cache.put("b", 2)
cache.get("a")
cache.put("c", 3)
cache.get("b")                               # None: "b" was evicted, "a" survived
```

- **`defaultdict` trap:** reading a missing key inserts it. Check with `key in d` or `d.get(key)`.
- `OrderedDict` is a dict plus a doubly linked list, which is why `move_to_end` and `popitem` are O(1). Be ready to build that by hand if the interviewer asks.

### Functions as strategies

```python
from typing import Callable

PricingRule = Callable[[int], float]             # hours -> fee

def flat(hours: int) -> float:
    return hours * 20.0
def capped(hours: int) -> float:
    return min(hours * 20.0, 100.0)

class Checkout:
    def __init__(self, pricing: PricingRule):
        self._pricing = pricing

    def fee(self, hours: int) -> float:
        return self._pricing(hours)

checkout = Checkout(flat)           # pass the function itself: no parentheses
checkout.fee(8)                     # 160.0

checkout = Checkout(capped)
checkout.fee(8)                     # 100.0

checkout = Checkout(lambda h: 0.0)  # a free day, no new class
checkout.fee(8)                     # 0.0
```

- **Function:** the strategy is one stateless operation.
- **Class (ABC or `Protocol`):** it has config or state, several methods, or needs a name on the class diagram.
- In an interview, say it: "In Python I'd pass a function here; if pricing grows config, it becomes a `PricingStrategy` class."

### Recognize-only

- `Generic[T]` / `TypeVar`: a typed container, e.g. `class LRUCache(Generic[K, V])`.
- `Self` (from `typing`): return type for builder methods and alternate constructors.
- `uuid.uuid4()` for ids, `datetime.now()` for timestamps, `itertools.count()` for sequence numbers.

---

## Review questions

Answer first, then open the answer to check.

1. You add `__eq__` to `Cell`, and `seen.add(Cell(1, 2))` starts raising `TypeError`. Why, and what are two fixes?

   <details><summary>Answer</summary>

   - Defining `__eq__` sets `__hash__` to `None`, so the object is unhashable.
   - Fix 1: define `__hash__` over the same fields `__eq__` compares.
   - Fix 2: make it a `@dataclass(frozen=True)`, which generates both.

   </details>

2. `Position` and `ParkingSpot`: which one gets value equality, which keeps identity, and how do you look up a spot?

   <details><summary>Answer</summary>

   - `Position` is a value object: two positions with the same row and column are the same thing. Frozen dataclass.
   - `ParkingSpot` is an entity: it stays the same spot while its state changes, so identity equality is right.
   - Look it up by id in a `dict[str, ParkingSpot]`, not by comparing objects.

   </details>

3. An object's `__hash__` uses a field that you change after adding the object to a set. What happens?

   <details><summary>Answer</summary>

   - The set stored it under the old hash. `obj in the_set` now looks in the wrong bucket and returns `False`, though the object is still inside.
   - So hash only fields that never change; that is why value objects are frozen.

   </details>

4. Elevator requests must be served by priority, first-come-first-served among equals. Which structure, and what breaks if you push `(priority, request)`?

   <details><summary>Answer</summary>

   - A heap of `(priority, seq, request)`, with `seq` from `itertools.count()`.
   - Without `seq`, two equal priorities make Python compare the `request` objects: `TypeError` if they have no `<`, and no FIFO guarantee even if they do.

   </details>

5. `find_spot()` when the lot is full, and `exit()` with an unknown ticket: raise or return? Why give the domain one base exception?

   <details><summary>Answer</summary>

   - Full lot: return `None`. It is a normal outcome and the caller branches on it.
   - Unknown ticket: raise. A rule is broken and it must not pass silently.
   - One base (`ParkingError`) lets callers catch every domain error with one `except` without also swallowing bugs such as `AttributeError`.

   </details>

6. Why a `deque` and not a list for a queue? What is the `defaultdict` trap? How does `OrderedDict` give an O(1) LRU cache?

   <details><summary>Answer</summary>

   - `deque.popleft()` is O(1); `list.pop(0)` shifts every element, O(n).
   - Reading a missing key on a `defaultdict` inserts it, so a lookup can grow the dict.
   - `move_to_end(key)` on every get and put, and `popitem(last=False)` to evict. Both are O(1) because it is a dict plus a doubly linked list.

   </details>

7. Pricing as a plain function or as a `PricingStrategy` class: when do you choose each?

   <details><summary>Answer</summary>

   - Function: one stateless operation (`hours -> fee`). Less code, and a lambda covers a one-off rule.
   - Class: the rule carries config or state, has several methods, or needs a name on the class diagram.

   </details>
