# Facade

Source: Hello Interview LLD → Design Patterns, and my teach-back on 21 Nov 2026. The crux and both uses were mine; "what goes wrong without one" was written by Claude at my request, so teach that part back on review.

## What this note covers

- The problem: callers that know too much
- What a facade is, and why I already build one in every LLD round
- A front door, not a wall
- The god-class trap

## Review points

- **What it is:** a class that coordinates several components behind a few simple methods. Hello Interview: "just a coordinator class that hides complexity".
- **Two uses:** an orchestrator designed from scratch (`Game`, `ParkingLot`), or a simpler front over existing messy code.
- **The problem it solves:** without it, every caller knows the steps, their order and the cleanup on failure, so a change in one subsystem breaks every caller.
- **The facade orders the steps; the subsystems own the rules.** A facade full of business logic is a god class.
- **A front door, not a wall:** subsystems stay usable directly.
- **In interviews you rarely name it.** Build the clean orchestrator and spend the words on the patterns that carry the design.

---

## The discussion

### "Checkout needs inventory, payment and shipping. Why not just call them?"

You can, once. Here are the three subsystems:

```python
class Inventory:
    def __init__(self) -> None:
        self.stock = {"book": 2}

    def reserve(self, item: str) -> None:
        if self.stock.get(item, 0) == 0:
            raise ValueError(f"{item} is out of stock")
        self.stock[item] -= 1

    def release(self, item: str) -> None:
        self.stock[item] += 1

class PaymentGateway:
    def __init__(self, balances: dict[str, float]):
        self.balances = balances

    def charge(self, user: str, amount: float) -> None:
        if self.balances.get(user, 0) < amount:
            raise ValueError("payment declined")
        self.balances[user] -= amount

class Shipping:
    def schedule(self, user: str, item: str) -> str:
        return f"shipping {item} to {user}"
```

And a caller that uses them directly:

```python
def web_checkout(inventory: Inventory, payments: PaymentGateway, shipping: Shipping,
                 user: str, item: str, amount: float) -> str:
    inventory.reserve(item)
    try:
        payments.charge(user, amount)
    except ValueError:
        inventory.release(item)          # forget this and stock leaks
        raise
    return shipping.schedule(user, item)

# mobile_checkout() and admin_reorder() repeat all of it
```

Every caller must know the order (reserve before charging), the cleanup (release if payment fails) and three classes' APIs. Then the payment team adds a fraud check that must run before `charge`. Now you're editing every caller, and the one you miss is the bug.

### "So what does a facade do?"

It owns that sequence, once:

```python
class Checkout:
    def __init__(self, inventory: Inventory, payments: PaymentGateway, shipping: Shipping):
        self._inventory = inventory
        self._payments = payments
        self._shipping = shipping

    def place_order(self, user: str, item: str, amount: float) -> str:
        self._inventory.reserve(item)
        try:
            self._payments.charge(user, amount)
        except ValueError:
            self._inventory.release(item)
            raise
        return self._shipping.schedule(user, item)

inventory = Inventory()
payments = PaymentGateway({"alice": 50.0, "bob": 5.0})
checkout = Checkout(inventory, payments, Shipping())

checkout.place_order("alice", "book", 12.5)     # "shipping book to alice"
inventory.stock["book"]                         # 1
```

And when bob's card is declined:

```python
try:
    checkout.place_order("bob", "book", 12.5)
except ValueError as error:
    print(error)                                # payment declined
inventory.stock["book"]                         # still 1: the facade released it
```

Callers now know one method. The fraud check is one edit, inside `place_order`.

### "Isn't that just… a normal class that calls other classes?"

Yes, and that's the point Hello Interview makes. You already build facades in every LLD round without naming them:

- **Tic Tac Toe's `Game`:** `make_move(row, col)` checks the game is in progress, places the mark on the `Board`, checks for a win, then a draw, then switches the `Player`. The caller just calls `make_move`.
- **`ParkingLot`:** coordinates floors, spots and tickets behind `park()` and `leave()`.

The name matters more in the second use: you inherit a subsystem with an awkward API and put a simpler front on it.

### "`make_move` has grown to 80 lines. What went wrong?"

My first answer was "split it into a validation call and a move call". That helps readability, but if those become private methods of `Game`, `Game` still owns everything, just in smaller pieces.

Ask *who owns* each piece. Checking rows, columns and diagonals is a board rule, so it goes in `Board.check_win(row, col)`. `Game` only decides the order of the steps.

**The facade orders the steps; the subsystems own the rules.** A facade with business logic in it is a god class with a nice name.

### "Should the facade stop callers from using `Inventory` directly?"

I said yes at first. Two different things are getting mixed:

- **Callers shouldn't *need* to know the internals.** That's the facade's purpose.
- **Callers shouldn't be *able* to reach them.** That's not what a facade does. It's a simpler front door, not a wall. Tests, admin tools and anything needing finer control can still use `Inventory` directly.

For an orchestrator built from scratch, keeping `_board` private is a fine choice, but that's ordinary encapsulation, not the pattern.

### Interviewer: "Which patterns did you use?"

Not just "Facade". Something like: "`ParkingLot` is the entry point that coordinates floors, spots and tickets, so it's a facade." Then move on to the patterns that carry the design, like Strategy for pricing. Hello Interview: build it naturally, name it if it helps, don't worry if you never do.

### Gotchas

- **The god facade.** Rules creeping into the facade. Push them down to the subsystem that owns the data.
- **Forgetting cleanup on partial failure.** Coordinating steps means owning what happens when step 2 fails after step 1 succeeded.
- **A facade over one class** is just indirection. It earns its place when it coordinates several.
- **Announcing it as if it were clever.** Interviewers expect orchestrators; naming one isn't a point scored.

### Connections

- Law of Demeter: callers talk to one friend, `Checkout`, instead of reaching into three ([03_general_principles.md](../../01_lld_in_a_hurry/03_general_principles.md)).
- Decorator keeps one object's interface and adds behaviour ([02_decorator.md](02_decorator.md)); a facade creates a new, simpler interface over several objects.
- Facade vs Adapter: an adapter wraps *one* object and converts its interface to the one callers expect; a facade sits over *several* and offers a new, simpler one ([01_adapter.md](01_adapter.md)).

---

## Review questions

Answer first, then open the answer to check.

1. What goes wrong without a facade when three callers each run checkout themselves?

   <details><summary>Answer</summary>

   - Each caller knows the order, the cleanup on failure and three APIs.
   - A change in one subsystem (a fraud check before charging) means editing every caller, and the one you miss is a bug.

   </details>

2. `Game.make_move` is 80 lines and checks rows, columns and diagonals itself. What's wrong, and where does that code go?

   <details><summary>Answer</summary>

   - `Game` has become a god class. Splitting it into private helpers in `Game` doesn't fix ownership.
   - Win checking is a board rule: `Board.check_win(row, col)`. The facade only orders the steps.

   </details>

3. Should a facade prevent callers from using the subsystems directly?

   <details><summary>Answer</summary>

   - No. It removes the *need* to know them, not the *ability*. Tests and code needing finer control can still use them.
   - Making parts private in an orchestrator you built is ordinary encapsulation, a separate choice.

   </details>

4. `place_order` reserves stock, then payment fails. What must the facade do, and why is that a reason to have one?

   <details><summary>Answer</summary>

   - Release the reserved stock, then re-raise.
   - That cleanup is exactly what each caller would otherwise have to remember. In the facade it's written once.

   </details>

5. Your parking lot design has a `ParkingLot` class coordinating floors, spots and tickets. The interviewer asks which patterns you used. What do you say?

   <details><summary>Answer</summary>

   - One sentence: it's the entry point coordinating the parts, which is a facade.
   - Then the patterns that carry the design, like Strategy for pricing. Naming the facade isn't the point.

   </details>
