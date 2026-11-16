# General design principles

Source: Hello Interview LLD → In a Hurry → Design Principles (part 1 of 2). Learned 10 Nov 2026. Next: [04_solid.md](04_solid.md).

## What this note covers

- KISS, DRY and YAGNI: what each says, and when each stops applying
- Separation of Concerns, with the TicTacToe split
- Law of Demeter, and why fluent chains don't break it
- The tensions between the principles and how to state the trade-off in an interview

## Review points

- Two groups: **general** (this note) and **object-oriented** (SOLID; OOD gets more interview attention). The three general ones to remember: KISS, DRY, YAGNI.
- Interviewers want you to *apply* the principles and explain your calls; reciting acronyms earns nothing.
- **KISS:** simplest thing that works. Add complexity only when simplicity breaks.
- **DRY:** same *logic* goes in one place. Code that merely *looks* similar can stay duplicated.
- **YAGNI:** think ahead, don't build ahead.
- **SoC:** each part owns one responsibility and doesn't know the others' internals.
- **Law of Demeter:** talk only to your immediate friends. Don't reach through a chain of different objects.

**What each one buys you** (the article's cheat sheet):

| Principle | Buys you |
|---|---|
| KISS | Start simple, add complexity only when needed |
| DRY | Less duplication, easier maintenance |
| YAGNI | Build for today, not hypothetical futures |
| SoC | Independent testing and changes |
| Law of Demeter | Less coupling, hidden internal structure |

- Senior signal: name the tension (DRY vs KISS, YAGNI vs extensibility) and say which way you lean *and why*.
- ⚠ **My weak spot (missed twice):** *when not to abstract*. With one or two stable cases, keep it simple and name the seam. Ask whether variation is coming before you build an interface. Test this first in a review.

---

### KISS: Keep It Simple, Stupid
- **Crux:** pick the simplest solution that works. If a conditional does the job, don't write a strategy pattern.
- **Most-violated principle in LLD interviews.** Over-engineering usually comes from showing off patterns (factories, builders, decorators), and interviewers notice it.
- **When to add complexity:**
  - a class grows into a god class (about 500 lines, many responsibilities)
  - adding one variant (e.g. a payment method) means editing several places

### DRY: Don't Repeat Yourself
- **Crux:** logic that is *conceptually the same* lives in one place, so a fix or a rule change happens once.
- **Don't overdo it.** If code is textually similar but means different things, sharing it creates *artificial coupling*: a change made for one breaks the other.
  - Example: `ParkingTicket` computes `hours * rate` and `LibraryLoan` computes `days * fine`. Keep them separate; they will evolve differently.

```python
# DRY: same rule, one place
def is_valid_email(email: str) -> bool:
    return "@" in email and "." in email.split("@")[-1]

class User:
    def __init__(self, email: str):
        if not is_valid_email(email):
            raise ValueError("bad email")
        self.email = email

class Newsletter:
    def subscribe(self, email: str):
        if not is_valid_email(email):
            raise ValueError("bad email")

# NOT DRY-worthy: looks alike, means different things. Keep separate.
class ParkingTicket:
    def fee(self) -> float:
        return self.hours * self.hourly_rate     # may add peak pricing
class LibraryLoan:
    def fine(self) -> float:
        return self.days_late * self.fine_per_day  # may add a cap
```
- **DRY vs KISS:** sometimes duplicating is simpler than adding an abstraction. A line to use in an interview: *"I'll keep this in `User` for now. If it shows up 3–4 times, I'll pull it into a shared validator."*

### YAGNI: You Aren't Gonna Need It
- **Crux:** think ahead, but don't build ahead. Design with extension in mind and implement only what the requirements ask for.
- **Why:** guesses about the future are usually wrong. You end up maintaining dead code, and the real requirement still doesn't fit.
- **In interviews:** no valet parking or EV charging unless the requirements ask for it. Save extensions for when the interviewer asks "how would you extend this?", then *talk* through them.
- **Example, a parking lot with one pricing rule:**
  - Don't build a `PricingStrategy` interface on a guess that rules will change.
  - Ask the interviewer: "Will there be multiple pricing rules?"
    - If yes, the strategy is justified.
    - If no, keep the fee calculation in a single `calculate_fee()` method. That is the one seam where a strategy can slot in later.

```python
class ParkingLot:
    HOURLY_RATE = 20

    def calculate_fee(self, ticket: Ticket) -> float:   # the seam: only place pricing lives
        return ticket.hours() * self.HOURLY_RATE

# Pricing inside ParkingLot is a 2nd responsibility (SRP), accepted on purpose (KISS):
# one line of logic doesn't earn its own class yet. Say that out loud in the interview.

# Later, only if a 2nd rule appears:
#   calculate_fee() -> self.pricing.calculate(ticket)   (PricingStrategy injected)
```

### Separation of Concerns (SoC)
- **Problem:** if one class mixes UI, input, rules and storage, every change risks breaking unrelated things, and you can't test any one part alone.
- **Crux:** split the code by responsibility (UI / business logic / data access). Each part talks to the others through a small interface and never uses their internals.
- **Example: TicTacToe.**
  - Bad: `play()` prints the board, reads `input()` and checks for a win, all in one method. Win logic can't be tested without stdin, and moving to a GUI means rewriting the game loop.
  - Good: `TicTacToe(board, display, input_handler)`. Each part changes on its own:
    - console → GUI: only `InputHandler` changes
    - new look: only `Display` changes
    - new win rule: only `Board` changes

```python
# Bad: display + input + rules in one method
class TicTacToe:
    def play(self):
        while True:
            for row in self.board:
                print(row)          # display
            r, c = int(input()), int(input())          # input
            self.board[r][c] = "X"
            if self.board[0][0] == self.board[1][1] == self.board[2][2]:  # rules
                print("Winner!")
                break

# Good: each concern in its own class, the game just coordinates
class TicTacToe:
    def __init__(self, board: Board, display: Display, input_handler: InputHandler):
        self.board = board
        self.display = display
        self.input_handler = input_handler

    def play(self):
        while not self.board.is_game_over():           # win OR full board (draw)
            self.display.render(self.board)
            self.board.make_move(self.input_handler.get_next_move())
        self.display.show_result(self.board.get_winner())   # None = draw
```
- **SoC vs SRP:** SoC is the broad idea and applies at any level (layers, modules, classes). SRP is SoC applied to one class: *one reason to change*.

### Law of Demeter (principle of least knowledge)
- **Problem:** `garage.get_floor(2).get_spot(14).get_vehicle().license_plate` makes the caller depend on the internal structure of Garage, Floor, Spot and Vehicle. If any of them reorganizes, the caller breaks.
- **Crux:** a method talks only to its immediate friends: itself, its fields, its parameters, and objects it creates. It doesn't reach through them to strangers.
- **Fix:**
  - Give the top object a method that does the navigation for the caller: `garage.get_plate_at(floor, spot)`. Stricter form: each object delegates one step down (`floor.get_plate_at(spot)` → `spot.get_plate()`).
  - Or return exactly the data the caller needs, not an object graph it has to dig through.
- **Not a violation: fluent chaining.** In `query.where(...).order_by(...).limit(10)` and `builder.set_name().build()`, every call returns the *same type*, so no internal structure leaks. The rule is about crossing *different* object types.

```python
# Bad: caller knows Garage -> Floor -> Spot -> Vehicle
plate = garage.get_floor(2).get_spot(14).get_vehicle().license_plate

# Good: each object talks only to its direct friend and delegates one step down
class Spot:
    def get_plate(self) -> str | None:
        return self.vehicle.license_plate if self.vehicle else None   # vehicle is Spot's own field

class Floor:
    def get_plate_at(self, spot: int) -> str | None:
        return self.spots[spot].get_plate()

class Garage:
    def get_plate_at(self, floor: int, spot: int) -> str | None:
        return self.floors[floor].get_plate_at(spot)

plate = garage.get_plate_at(2, 14)

# Fine: fluent chaining, every call returns the same Query type
query.where("age > 30").order_by("name").limit(10)
```

---

## Review questions

Answer first, then open the answer to check.

1. When does DRY hurt? Give an example of similar code that should stay duplicated.

   <details><summary>Answer</summary>

   - When code only *looks* alike but means different things. Sharing it creates artificial coupling: a change made for one breaks the other.
   - Example: `ParkingTicket` (`hours * rate`) and `LibraryLoan` (`days * fine`) will evolve differently, so keep them separate.
   - Also when the abstraction is more complex than two copies (DRY vs KISS). Extract when it shows up 3–4 times.

   </details>

2. The interviewer sees you adding a factory and a strategy for a problem with one variant. Which principles are you breaking, and what should trigger adding them?

   <details><summary>Answer</summary>

   - KISS and YAGNI.
   - Triggers: a real second or third variant, requirements that say variants will keep growing, or adding a variant means editing several places.

   </details>

3. Parking lot with one pricing rule: interface now or not? What do you ask, and where do you leave the seam? The interviewer then says pricing inside `ParkingLot` violates SRP: defend it, and say what would change your mind.

   <details><summary>Answer</summary>

   - Not now. Ask: "Will there be multiple pricing rules?" If yes, a `PricingStrategy` is justified.
   - If no, keep pricing in one `calculate_fee()` method. That is the seam.
   - Defence: yes, it's a second responsibility, accepted on purpose. One line of logic doesn't earn its own class (KISS).
   - Change my mind: a second pricing rule appears. Then `calculate_fee()` delegates to an injected `PricingStrategy`.

   </details>

4. What concrete signals tell you to split a class or add a pattern, and what tells you to stop splitting?

   <details><summary>Answer</summary>

   - Split: the class is becoming a god class (about 500 lines, many responsibilities), adding one variant means editing several places, or a part can't be tested alone.
   - Stop: one class does the job without getting messy, the new class would hold a line or two, or there are only one or two stable cases and no stated variation.

   </details>

5. TicTacToe does display, input and win-checking in one `play()`. Name two concrete problems and the split that fixes them. Is that split SoC or SRP, and what is the difference?

   <details><summary>Answer</summary>

   - Problems: win logic can't be tested without stdin, and moving to a GUI means rewriting the game loop.
   - Split: `Board` (rules), `Display`, `InputHandler`; `TicTacToe` only coordinates.
   - Both. SoC is the broad idea at any level (layers, modules, classes). SRP is SoC applied to one class: one reason to change.

   </details>

6. Is `query.where().order_by().limit()` a Law of Demeter violation? Is `order.get_customer().get_address().get_zip()`? Why, and how do you fix the second?

   <details><summary>Answer</summary>

   - First: no. Every call returns the same type, so no internal structure leaks.
   - Second: yes. The caller depends on the internals of `Order`, `Customer` and `Address`, and breaks if any of them reorganizes.
   - Fix: `order.get_customer_zip()` does the navigation (strict form: each object delegates one step down), or return just the data the caller needs.

   </details>
