# Strategy

Source: Hello Interview LLD → Design Patterns, and my teach-back on 21 Nov 2026 (not clean yet). Sections marked **(missed)** are what I didn't get in the teach-back; re-teach those on review.

## What this note covers

- The `if/elif` on type that Strategy replaces
- Holds the strategy, hands the work to it, swaps it at runtime
- Where the `if/elif` actually goes
- Why not subclass the context
- Strategy plus Decorator: pricing with an EV discount
- Function or class
- When a plain `if` is better

## Review points

- **What it is:** different ways of doing the same thing, each behind one interface. The context *holds* one and *hands the work to it*, and can swap it at runtime.
- **Hello Interview:** the most common pattern in LLD interviews. A pile of `if/elif` on type is a strategy waiting to happen.
- **The `if/elif` doesn't vanish.** Choosing the strategy moves to one place at the edge (setup, config, a factory). The business method stops branching.
- **Why not subclass the context:** it can't switch at runtime, and subclasses multiply when a second thing varies (3 × 3 = 9 vs 3 + 3 = 6).
- **Something applied on top of any strategy** (an EV discount) is a Decorator around the strategy, not another strategy.
- **In Python a stateless strategy can be a function.** Use a class when it has settings or several methods.
- **Count variants, not methods.** Two ways that will never change: a plain `if`.
- **Factory vs Strategy:** Factory picks which class to create. Strategy picks which behaviour an existing object uses.

---

## The discussion

### "Checkout has an `if/elif` per payment type. What's wrong with that?" (missed)

```python
def checkout(payment_type: str, amount: float) -> str:
    if payment_type == "card":
        return f"charged {amount} to card"
    elif payment_type == "wallet":
        return f"debited {amount} from wallet"
    else:
        raise ValueError(f"unknown payment type: {payment_type}")
```

Fine with two branches. Then come UPI, net banking and pay-later, each with real logic inside its branch:

- every new type edits `checkout`, the method that every order runs through;
- the method grows without limit;
- you can't test the wallet logic without going through `checkout`.

### "So what does Strategy do?"

Each way gets its own class behind one interface, and the context holds one and hands the work to it:

```python
from abc import ABC, abstractmethod
from typing import Callable

class PaymentStrategy(ABC):
    @abstractmethod
    def pay(self, amount: float) -> str: ...

class CardPayment(PaymentStrategy):
    def pay(self, amount: float) -> str:
        return f"charged {amount} to card"

class WalletPayment(PaymentStrategy):
    def pay(self, amount: float) -> str:
        return f"debited {amount} from wallet"

class ShoppingCart:
    def __init__(self, payment: PaymentStrategy):
        self._payment = payment                 # holds it

    def set_payment(self, payment: PaymentStrategy) -> None:
        self._payment = payment                 # swappable at runtime

    def checkout(self, amount: float) -> str:
        return self._payment.pay(amount)        # hands the work to it

cart = ShoppingCart(CardPayment())
cart.checkout(100.0)            # "charged 100.0 to card"
cart.set_payment(WalletPayment())
cart.checkout(50.0)             # "debited 50.0 from wallet"
```

`checkout` has no branches. A new payment type is a new class; `ShoppingCart` never changes. As Hello Interview puts it, "this is just polymorphism with a pattern name".

### "Doesn't the `if/elif` just move somewhere else?" (missed)

Yes, and say so before the interviewer does. Something still has to turn "the user picked wallet" into `WalletPayment()`. The gain is *where* that happens: once, at the edge, instead of inside every business method.

That "somewhere" is often a factory, which is how the two patterns work together:

```python
def create_payment(kind: str) -> PaymentStrategy:
    if kind == "card":
        return CardPayment()
    if kind == "wallet":
        return WalletPayment()
    raise ValueError(f"unknown payment type: {kind}")

payment = create_payment("wallet")      # Factory: which class
cart = ShoppingCart(payment)            # Strategy: the cart uses it
cart.checkout(30.0)                     # "debited 30.0 from wallet"
```

### "Why not subclass the cart, `CardCart` and `WalletCart`?" (missed)

My answer was "open/closed breaks". That's a label, not a failure, and arguably wrong: adding `UpiCart` edits nothing. The real failures:

- **It can't switch at runtime.** The user changes payment method halfway through checkout. A cart's class is fixed, so you'd build a new cart and copy the items. With Strategy it's `cart.set_payment(...)`.
- **Subclasses multiply when a second thing varies.** 3 payments × 3 discount types = 9 subclasses (`WalletStudentDiscountCart`…). As strategies: 3 + 3 = 6 classes, combined freely. The same explosion as in Decorator.

Composition over inheritance, in one example.

### "Design parking lot pricing: hourly, a flat weekend rate, an EV discount." (missed)

My answer, "have a price strategy and pass the right one", only restated the pattern. Name the pieces:

```python
class PricingStrategy(ABC):
    @abstractmethod
    def calculate(self, hours: float) -> float: ...

class HourlyPricing(PricingStrategy):
    def __init__(self, rate: float):
        self._rate = rate

    def calculate(self, hours: float) -> float:
        return self._rate * hours

class WeekendFlatPricing(PricingStrategy):
    def __init__(self, fee: float):
        self._fee = fee

    def calculate(self, hours: float) -> float:
        return self._fee

class ParkingLot:
    def __init__(self, pricing: PricingStrategy):
        self._pricing = pricing

    def exit(self, hours: float) -> float:
        return self._pricing.calculate(hours)
```

And the trap: the EV discount isn't a third way to price. It applies *on top of* hourly or weekend. Making it a strategy forces `EvHourlyPricing` and `EvWeekendPricing`, the explosion again. It's a **Decorator around the strategy**: same interface, wraps any pricing.

```python
class EvDiscount(PricingStrategy):
    def __init__(self, inner: PricingStrategy, percent: float):
        self._inner = inner
        self._percent = percent

    def calculate(self, hours: float) -> float:
        full = self._inner.calculate(hours)
        return full * (1 - self._percent / 100)

hourly = HourlyPricing(rate=40)
weekday_lot = ParkingLot(hourly)
weekday_lot.exit(3)                     # 120

ev_weekend = EvDiscount(WeekendFlatPricing(fee=100), percent=25)
ev_lot = ParkingLot(ev_weekend)
ev_lot.exit(3)                          # 75.0
```

A new pricing rule is one new class plus one line where pricing is chosen. `ParkingLot` doesn't change.

### "In Python, does a strategy need a class?" (missed)

No. I answered a different question in the teach-back. A strategy is "something with the right signature", and a function qualifies:

```python
def night_flat_fee(hours: float) -> float:
    return 50.0

class Meter:
    def __init__(self, pricing: Callable[[float], float]):
        self._pricing = pricing

    def charge(self, hours: float) -> float:
        return self._pricing(hours)

meter = Meter(night_flat_fee)           # no parentheses: pass the function
meter.charge(3)                         # 50.0
```

Still Strategy, minus the ceremony. Use a **class** when the strategy has settings or state (`HourlyPricing(rate=40)`), needs several methods, or needs to be wrapped like `EvDiscount`. Use a **function** for a stateless one-method rule. In an interview, the class version is the safe default because it's what interviewers expect to see; mention the function version as the Python shortcut.

### Interviewer: "Two payment methods that will never change. Strategy?" (missed)

No. A plain `if` is simpler. I said "if there won't be new methods", which counts the wrong thing: what matters is new *variants* (ways of doing it), not new methods. Strategy pays when variants get added or swapped.

### "Factory and Strategy both remove an `if/elif`. What's the difference?"

- **Factory** decides *which class to create* from a key. It runs once, at creation.
- **Strategy** decides *which behaviour an existing object uses*. The object holds it, delegates to it and can swap it.

They often appear together: a factory creates the strategy, the context uses it.

### Gotchas

- **A context with no strategy.** Hello Interview's cart starts with `None` and a setter, so `checkout` before `set_payment` crashes. Take the strategy in the constructor.
- **Passing the whole context into the strategy.** Pass only what it needs (`hours`, not the `ParkingLot`), or the strategy gets coupled to everything.
- **A "strategy" that's really a modifier.** If it applies on top of any strategy, it's a decorator.
- **Strategy for every `if`.** Two fixed branches don't need classes.

### Connections

- Polymorphism plus composition over inheritance ([06_polymorphism_inheritance.md](../../01_lld_in_a_hurry/06_polymorphism_inheritance.md)); open/closed ([04_solid.md](../../01_lld_in_a_hurry/04_solid.md)).
- Factory creates the strategy ([02_factory.md](../01_creational/02_factory.md)); Decorator wraps it ([02_decorator.md](../02_structural/02_decorator.md)).
- Strategy vs State: to fill in when I do State.

---

## Review questions

Answer first, then open the answer to check.

1. "Strategy removes the `if/elif`." True? Where does the choice happen now?

   <details><summary>Answer</summary>

   - Not quite: choosing the strategy still needs a decision.
   - It moves to one place at the edge (setup, config, or a factory), so business methods like `checkout` stop branching.

   </details>

2. Why not `CardCart(ShoppingCart)` and `WalletCart(ShoppingCart)`? Give the concrete failures, not a principle's name.

   <details><summary>Answer</summary>

   - Can't switch at runtime: changing payment mid-checkout means a new cart and copying items.
   - Multiplies with a second varying thing: 3 payments × 3 discounts = 9 subclasses, versus 6 strategy classes.

   </details>

3. Parking pricing is hourly or weekend-flat, and EV owners get 25% off either. Which pieces, and why isn't the discount a strategy?

   <details><summary>Answer</summary>

   - `PricingStrategy` with `HourlyPricing` and `WeekendFlatPricing`; `ParkingLot` holds one and calls `calculate`.
   - The discount applies on top of any pricing, so it's a decorator: `EvDiscount(inner, percent)` implements `PricingStrategy` and wraps another. As a strategy it would need one class per combination.

   </details>

4. When should a Python strategy be a function, and when a class?

   <details><summary>Answer</summary>

   - Function: stateless, one method, `Callable[[float], float]`.
   - Class: it has settings or state (`rate=40`), several methods, or must be wrapped by a decorator. In an interview, default to the class and mention the function shortcut.

   </details>

5. Two payment methods, fixed forever. Strategy or `if`? What do you count to decide?

   <details><summary>Answer</summary>

   - A plain `if`.
   - Count variants (ways of doing it) and how often they change, not methods.

   </details>
