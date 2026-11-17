# Builder

Source: Hello Interview LLD → Design Patterns, and my teach-back on 17 Nov 2026 (clean on the second pass).

## What this note covers

- What a builder is, and why Python needs it far less than Java
- The three cases where it still earns its place
- Why validation lives in `build()`
- Builder versus Factory

## Review points

- **What it is:** a separate, mutable builder object collects choices one method call at a time; `build()` validates them and returns the real object.
- **In Python, try keyword arguments and defaults first.** They already fix the unreadable ten-parameter call.
- **Use a builder when** the choices are gathered across several statements or branches, a rule spans several fields, or the finished object must be immutable.
- **`build()` is the single place for validation,** because only it sees all the fields together.
- **Factory picks *which class*. Builder configures *one object* step by step.**
- Rare in LLD interviews: mostly API clients, queries and configuration objects.

---

## The discussion

### "Ten constructor parameters and I only want to set two. What's the problem?"

In Java, the call site: `new HttpRequest(url, null, null, null, 30, null)`. Nobody can tell which `null` is which, and that is the pain Builder was invented for.

In Python it isn't a problem. Keyword arguments and defaults already give a readable call:

```python
from dataclasses import dataclass
from typing import Mapping

@dataclass(frozen=True)
class RetryPolicy:
    attempts: int = 3
    delay_seconds: float = 1.0
    backoff: float = 2.0

policy = RetryPolicy(attempts=5)        # the other two keep their defaults
```

That was my first answer in the teach-back, and it's right. If this covers your case, stop here: no builder.

### "So is Builder dead in Python?"

No. Building *incrementally* is still something one constructor call can't do. Three concrete cases:

- **The choices are gathered over time.** A header added inside an `if`, a filter added in a loop. A constructor call can't be spread across statements; a builder can.
- **A rule spans several fields.** "A POST needs a body" can't be checked while setting either one alone.
- **The finished object should be immutable.** The builder is the mutable draft; what comes out is frozen.

```python
@dataclass(frozen=True)
class HttpRequest:
    url: str
    method: str
    headers: Mapping[str, str]
    body: str | None
    timeout: int

class HttpRequestBuilder:
    def __init__(self, url: str):               # required values go in the constructor
        self._url = url
        self._method = "GET"
        self._headers: dict[str, str] = {}
        self._body: str | None = None
        self._timeout = 30

    def method(self, method: str) -> "HttpRequestBuilder":
        self._method = method
        return self                              # same builder back, so calls chain

    def header(self, name: str, value: str) -> "HttpRequestBuilder":
        self._headers[name] = value
        return self

    def body(self, body: str) -> "HttpRequestBuilder":
        self._body = body
        return self

    def timeout(self, seconds: int) -> "HttpRequestBuilder":
        self._timeout = seconds
        return self

    def build(self) -> HttpRequest:
        if self._method == "POST" and self._body is None:
            raise ValueError("a POST request needs a body")
        return HttpRequest(self._url, self._method, dict(self._headers), self._body, self._timeout)
```

Using it, with a choice made in a branch:

```python
auth_token = "abc123"

builder = HttpRequestBuilder("https://api.example.com/orders")
builder.method("POST").header("Content-Type", "application/json")
if auth_token:
    builder.header("Authorization", f"Bearer {auth_token}")
builder.body('{"item": "book"}')

request = builder.build()
request.method                          # "POST"
len(request.headers)                    # 2
```

The methods are called on the *builder*, not on the request. No `HttpRequest` exists until `build()` runs.

### "Why validate in `build()` and not in each method?"

`build()` is the one place that sees every field together. A required-together rule can't be checked in separate method calls: when `method("POST")` runs, the body may simply not have been set *yet*.

And because the product is frozen, the payoff is strong: an `HttpRequest` either doesn't exist or is valid. It can never be half-built.

### "`builder.method(...).header(...)`: isn't that chain a Law of Demeter violation?"

No. Every call returns the same builder, so no internal structure leaks. Demeter is about reaching through *different* types.

### "A `ParkingTicket` with a plate, an entry time and a spot. Builder?"

No. Three fields, all required, nothing optional and nothing to gather over time. A constructor or a dataclass does the job.

The page's rule of thumb: if the interviewer didn't describe a complex object with lots of optional details, Builder probably isn't needed. Most interview objects have 2–4 required fields.

### "Factory and Builder both create objects. What's the difference?"

- **Factory** returns different classes depending on a key, and gives that decision one home.
- **Builder** produces one class, configured incrementally.

### Gotchas

- **Forgetting `return self`.** The chain then breaks with `'NoneType' object has no attribute ...`.
- **Required values as optional steps.** Put them in the builder's constructor, as `url` is here, so they can't be forgotten.
- **Sharing mutable state with the product.** `build()` copies the headers (`dict(self._headers)`), so reusing the builder later can't change a request that was already built.
- **A builder for a product that has setters.** If the object can be changed after `build()`, the validation guaranteed nothing.
- **Reaching for it in Python out of Java habit.** Keyword arguments first.

### Connections

- Fluent chaining is the same idea the Law of Demeter note calls fine ([03_general_principles.md](../../01_lld_in_a_hurry/03_general_principles.md)).
- The product is a value object: a frozen dataclass ([07_python_oop_toolkit.md](../../01_lld_in_a_hurry/07_python_oop_toolkit.md)).

---

## Review questions

Answer first, then open the answer to check.

1. Java needs a builder for a ten-parameter constructor. Why doesn't Python, and what three cases still justify one?

   <details><summary>Answer</summary>

   - Keyword arguments and defaults already make the call readable: `RetryPolicy(attempts=5)`.
   - Still justified when choices are gathered across statements or branches, when a rule spans several fields, or when the finished object must be immutable.

   </details>

2. Why does "a POST needs a body" belong in `build()` and not in `method()` or `body()`?

   <details><summary>Answer</summary>

   - Neither method sees both fields, and the other one may not have been called yet.
   - `build()` sees everything at once, so it is the single place for validation. With a frozen product, an invalid request can never exist.

   </details>

3. You build one request, add a header to the same builder, and build a second. The first request's headers changed too. What went wrong?

   <details><summary>Answer</summary>

   - `build()` passed the builder's own dict to the product, so both share it.
   - Copy mutable state in `build()`: `dict(self._headers)`.

   </details>

4. The interviewer asks for a `ParkingTicket` with a plate, an entry time and a spot, and you reach for a builder. What should stop you?

   <details><summary>Answer</summary>

   - Three required fields, nothing optional, nothing gathered over time. A constructor or dataclass is enough.
   - A builder here is over-engineering, which interviewers notice.

   </details>

5. Factory or Builder: creating the right `Notification` subclass from `"sms"`; assembling a search query from filters chosen at runtime.

   <details><summary>Answer</summary>

   - `"sms"` → subclass: Factory. It decides which class.
   - Search query: Builder. One class, configured step by step.

   </details>
