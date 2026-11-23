# Design patterns: overview

Source: Hello Interview LLD → Design Patterns (intro, category intros and "Wrapping Up"). Drafted by Claude on 22 Nov 2026 from its explanation of the three categories; teach the categories back before treating them as learned.

## What this note covers

- What a pattern is, and how many actually matter
- How many to use in one design
- The three categories, and a test for telling them apart
- The cheat sheet: each pattern, when to use it, the words that signal it

## Review points

- **A pattern is a name for a structure good design produces anyway.** Of the Gang of Four's 23 (1994, C++ and Smalltalk), interviews ask about roughly five.
- **Don't force them.** Most good designs use none, or one or two. Reaching for three or more usually means over-engineering.
- **Where I'm interviewing matters:** in the US, LLD rounds judge the design, not pattern names; in India, interviewers more often ask about patterns directly. Know the names and when *not* to use each.
- **Creational:** how an object gets made. **Structural:** how objects are wired together. **Behavioural:** who does the work and how objects talk.
- **Test:** if the class diagram *is* the point (who wraps whom), it's structural. If the diagram is boring and the point is what happens at runtime, it's behavioural.

---

## The discussion

### "There are 23 patterns. Do I need all of them?"

No. The Gang of Four book is from 1994, written for C++ and Smalltalk. Since then:

- languages absorbed some patterns: iterators are a built-in, not a pattern;
- the move from deep inheritance to composition and functions made others obsolete.

Hello Interview's count: you'll be asked about maybe five. The eight on the page matter most; Composite, Adapter, Chain of Responsibility and Command are a second tier, worth knowing for pattern-name questions and a few LLD problems.

### "So should I use as many as I can, to show I know them?"

The opposite. Patterns should fall out of good design decisions, not drive them. The most common mistake is fitting a pattern where it doesn't belong, and interviewers read it as over-engineering.

Rule of thumb from the page: most interview-ready designs use no patterns, or one or two. Solve the problem cleanly first, then name the pattern if it fits.

### "Why are they split into three groups?"

Each group answers a different question:

| Group | The question | Hello Interview's wording | Patterns |
|---|---|---|---|
| **Creational** | *How* does an object get made, and which one? | "control how objects get created"; hide construction details, keep code from being tied to concrete classes | Factory, Builder, Singleton |
| **Structural** | *How are objects wired together?* Who wraps or holds whom, and what interface the outside sees | "how objects connect to each other" | Decorator, Facade, Composite, Adapter |
| **Behavioural** | *Who does the work, and how do objects talk?* Which algorithm runs, who hears about a change, what's allowed now | "how objects interact and distribute responsibilities"; "the flow of control" | Strategy, Observer, State, Chain of Responsibility, Command |

### "Structural and behavioural both use composition. How do I tell them apart?"

Draw the class diagram:

- **Structural:** the diagram *is* the point. A decorator wraps an object with the same interface; a facade sits in front of several. At runtime, calls mostly just pass through.
- **Behavioural:** the diagram is boring, often just "a class holds an interface". The point is what happens *while running*: which implementation runs, who gets notified, which state comes next.

The tricky pair is **Decorator vs Strategy**, which look the same on paper:

- **Decorator** wraps an object and has the *same interface* as it. You stack decorators to add layers around the real work.
- **Strategy** is a pluggable piece the context *hands a decision to*. You swap strategies; you don't stack them.

Ask: am I adding a layer around something (structural), or choosing which way to do something (behavioural)?

### "How much does knowing the group matter in an interview?"

Little. Nobody asks "is Observer behavioural?". They ask "which pattern fits here, and why?". The groups help you organise review and give each pattern a one-line purpose.

### The cheat sheet

| Pattern | Use it when | Signal in the requirements | Note |
|---|---|---|---|
| Factory | Callers shouldn't care which concrete class gets created, and the choice is made in several places or arrives as data | "different notification types", "multiple payment methods" | [01_factory.md](01_creational/01_factory.md) |
| Builder | An object has lots of optional fields, rules across fields, or is assembled over several steps | "configurable request", "optional settings" | [02_builder.md](01_creational/02_builder.md) |
| Singleton | Exactly one shared instance across the system (rare in interviews; usually inject instead) | "one shared config / pool / cache" | [03_singleton.md](01_creational/03_singleton.md) |
| Decorator | Optional behaviours layered at runtime without a subclass per combination | "optional features", "stack behaviours", "add logging / encryption to some" | [01_decorator.md](02_structural/01_decorator.md) |
| Facade | Hide internal complexity behind a simple entry point | an orchestrator: `Game`, `ParkingLot`, `Checkout` | [02_facade.md](02_structural/02_facade.md) |
| Strategy | Replace `if/elif` on type with interchangeable behaviours | "different ways to price / pay / route" | [01_strategy.md](03_behavioural/01_strategy.md) |
| Observer | Several components react to one event | "notify", "update multiple components" | [02_observer.md](03_behavioural/02_observer.md) |
| State machine | Behaviour depends on the current state, and transitions get messy | "state", "status", "stage" repeated; vending machine, workflow | [03_state.md](03_behavioural/03_state.md) |

Not on the Hello Interview page, but worth knowing (especially for pattern-name questions in India):

| Pattern | Use it when | Signal in the requirements | Note |
|---|---|---|---|
| Composite | Items and groups of items must be treated the same, in a tree | "folders inside folders", "nested", "total over the hierarchy" | [03_composite.md](02_structural/03_composite.md) |
| Adapter | A class you can't change has the wrong interface | "integrate a third-party / legacy API" | [04_adapter.md](02_structural/04_adapter.md) |
| Chain of responsibility | A request passes along handlers until handled, or through each | "escalate", "approval limits", "middleware", "dispense notes" | [04_chain_of_responsibility.md](03_behavioural/04_chain_of_responsibility.md) |
| Command | Actions must be stored, undone, queued or replayed | "undo/redo", "queue jobs", "macros" | [05_command.md](03_behavioural/05_command.md) |
 Recognize-only, no notes: proxy, template method, iterator, abstract factory.

### Gotchas

- **Pattern-first design.** Start from the requirements; name patterns afterwards.
- **Three or more patterns in one design.** Probably forced.
- **Naming without judgement.** In an Indian interview, "I used Strategy" isn't enough; say what varied and why a plain `if` wasn't enough.
- **Treating the groups as the content.** The useful part is each pattern's "use it when", not its category.

---

## Review questions

Answer first, then open the answer to check.

1. Why do modern interviews cover about five patterns, not 23?

   <details><summary>Answer</summary>

   - Languages absorbed some (iterators are built in), and the move from deep inheritance to composition and functions made others obsolete.
   - Interviewers ask about the few that still show up in real designs.

   </details>

2. Your design uses Factory, Builder, Strategy and Observer. What should you suspect?

   <details><summary>Answer</summary>

   - Over-engineering: most good designs use none, or one or two.
   - Check each against the requirement that's supposed to need it, and drop the ones that are only there to show knowledge.

   </details>

3. In one line each: what question does each of the three groups answer?

   <details><summary>Answer</summary>

   - Creational: how an object gets made, and which one.
   - Structural: how objects are wired together.
   - Behavioural: who does the work and how objects talk at runtime.

   </details>

4. Decorator and Strategy both hold an object behind an interface. Why is one structural and the other behavioural?

   <details><summary>Answer</summary>

   - Decorator adds a layer around an object with the same interface; the wrapping structure is the point, and decorators stack.
   - Strategy is a pluggable decision the context delegates to; the point is which behaviour runs, and strategies are swapped, not stacked.

   </details>

5. Match each to a pattern: "send alerts by email, SMS or push, chosen per user"; "log and encrypt writes, but only for some data sources"; "an order moves from paid to shipped to delivered".

   <details><summary>Answer</summary>

   - Alerts by type: Factory to create the right notifier (Strategy if one object swaps how it sends).
   - Log and encrypt some sources: Decorator.
   - Order lifecycle: State machine.

   </details>
