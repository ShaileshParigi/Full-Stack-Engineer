# Day 23 — Exceptions & Resource Management

_Why this matters:_ At 3am you don't read your code — you read a stack trace. If
the exception was wrapped badly, the real cause is missing and the trace says
"could not place order" with no clue why. If a resource wasn't closed, the service
slowly runs out of database connections and falls over hours later with an error
that mentions neither. If a `catch` block is empty, the bug is invisible *and*
the data is wrong. Spring's `@Transactional` rollback rules, REST error handling
(Phase 6) and every library you'll use are built on how Java exceptions work.

---

## The one-paragraph version

When something goes wrong, a method can't just return an error code that every
caller must remember to check — so Java lets it **throw an exception**: a special
object that immediately stops the method and travels *up* the chain of callers
until someone **catches** it. If nobody does, the thread dies and prints a **stack
trace**. Exceptions come in two families: **checked** (the compiler forces you to
handle or declare them — `IOException`) and **unchecked** (it doesn't —
`NullPointerException`, `IllegalArgumentException`). Code that *opens* something —
a file, a network connection, a database connection — must *close* it even when
something fails halfway, or the thing leaks; `finally` and the cleaner
**try-with-resources** exist for exactly that. When you catch an exception and
throw a different one, always attach the original as the **cause**, so the whole
story survives in the stack trace. And never catch an exception just to ignore it.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **exception** | An object describing "something went wrong", thrown to abort the current work | An instance of `Throwable` |
| **throw** | Raise the alarm | `throw new IllegalStateException("…")` |
| **catch** | Handle the alarm at this level | `catch (SomeException e) { … }` |
| **stack trace** | The list of method calls that were active when the exception was created | `e.printStackTrace()`; first line is the message, then `at Class.method(File:line)` |
| **propagate** | Let an exception keep travelling up to the caller | The default when you don't catch it |
| **`Error`** | A problem so serious your code shouldn't try to handle it | `OutOfMemoryError`, `StackOverflowError` (Day 21) |
| **`Exception`** | A problem your code might reasonably handle | Parent of both checked and unchecked exceptions |
| **checked exception** | The compiler makes you deal with it | `Exception` subclasses that aren't `RuntimeException`: must be caught or declared with `throws` |
| **unchecked exception** | The compiler doesn't care | `RuntimeException` and its subclasses |
| **`throws`** | A method's declaration "I might throw this checked exception" | Part of the method signature |
| **`finally`** | Code that runs whatever happens in the `try` | Runs after `try`/`catch`, on success, failure, or `return` |
| **resource** | Something that must be released after use | A file, socket, DB connection, lock |
| **`AutoCloseable`** | "I know how to release myself" | Interface with one method, `void close() throws Exception` |
| **try-with-resources** | `try` that closes resources for you | `try (Res r = open()) { … }` |
| **suppressed exception** | A second exception that happened while handling the first | Attached to the first via `addSuppressed`; read with `getSuppressed()` |
| **cause** | The earlier exception that led to this one | `new X("msg", cause)`; read with `getCause()` |
| **wrapping** | Catching one exception and throwing another that carries the first as its cause | Used to cross a layer boundary |
| **swallowing** | Catching an exception and doing nothing | An empty or log-and-continue `catch` — the worst anti-pattern |

---

## Before you read: four questions

1. A method has `try { return 1; } finally { return 2; }`. What does it return?
2. You open a database connection, run a query that throws, and the line that closes
   the connection comes after the query. Is the connection closed?
3. What does the line `Caused by:` in a stack trace tell you, and what happens to it
   if you write `throw new RuntimeException("failed")` inside a catch block?
4. `catch (Exception e) { }` — what's wrong with it?

<details><summary>Answers (try first)</summary>

1. `2`. Part 3.
2. No. Part 4.
3. It's the earlier exception that triggered this one; without passing it to the new
   exception, it's lost. Part 6.
4. It hides every failure, including bugs you didn't know could happen. Part 7.

</details>

---

## Part 1 — The hierarchy

**In plain words:** all the things that can be "thrown" form a family tree. At the
root is `Throwable`. It splits into **`Error`** (the JVM is in trouble — don't
handle it) and **`Exception`** (your program had a problem — maybe handle it).
`Exception` splits again: **`RuntimeException`** and its children are "unchecked"
(programming mistakes: null, bad argument, bad index); everything else under
`Exception` is "checked" (outside-world problems: file missing, network down).

```
                          Throwable
                  ┌──────────┴───────────┐
                Error                 Exception
        (JVM trouble: don't catch)   ┌─────┴──────────────────┐
     OutOfMemoryError            RuntimeException      IOException, SQLException,
     StackOverflowError          (UNCHECKED)           TimeoutException ...
                                 NullPointerException  (CHECKED)
                                 IllegalArgumentException
                                 IllegalStateException
                                 IndexOutOfBoundsException
```

Print the real parent chains:

```java
import java.io.IOException;

public class Hierarchy {
    static void chain(Class<?> c) {
        StringBuilder sb = new StringBuilder();
        for (Class<?> k = c; k != null; k = k.getSuperclass()) {
            if (sb.length() > 0) sb.append(" -> ");
            sb.append(k.getSimpleName());
        }
        System.out.println(sb);
    }

    public static void main(String[] args) {
        chain(IOException.class);
        chain(IllegalArgumentException.class);
        chain(NullPointerException.class);
        chain(OutOfMemoryError.class);
        System.out.println("IOException is a RuntimeException? "
                + RuntimeException.class.isAssignableFrom(IOException.class));
    }
}
```

```
IOException -> Exception -> Throwable -> Object
IllegalArgumentException -> RuntimeException -> Exception -> Throwable -> Object
NullPointerException -> RuntimeException -> Exception -> Throwable -> Object
OutOfMemoryError -> VirtualMachineError -> Error -> Throwable -> Object
IOException is a RuntimeException? false
```

Where the line falls:

| | Checked | Unchecked |
|---|---------|-----------|
| **Base** | `Exception` (not `RuntimeException`) | `RuntimeException` |
| **Compiler** | Must catch or declare `throws` | Doesn't check |
| **Meaning** | "Something outside your control may fail; you can plausibly recover" | "You (the programmer) broke a rule" |
| **Examples** | `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException`, `IllegalStateException` |

> **Say this in an interview:** "`Throwable` splits into `Error`, which signals
> JVM-level failure, and `Exception`. Checked exceptions are `Exception` subclasses
> other than `RuntimeException`; the compiler forces callers to handle or declare
> them. Unchecked exceptions extend `RuntimeException` and are for programming
> errors."

---

## Part 2 — Checked vs unchecked: a defensible position

**In plain words:** checked exceptions were meant to make failure impossible to
forget: if `readFile` can fail, the compiler won't let you ignore it. In practice
it became noisy: layers of methods that can't do anything about an `SQLException`
still have to declare it, or catch it and write boilerplate. And lambdas (next week)
can't throw checked exceptions at all, which makes them painful with streams.

The debate:

| For checked | Against checked |
|-------------|-----------------|
| Failure is visible in the signature | Pollutes every method in the chain with `throws` |
| Forces you to think about recoverable errors | Callers usually just wrap and rethrow |
| | Breaks lambdas and functional interfaces (Phase 3, Week 2) |
| | Leaks implementation details (`SQLException` in a "domain" interface) |

**A position you can defend** (and which Spring itself follows — it wraps
`SQLException` in an unchecked `DataAccessException`):

- Use **unchecked** exceptions for almost everything in application code.
- Use **checked** only when the *caller can genuinely do something different*
  (retry, fall back) and you want the compiler to insist they consider it.
- Throw the **standard** exceptions where they fit: `IllegalArgumentException` (bad
  input value), `IllegalStateException` (the object isn't in a state where this call
  makes sense), `UnsupportedOperationException`, `NullPointerException` via
  `Objects.requireNonNull(x, "x must not be null")`.
- At a layer boundary, **translate**: catch the low-level exception and throw one
  that belongs to *your* layer (Part 6).

> **Say this in an interview:** "Checked exceptions force handling at compile time,
> but they leak implementation details and don't compose with lambdas, so I default
> to unchecked exceptions and use checked ones only where the caller can meaningfully
> recover. Frameworks like Spring translate checked exceptions to unchecked ones."

---

## Part 3 — `try` / `catch` / `finally`, and exactly what `finally` guarantees

**In plain words:** `try` marks the risky bit. `catch` is "if *that* goes wrong,
do this". `finally` is "no matter what happened — success, failure, early
`return` — do this last". It's meant for tidying up.

```
try {
    risky();
} catch (FileNotFoundException e) {   // most specific first
    ...
} catch (IOException e) {             // then broader ones
    ...
} finally {
    cleanup();                         // runs in every case
}
```

- Catch clauses are checked **top to bottom**; the first that matches wins, so
  subclasses must come *before* parents (the compiler rejects the wrong order).
- `catch (A | B e)` handles two unrelated types identically.
- An exception not matched by any `catch` keeps travelling up, **after `finally` runs**.

**What `finally` guarantees:** it runs after the `try` block completes normally,
after a `catch` handles something, or when an exception is passing through. It runs
**even if the `try` has a `return`**. It does *not* run if the JVM itself stops —
`System.exit()`, the process being killed, a power cut.

Now the surprises:

```java
public class FinallyRules {
    @SuppressWarnings("finally")
    static int overrides() {
        try {
            return 1;
        } finally {
            return 2;
        }
    }

    @SuppressWarnings("finally")
    static int swallows() {
        try {
            throw new IllegalStateException("important failure");
        } finally {
            return 3;
        }
    }

    static int order() {
        int x = 10;
        try {
            return x;
        } finally {
            x = 99;
            System.out.println("finally ran, x is now " + x);
        }
    }

    public static void main(String[] args) {
        System.out.println("overrides(): " + overrides());
        System.out.println("swallows():  " + swallows() + "   (the exception vanished)");
        System.out.println("order():     " + order() + "   (return value was captured before finally)");
    }
}
```

```
overrides(): 2
swallows():  3   (the exception vanished)
finally ran, x is now 99
order():     10   (return value was captured before finally)
```

1. A `return` in `finally` **replaces** the `return` in `try` — the answer to
   question 1.
2. A `return` in `finally` also **erases a thrown exception**. The failure just
   disappears. Never `return`, `break` or `throw` from `finally`.
3. `return x` evaluates `x` *first* (10), then `finally` runs and changes `x`
   to 99 — but the value being returned was already captured. (If `x` were a
   mutable object, `finally` could change what the caller sees *inside* the
   object; reassigning the variable can't.)

> **Say this in an interview:** "`finally` runs whether the try completes normally,
> returns, or throws, and it's for releasing resources. It doesn't run if the JVM
> exits. A `return` or `throw` inside `finally` overrides the original outcome, so
> I never do that."

---

## Part 4 — Resources and try-with-resources

**In plain words:** some things have to be given back when you're done: a file
handle, a socket, a database connection. The pool has a limited number; if each
failed request forgets to give one back, the pool slowly empties and the whole
service stops. "Close it even when something went wrong" is the job. The old way
is `finally`; the modern way is **try-with-resources**, where you declare the thing
in parentheses and Java closes it for you.

Any class that implements `AutoCloseable` (one method, `close()`) can be used this
way: `InputStream`, `Connection`, `Scanner`, JDBC `Statement`/`ResultSet`, and your
own. Here's the plan's practice exercise — a resource that logs when it opens and
closes, used three ways while the work fails:

```java
public class Resources {
    static class Conn implements AutoCloseable {
        final String name;
        Conn(String name) { this.name = name; System.out.println("  open  " + name); }
        void query() { throw new IllegalStateException("query failed on " + name); }
        @Override public void close() { System.out.println("  close " + name); }
    }

    static void leaky() {
        Conn c = new Conn("A");
        c.query();
        c.close();
    }

    static void manualFinally() {
        Conn c = new Conn("B");
        try {
            c.query();
        } finally {
            c.close();
        }
    }

    static void twr() {
        try (Conn c1 = new Conn("C1"); Conn c2 = new Conn("C2")) {
            c2.query();
        }
    }

    static void run(String label, Runnable r) {
        System.out.println(label);
        try {
            r.run();
        } catch (IllegalStateException e) {
            System.out.println("  caught: " + e.getMessage());
        }
    }

    public static void main(String[] args) {
        run("1. no finally:", Resources::leaky);
        run("2. try/finally:", Resources::manualFinally);
        run("3. try-with-resources (closes in reverse order):", Resources::twr);
    }
}
```

```
1. no finally:
  open  A
  caught: query failed on A
2. try/finally:
  open  B
  close B
  caught: query failed on B
3. try-with-resources (closes in reverse order):
  open  C1
  open  C2
  close C2
  close C1
  caught: query failed on C2
```

- **Case 1:** `A` is opened, the query throws, the `close()` line never runs. **That's
  a leak.** Do this a few hundred times and the connection pool is empty.
- **Case 2:** `finally` fixes it, at the cost of boilerplate — and it gets ugly with
  two resources (nested try/finally, and the second `close()` may throw).
- **Case 3:** one line of syntax; resources are closed **automatically, in reverse
  order of opening** (C2 then C1, like unstacking plates), even when the body
  throws. It's the form to use *always*.

The `Runnable r` / `Resources::leaky` bits are a method reference and a functional
interface — Week 2 explains them; for now they just mean "run this method".

### Suppressed exceptions — the second failure

What if the body throws, *and* `close()` throws too? The old `finally` way loses the
first exception entirely. Try-with-resources keeps both:

```java
public class Suppressed {
    static class Res implements AutoCloseable {
        @Override public void close() {
            throw new IllegalStateException("close failed");
        }
    }

    @SuppressWarnings("finally")
    static void oldWay() {
        Res r = new Res();
        try {
            throw new RuntimeException("body failed");
        } finally {
            r.close();
        }
    }

    static void newWay() {
        try (Res r = new Res()) {
            throw new RuntimeException("body failed");
        }
    }

    public static void main(String[] args) {
        try {
            oldWay();
        } catch (Exception e) {
            System.out.println("old way caught: " + e.getMessage() + "   <- the real cause is gone");
        }

        try {
            newWay();
        } catch (Exception e) {
            System.out.println("try-with-resources caught: " + e.getMessage());
            for (Throwable s : e.getSuppressed()) {
                System.out.println("  suppressed: " + s.getMessage());
            }
        }
    }
}
```

```
old way caught: close failed   <- the real cause is gone
try-with-resources caught: body failed
  suppressed: close failed
```

The primary exception — the one that actually explains what went wrong — is
`body failed`. With plain `finally`, the later `close failed` replaced it, and you'd
debug the wrong problem. With try-with-resources the first exception is thrown and
the second is **suppressed** (attached to it, visible in the stack trace as
`Suppressed:`).

> **Say this in an interview:** "Try-with-resources closes any `AutoCloseable` in
> reverse order, even on exceptions, and keeps the original exception primary while
> attaching close failures as suppressed. It replaces try/finally for resource
> management and avoids losing the root cause."

---

## Part 5 — Custom exceptions: when they earn their place

**In plain words:** write your own exception type when callers need to **tell
failures apart and react differently** — or when the name says something the
standard ones can't. Don't write one just for the sake of it.

Good reasons:

- The caller will `catch` that type specifically (`OrderNotFoundException` → HTTP 404;
  `InsufficientStockException` → HTTP 409).
- You want to carry **data** with the failure (the order id, the missing field).
- You're crossing a layer boundary (Part 6).

Bad reasons: `MyException`, a custom type for every method, a class that adds
nothing over `IllegalArgumentException`.

```
public class OrderNotFoundException extends RuntimeException {
    private final long orderId;

    public OrderNotFoundException(long orderId) {
        super("No order with id " + orderId);
        this.orderId = orderId;
    }

    public long getOrderId() { return orderId; }
}
```

(Extending `RuntimeException` makes it unchecked, per Part 2. Always provide a
constructor that takes a `cause` — next part.) In Phase 6 you'll map this class to
a 404 in one place, a pattern called an exception handler.

---

## Part 6 — Wrapping, causes, and reading a stack trace

**In plain words:** imagine a form passing through three departments, each stamping
"rejected" and handing it on. If every stamp only says "rejected", by the end nobody
knows why. Each department should write *its own* summary **and staple the previous
note to it**. In Java that stapling is passing the original exception as the
**cause**: `new RepositoryException("could not load row 42", e)`.

Layers should speak their own language. A web controller shouldn't be handling
`SQLException`; a repository should translate it into something meaningful at its
level, *keeping the original attached*:

```java
public class Chain {
    static class RepositoryException extends RuntimeException {
        RepositoryException(String message, Throwable cause) { super(message, cause); }
    }

    static class OrderException extends RuntimeException {
        final long orderId;
        OrderException(String message, long orderId, Throwable cause) {
            super(message, cause);
            this.orderId = orderId;
        }
    }

    static String lowLevel() {
        throw new java.io.UncheckedIOException(new java.io.IOException("connection reset by peer"));
    }

    static String repository(long id) {
        try {
            return lowLevel();
        } catch (java.io.UncheckedIOException e) {
            throw new RepositoryException("could not load row " + id, e);
        }
    }

    static String service(long id) {
        try {
            return repository(id);
        } catch (RepositoryException e) {
            throw new OrderException("could not place order", id, e);
        }
    }

    public static void main(String[] args) {
        try {
            service(42);
        } catch (OrderException e) {
            e.printStackTrace(System.out);
            System.out.println("---- walking the chain ----");
            for (Throwable t = e; t != null; t = t.getCause()) {
                System.out.println(t.getClass().getSimpleName() + ": " + t.getMessage());
            }
        }
    }
}
```

```
Chain$OrderException: could not place order
	at Chain.service(Chain.java:30)
	at Chain.main(Chain.java:36)
Caused by: Chain$RepositoryException: could not load row 42
	at Chain.repository(Chain.java:22)
	at Chain.service(Chain.java:28)
	... 1 more
Caused by: java.io.UncheckedIOException: java.io.IOException: connection reset by peer
	at Chain.lowLevel(Chain.java:15)
	at Chain.repository(Chain.java:20)
	... 2 more
Caused by: java.io.IOException: connection reset by peer
	... 4 more
---- walking the chain ----
OrderException: could not place order
RepositoryException: could not load row 42
UncheckedIOException: java.io.IOException: connection reset by peer
IOException: connection reset by peer
```

**How to read it — bottom to top for the cause, top to bottom for the path:**

1. **The first line is the outermost exception** — what the caller got: "could not
   place order". Useful to a user, useless for diagnosis.
2. **Each `Caused by:` goes one level deeper** into the story. **The last
   `Caused by:` is the root cause**: "connection reset by peer" — the network. That's
   the line to start debugging from.
3. Under each exception, the `at Class.method(File.java:line)` lines are the call
   stack **where that exception was created**, most recent call first. The first
   `at` line is where it was thrown.
4. **`... N more`** means "the last N frames are identical to the enclosing
   exception's, so I left them out", not that information is missing.

If `repository` had written `throw new RepositoryException("could not load row " +
id)` **without `e`**, the output would end at `could not load row 42` and the words
"connection reset by peer" would never reach the log. That's the single most common
way to make an incident undiagnosable. **Always pass the cause.**

Related habits:

- **Log or throw, not both.** If you log an exception *and* rethrow it, the same
  stack trace appears once per layer and the log fills with duplicates. Log it
  where you finally handle it (usually one place near the top).
- **Don't lose the stack trace by throwing `new X(e.getMessage())`** — that copies
  only the text. Pass `e` itself.
- **Include context in the message:** the id, the value, the operation. "failed" is
  useless; "could not load row 42" is a search term.
- `Objects.requireNonNull(x, "x")` and early argument checks make failures happen
  close to the mistake instead of far from it.

> **Say this in an interview:** "When translating exceptions across layers I wrap
> the original as the cause so the root cause survives in the `Caused by:` chain,
> and I read a stack trace by finding the deepest cause first. I log an exception
> once, where it's handled, rather than at every layer."

---

## Part 7 — Anti-patterns

**In plain words:** most exception bugs aren't about the mechanics; they're about
hiding the failure. A good `catch` does one of three things: **recover** (try
something else), **translate** (throw a better exception, with the cause), or
**clean up and rethrow**. A `catch` that does none of these is a bug.

| Anti-pattern | Why it hurts | Instead |
|--------------|--------------|---------|
| **Empty `catch`** | The failure disappears; the program carries on with wrong data | Handle it, or let it propagate |
| **`catch (Exception e)`** (or `Throwable`) | Catches bugs you didn't expect — and `NullPointerException`s — and treats them like the one failure you planned for | Catch the specific type |
| **Log and continue** (`e.printStackTrace()` then move on) | Looks handled; isn't. Logs of a failure that the code then ignores | Recover for real, or rethrow |
| **Exceptions as control flow** (`try { parse } catch { try next }` in a hot loop) | Slow: building a stack trace is expensive; and unreadable | Check first (`if (s.isEmpty())`) |
| **`throw new X(e.getMessage())`** | Drops the stack trace and the cause | `throw new X("context", e)` |
| **`return` in `finally`** | Overrides the real result, erases exceptions (Part 3) | Don't |
| **Catching `Throwable`/`Error`** | You'll swallow `OutOfMemoryError` | Let `Error`s through |
| **Declaring `throws Exception`** on everything | Tells callers nothing | Declare the specific type, or go unchecked |

See an empty catch do damage:

```java
public class Swallow {
    static int parse(String s) {
        try {
            return Integer.parseInt(s);
        } catch (Exception e) {
        }
        return 0;
    }

    static int parseLogged(String s) {
        try {
            return Integer.parseInt(s);
        } catch (NumberFormatException e) {
            throw new IllegalArgumentException("quantity must be a whole number, got '" + s + "'", e);
        }
    }

    public static void main(String[] args) {
        System.out.println("parse(\"12\")  = " + parse("12"));
        System.out.println("parse(\"1 2\") = " + parse("1 2") + "   <- silently wrong: an order for 0 items");
        try {
            parseLogged("1 2");
        } catch (IllegalArgumentException e) {
            System.out.println("parseLogged: " + e.getMessage());
            System.out.println("  cause: " + e.getCause());
        }
    }
}
```

```
parse("12")  = 12
parse("1 2") = 0   <- silently wrong: an order for 0 items
parseLogged: quantity must be a whole number, got '1 2'
  cause: java.lang.NumberFormatException: For input string: "1 2"
```

`parse` made a bad input into a valid-looking `0`. Nothing was logged, nothing
failed, and an order for zero items went into the system. `parseLogged` names the
problem, says what the value was, and keeps the original as the cause.

> **Say this in an interview:** "A `catch` should recover, translate with the cause
> attached, or clean up and rethrow. Empty catches and `catch (Exception)` hide
> bugs. I also avoid using exceptions for normal control flow because they're slow
> and hard to follow."

---

## Hands-on

1. **Required — resource logging** (the plan's practice). Run `Resources`. Then add
   a fourth case: a try-with-resources where *both* the body and `close()` throw,
   and print `getSuppressed()`. Write one sentence explaining why case 1 would have
   been a production incident.
2. **Required — a three-level chain.** Run `Chain`. Then break it on purpose: change
   `throw new RepositoryException("could not load row " + id, e)` to omit `e`. Run it
   and look at what's gone from the trace. Restore it.
3. In `FinallyRules`, add `System.exit(0);` inside the `try` of a new method and put a
   `System.out.println` in its `finally`. Does the message print?
4. Write `static void process(String path) throws IOException` using
   `Files.readAllLines(Path.of(path))` and call it with a missing file, once catching
   `IOException` specifically and once with `catch (Exception e)`. Add a
   `NullPointerException`-causing line and see which catch hides it.
5. Write a custom `InsufficientStockException extends RuntimeException` with
   `sku`, `requested` and `available` fields and a helpful message. Throw it from a
   small `Inventory.remove(sku, qty)` method.
6. Look at a real stack trace from any Java project you've worked on. Find the
   outermost exception, the last `Caused by:`, and the first `at` line of your own
   code.

---

## Gotchas

- **Checked exceptions inside lambdas.** `list.forEach(x -> Files.readString(x))` won't
  compile — `forEach`'s lambda can't throw a checked exception. You'll wrap in a
  try/catch inside the lambda. (A good reason for the defaults in Part 2.)
- **A `catch` for an exception that can't be thrown** by the `try` is a *compile
  error* for checked ones (it's dead code), but silently allowed for unchecked ones.
- **Resource variables are implicitly `final`** in a try-with-resources header; you
  can't reassign them. You can also use an *existing* effectively-final variable:
  `try (conn) { … }`.
- **Order of closing is reverse of opening.** If one resource depends on another
  (a `Statement` on a `Connection`), declare them in dependency order.
- **Don't close a resource you didn't open.** If a method receives a `Connection` it
  was handed, closing it is the owner's job.
- **Stack traces cost.** Creating an exception walks the stack; that's why they're
  slow as control flow. Never throw in a tight loop as a normal path.
- **`NullPointerException`s** now come with a helpful message in Java 21 ("Cannot
  invoke `String.length()` because `s` is null") — read it.
- **`@Transactional` rolls back on unchecked exceptions by default, *not* on
  checked ones** (Phase 7). This is the practical reason the checked/unchecked
  choice matters in Spring.

---

## Recap

- `Throwable` → `Error` (don't handle) and `Exception`. `RuntimeException` and its
  children are **unchecked**; other `Exception`s are **checked**.
- Default to unchecked in application code; use checked only for recoverable cases
  the caller must consider.
- `finally` always runs (except on JVM exit) — and a `return`/`throw` in it overrides
  the real outcome. Don't.
- **Try-with-resources** closes `AutoCloseable`s in reverse order and keeps the
  original exception primary, with close failures **suppressed**.
- When you translate an exception, **pass the cause**. Read traces from the last
  `Caused by:` up.
- A `catch` recovers, translates, or cleans up and rethrows. Anything else is a bug.

Re-answer the four questions at the top.

---

## Say it out loud

1. "Describe the exception hierarchy and the difference between checked and
   unchecked exceptions."
2. "What does `finally` guarantee, and what's the danger of `return` inside it?"
3. "Why is try-with-resources better than try/finally?"
4. "What's a suppressed exception?"
5. "You're given a stack trace with three `Caused by:` sections. How do you read it?"
6. "What's wrong with `catch (Exception e) { }`, and what should you do instead?"

---

## Q&A

_Follow-up questions from this lesson are appended here._
