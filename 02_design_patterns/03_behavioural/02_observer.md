# Observer

Source: Hello Interview LLD → Design Patterns, and my teach-back on 22 Nov 2026 (not clean yet). Sections marked **(missed)** are what I didn't get; re-teach those on review, especially the crux and why the loop uses a copy.

## What this note covers

- The problem: a subject that calls every dependent itself
- The crux: the subject knows only an interface and a list
- One failing observer, an observer that detaches itself, an observer that changes the subject
- Function observers in Python
- Observer vs pub-sub

## Review points

- **What it is:** observers attach to a subject; when the subject changes, it calls `update` on each. Hello Interview: a top-tier interview pattern. Signal words: "notify", "update multiple components".
- **The problem:** without it, `place()` calls inventory, email and analytics itself, so every new listener edits the subject.
- **The crux:** the subject knows only the `Observer` interface and a list. Not who's listening, not what they do. Attach and detach at runtime.
- **It's synchronous:** `notify` returns only after every observer has run. Catch exceptions per observer, so one failure doesn't silence the rest.
- **Loop over a copy** (`list(self._observers)`), or an observer that detaches itself makes the next one get skipped.
- **An observer that changes the subject** loops forever. Notify only on an actual change.
- **Observer is pub-sub inside one process, without a broker.**

---

## The discussion

### "An order is placed and three things need to know. Why not just call them?"

```python
def place_order(order_id: str) -> None:
    ...                                 # save the order
    inventory.reserve(order_id)
    email.send_receipt(order_id)
    analytics.track(order_id)
```

It works, and every line couples the order code to another team's class. Product asks for an SMS too: you edit `place_order`. Analytics is retired: you edit it again. The order code knows everyone who cares about orders, and it shouldn't need to.

### "So what does Observer change?" (missed: the crux)

The subject stops knowing *who* is listening. It keeps a list of things that implement one interface, and tells each of them:

```python
from abc import ABC, abstractmethod
from typing import Callable

class OrderObserver(ABC):
    @abstractmethod
    def on_order_placed(self, order_id: str) -> None: ...

class OrderService:
    def __init__(self) -> None:
        self._observers: list[OrderObserver] = []

    def attach(self, observer: OrderObserver) -> None:
        self._observers.append(observer)

    def detach(self, observer: OrderObserver) -> None:
        self._observers.remove(observer)

    def place(self, order_id: str) -> None:
        # ... save the order
        self._notify(order_id)

    def _notify(self, order_id: str) -> None:
        for observer in list(self._observers):          # a copy: see below
            try:
                observer.on_order_placed(order_id)
            except Exception as error:                  # one failure doesn't stop the rest
                print(f"observer failed: {error}")
```

SMS arrives: one new observer class and one `attach` call at setup. `OrderService` doesn't change. That's the whole pattern; the rest of this note is about what goes wrong around `_notify`.

### "The email server is down. Does analytics still hear about the order?"

Only if `_notify` catches per observer. Without the `try`, the email observer's exception escapes the loop, and every observer after it never runs. My answer was right: wrap each call.

My first version collected the failures and returned them. Logging is the usual choice: the caller placed an order, and an email outage isn't its problem.

### "An observer detaches itself inside `update`. What can go wrong?" (missed)

I didn't know this one. A `for` loop over a list walks by position. Observers are `[coupon, email, inventory]`:

1. Position 0: `coupon` runs and detaches itself. The list is now `[email, inventory]`.
2. Position 1 is now `inventory`. **`email` is silently skipped.** No error, just a missing receipt.

`list(self._observers)` loops over a snapshot taken before anyone runs, so detaching changes the real list but not the one being walked. One-shot observers ("send a coupon on the first order, then stop") make this a real bug, not a puzzle.

All three cases together:

```python
class FirstOrderCoupon(OrderObserver):
    def __init__(self, service: OrderService):
        self._service = service
        self.sent: list[str] = []

    def on_order_placed(self, order_id: str) -> None:
        self.sent.append(order_id)
        self._service.detach(self)                      # one-shot

class EmailReceipt(OrderObserver):
    def on_order_placed(self, order_id: str) -> None:
        raise ConnectionError("smtp down")

class Inventory(OrderObserver):
    def __init__(self) -> None:
        self.reserved: list[str] = []

    def on_order_placed(self, order_id: str) -> None:
        self.reserved.append(order_id)

service = OrderService()
coupon = FirstOrderCoupon(service)
inventory = Inventory()
service.attach(coupon)
service.attach(EmailReceipt())
service.attach(inventory)

service.place("o1")         # prints: observer failed: smtp down
service.place("o2")         # prints: observer failed: smtp down
coupon.sent                 # ["o1"]: sent once, then detached
inventory.reserved          # ["o1", "o2"]: reached despite the detach and the failure
```

### "What if an observer changes the subject?" (my answer: infinite recursion)

Right. `set_price` notifies, the observer calls `set_price`, which notifies, and so on until `RecursionError`. Two defences:

- observers don't change the subject inside `update`;
- the subject notifies **only when the value actually changed**, which stops loops that settle on a value.

This one also shows the Python shortcut: an observer can be a plain function.

```python
class Stock:
    def __init__(self) -> None:
        self.price = 0.0
        self._observers: list[Callable[[float], None]] = []

    def attach(self, observer: Callable[[float], None]) -> None:
        self._observers.append(observer)

    def set_price(self, price: float) -> None:
        if price == self.price:
            return                                      # no change, no notification
        self.price = price
        for observer in list(self._observers):
            observer(price)

stock = Stock()

def round_to_whole(price: float) -> None:
    stock.set_price(round(price))                       # changes the subject

stock.attach(round_to_whole)
stock.set_price(10.4)
stock.price                 # 10: the second call changed nothing, so the loop stopped
```

Use functions when observers are stateless one-liners; use the `Observer` class when they hold state or the interface has several events. Same trade-off as Strategy.

### "Isn't this just pub-sub?" (missed)

Same idea, with a middleman:

| | Observer | Pub-sub |
|---|---|---|
| Who knows whom | The subject holds its observers | Neither side knows the other; both know only a **broker** and a **topic** (`"order.placed"`) |
| Where | One process | Across services (Kafka, RabbitMQ, Redis pub-sub) |
| Delivery | Synchronous: `notify` returns after all have run | Usually asynchronous: publish and move on |
| Failure | Your problem, right now | The broker can store, retry and redeliver |

So my first line, "Observer helps achieve event-driven architecture", oversells it. Say: **Observer is pub-sub inside one process.** Pub-sub proper comes in HLD and in the pub-sub queue LLD problem.

### Gotchas

- **Looping over the live list.** A self-detaching observer makes the next one skip. Loop over `list(...)`.
- **No per-observer `try`.** One broken observer silences every observer after it.
- **Slow observers.** It's synchronous: a 2-second email call makes `place()` take 2 seconds. Move slow work to a queue.
- **Observers that change the subject.** Infinite loop. Notify only on real change.
- **Forgotten observers.** Never detached, they keep receiving updates and keep their objects alive.
- **Relying on notification order.** Treat it as unordered. If B must run after A, that's a direct call, not two observers.
- **Hard to trace.** "What happens when an order is placed?" means finding every `attach`.

### Connections

- Strategy holds *one* behaviour and uses its result; Observer holds *many* listeners and ignores their results ([01_strategy.md](01_strategy.md)).
- Open/closed: a new listener is a new class, the subject unchanged ([04_solid.md](../../01_lld_in_a_hurry/04_solid.md)).
- Pub-sub and message queues: Track 3 (HLD).

---

## Review questions

Answer first, then open the answer to check.

1. Order placement calls inventory, email and analytics directly. What's the problem, and what does the subject know after switching to Observer?

   <details><summary>Answer</summary>

   - The order code is coupled to every listener; adding SMS or removing analytics edits it.
   - After: only the `OrderObserver` interface and a list. Not who's listening or what they do.

   </details>

2. The email observer raises. What happens to analytics, and what do you do with the error?

   <details><summary>Answer</summary>

   - Without a per-observer `try`, the exception escapes the loop and analytics never runs.
   - Catch per observer and log. Don't return it to the caller: placing an order shouldn't fail on an email outage.

   </details>

3. Observers `[coupon, email, inventory]`. The coupon detaches itself during notify. Which observer misses the event, why, and what's the fix?

   <details><summary>Answer</summary>

   - `email`. The loop walks by position: removing position 0 shifts `email` into it, and the loop moves on to position 1, which is `inventory`.
   - Loop over a snapshot: `for observer in list(self._observers)`.

   </details>

4. An observer calls `stock.set_price(round(price))` inside its update. What happens, and how do you defend against it?

   <details><summary>Answer</summary>

   - Each `set_price` notifies again: infinite recursion, `RecursionError`.
   - Notify only when the value changes, and don't let observers change the subject in `update`.

   </details>

5. Observer vs pub-sub with Kafka: three differences.

   <details><summary>Answer</summary>

   - Coupling: the subject holds observer references; in pub-sub both sides know only a broker and a topic.
   - Scope and timing: one process and synchronous, versus across services and usually asynchronous.
   - Failure: an observer's failure is handled right now; a broker can store, retry and redeliver.

   </details>
