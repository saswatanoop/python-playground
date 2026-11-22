# State machine (State pattern)

Source: Hello Interview LLD → Design Patterns. Drafted by Claude on 22 Nov 2026 at my request; my teach-back so far is only the page's one-line definition. Teach it back before treating it as learned.

## What this note covers

- The `if status ==` in every method that a state machine replaces
- Drawing the diagram first
- One class per state, and who decides the next state
- State vs Strategy
- The transition table: often the better Python answer
- Adding a state

## Review points

- **What it is:** an object whose behaviour depends on its current state, with rules for moving between states. Hello Interview: less common than Strategy or Observer, but when it appears it's usually *the centerpiece* of the design.
- **Signal:** the word "state" (or status, stage, mode) appears several times in the requirements. Vending machines, orders, document workflows, games, elevators.
- **Draw the state diagram first.** States as circles, transitions as labelled arrows. Interviewers like seeing it.
- **State pattern:** one class per state. The context holds the current state and delegates every action to it; **each state decides the next state**.
- **State vs Strategy:** same structure. A strategy is chosen from outside and never swaps itself; states replace themselves as the object moves.
- **Transition table** (`dict[(state, action)] -> next_state`): simpler when states differ only in which moves are allowed. Classes when each state *behaves* differently.

---

## The discussion

### "An order can be paid, shipped, delivered or cancelled. What's wrong with checking the status in each method?"

The naive version:

```python
class InvalidTransition(Exception):
    pass

class NaiveOrder:
    def __init__(self) -> None:
        self.status = "created"
        self.refunded = False

    def ship(self) -> None:
        if self.status == "paid":
            self.status = "shipped"
        else:
            raise InvalidTransition(f"can't ship a {self.status} order")

    def cancel(self) -> None:
        if self.status == "created":
            self.status = "cancelled"
        elif self.status == "paid":
            self.refunded = True
            self.status = "cancelled"
        else:
            raise InvalidTransition(f"can't cancel a {self.status} order")

    # pay() and deliver() repeat the same shape
```

Four actions × five states, and the rules are scattered across every method. Add a "returned" state and you touch every method, and the one you forget lets a cancelled order get shipped. Nobody can see the whole lifecycle in one place.

### "Where do I start?"

With the diagram, before any code:

```text
 created ──pay──▶ paid ──ship──▶ shipped ──deliver──▶ delivered
    │              │
  cancel         cancel (refund)
    ▼              ▼
 cancelled ◀───────┘
```

Every arrow is a legal transition; anything without an arrow is illegal. Hello Interview: if there's a state machine in your solution, the interview is probably organised around it, so talk through this picture first.

### "So what does the State pattern do?"

Each state becomes a class that knows which actions it allows and which state comes next. The base class rejects everything, so each state overrides only its arrows:

```python
class OrderState:
    name = "unknown"

    def pay(self, order: "Order") -> None:
        raise InvalidTransition(f"can't pay a {self.name} order")

    def ship(self, order: "Order") -> None:
        raise InvalidTransition(f"can't ship a {self.name} order")

    def deliver(self, order: "Order") -> None:
        raise InvalidTransition(f"can't deliver a {self.name} order")

    def cancel(self, order: "Order") -> None:
        raise InvalidTransition(f"can't cancel a {self.name} order")

class Created(OrderState):
    name = "created"

    def pay(self, order: "Order") -> None:
        order.state = Paid()

    def cancel(self, order: "Order") -> None:
        order.state = Cancelled()

class Paid(OrderState):
    name = "paid"

    def ship(self, order: "Order") -> None:
        order.state = Shipped()

    def cancel(self, order: "Order") -> None:
        order.refunded = True                   # behaviour that differs by state
        order.state = Cancelled()

class Shipped(OrderState):
    name = "shipped"

    def deliver(self, order: "Order") -> None:
        order.state = Delivered()

class Delivered(OrderState):
    name = "delivered"                          # terminal: overrides nothing

class Cancelled(OrderState):
    name = "cancelled"                          # terminal

class Order:
    def __init__(self, order_id: str):
        self.order_id = order_id
        self.refunded = False
        self.state: OrderState = Created()

    def pay(self) -> None:
        self.state.pay(self)                    # delegate to the current state

    def ship(self) -> None:
        self.state.ship(self)

    def deliver(self) -> None:
        self.state.deliver(self)

    def cancel(self) -> None:
        self.state.cancel(self)
```

```python
order = Order("o1")
order.pay()
order.ship()
order.state.name            # "shipped"

try:
    order.cancel()
except InvalidTransition as error:
    print(error)            # can't cancel a shipped order

refund_order = Order("o2")
refund_order.pay()
refund_order.cancel()
refund_order.refunded       # True: cancelling from Paid refunds
```

`Order` has no `if` on status at all. Each state's rules live in one class, and an illegal action can't slip through: the base class rejects it.

### "This looks exactly like Strategy. What's the difference?"

The structure is identical: a context holds an object behind an interface and delegates to it. The difference is who changes it and why:

| | Strategy | State |
|---|---|---|
| Who picks it | The client, from outside (`cart.set_payment(...)`) | The states themselves, as actions happen |
| Do they know each other? | No. `CardPayment` has never heard of `WalletPayment` | Yes. `Created` knows the next state is `Paid` |
| The question it answers | *How* do I do this one thing? | *What* is allowed right now, and what comes next? |

Quick test: if the swap happens because of something the object went through, it's State. If someone outside chooses, it's Strategy.

### "Do I really need a class per state?"

Often not, in Python. If states differ only in *which transitions are allowed*, a table says it all and reads like the diagram:

```python
from enum import Enum

class Status(Enum):
    CREATED = "created"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

TRANSITIONS: dict[tuple[Status, str], Status] = {
    (Status.CREATED, "pay"): Status.PAID,
    (Status.CREATED, "cancel"): Status.CANCELLED,
    (Status.PAID, "ship"): Status.SHIPPED,
    (Status.PAID, "cancel"): Status.CANCELLED,
    (Status.SHIPPED, "deliver"): Status.DELIVERED,
}

class TableOrder:
    def __init__(self) -> None:
        self.status = Status.CREATED

    def apply(self, action: str) -> None:
        key = (self.status, action)
        if key not in TRANSITIONS:
            raise InvalidTransition(f"can't {action} a {self.status.value} order")
        self.status = TRANSITIONS[key]

simple = TableOrder()
simple.apply("pay")
simple.apply("ship")
simple.status               # Status.SHIPPED
```

Choose by what varies:

- **Only the allowed moves differ** → table. One line per arrow, the whole machine visible at once.
- **Each state does different work** (cancel from `Paid` refunds; a vending machine in `HasCoin` reacts to a button differently than in `NoCoin`) → classes.

Both are fine in an interview. Start with the diagram, then say which one you're choosing and why.

### Interviewer: "Add a Returned state: delivered orders can be returned."

- **Classes:** a new `Returned` class, plus a `return_order` method on the base class, on `Order`, and an override in `Delivered`. The states reference each other, so adding one does touch its neighbours. Don't claim it's perfectly open/closed.
- **Table:** one enum value and one line, `(Status.DELIVERED, "return"): Status.RETURNED`.

### Gotchas

- **Skipping the diagram.** It's the fastest way to show the interviewer you understand the lifecycle, and to catch missing transitions.
- **Silently ignoring illegal actions.** Raise, or return a clear message. Hello Interview's vending machine prints "Insert coin first".
- **Data in the states.** Keep order data (`refunded`, items, amount) on the context. States hold behaviour only, so they stay small and shareable.
- **Forgetting terminal states.** `Delivered` and `Cancelled` must reject everything.
- **Classes where a table would do.** Five classes that each only set the next state are the table, written the long way.

### Connections

- Strategy: same shape, different driver ([01_strategy.md](01_strategy.md)).
- Observer: a state change is a natural event to notify on ([02_observer.md](02_observer.md)).
- LLD problems that are built around one: vending machine, elevator, Connect Four's game state, Amazon Locker.

---

## Review questions

Answer first, then open the answer to check.

1. Every method of `Order` starts with `if self.status == ...`. What goes wrong as the lifecycle grows?

   <details><summary>Answer</summary>

   - The rules are scattered across every method; a new state touches all of them, and a missed check lets an illegal transition through.
   - Nobody can see the whole lifecycle in one place.

   </details>

2. In the State pattern, who decides the next state, and what does the context do?

   <details><summary>Answer</summary>

   - The current state object: `Created.pay` sets `order.state = Paid()`.
   - The context holds the current state and delegates every action to it, with no `if` on status.

   </details>

3. State and Strategy have the same class diagram. How do you tell them apart?

   <details><summary>Answer</summary>

   - Strategy is chosen from outside and never swaps itself; strategies don't know each other.
   - States replace themselves as actions happen and know their successors. Strategy answers "how", State answers "what's allowed now, and what's next".

   </details>

4. Transition table or a class per state? What decides it?

   <details><summary>Answer</summary>

   - Table when states differ only in which transitions are allowed: one line per arrow.
   - Classes when states behave differently (cancel from `Paid` refunds; a vending machine reacts differently per state).

   </details>

5. Add a Returned state for delivered orders. What changes in each version, and is the class version open/closed?

   <details><summary>Answer</summary>

   - Table: one enum value and one line.
   - Classes: a new `Returned` class, a new action on the base and `Order`, and an override in `Delivered`. States reference each other, so neighbours change: not fully open/closed.

   </details>
