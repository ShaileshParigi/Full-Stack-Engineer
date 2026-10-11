# Day 25 — Optional & Null Handling + Week 5 Review

_Why this matters:_ `NullPointerException` is the most common exception in Java
programs, and the person who invented `null` called it his "billion-dollar mistake".
The problem isn't the exception — it's that a `String` in a method signature might
secretly be "a string, or nothing", and the compiler says nothing about it. `Optional`
is Java's way of putting "might be nothing" **into the type**, for return values.
It's also one of the most *misused* classes in the JDK: code reviewers reject
`.get()`, `Optional` fields and `Optional` parameters on sight, and interviewers ask
why. Today you learn both the tool and its limits, then close Week 5.

> **A heads-up about syntax.** `Optional` is built around passing small pieces of
> code into methods: `name -> name.length() > 3` and `String::toUpperCase`. Those are
> **lambdas** and **method references**, and Week 6 (Days 26–30) derives them from
> scratch. For today, read `x -> something-with-x` as "a tiny unnamed function: take
> `x`, produce `something`", and `String::toUpperCase` as shorthand for
> `s -> s.toUpperCase()`. That's enough to follow everything below.

---

## The one-paragraph version

`null` means "no object here" — but nothing in a method's signature tells you
whether `null` might come back, so you either check *everywhere* (noisy) or
*nowhere* (crashes with `NullPointerException`, often far from the cause).
`Optional<T>` is a small box that holds either one `T` or nothing, used as a
**return type** to say out loud "this might not have an answer". Instead of
unpacking it with an `if`, you describe what to do with the value *if it's there*
using `map`, `filter` and `flatMap`, and finish with a way out: `orElse` (a default),
`orElseGet` (a lazily computed default), or `orElseThrow` (this is an error). Used
this way it removes chains of null checks. Used wrongly — `.get()` without checking,
as a field, as a parameter, wrapped around a collection — it is just `null` with
more typing. Everywhere else, plain `null` with a documented contract, plus
`Objects.requireNonNull` to fail fast at the boundary, is the honest tool.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **`null`** | "No object here" | A reference that points at nothing; calling a method on it throws `NullPointerException` |
| **`NullPointerException` (NPE)** | You asked nothing to do something | Unchecked; thrown when dereferencing `null`, unboxing `null`, etc. |
| **`Optional<T>`** | A box that holds one `T` or is empty | Immutable container class, `java.util.Optional` (Java 8) |
| **present / empty** | Box has a value / box has none | `isPresent()` / `isEmpty()`; `Optional.empty()` |
| **`map`** | "If there's a value, transform it" | `Optional<T>.map(T → R)` returns `Optional<R>`; empty stays empty |
| **`flatMap`** | Like `map`, for a step that itself returns an `Optional` | Avoids `Optional<Optional<R>>` |
| **`filter`** | "Keep the value only if it passes this test" | Becomes empty when the test fails |
| **lambda** | A tiny unnamed function you pass to a method | `x -> x.length() > 3` (full treatment: Day 26) |
| **method reference** | Shorthand for a lambda that just calls one method | `String::toUpperCase` (Day 28) |
| **`Supplier<T>`** | A thing you can ask for a value, when you want one | A lambda with no input that returns a `T`: `() -> compute()` (Day 27) |
| **eager / lazy** | Computed immediately / computed only when needed | `orElse(x)` evaluates `x` always; `orElseGet(supplier)` only if empty |
| **fail fast** | Reject bad input at the door, not deep inside | `Objects.requireNonNull(x, "message")` |
| **contract** | The documented promise of a method | "never returns null", "param may be null" — Day 22 |
| **boundary** | Where data enters your code from outside | Public method params, constructor args, request bodies, DB results |

---

## Before you read: four questions

1. A method `Customer findCustomer(long id)` returns `null` when nobody has that id.
   What's wrong with this *design*, beyond the crash risk?
2. `optional.get()` on an empty `Optional` — what happens? Is that better than a
   `null`?
3. `opt.orElse(buildExpensiveDefault())` — when does `buildExpensiveDefault()` run?
4. Your class has a field `Optional<String> nickname`. A reviewer says "no". Why?

<details><summary>Answers (try first)</summary>

1. The signature doesn't tell callers "might be absent", so each caller must remember
   to check. (Part 1.)
2. `NoSuchElementException`. Not really better: you've swapped one unchecked
   exception for another. (Part 4.)
3. **Always** — even if the `Optional` has a value. (Part 3.)
4. `Optional` is meant for return values; as a field it's extra memory, not
   serializable, and can itself be `null`. (Part 4.)

</details>

---

## Part 1 — `null` is a design problem

**In plain words:** a method signature is a promise. `String name()` *looks* like it
promises a string. If it sometimes returns `null`, the promise is secretly "a string,
or nothing" — and the only way to know is to read the method's code or documentation,
every time. Every caller has to guess whether to check. Guess "no" wrongly and you
get a crash at 3am, on a line that has nothing wrong with it.

Here's the shape of the problem — an `Order` has a `Customer`, who has an `Address`,
which has a `city`, and any of them might be missing:

```java
import java.util.Optional;

public class NullChain {
    static class Address {
        private final String city;
        Address(String city) { this.city = city; }
        String getCity() { return city; }
    }

    static class Customer {
        private final Address address;
        Customer(Address address) { this.address = address; }
        Address getAddress() { return address; }
    }

    static class Order {
        private final Customer customer;
        Order(Customer customer) { this.customer = customer; }
        Customer getCustomer() { return customer; }
    }

    static String cityNaive(Order order) {
        return order.getCustomer().getAddress().getCity();
    }

    static String cityWithNullChecks(Order order) {
        if (order != null) {
            Customer customer = order.getCustomer();
            if (customer != null) {
                Address address = customer.getAddress();
                if (address != null) {
                    String city = address.getCity();
                    if (city != null) {
                        return city;
                    }
                }
            }
        }
        return "unknown";
    }

    static String cityWithOptional(Order order) {
        return Optional.ofNullable(order)
                .map(Order::getCustomer)
                .map(Customer::getAddress)
                .map(Address::getCity)
                .orElse("unknown");
    }

    public static void main(String[] args) {
        Order complete = new Order(new Customer(new Address("Pune")));
        Order noAddress = new Order(new Customer(null));
        Order noCustomer = new Order(null);

        System.out.println("complete,   nested ifs: " + cityWithNullChecks(complete));
        System.out.println("noAddress,  nested ifs: " + cityWithNullChecks(noAddress));
        System.out.println("noCustomer, nested ifs: " + cityWithNullChecks(noCustomer));
        System.out.println();
        System.out.println("complete,   Optional:   " + cityWithOptional(complete));
        System.out.println("noAddress,  Optional:   " + cityWithOptional(noAddress));
        System.out.println("noCustomer, Optional:   " + cityWithOptional(noCustomer));
        System.out.println("null order, Optional:   " + cityWithOptional(null));
        System.out.println();

        try {
            cityNaive(noAddress);
        } catch (NullPointerException e) {
            System.out.println("naive chain on noAddress -> NullPointerException:");
            System.out.println("  " + e.getMessage());
        }
    }
}
```

```
complete,   nested ifs: Pune
noAddress,  nested ifs: unknown
noCustomer, nested ifs: unknown

complete,   Optional:   Pune
noAddress,  Optional:   unknown
noCustomer, Optional:   unknown
null order, Optional:   unknown

naive chain on noAddress -> NullPointerException:
  Cannot invoke "NullChain$Address.getCity()" because the return value of "NullChain$Customer.getAddress()" is null
```

Three versions of one task:

1. **`cityNaive`** — the one-liner everyone writes first. It works until any link is
   `null`, then crashes. Since Java 15, the exception message tells you *exactly
   which* link was null (the last line) — read it, it's genuinely useful.
2. **`cityWithNullChecks`** — correct, and the code walks off the right edge of the
   screen. This is the "pyramid of doom". The real work (reading `getCity()`) is
   buried under the checking.
3. **`cityWithOptional`** — same behaviour, flat. This is today's practice exercise,
   done. How it works is Part 3.

The design point: the "null chain" is the symptom. The disease is that nothing in
`Order.getCustomer()`'s signature said "might be absent".

> **Say this in an interview:** "`null` is a design problem because a method's type
> doesn't say whether it can be absent, so every caller must guess whether to check.
> `Optional` moves 'might be absent' into the return type, where the compiler and the
> reader can see it."

---

## Part 2 — What `Optional` is

**In plain words:** a small, sealed lunchbox. Open it and it either holds one item or
it's empty. A method returning `Optional<Customer>` is a method saying: *"I'll hand
you a box. It might be empty. Deal with that."* You can't use the customer without
going through the box, so you can't forget. That's the entire idea.

```
Optional<String> a = Optional.of("alice");          // a box with "alice" in it
Optional<String> b = Optional.empty();              // an empty box
Optional<String> c = Optional.ofNullable(maybeNull);// box holding it, or empty if it was null
```

| Factory | Use it when |
|---------|-------------|
| `Optional.of(x)` | `x` is definitely not null — **throws NPE if it is** |
| `Optional.empty()` | You have nothing to return |
| `Optional.ofNullable(x)` | `x` came from somewhere that might give `null` (a map lookup, an old API) |

The rule of thumb for **where it belongs**: **as the return type of a method that
might legitimately have no result** — "find a user by id", "the first element
matching…", "look up this setting". The JDK's own design intent, stated by its
authors, was precisely that: *a limited mechanism for return types where there needs
to be a clear way to say "no result"*. Everything outside that (fields, parameters,
collection elements) is a misuse — Part 4.

The JDK itself returns `Optional` from `stream.findFirst()`, `stream.max()`
and Spring Data's `findById` — you'll meet it constantly.

---

## Part 3 — Working with a value inside the box

**In plain words:** you don't open the box and look; you tell it what to do. "If
there's a value, turn it into upper case." "If there's a value, keep it only if it's
long enough." "Whatever happens, give me a string at the end — this default if empty."
If the box is empty, each "if there's a value" step is simply skipped, and the empty
box flows through to the end. That's why the Part 1 chain doesn't crash: the first
missing link makes the whole thing empty, and the later `map` steps never run.

A tour of the methods, with real output:

```java
import java.util.HashMap;
import java.util.Map;
import java.util.NoSuchElementException;
import java.util.Optional;

public class OptionalTour {
    static final Map<Integer, String> USERS = new HashMap<>();

    static Optional<String> findUser(int id) {
        return Optional.ofNullable(USERS.get(id));
    }

    static Optional<String> emailFor(String user) {
        return user.equals("alice") ? Optional.of("alice@example.com") : Optional.empty();
    }

    public static void main(String[] args) {
        USERS.put(1, "alice");
        USERS.put(2, "bob");

        Optional<String> present = findUser(1);
        Optional<String> absent = findUser(99);
        System.out.println("present:           " + present);
        System.out.println("absent:            " + absent);
        System.out.println("present.isPresent: " + present.isPresent() + ", absent.isEmpty: " + absent.isEmpty());

        System.out.println("map:               " + present.map(String::toUpperCase));
        System.out.println("map on empty:      " + absent.map(String::toUpperCase));

        System.out.println("map vs flatMap:");
        Optional<Optional<String>> nested = present.map(OptionalTour::emailFor);
        Optional<String> flat = present.flatMap(OptionalTour::emailFor);
        System.out.println("  map(emailFor):     " + nested);
        System.out.println("  flatMap(emailFor): " + flat);
        System.out.println("  bob flatMap:       " + findUser(2).flatMap(OptionalTour::emailFor));

        System.out.println("filter keeps:      " + present.filter(n -> n.length() > 3));
        System.out.println("filter drops:      " + present.filter(n -> n.length() > 10));

        System.out.println("orElse:            " + absent.orElse("guest"));
        System.out.println("or(...):           " + absent.or(() -> findUser(2)));

        present.ifPresent(n -> System.out.println("ifPresent ran for " + n));
        absent.ifPresentOrElse(
                n -> System.out.println("not printed"),
                () -> System.out.println("ifPresentOrElse: nobody there"));

        try {
            absent.orElseThrow(() -> new IllegalArgumentException("no user with id 99"));
        } catch (IllegalArgumentException e) {
            System.out.println("orElseThrow:       " + e.getMessage());
        }

        try {
            absent.get();
        } catch (NoSuchElementException e) {
            System.out.println("get() on empty:    NoSuchElementException: " + e.getMessage());
        }

        try {
            Optional.of(null);
        } catch (NullPointerException e) {
            System.out.println("Optional.of(null): NullPointerException");
        }
        System.out.println("ofNullable(null):  " + Optional.ofNullable(null));
    }
}
```

```
present:           Optional[alice]
absent:            Optional.empty
present.isPresent: true, absent.isEmpty: true
map:               Optional[ALICE]
map on empty:      Optional.empty
map vs flatMap:
  map(emailFor):     Optional[Optional[alice@example.com]]
  flatMap(emailFor): Optional[alice@example.com]
  bob flatMap:       Optional.empty
filter keeps:      Optional[alice]
filter drops:      Optional.empty
orElse:            guest
or(...):           Optional[bob]
ifPresent ran for alice
ifPresentOrElse: nobody there
orElseThrow:       no user with id 99
get() on empty:    NoSuchElementException: No value present
Optional.of(null): NullPointerException
ofNullable(null):  Optional.empty
```

The vocabulary:

| Method | What it does | Use for |
|--------|--------------|---------|
| `map(f)` | If present, apply `f`, wrap the result; if empty, stay empty | Transforming the value |
| `flatMap(f)` | Like `map`, but `f` itself returns an `Optional`, and it won't nest | A step that might *also* have no answer |
| `filter(p)` | Keep the value only if `p` is true, otherwise become empty | Conditions |
| `orElse(x)` | The value, or `x` | A cheap constant default |
| `orElseGet(supplier)` | The value, or call the supplier | A default that costs something |
| `orElseThrow(supplier)` | The value, or throw what the supplier makes | "Absent is an error here" |
| `or(supplier)` | This `Optional`, or another `Optional` (Java 9) | Fallback sources |
| `ifPresent(consumer)` | Run code if there's a value | Side effects, like logging |
| `ifPresentOrElse(c, r)` | One or the other (Java 9) | Same, with an else |
| `isPresent()` / `isEmpty()` | Is there a value? | Rarely — see Part 4 |
| `get()` | The value, or throw if empty | Almost never — see Part 4 |

**`map` vs `flatMap`.** `map(emailFor)` returned `Optional<Optional<String>>` — a box
inside a box — because `emailFor` already returns an `Optional`, and `map` wraps
whatever it gets. `flatMap` is `map` followed by "unwrap one layer". The rule: if
the function you're applying returns an `Optional`, use `flatMap`; if it returns a
plain value, use `map`. (You'll meet the same `flatMap` idea with streams on Day 34
and `CompletableFuture` in Phase 5.)

### `orElse` vs `orElseGet` — the trap

`orElse(x)` takes a **value**. Java evaluates every argument *before* calling the
method, so `x` is computed whether or not the `Optional` needs it.
`orElseGet(supplier)` takes a **recipe** (a `Supplier`: "call this if you need a
value"), and runs it only when the box is empty. Prove it with a logging default —
the plan's third exercise:

```java
import java.util.Optional;

public class OrElseLazy {
    static int calls = 0;

    static String expensiveDefault() {
        calls++;
        System.out.println("  ...computing the default (call #" + calls + ")");
        return "default";
    }

    public static void main(String[] args) {
        Optional<String> present = Optional.of("real value");
        Optional<String> empty = Optional.empty();

        System.out.println("orElse on a PRESENT optional:");
        String a = present.orElse(expensiveDefault());
        System.out.println("  result: " + a);

        System.out.println("orElseGet on a PRESENT optional:");
        String b = present.orElseGet(() -> expensiveDefault());
        System.out.println("  result: " + b);

        System.out.println("orElse on an EMPTY optional:");
        String c = empty.orElse(expensiveDefault());
        System.out.println("  result: " + c);

        System.out.println("orElseGet on an EMPTY optional:");
        String d = empty.orElseGet(() -> expensiveDefault());
        System.out.println("  result: " + d);

        System.out.println("total times the default was computed: " + calls + " (needed: 2)");
    }
}
```

```
orElse on a PRESENT optional:
  ...computing the default (call #1)
  result: real value
orElseGet on a PRESENT optional:
  result: real value
orElse on an EMPTY optional:
  ...computing the default (call #2)
  result: default
orElseGet on an EMPTY optional:
  ...computing the default (call #3)
  result: default
total times the default was computed: 3 (needed: 2)
```

Line 1 is the bug: the default was computed and thrown away although the value was
present. With a constant (`"guest"`) this is harmless. With a **database call, an
HTTP request, or an object creation** it's wasted work on every request — or worse a
**side effect**: `orElse(userRepository.save(newUser()))` creates a user *every
time*, whether needed or not. That's a real production bug.

| | Evaluates its argument | Use when |
|---|---|---|
| `orElse(x)` | Always | `x` is a constant or an already-computed variable |
| `orElseGet(() -> x())` | Only if empty | `x()` is anything that costs time, I/O, or has side effects |
| `orElseThrow(() -> new E(...))` | Only if empty | Absent is an error |

> **Say this in an interview:** "`map` transforms the value if present, `flatMap` does
> the same for a function that returns an `Optional` so you don't nest them, and
> `filter` makes it empty on failure. `orElse` evaluates its argument eagerly even
> when the value is present; for anything non-trivial I use `orElseGet`, which takes
> a `Supplier` and runs it only when empty."

---

## Part 4 — Anti-patterns

**In plain words:** every one of these takes the lunchbox and wastes the point of it.
The giveaway is the same each time: you've gone back to checking "is it there?" with
an `if`, which is exactly what `null` already let you do, with more ceremony.

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

public class AntiPatterns {
    static Optional<String> find(String key) {
        return key.equals("a") ? Optional.of("found-a") : Optional.empty();
    }

    // Anti-pattern 1: isPresent() then get() - a null check wearing a costume
    static String badLookup(String key) {
        Optional<String> r = find(key);
        if (r.isPresent()) {
            return r.get().toUpperCase();
        } else {
            return "NONE";
        }
    }

    static String goodLookup(String key) {
        return find(key).map(String::toUpperCase).orElse("NONE");
    }

    // Anti-pattern 2: Optional as a parameter - three states for the caller to get wrong
    static String greetBad(Optional<String> name) {
        return "Hello, " + name.orElse("stranger");
    }

    static String greet(String name) {
        return "Hello, " + (name != null ? name : "stranger");
    }

    static String greet() {
        return greet(null);
    }

    // Anti-pattern 3: Optional wrapping a collection - empty collection already means "none"
    static Optional<List<String>> tagsBad(boolean any) {
        return any ? Optional.of(List.of("x", "y")) : Optional.empty();
    }

    static List<String> tags(boolean any) {
        return any ? List.of("x", "y") : List.of();
    }

    // Anti-pattern 4: Optional where a plain null check is clearer
    static String worse(String s) {
        return Optional.ofNullable(s).map(String::trim).orElse("");
    }

    static String better(String s) {
        return s == null ? "" : s.trim();
    }

    // Anti-pattern 5: Optional as a field
    static class BadUser {
        Optional<String> nickname = Optional.empty();
    }

    static class GoodUser {
        private String nickname;
        Optional<String> getNickname() { return Optional.ofNullable(nickname); }
    }

    public static void main(String[] args) {
        System.out.println("badLookup(a)  = " + badLookup("a") + ", goodLookup(a)  = " + goodLookup("a"));
        System.out.println("badLookup(zz) = " + badLookup("zz") + ", goodLookup(zz) = " + goodLookup("zz"));
        System.out.println();

        System.out.println("greetBad(null) -> ");
        try {
            System.out.println(greetBad(null));
        } catch (NullPointerException e) {
            System.out.println("  NullPointerException: the parameter was STILL nullable");
        }
        System.out.println(greet("alice") + " / " + greet());
        System.out.println();

        System.out.println("tagsBad(false) = " + tagsBad(false) + "   tags(false) = " + tags(false));
        System.out.println();

        System.out.println("worse(\"  hi \")  = '" + worse("  hi ") + "'   better(\"  hi \") = '" + better("  hi ") + "'");
        System.out.println();

        BadUser bad = new BadUser();
        System.out.println("BadUser.nickname field: " + bad.nickname);
        System.out.println("GoodUser.getNickname(): " + new GoodUser().getNickname());
    }
}
```

```
badLookup(a)  = FOUND-A, goodLookup(a)  = FOUND-A
badLookup(zz) = NONE, goodLookup(zz) = NONE

greetBad(null) -> 
  NullPointerException: the parameter was STILL nullable
Hello, alice / Hello, stranger

tagsBad(false) = Optional.empty   tags(false) = []

worse("  hi ")  = 'hi'   better("  hi ") = 'hi'

BadUser.nickname field: Optional.empty
GoodUser.getNickname(): Optional.empty
```

| # | Anti-pattern | Why it's wrong | Do this instead |
|---|--------------|----------------|-----------------|
| 1 | **`isPresent()` then `get()`** | It's `if (x != null) x.foo()` with extra steps — and forgetting the check gives you `NoSuchElementException`, which is no better than an NPE | `map` / `orElse` / `orElseThrow` / `ifPresent` |
| 2 | **`.get()` with no check** | Throws `NoSuchElementException: No value present` — no context, no better than `null` | `orElseThrow(() -> new X("what was missing, and for which id"))` |
| 3 | **`Optional` as a method parameter** | Callers can pass `Optional.of(x)`, `Optional.empty()` — **or `null`** (as `greetBad(null)` just did). Three states, one of which crashes. Awkward to call, too | Overloads (`greet()` and `greet(String)`), or a plain nullable parameter with a documented contract |
| 4 | **`Optional` as a field** | Not `Serializable`; costs an extra object per field; a JPA/Jackson mapping headache; and the field itself can still be `null` | Store a nullable field; **return** `Optional.ofNullable(field)` from the getter (`GoodUser`) |
| 5 | **`Optional<List<T>>`, `Optional<Map<…>>`, `Optional<String>` for "empty text"** | An empty collection *already means "nothing"*. Now you have two empties (`Optional.empty()` and `Optional.of(emptyList)`) | Return an empty collection (`tags(false)` → `[]`) — never `null` either |
| 6 | **`Optional` for everything, including trivial null checks** (`worse`) | Allocates a box and builds a pipeline to do `s == null ? "" : s.trim()` | Plain `null` check when the logic is a single check on a local value |
| 7 | **`Optional.of(x)` with a possibly-null `x`** | Throws NPE immediately | `ofNullable` |
| 8 | **Returning `null` from a method that returns `Optional`** | Defeats the entire point; callers now must check the box *and* the contents | Return `Optional.empty()` |
| 9 | **`Optional<Integer>` in hot code** | Boxing + box = two allocations | `OptionalInt`, `OptionalLong`, `OptionalDouble` |

The test for the "worse" version (anti-pattern 6, today's second practice item): if
the `Optional` is created and consumed **in the same expression** on a single value,
with a single transformation and a default, ask whether `s == null ? "" : s.trim()`
reads better. It does. `Optional` earns its place when it **crosses a method
boundary** (as a return type) or when it **replaces a chain** of dependent checks
(Part 1's three-level walk). It's a bad trade for a one-step check.

> **Say this in an interview:** "I use `Optional` for return values that might have no
> result, and compose with `map`/`flatMap`/`orElseGet`. I avoid `get()`, I don't use
> it for fields, parameters or collections, and I don't wrap a single null check in
> one — a plain `null` check is clearer there."

---

## Part 5 — When plain `null` is fine, and validating at the boundary

**In plain words:** `Optional` isn't a ban on `null`. It's a signal for one
situation. Everywhere else, the question is: *where do bad values get stopped?* The
cleanest answer is **at the door** — check inputs the moment they enter your code and
reject them loudly — so the rest of your code can trust what it holds.

Plain `null` is reasonable when:

- It's a **private, local** detail — a variable you set to `null` and test a few lines
  later.
- A **field is genuinely optional** (`nickname`), with the getter returning
  `Optional` to callers (Anti-pattern 4's fix).
- You're implementing an interface or a framework that **requires** `null` (many
  frameworks, JSON mapping, JDBC `getString` for NULL columns).
- A method's **contract** says so, in one line of Javadoc: "`@param name may be
  null`" or "`@return the user, or null if absent`". An *undocumented* `null` is the
  bug, not `null` itself.

### Fail fast with `Objects.requireNonNull`

If a constructor stores a `null` it was handed, the program **appears to work** — and
crashes much later, in some other method, when that field is first used. The error
points at the *use*, not at the *mistake*. `Objects.requireNonNull(x, "message")`
throws immediately, at the real culprit, with a message you wrote:

```java
import java.util.Objects;

public class Boundaries {
    static class Notifier {
        private final String recipient;

        Notifier(String recipient) {
            this.recipient = Objects.requireNonNull(recipient, "recipient must not be null");
        }

        int recipientLength() {
            return recipient.length();
        }
    }

    static class LazyNotifier {
        private final String recipient;

        LazyNotifier(String recipient) {
            this.recipient = recipient;
        }

        int recipientLength() {
            return recipient.length();
        }
    }

    static String describe(String nickname) {
        String shown = Objects.requireNonNullElse(nickname, "(none)");
        return "nickname: " + shown;
    }

    public static void main(String[] args) {
        System.out.println("-- fail fast, at the boundary --");
        try {
            new Notifier(null);
        } catch (NullPointerException e) {
            System.out.println("constructor says: " + e.getMessage());
        }

        System.out.println("-- fail late, far from the cause --");
        LazyNotifier lazy = new LazyNotifier(null);
        System.out.println("constructed without complaint...");
        try {
            lazy.recipientLength();
        } catch (NullPointerException e) {
            System.out.println("...and much later: " + e.getMessage());
        }

        System.out.println("-- defaults --");
        System.out.println(describe("zed"));
        System.out.println(describe(null));
        System.out.println("Objects.equals(null, null) = " + Objects.equals(null, null));
        System.out.println("Objects.toString(null, \"-\") = " + Objects.toString(null, "-"));
    }
}
```

```
-- fail fast, at the boundary --
constructor says: recipient must not be null
-- fail late, far from the cause --
constructed without complaint...
...and much later: Cannot invoke "String.length()" because "this.recipient" is null
-- defaults --
nickname: zed
nickname: (none)
Objects.equals(null, null) = true
Objects.toString(null, "-") = -
```

In the lazy version the `null` went in silently; in a real service, "much later"
might be a different request, a different thread, an hour afterwards. Fail-fast turns
a mystery into a one-line fix.

The `Objects` toolbox:

| Method | Does |
|--------|------|
| `requireNonNull(x, "msg")` | Return `x`, or throw NPE with `msg` |
| `requireNonNullElse(x, dflt)` | `x` if non-null, else `dflt` |
| `requireNonNullElseGet(x, supplier)` | Same, with a lazy default |
| `Objects.equals(a, b)` | Null-safe `equals` (Day 22) |
| `Objects.toString(x, dflt)` | Null-safe `toString` |
| `Objects.isNull(x)` / `nonNull(x)` | Handy as method references in streams |

Where to validate: **public API boundaries** — constructors, public method params,
incoming request data. Inside private code that already trusts its callers, don't
re-check. Spring has the same idea built in: `@NonNull`, and `@Valid` on request
bodies (Phase 6).

Three more habits:

- **Return empty collections, never `null`.** `return List.of();` Callers can loop
  without a check.
- **Compare with the literal first**: `"admin".equals(role)` can't NPE;
  `role.equals("admin")` can.
- **Prefer primitives for things that can't be absent** (`int`, not `Integer`) — Day
  21's unboxing trap.

> **Say this in an interview:** "I validate at the boundary with
> `Objects.requireNonNull` so bad input fails immediately, with a clear message,
> rather than as a distant NPE. I return empty collections instead of `null`, I use
> `Optional` for absent return values, and I document nullability in the contract
> everywhere else."

---

## Hands-on

1. **Required — flatten a null chain** (the plan's practice). Take `cityWithNullChecks`
   and rewrite it yourself, from scratch and without looking, as an `Optional` chain.
   Then add a fourth level — `Address` gets a `getPostcode()` — in both styles and
   count how many lines each change costs.
2. **Required — the worse version.** Find a spot where `Optional` makes things worse
   (`worse` in `AntiPatterns` is one). Write down in one sentence the rule you'd give a
   colleague for when *not* to use it.
3. **Required — `orElse` vs `orElseGet`.** Run `OrElseLazy`. Then change it so
   `expensiveDefault()` increments a counter and creates a `new Object()`; predict the
   counter after each of the four calls *before* running.
4. Write a method `Optional<String> firstLongWord(List<String> words, int min)` using a
   loop (no streams yet) that returns the first word with at least `min` characters.
   Call it and handle the result **only** with `map`/`orElse`/`orElseThrow` — no
   `isPresent`.
5. Replace the `map.get(key)` + `if (v != null)` in a small program of yours with
   `Optional.ofNullable(map.get(key))`. Then read `Map.getOrDefault` and
   `Map.computeIfAbsent` (Day 32) and decide which is better.
6. Add `Objects.requireNonNull` to the constructor of one of your own classes and
   write a call that triggers it. Read the stack trace — does it point at the caller's
   mistake?
7. Search a project you work on for `.get()` on an `Optional` (look for `findById(…).get()`
   — Spring Data — a very common one). Rewrite one with `orElseThrow`.

---

## Gotchas

- **`Optional.of(null)` throws.** Use `ofNullable` for anything that can be null.
- **`Optional` is not a `null` replacement everywhere** — the `Optional` reference
  itself *can* still be `null` if someone returns it carelessly. Never return `null`
  from an `Optional`-returning method.
- **`Optional` isn't `Serializable`** — don't put one in a field that gets serialized
  (HTTP session, caches), or in a JPA entity field.
- **Don't compare `Optional`s with `==`.** It's an object; use `equals`, or better,
  unwrap with `orElse`.
- **JSON:** Jackson can serialize `Optional` only with a module registered; as a DTO
  field it's usually a smell. Use `null` or omit the property.
- **`Optional<Integer>` vs `OptionalInt`** — the primitive variants (`OptionalInt`,
  `OptionalDouble`) exist to avoid boxing; streams of primitives (`IntStream.max()`)
  return them.
- **`orElse(throw …)` doesn't exist.** Throwing is `orElseThrow`; a no-argument
  `orElseThrow()` (Java 10) throws `NoSuchElementException` — fine for tests, vague in
  production.
- **Streams of `Optional`s:** `stream.map(this::find)` gives `Stream<Optional<T>>`;
  `flatMap(opt -> opt.stream())` (Java 9) flattens them. Day 34.
- **Debugging output.** `Optional[alice]` and `Optional.empty` are the `toString`
  forms — recognise them in logs.

---

## Recap

- `null` is a design problem: the type doesn't say "might be absent", so callers must
  guess.
- `Optional<T>` = a box with one `T` or nothing, used for **return values** that might
  have no result.
- Work with the contents using `map` / `flatMap` / `filter`, finish with `orElse` (a
  constant), `orElseGet` (a lazy default), or `orElseThrow` (absent is an error).
- `orElse` always evaluates its argument; `orElseGet` only when empty — use it for
  anything that costs something or has side effects.
- Don't: `get()`, `isPresent()`+`get()`, `Optional` fields or parameters,
  `Optional<List<…>>`, `Optional.of(maybeNull)`, a single trivial null check wrapped
  in `Optional`.
- Plain `null` with a documented contract is fine locally; validate at the boundary
  with `Objects.requireNonNull`; return empty collections, not `null`.

---

## Week 5 review — how Java works underneath

Five days, one picture. If you can hold the picture, Phase 5 (concurrency) and Phase
7 (JPA) will make sense; if you can't, re-read the day marked.

| Day | The idea in one line | The trap to remember |
|-----|----------------------|----------------------|
| **21 JVM & memory** | Variables hold **references**; objects live on the **heap**, locals and call frames on each thread's **stack**; the GC reclaims what's unreachable | Java is pass-by-value of the *reference*; `Integer` caches -128..127; a static cache is a leak |
| **22 equals/hashCode/String** | `equals` defines "same meaning"; `hashCode` must agree, because hash collections pick a bucket *first* | `equals` without `hashCode` → duplicates & lost keys, silently; mutating a key loses it; `+=` in a loop is quadratic |
| **23 Exceptions** | Throw to abort, catch to recover or translate; **try-with-resources** for anything closable; always pass the **cause** | `finally` + `return` overrides; empty `catch`; losing the cause |
| **24 Generics** | Compile-time type checking; **erased** at runtime; invariant; wildcards **PECS** | Raw types → heap pollution; `new T[]` illegal; `List<String>` isn't a `List<Object>` |
| **25 Optional/null** | `Optional` for "might have no result" return values; validate at the boundary | `get()`, `orElse` eager, no `Optional` fields/params |

The threads that connect them (these are what make it one subject):

- **Reference vs object** (21) → `==` vs `equals` (22) → a `null` reference (25) →
  a *typed* reference that erasure has made untyped at runtime (24).
- **The heap is shared** (21) → why a mutable key (22) or a mutable shared object
  breaks things → why **immutability** is the cure (Day 38) → why Phase 5 starts with
  it.
- **Fail at the right place.** A `ClassCastException` three methods away (24), a lost
  exception (23), a late NPE (25), a missing `hashCode` giving wrong answers with no
  error (22) — every one is a failure that shows up far from its cause. The week's
  skill is **making failures loud and local**: compile-time generics, fail-fast
  `requireNonNull`, causes attached, `final` fields.

### Check yourself — ten questions

Answer out loud, no scrolling. The ones you can't answer tell you which day to
re-read.

1. Draw what happens in memory when you pass a `List` to a method that reassigns its
   parameter. (21)
2. Why is `Integer a = 128, b = 128; a == b` false? (21)
3. What are GC roots, and how does a `static` field cause a leak? (21)
4. Two equal objects in a `HashSet` and it holds both. Diagnose. (22)
5. Why must you not mutate a `HashMap` key? (22)
6. What does try-with-resources do that `finally` doesn't? (23)
7. Read a three-level `Caused by:` chain and find the root. (23)
8. Why can't you write `new T[10]`, and what's the underlying reason? (24)
9. Which of `? extends` / `? super` lets you add, and why? (24)
10. `orElse` vs `orElseGet` — a concrete case where it matters. (25)

> **A quiz is available.** If you'd like a written Week 5 quiz (like the Week 1/2 ones)
> to take away, say "quiz" and I'll create `day-25-week-5-quiz.md`.

---

## Say it out loud

1. "Why is `null` considered a design problem?"
2. "What is `Optional` for, and where does it *not* belong?"
3. "What's the difference between `map` and `flatMap` on an `Optional`?"
4. "Why is `isPresent()` followed by `get()` an anti-pattern?"
5. "`orElse` versus `orElseGet` — when does it matter?"
6. "What does `Objects.requireNonNull` buy you?"

---

## Q&A

_Follow-up questions from this lesson are appended here._
