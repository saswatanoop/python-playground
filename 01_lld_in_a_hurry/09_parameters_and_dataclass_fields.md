# Constructor parameters and dataclass fields

Source: drafted by Claude on 17 Nov 2026 from my question while studying Builder. Not from a teach-back yet: teach it back, then reword. Dataclass basics (`frozen`, `__post_init__`, `order`) are in [07_python_oop_toolkit.md](07_python_oop_toolkit.md).

## What this note covers

- Keyword arguments and default values in a constructor
- Keyword-only parameters, and the mutable-default trap
- Required and optional dataclass fields, and the ordering rule
- `field(...)` options and `replace()`
- What a dataclass does not do for you

## Review points

- **Any parameter can be passed by name.** A parameter with `= value` is optional; one without is required.
- **A bare `*` makes everything after it keyword-only.** Use it for optional settings, so call sites stay readable and you can add parameters later without breaking callers.
- **Never use a mutable default** (`items=[]`). Defaults are created once, when the function is defined. Use `None` and build the list inside.
- **Dataclass:** a field with no default is required; a field with `= value` is optional. Required fields must come first, unless you pass `kw_only=True`.
- **`field(default_factory=...)`** for a fresh value per instance: lists, ids, timestamps.
- **`replace(obj, x=...)`** makes a changed copy of a frozen dataclass.
- **Not enforced:** type hints. `frozen=True` is shallow: a list inside can still be mutated.

---

### Keyword arguments and defaults

```python
class HttpClient:
    def __init__(self, base_url: str, *, timeout: int = 30, retries: int = 3):
        self.base_url = base_url
        self.timeout = timeout
        self.retries = retries

client = HttpClient("https://api.example.com", retries=5)
client.timeout      # 30: kept its default
client.retries      # 5
```

- **Required:** `base_url` has no default, so the caller must pass it.
- **Optional:** `timeout` and `retries` have defaults, so the caller names only what differs.
- **Keyword-only:** everything after the bare `*` must be passed by name. `HttpClient("https://api.example.com", 10)` raises `TypeError: HttpClient.__init__() takes 2 positional arguments but 3 were given`.
- **Why bother with `*`:** `HttpClient(url, 10, 3)` tells the reader nothing. Named settings read well, and their order can change later without breaking anyone.
- **Order in the signature:** parameters without defaults come before those with. After the `*`, the order is free, and a keyword-only parameter without a default is required by name.
- This is what makes Builder rare in Python (see [02_builder.md](../02_design_patterns/01_creational/02_builder.md)).

### The mutable-default trap

```python
class BadCart:
    def __init__(self, items: list[str] = []):      # one list, created once
        self.items = items

first = BadCart()
second = BadCart()
first.items.append("book")
second.items        # ["book"]: both carts share the same list

class Cart:
    def __init__(self, items: list[str] | None = None):
        if items is None:
            self.items = []
        else:
            self.items = list(items)                # copy, so the caller's list isn't shared either

third = Cart()
fourth = Cart()
third.items.append("book")
fourth.items        # []
```

- Defaults are evaluated **once, when the `def` runs**, not on each call. The same goes for `created_at=datetime.now()`: every object gets the time the class was loaded.
- The fix is always the same: default to `None`, create the value inside.

### Dataclass: required and optional fields

```python
from dataclasses import dataclass, field, replace
from datetime import datetime
from uuid import uuid4

@dataclass
class Ticket:
    plate: str              # required: no default
    spot: int               # required
    hours: int = 1          # optional: has a default

ticket = Ticket("KA01AB1234", 14)
ticket.hours        # 1
```

- **Ordering rule:** required fields first. Put `hours: int = 1` above `spot: int` and the class fails to define: `TypeError: non-default argument 'spot' follows default argument 'hours'`.
- The generated `__init__` takes the fields in the order written, so callers can pass them by position or by name.

### `kw_only`, `field(...)` and `replace()`

```python
@dataclass(frozen=True, kw_only=True)
class Booking:
    user_id: str                                                 # required, by name
    seats: int = 1
    id: str = field(default_factory=lambda: uuid4().hex)         # fresh id per instance
    created_at: datetime = field(default_factory=datetime.now)   # fresh time per instance
    notes: list[str] = field(default_factory=list)
    card_token: str = field(default="", repr=False)              # kept out of repr and logs

booking = Booking(user_id="u1", seats=2)
bigger = replace(booking, seats=4)      # frozen, so make a changed copy
bigger.seats            # 4
booking.seats           # 2
bigger.id == booking.id # True: everything else is copied
```

- **`kw_only=True`:** every field must be passed by name, and the ordering rule no longer applies. Good for anything with more than three or four fields.
- **`default_factory`:** a function called for each new instance. `datetime.now` is passed *without* parentheses: you hand over the function, and the dataclass calls it.
- **`field` options worth knowing:** `repr=False` (secrets), `compare=False` (left out of `==` and ordering, see [08](08_python_objects_and_containers.md)), `init=False` (not a constructor parameter; set it in `__post_init__`).
- **`replace()`** is how you "modify" a value object: you get a new one.

### What a dataclass doesn't do

- **It doesn't check types.** `Ticket("KA01AB1234", "fourteen")` is accepted. Validate in `__post_init__` if it matters.
- **`frozen=True` is shallow.** `booking.notes.append("late")` works, because the list itself isn't frozen. Use a `tuple` field for a truly immutable value object.
- **`frozen=True` blocks assignment in `__post_init__` too.** For a derived value on a frozen class, use a `@property`.
- **Inheritance:** if a parent dataclass has a field with a default, every child field needs one too, unless you use `kw_only=True`.
- Recognize-only: `slots=True` (less memory, no new attributes), `asdict(obj)` (to a plain dict).

---

## Review questions

Answer first, then open the answer to check.

1. `def __init__(self, base_url, *, timeout=30, retries=3)`. What does the `*` do, and why is it worth adding?

   <details><summary>Answer</summary>

   - Everything after it is keyword-only: `HttpClient(url, 10)` raises `TypeError`.
   - Call sites name each setting, so they read well, and you can add or reorder optional parameters later without breaking callers.

   </details>

2. Two `BadCart()` objects end up sharing items. Why, and what is the fix? What other default has the same problem?

   <details><summary>Answer</summary>

   - `items=[]` is evaluated once, when the function is defined, so every call gets the same list.
   - Fix: default to `None` and create the list inside the constructor.
   - Same problem: `created_at=datetime.now()` as a default; every object gets the load-time timestamp.

   </details>

3. In a dataclass, how do you mark a field required, and what happens if `hours: int = 1` is written above `spot: int`? Two ways to fix it?

   <details><summary>Answer</summary>

   - Required means no default.
   - The class fails to define: `TypeError: non-default argument 'spot' follows default argument 'hours'`.
   - Fix: move required fields first, or use `@dataclass(kw_only=True)`.

   </details>

4. A `Booking` needs a unique id and a creation time, set automatically. How, and why not `id: str = uuid4().hex`?

   <details><summary>Answer</summary>

   - `field(default_factory=lambda: uuid4().hex)` and `field(default_factory=datetime.now)`: the factory runs for each new instance.
   - `= uuid4().hex` is evaluated once, when the class is defined, so every booking would share one id.

   </details>

5. `Booking` is `frozen=True`. How do you change the seat count, and can anything inside it still be mutated?

   <details><summary>Answer</summary>

   - `replace(booking, seats=4)` returns a new object; the original is unchanged.
   - Yes: `frozen` is shallow. `booking.notes.append(...)` still works because the list isn't frozen. Use a `tuple` for a truly immutable value.

   </details>
