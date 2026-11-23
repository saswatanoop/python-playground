# Adapter

Source: not on Hello Interview's Design Patterns page. Taught by Claude on 22 Nov 2026 in the page's style; teach-back pending.

## What this note covers

- The mismatch Adapter fixes: a class you can't change, with the wrong interface
- What the adapter translates
- Adapter vs Facade vs Decorator

## Review points

- **What it is:** a wrapper that makes an existing class fit the interface your code expects, by translating calls (names, arguments, units, return types).
- **Use it when** you integrate a third-party or legacy class you can't or shouldn't edit.
- **Signal:** "integrate an external payment gateway / SMS provider / legacy system", "the APIs don't match".
- **Payoff:** your code depends on your own interface. Swap the vendor and only the adapter changes.
- **Adapter changes the interface. Decorator keeps it and adds behaviour. Facade makes a new, simpler one over several objects.**

---

## The discussion

### "Our code pays through `PaymentGateway.pay(rupees)`. The new vendor's SDK has a different API. Now what?"

Our interface, which checkout and refunds already use:

```python
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    @abstractmethod
    def pay(self, user: str, rupees: float) -> str: ...      # returns a payment id
```

The vendor's SDK, which we can't edit:

```python
class VendorPayClient:
    def create_charge(self, amount_paise: int, customer_ref: str) -> dict:
        return {"status": "ok", "charge_id": f"ch_{customer_ref}_{amount_paise}"}
```

Different method name, paise instead of rupees, arguments in another order, a dict instead of an id. Calling the SDK directly from checkout spreads those details everywhere, and the next vendor change touches every caller.

### "So what does the adapter do?"

It implements *our* interface and translates each call into the vendor's:

```python
class VendorPayAdapter(PaymentGateway):
    def __init__(self, client: VendorPayClient):
        self._client = client

    def pay(self, user: str, rupees: float) -> str:
        paise = round(rupees * 100)                             # units
        response = self._client.create_charge(paise, user)      # name and argument order
        if response["status"] != "ok":
            raise RuntimeError("payment failed")
        return response["charge_id"]                            # their dict → our id

client = VendorPayClient()
gateway = VendorPayAdapter(client)
gateway.pay("alice", 12.5)      # "ch_alice_1250"
```

Checkout keeps calling `gateway.pay(...)` and never learns the vendor exists. Next year's vendor is one new adapter.

### "Adapter, Facade, Decorator: all three wrap things. How do I keep them apart?"

| | What it wraps | Interface the caller sees | Why |
|---|---|---|---|
| **Adapter** | One object | A *different* one: the interface the caller already expects | Make an incompatible class fit |
| **Decorator** | One object | The *same* one as the wrapped object | Add behaviour, stackable |
| **Facade** | Several objects | A *new, simpler* one | Hide coordination |

One-liner: Adapter converts, Decorator adds, Facade simplifies.

### "Is this worth a class in Python?"

If the mismatch is one function, a small function or a lambda can adapt it. Use the class when the vendor needs setup (a client object), there are several methods to translate, or your code expects an interface it can type-check. In an LLD interview, the class is what's expected.

### Gotchas

- **Leaking the vendor's types.** Returning their `dict` or raising their exceptions brings the coupling back. Translate results and errors too.
- **Business logic in the adapter.** It translates; fraud checks and retries belong elsewhere.
- **Units.** Rupees vs paise, seconds vs milliseconds: the classic adapter bug. Name the units in parameter names.
- **Confusing it with Facade.** One object, interface changed: Adapter. Many objects, new simple entry point: Facade.

### Connections

- Dependency inversion: your code owns the interface, the vendor is plugged in behind it ([04_solid.md](../../01_lld_in_a_hurry/04_solid.md)).
- Fits with Strategy: each adapter is one interchangeable `PaymentGateway` ([01_strategy.md](../03_behavioural/01_strategy.md)).
- Facade vs Adapter, in the Facade note ([03_facade.md](03_facade.md)).

---

## Review questions

Answer first, then open the answer to check.

1. A vendor SDK charges in paise with `create_charge(amount_paise, customer_ref)` and returns a dict. Your code expects `pay(user, rupees) -> str`. What exactly does the adapter translate?

   <details><summary>Answer</summary>

   - The method name, the argument order, rupees to paise, and the dict to a payment id.
   - Also the vendor's errors into yours, so nothing vendor-specific leaks out.

   </details>

2. Why not call the SDK directly from checkout?

   <details><summary>Answer</summary>

   - Vendor details spread into every caller, and a vendor change touches all of them.
   - With an adapter, callers depend on your interface; a new vendor is one new adapter.

   </details>

3. Adapter, Decorator, Facade: one line each on what changes about the interface.

   <details><summary>Answer</summary>

   - Adapter: one object, interface converted to the one callers expect.
   - Decorator: one object, same interface, behaviour added.
   - Facade: several objects behind a new, simpler interface.

   </details>
