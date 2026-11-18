# Singleton

Source: Hello Interview LLD → Design Patterns, and my teach-back on 18 Nov 2026 (clean on the third pass).

## What this note covers

- What a singleton guarantees, and the problem it solves
- The three Python versions: a module, `__new__`, and `__new__` with a lock
- The `__init__` trap
- Why it's called an anti-pattern, and what to do instead

## Review points

- **What it is:** one instance of a class per process, plus one global way to reach it.
- **The problem:** some resources must exist once and be shared, like a cache, a config, or a connection pool. Two copies waste memory and drift apart.
- **In Python, use a module.** A module runs once and is cached, so a module-level object is already a singleton, and it's thread-safe for free.
- **The class version** keeps the instance in a class attribute and returns it from `__new__`. **Trap:** `__init__` still runs on every call.
- **Thread-safe version:** double-checked locking. Check, take the lock, check again. Interviewers rarely ask you to write it.
- **Interview default:** "Should this be a singleton?" Usually no. Create one at startup and inject it.

---

## The discussion

### "Why would I ever want only one object?"

Because some things go wrong as soon as there are two. Take an in-memory price cache. If two parts of the app each create their own:

- one part updates a price and the other keeps serving the old one;
- the same data is held twice in memory.

The rule "only one, shared by everyone" is what Singleton enforces. The same goes for a config object, a connection pool and a logger.

### "So how do I make one in Python?"

The easy way: you probably already have. Put the object in a module:

```python
# price_cache.py
class PriceCache:
    def __init__(self) -> None:
        self._prices: dict[str, float] = {}

    def set(self, item: str, price: float) -> None:
        self._prices[item] = price

    def get(self, item: str) -> float | None:
        return self._prices.get(item)

cache = PriceCache()            # created once, when the module is first imported
```

```python
# anywhere else
# from price_cache import cache
cache.set("book", 12.5)
cache.get("book")               # 12.5, for every importer
```

Python runs a module's code once and caches the module in `sys.modules`. Every later `import` gets that same module object, so every importer gets the same `cache`. No pattern code at all.

Hello Interview says the same: singletons aren't idiomatic in Python, and a module-level instance is the Python answer.

### "And the class version, if the interviewer wants to see one?"

The class remembers its instance and hands it back instead of making a new one:

```python
class DatabaseConnection:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

first = DatabaseConnection()
second = DatabaseConnection()
first is second                 # True
```

`__new__` is what actually creates the object; `__init__` only fills it in afterwards. Overriding `__new__` lets the class return the existing object.

### "I pass a path the second time. The first path wins, right?"

That's what I said at first, and it's wrong. `DatabaseConnection()` calls `__new__` *and then* `__init__`, every time. `__new__` returns the old object, and `__init__` runs on it again:

```python
class Config:
    _instance = None

    def __new__(cls, path: str):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, path: str):
        self.path = path        # runs on every Config(...) call

first = Config("a.yaml")
second = Config("b.yaml")
first is second                 # True: still one object
first.path                      # "b.yaml": silently overwritten
```

Still one object, but the second caller re-initialized it under everyone's feet. The fix: a singleton takes no constructor arguments and loads its own settings. (Or guard `__init__` with an `_initialized` flag, which is uglier.)

### "What about two threads calling it at the same time?"

Both can see `_instance is None` before either one sets it, and you get two instances. The fix is double-checked locking:

```python
import threading

class ConnectionPool:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:               # cheap check, no lock
            with cls._lock:
                if cls._instance is None:       # re-check: another thread may have won
                    cls._instance = super().__new__(cls)
        return cls._instance

pool = ConnectionPool()
```

- **First check:** once the instance exists, every call skips the lock. No wasted cycles on the common path.
- **Second check:** a thread that waited for the lock may find that the thread ahead of it already created the instance.

The module version needs none of this. Python holds a per-module lock while a module's code runs, so a second thread importing at the same moment waits, then gets the finished module.

Hello Interview's note: interviewers don't expect you to write the thread-safe version in an LLD round. Knowing why it exists is enough.

### Interviewer: "Should the `ParkingLot` be a singleton?"

Usually no, and they're often checking whether you know that. My answer: only one place creates the `ParkingLot`, so there's nothing to guard against. Create it once at startup and pass it to whoever needs it.

That gives you "only one" without the costs that earned Singleton its bad name:

- **Hidden dependencies.** Any code can grab it without the constructor saying so.
- **Tests leak into each other.** All tests in a run share one process, so the singleton's state outlives each test. One test fills the cache, and the next test sees it. They pass alone and fail together.
- **No fakes.** You can't hand a test its own instance.
- **Two jobs in one class:** its real work, plus policing how many of itself exist.

Say yes only when the interviewer explicitly wants one shared instance across the whole system.

### Gotchas

- **`__init__` runs every time.** The `__new__` singleton gets re-initialized by every call with arguments.
- **Testing pain.** State outlives the test. If you're stuck with a singleton, give tests a reset hook. Better: inject it.
- **Thread safety:** the plain `__new__` version races. The module version doesn't.
- **"It's a manager, so it's a singleton."** Being created once is not a reason. Enforcing "once" is what the pattern adds, and you rarely need it enforced.

### Connections

- Injection over globals: the same answer as in Factory ([02_factory.md](02_factory.md)). Create once, pass in.
- Thread-safe creation is a race condition and a lock (Track 1 concurrency, [02_correctness.py](../../03_concurrency/02_correctness.py)).

---

## Review questions

Answer first, then open the answer to check.

1. What problem does Singleton solve? Give an example where two instances cause a bug.

   <details><summary>Answer</summary>

   - Some resources must exist once and be shared.
   - Two in-memory caches: one part of the app updates a price, the other keeps serving the stale one, and the data sits in memory twice.

   </details>

2. What's the Pythonic way to get a singleton, and why is it thread-safe without a lock?

   <details><summary>Answer</summary>

   - A module-level instance. A module runs once and is cached in `sys.modules`, so every importer gets the same object.
   - Python holds a lock while a module's code runs, so a second importing thread waits and then gets the finished module.

   </details>

3. With the `__new__` singleton, `Config("a.yaml")` then `Config("b.yaml")`. How many objects, and what is `path`? How do you prevent it?

   <details><summary>Answer</summary>

   - One object, but `path` is `"b.yaml"`: `__init__` runs after `__new__` on every call.
   - Prevent it: no constructor arguments (the singleton loads its own settings), or an `_initialized` guard in `__init__`.

   </details>

4. In double-checked locking, why check before taking the lock, and why check again inside it?

   <details><summary>Answer</summary>

   - Before: once the instance exists, skip the lock entirely, so the common path is cheap.
   - Inside: a thread that waited for the lock may find another thread already created the instance while it waited.

   </details>

5. "Make the `ParkingLot` a singleton." What do you say?

   <details><summary>Answer</summary>

   - Usually no. Nothing else creates it, so there's nothing to guard. Create it once at startup and inject it.
   - A singleton hides dependencies and leaks state between tests. Use one only if the interviewer explicitly wants a single shared instance for the whole system.

   </details>
