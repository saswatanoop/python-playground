# Chain of responsibility

Source: not on Hello Interview's Design Patterns page. Taught by Claude on 22 Nov 2026 in the page's style; teach-back pending. The syllabus planned for me to write this one with hints; I asked for it taught instead.

## What this note covers

- Passing a request along a line of handlers
- Two flavours: stop at the first handler, or each does its part
- ATM cash dispensing
- A list and a loop as the Python shortcut

## Review points

- **What it is:** a request passes along a chain of handlers. Each one handles it, passes it on, or both.
- **The crux:** the sender knows only the first handler. Each handler knows only the next. Adding or reordering handlers doesn't touch the sender.
- **Two flavours:** *first match stops* (approval by amount, support escalation) and *each does its part* (middleware, ATM notes, validation pipeline).
- **Signal:** "escalate", "levels", "pipeline", "middleware", "approval limits", "dispense notes".
- **Trap:** the request falls off the end of the chain unhandled. Decide what happens there.
- **Python shortcut:** a list of handlers in a loop is often simpler than `next` pointers.

---

## The discussion

### "Purchases up to ₹10,000 need a manager, up to ₹1,00,000 a director, above that the CEO. Where does that logic go?"

The naive version is one function with every rule in it:

```python
def approve_naive(amount: float) -> str:
    if amount <= 10_000:
        return "manager"
    elif amount <= 100_000:
        return "director"
    else:
        return "ceo"
```

Fine for three fixed levels. It hurts when each level does real work (checks budgets, logs, notifies), levels change per department, or a new level gets inserted in the middle. Every change edits this one function.

### "So what does a chain look like?"

Each handler decides "mine or not", and otherwise passes the request to the next one. This is the **first match stops** flavour:

```python
class Approver:
    def __init__(self, title: str, limit: float):
        self.title = title
        self._limit = limit
        self._next: "Approver | None" = None

    def set_next(self, approver: "Approver") -> "Approver":
        self._next = approver
        return approver                                 # lets you link the next one

    def approve(self, amount: float) -> str:
        if amount <= self._limit:
            return self.title
        if self._next is None:
            raise ValueError(f"nobody can approve {amount}")
        return self._next.approve(amount)

manager = Approver("manager", 10_000)
director = Approver("director", 100_000)
ceo = Approver("ceo", 10_000_000)
manager.set_next(director)
director.set_next(ceo)

manager.approve(5_000)          # "manager"
manager.approve(50_000)         # "director"
```

The caller only ever talks to `manager`. Inserting a "senior manager" level is two `set_next` calls; no approver's code changes.

### "And the other flavour?"

**Each does its part, then passes on.** The ATM classic: dispense ₹1,800 using ₹500, ₹200 and ₹100 notes. Each handler gives out as many of its note as fit and passes the remainder down:

```python
class NoteDispenser:
    def __init__(self, note: int):
        self._note = note
        self._next: "NoteDispenser | None" = None

    def set_next(self, dispenser: "NoteDispenser") -> "NoteDispenser":
        self._next = dispenser
        return dispenser

    def dispense(self, amount: int, result: dict[int, int]) -> None:
        count = amount // self._note
        if count > 0:
            result[self._note] = count
        remainder = amount - count * self._note
        if remainder == 0:
            return
        if self._next is None:
            raise ValueError(f"can't dispense the last {remainder}")
        self._next.dispense(remainder, result)

five_hundred = NoteDispenser(500)
two_hundred = NoteDispenser(200)
hundred = NoteDispenser(100)
five_hundred.set_next(two_hundred)
two_hundred.set_next(hundred)

notes: dict[int, int] = {}
five_hundred.dispense(1800, notes)
notes               # {500: 3, 200: 1, 100: 1}
```

Middleware is the same flavour: auth, then rate limiting, then logging, each doing its bit before passing the request on (or stopping it).

### "Do I need `set_next` in Python?"

Often not. A list in order and a loop gives the same behaviour with less wiring:

```python
approvers = [manager, director, ceo]
```

Loop through them and return the first whose limit fits. Use linked handlers when each handler must decide *whether* and *how* to call the next (middleware that can stop the request, or change it on the way). In an interview, the linked version is the classic shape; mention the list as the Python simplification.

### Gotchas

- **Falling off the end.** The last handler must handle or raise. A silent `None` is a lost request.
- **Order matters.** Put the ₹100 dispenser first and it pays out everything in ₹100 notes. Build the chain in one place so the order is visible.
- **Long chains are hard to debug.** "Who handled this?" means stepping through every link; log the handler that acted.
- **A chain for two fixed rules.** An `if` is fine.

### Connections

- Like Decorator, each link holds the next one and passes calls on. But a decorator always passes on and adds behaviour; a chain handler may stop the request ([02_decorator.md](../02_structural/02_decorator.md)).
- Express middleware, logging levels, validation pipelines.
- LLD problems: ATM, logging service (log levels), support ticket escalation.

---

## Review questions

Answer first, then open the answer to check.

1. What does the sender know in a chain of responsibility, and what does that buy you?

   <details><summary>Answer</summary>

   - Only the first handler; each handler knows only the next.
   - Handlers can be added, removed or reordered without touching the sender or the other handlers' code.

   </details>

2. Approval by amount and ATM note dispensing are both chains. How do they differ?

   <details><summary>Answer</summary>

   - Approval: first match stops. One handler takes the request.
   - ATM: each does its part. Every dispenser hands out what it can and passes the remainder on.

   </details>

3. The ATM is asked for ₹1,850. What happens, and where should that be caught?

   <details><summary>Answer</summary>

   - The ₹100 dispenser is left with 50 and no next handler, so it raises.
   - Better to validate up front (amount must be a multiple of 100) before dispensing anything; a real ATM must never half-dispense.

   </details>

4. Linked handlers with `set_next`, or a list and a loop?

   <details><summary>Answer</summary>

   - A list and a loop when each handler just says "mine or not" in order.
   - Linked handlers when a handler must decide whether and how to call the next one, such as middleware that stops or changes the request.

   </details>
