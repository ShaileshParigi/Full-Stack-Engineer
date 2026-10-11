# Day 27 — The Functional Interface Toolkit

_Why this matters:_ Yesterday you learned that a lambda is "an implementation of a
functional interface". Today is the catalogue of the ones you'll actually meet. Every
signature in streams (`filter(Predicate)`, `map(Function)`, `forEach(Consumer)`,
`reduce(BinaryOperator)`), `Optional` (`orElseGet(Supplier)`), `Map`
(`computeIfAbsent(Function)`, `merge(BinaryOperator)`) and Spring uses one of about
six shapes. Learn the six and the whole API becomes readable: you glance at a method's
parameter type and know what kind of lambda it wants. And you'll understand — finally —
*why* `orElseGet` is lazy and `orElse` isn't.

---

## The one-paragraph version

Instead of every library inventing its own one-method interface, the JDK ships a
package, `java.util.function`, of ready-made ones, named by **what the lambda does**,
not by who uses it. If your lambda **takes a value and returns a different one**, it's
a **`Function`**. If it **takes a value and answers yes/no**, it's a **`Predicate`**.
If it **takes nothing and produces a value** — typically *later*, only if needed — it's
a **`Supplier`**. If it **takes a value and returns nothing** (it just *does* something),
it's a **`Consumer`**. Add a `Bi` prefix for two inputs; `UnaryOperator` and
`BinaryOperator` for "same type in, same type out". Because these interfaces carry
extra default methods (`andThen`, `and`, `negate`), small lambdas can be **combined**
into bigger ones, like Lego, instead of nesting calls. And because boxing costs real
time, each has primitive variants (`IntPredicate`, `ToIntFunction`, …) for `int`,
`long` and `double`.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **`Function<T, R>`** | "Give me a T, I'll give you an R" | Method `R apply(T t)` |
| **`Predicate<T>`** | "Give me a T, I'll say yes or no" | Method `boolean test(T t)` |
| **`Supplier<T>`** | "Ask me and I'll produce a T" | Method `T get()`; no input |
| **`Consumer<T>`** | "Give me a T, I'll do something with it" | Method `void accept(T t)`; returns nothing |
| **`BiFunction` / `BiPredicate` / `BiConsumer`** | The same, with two inputs | `apply(T, U)`, `test(T, U)`, `accept(T, U)` |
| **`UnaryOperator<T>`** | A function whose input and output are the same type | `extends Function<T, T>` |
| **`BinaryOperator<T>`** | Combine two Ts into one T | `extends BiFunction<T, T, T>` |
| **side effect** | Something a function does *besides* returning a value | Printing, writing a file, changing a field |
| **pure function** | Same input → same output, no side effects | Safe to call any number of times, in any order |
| **composition** | Building a bigger function by chaining smaller ones | `andThen`, `compose`, `and`, `or`, `negate` |
| **`default` method** | A method with a body inside an interface | How `andThen` can exist on an interface (full story Day 29) |
| **lazy** | Not computed until needed | A `Supplier` is a "recipe", not the dish |
| **primitive specialisation** | A version of the interface for `int`/`long`/`double` | `IntPredicate`, `ToIntFunction<T>`, `IntSupplier`… avoids boxing |
| **arity** | How many inputs | 0 = Supplier, 1 = Function/Predicate/Consumer, 2 = `Bi…` |

---

## Before you read: four questions

1. You're writing `Optional.orElseGet(…)`. What kind of lambda goes in the brackets —
   what does it take, what does it return?
2. `f.andThen(g)` and `f.compose(g)` — which one runs `f` first?
3. `Predicate<User>`: how would you build "is an adult AND is active" from two
   existing predicates, without writing a new lambda?
4. Why does `IntPredicate` exist when `Predicate<Integer>` already works?

<details><summary>Answers (try first)</summary>

1. A `Supplier`: takes nothing, returns the value. (Part 3.)
2. `andThen` runs `f` first, then `g`. `compose` runs `g` first, then `f`. (Part 1.)
3. `adult.and(active)`. (Part 2.)
4. To avoid boxing every `int` into an `Integer`. (Part 6.)

</details>

---

## Part 0 — How to choose: look at the shape

Don't memorise names. Memorise **the shape of the arrow**:

| Your lambda looks like… | Shape | Interface | Method |
|--------------------------|-------|-----------|--------|
| `s -> s.length()` | one in, a value out (different type) | `Function<String, Integer>` | `apply` |
| `n -> n > 18` | one in, boolean out | `Predicate<Integer>` | `test` |
| `() -> new ArrayList<>()` | nothing in, a value out | `Supplier<List<…>>` | `get` |
| `s -> System.out.println(s)` | one in, nothing out | `Consumer<String>` | `accept` |
| `(a, b) -> a + b` | two in, one out | `BiFunction<A, B, R>` (or `BinaryOperator<T>` if all the same type) | `apply` |
| `s -> s.trim()` where in = out type | one in, same type out | `UnaryOperator<String>` | `apply` |

Naming pattern for the other ~35 interfaces: **`[type prefix][base name]`**.
`Bi` = two inputs. `Int`/`Long`/`Double` = primitive input (`IntPredicate`). `ToInt`/
`ToLong`/`ToDouble` = primitive *output* (`ToIntFunction<T>`). Once you know the six
bases, the rest are readable on sight.

Two cousins that live elsewhere: `Runnable` (`void run()` — nothing in, nothing out;
Phase 5) and `Callable<V>` (`V call()` — like `Supplier` but may throw checked
exceptions; Phase 5).

---

## Part 1 — `Function<T, R>`: transform

**In plain words:** a little machine with one input slot and one output slot. Put a
string in, get its length out. The output can be a different type from the input.

Two methods turn small functions into bigger ones, and the names trip everyone up, so
here they are with a trace of what actually runs, in order:

```java
import java.util.List;
import java.util.function.Function;

public class FunctionDemo {
    public static void main(String[] args) {
        Function<String, Integer> length = s -> s.length();
        System.out.println("apply:  " + length.apply("hello"));

        Function<Integer, Integer> plusOne = n -> { System.out.println("  plusOne(" + n + ")"); return n + 1; };
        Function<Integer, Integer> timesTen = n -> { System.out.println("  timesTen(" + n + ")"); return n * 10; };

        System.out.println("plusOne.andThen(timesTen).apply(5):   first plusOne, then timesTen");
        System.out.println("  = " + plusOne.andThen(timesTen).apply(5));

        System.out.println("plusOne.compose(timesTen).apply(5):   first timesTen, then plusOne");
        System.out.println("  = " + plusOne.compose(timesTen).apply(5));

        // The plan's pipeline: trim -> lowercase -> truncate, composed rather than nested
        Function<String, String> trim = String::trim;
        Function<String, String> lower = String::toLowerCase;
        Function<String, String> truncate = s -> s.length() > 8 ? s.substring(0, 8) : s;

        String nested = truncate.apply(lower.apply(trim.apply("   HELLO, Wonderful World  ")));
        Function<String, String> normalise = trim.andThen(lower).andThen(truncate);
        String composed = normalise.apply("   HELLO, Wonderful World  ");
        System.out.println("nested:   '" + nested + "'");
        System.out.println("composed: '" + composed + "'");

        // A pipeline can be built from a list, at runtime
        List<Function<String, String>> steps = List.of(trim, lower, truncate);
        Function<String, String> built = Function.identity();
        for (Function<String, String> step : steps) {
            built = built.andThen(step);
        }
        System.out.println("built from a list: '" + built.apply("  ABCDEFGHIJKL ") + "'");
    }
}
```

```
apply:  5
plusOne.andThen(timesTen).apply(5):   first plusOne, then timesTen
  plusOne(5)
  timesTen(6)
  = 60
plusOne.compose(timesTen).apply(5):   first timesTen, then plusOne
  timesTen(5)
  plusOne(50)
  = 51
nested:   'hello, w'
composed: 'hello, w'
built from a list: 'abcdefgh'
```

| Method | Order | Reads as |
|--------|-------|----------|
| `f.andThen(g)` | `f` first, then `g` | "f, **and then** g" — **left to right**, the natural reading |
| `f.compose(g)` | `g` first, then `f` | "f **composed with** g" = f(g(x)) — the mathematical order |
| `Function.identity()` | returns its input | The "do nothing" starting point for building a chain |

Rule of thumb: **use `andThen`** and read left-to-right. `compose` exists for maths
people and for the occasional `parse.compose(trim)`.

Why bother with `andThen` when you could nest the calls? Compare the two lines in the
output:

```
truncate.apply(lower.apply(trim.apply(text)))     // nested: read it inside-out, right to left
trim.andThen(lower).andThen(truncate)             // composed: read it left to right
```

The bigger benefit: the composed pipeline is **a value**. You can store it in a field,
pass it to another method, or — as the last block did — **build it from a list at
runtime**. You can't do that with nested calls, which are fixed in the source. This is
exactly how a "text normalisation step" or "request enrichment chain" becomes
configurable.

> **Say this in an interview:** "`Function<T, R>` is a transformation from `T` to `R`.
> `andThen` chains functions left to right; `compose` is the reverse. Composition turns
> small functions into a pipeline that is itself a value, so I can build or configure
> it at runtime."

---

## Part 2 — `Predicate<T>`: test

**In plain words:** a yes/no question about one thing. "Is this user an adult?" "Is
this string empty?" Its superpower is that questions combine with `and`, `or` and
`not` — exactly how you'd phrase a rule in English — and each small question is
named and reusable.

```java
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.function.Predicate;

public class PredicateDemo {
    record User(String name, String email, int age, boolean active) {}

    static final Predicate<User> HAS_NAME = u -> u.name() != null && !u.name().isBlank();
    static final Predicate<User> VALID_EMAIL = u -> u.email() != null && u.email().contains("@");
    static final Predicate<User> ADULT = u -> u.age() >= 18;
    static final Predicate<User> ACTIVE = User::active;

    public static void main(String[] args) {
        User alice = new User("alice", "alice@example.com", 30, true);
        User kid = new User("tim", "tim@example.com", 12, true);
        User noMail = new User("bob", "bob-at-example", 40, true);
        User gone = new User("carl", "carl@example.com", 50, false);

        Predicate<User> canOrder = HAS_NAME.and(VALID_EMAIL).and(ADULT).and(ACTIVE);
        Predicate<User> blocked = canOrder.negate();
        Predicate<User> needsReview = ADULT.negate().or(ACTIVE.negate());

        for (User u : List.of(alice, kid, noMail, gone)) {
            System.out.printf("%-6s canOrder=%-5s blocked=%-5s needsReview=%s%n",
                    u.name(), canOrder.test(u), blocked.test(u), needsReview.test(u));
        }

        System.out.println();
        System.out.println("which rule failed? (named rules)");
        Map<String, Predicate<User>> rules = new LinkedHashMap<>();
        rules.put("has a name", HAS_NAME);
        rules.put("valid email", VALID_EMAIL);
        rules.put("is an adult", ADULT);
        rules.put("is active", ACTIVE);
        for (User u : List.of(kid, noMail, gone)) {
            rules.forEach((label, rule) -> {
                if (!rule.test(u)) System.out.println("  " + u.name() + " fails: " + label);
            });
        }

        System.out.println();
        System.out.println("short-circuit:");
        Predicate<String> loudFalse = s -> { System.out.println("  loudFalse ran"); return false; };
        Predicate<String> loudTrue = s -> { System.out.println("  loudTrue ran"); return true; };
        System.out.println("  false.and(true) = " + loudFalse.and(loudTrue).test("x"));
        System.out.println("  true.or(false)  = " + loudTrue.or(loudFalse).test("x"));

        System.out.println();
        Predicate<String> blank = String::isBlank;
        System.out.println("Predicate.not(String::isBlank) on \"hi\": " + Predicate.not(blank).test("hi"));
        System.out.println("Predicate.isEqual(\"a\") on \"a\": " + Predicate.isEqual("a").test("a"));
    }
}
```

```
alice  canOrder=true  blocked=false needsReview=false
tim    canOrder=false blocked=true  needsReview=true
bob    canOrder=false blocked=true  needsReview=false
carl   canOrder=false blocked=true  needsReview=true

which rule failed? (named rules)
  tim fails: is an adult
  bob fails: valid email
  carl fails: is active

short-circuit:
  loudFalse ran
  false.and(true) = false
  loudTrue ran
  true.or(false)  = true

Predicate.not(String::isBlank) on "hi": true
Predicate.isEqual("a") on "a": true
```

(`record User(...)` is a compact class — Day 36. `u.name()` is its getter. The
`printf` format `%-6s` means "left-aligned in 6 characters".)

This is today's first practice exercise, built up. What to notice:

- **One rule, one name.** `ADULT`, `ACTIVE`… each is testable alone and reused in
  several combinations. `canOrder` reads like the business rule.
- **`and`, `or`, `negate`** each return a **new** `Predicate`; the originals are
  untouched.
- **Short-circuit.** `and` stops at the first `false` and `or` at the first `true`,
  *like `&&` and `||`* — the output shows `loudTrue` never ran in the first case.
  Put the cheap or most-likely-to-fail rule first.
- **Named rules in a `Map`** let you report *which* rule failed — a plain combined
  predicate can only say yes or no. This pattern (a list of named rules) is how
  validation frameworks work.
- **`Predicate.not(String::isBlank)`** (Java 11) is the neat way to negate a method
  reference, which has no `.negate()` until it's been given a `Predicate` type.

> **Say this in an interview:** "A `Predicate<T>` is a boolean test on `T`. `and`, `or`
> and `negate` compose small named rules into bigger ones without new lambdas, with the
> same short-circuiting as `&&` and `||`."

---

## Part 3 — `Supplier<T>`: produce, lazily

**In plain words:** a `Supplier` is a **recipe, not a dish**. Writing the recipe
doesn't cook anything; whoever holds it cooks only if and when they decide to — by
calling `get()` — and cooks again every time they call. That "only if needed" is the
entire point, and it's the answer to a question you've had since Day 25.

```java
import java.util.Optional;
import java.util.function.Supplier;

public class SupplierDemo {
    static boolean debugEnabled = false;

    static String expensiveDescription() {
        System.out.println("  (building an expensive log message)");
        return "state dump ...";
    }

    static void debugEager(String message) {
        if (debugEnabled) System.out.println("DEBUG " + message);
    }

    static void debugLazy(Supplier<String> message) {
        if (debugEnabled) System.out.println("DEBUG " + message.get());
    }

    public static void main(String[] args) {
        Supplier<String> greeting = () -> {
            System.out.println("  supplier body ran");
            return "hello";
        };
        System.out.println("created the supplier - nothing has run yet");
        System.out.println("first get():  " + greeting.get());
        System.out.println("second get(): " + greeting.get() + "   (runs again every time)");

        System.out.println();
        System.out.println("logging with debug OFF:");
        System.out.println(" eager:");
        debugEager(expensiveDescription());
        System.out.println(" lazy:");
        debugLazy(() -> expensiveDescription());

        System.out.println();
        System.out.println("logging with debug ON:");
        debugEnabled = true;
        debugLazy(() -> expensiveDescription());

        System.out.println();
        System.out.println("this is exactly what makes Optional.orElseGet lazy:");
        Optional<String> present = Optional.of("value");
        String r = present.orElseGet(() -> { System.out.println("  NOT printed"); return "default"; });
        System.out.println("  result: " + r);
        Optional<String> empty = Optional.empty();
        r = empty.orElseGet(() -> { System.out.println("  default supplier ran"); return "default"; });
        System.out.println("  result: " + r);
    }
}
```

```
created the supplier - nothing has run yet
  supplier body ran
first get():  hello
  supplier body ran
second get(): hello   (runs again every time)

logging with debug OFF:
 eager:
  (building an expensive log message)
 lazy:

logging with debug ON:
  (building an expensive log message)
DEBUG state dump ...

this is exactly what makes Optional.orElseGet lazy:
  result: value
  default supplier ran
  result: default
```

This is the proof the plan asks for — a print inside a `Supplier`:

- Creating the supplier ran **nothing**. The body ran at each `get()`, and again on the
  second `get()`.
- **Eager (`debugEager`)**: Java must *evaluate the argument* `expensiveDescription()`
  **before** calling the method, so the expensive message was built even though debug
  was off and the result was thrown away. **Lazy (`debugLazy`)**: only a recipe was
  passed, and with debug off, nobody ever called `get()`.
- **That's the `orElse` / `orElseGet` difference from Day 25.** `orElse(x)` takes a
  value, so Java computes `x` first. `orElseGet(supplier)` takes a recipe and only
  calls `get()` when the `Optional` is empty. A "lazy" API is nothing more than "takes
  a `Supplier` instead of a value".

Where you'll see `Supplier` constantly: `orElseGet`, `orElseThrow(() -> new X())`,
`Objects.requireNonNull(x, () -> "message " + expensive)`,
`Collectors.toCollection(ArrayList::new)`, `CompletableFuture.supplyAsync(…)`
(Phase 5), and logging frameworks' `log.debug(() -> …)`.

Two cautions: a `Supplier` that returns a *different* value each call (`() -> new
Random().nextInt()`) is legitimate but means callers can't assume `get()` is repeatable;
and a Supplier is **not memoised** — it recomputes each time. If you want
compute-once-and-cache, you write that yourself (or use a library "lazy").

> **Say this in an interview:** "A `Supplier<T>` takes no arguments and produces a `T`
> on demand. Passing one instead of a value defers the computation until it's needed —
> that's why `orElseGet` doesn't evaluate its default when the value is present while
> `orElse` always does."

---

## Part 4 — `Consumer<T>`: do something

**In plain words:** it takes a value and returns nothing — the whole point is the side
effect: print it, save it, send it, add it to a list. `forEach` is the classic user.

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;
import java.util.function.BiConsumer;
import java.util.function.BiFunction;
import java.util.function.BiPredicate;
import java.util.function.BinaryOperator;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.UnaryOperator;

public class ConsumerAndBi {
    public static void main(String[] args) {
        List<String> audit = new ArrayList<>();
        Consumer<String> print = s -> System.out.println("  print: " + s);
        Consumer<String> record = audit::add;
        Consumer<String> both = print.andThen(record);
        both.accept("user logged in");
        both.accept("user logged out");
        System.out.println("audit trail: " + audit);

        System.out.println();
        BiFunction<String, Integer, String> repeat = (s, n) -> s.repeat(n);
        BiPredicate<String, Integer> longerThan = (s, n) -> s.length() > n;
        BiConsumer<String, Integer> show = (k, v) -> System.out.println("  " + k + " -> " + v);
        System.out.println("BiFunction:  " + repeat.apply("ab", 3));
        System.out.println("BiPredicate: " + longerThan.test("hello", 3));
        System.out.println("BiConsumer via Map.forEach:");
        Map<String, Integer> stock = new TreeMap<>(Map.of("apples", 5, "pears", 0));
        stock.forEach(show);

        System.out.println();
        UnaryOperator<String> shout = s -> s.toUpperCase();
        Function<String, String> sameThingAsFunction = shout;
        System.out.println("UnaryOperator is a Function<T,T>: " + sameThingAsFunction.apply("hey"));
        List<String> names = new ArrayList<>(List.of("ann", "bo"));
        names.replaceAll(shout);
        System.out.println("List.replaceAll(UnaryOperator):   " + names);

        BinaryOperator<Integer> max = BinaryOperator.maxBy(Integer::compare);
        BinaryOperator<Integer> sum = (a, b) -> a + b;
        System.out.println("BinaryOperator maxBy(3, 9):  " + max.apply(3, 9));
        System.out.println("BinaryOperator sum(3, 9):    " + sum.apply(3, 9));
        Map<String, Integer> counts = new TreeMap<>();
        for (String w : "a b a c a b".split(" ")) counts.merge(w, 1, sum);
        System.out.println("Map.merge(key, 1, sum):      " + counts);
    }
}
```

```
  print: user logged in
  print: user logged out
audit trail: [user logged in, user logged out]

BiFunction:  ababab
BiPredicate: true
BiConsumer via Map.forEach:
  apples -> 5
  pears -> 0

UnaryOperator is a Function<T,T>: HEY
List.replaceAll(UnaryOperator):   [ANN, BO]
BinaryOperator maxBy(3, 9):  9
BinaryOperator sum(3, 9):    12
Map.merge(key, 1, sum):      {a=3, b=2, c=1}
```

`Consumer.andThen(other)` runs both on the **same** input, in order: print, *then*
record. Unlike `Function.andThen`, the second doesn't receive the first's output —
a `Consumer` has no output. (So "do these two things with this value" is the shape.)

A rule worth stating: **`Function` and `Predicate` should be pure — no side effects.**
The library may call them any number of times, in any order, even in parallel. A
`Predicate` that prints or changes a counter will surprise you (the short-circuit demo
in Part 2 showed a predicate not even being called). **Side effects belong in
`Consumer`s**, and in terminal operations like `forEach`.

---

## Part 5 — The `Bi…` and "operator" family

**In plain words:** same shapes, more inputs — or "same type everywhere".

| Interface | Method | When you'll meet it |
|-----------|--------|---------------------|
| `BiFunction<T, U, R>` | `R apply(T, U)` | Combine two different things; `Map.compute`, `merge` |
| `BiPredicate<T, U>` | `boolean test(T, U)` | "Does this key/value pair qualify?" |
| `BiConsumer<T, U>` | `void accept(T, U)` | **`Map.forEach((key, value) -> …)`** — by far the most common |
| `UnaryOperator<T>` | `T apply(T)` | `List.replaceAll(…)`; any "T to T" step |
| `BinaryOperator<T>` | `T apply(T, T)` | `reduce`, `Map.merge` — combining two Ts into one |

**How they relate (it's inheritance, not a new idea):**

```
UnaryOperator<T>  extends  Function<T, T>
BinaryOperator<T> extends  BiFunction<T, T, T>
```

That's what the demo printed: a `UnaryOperator<String>` could be assigned straight to a
`Function<String, String>`. They exist for **readability** ("this is T → T, not some
arbitrary conversion") and to give `replaceAll` and `reduce` precise signatures.

`Map.merge(word, 1, sum)` is the idiom you'll use all the time: "put 1 if absent,
otherwise combine the old and new values with `sum`" — a word-count in one line, using
a `BinaryOperator`. (Day 32 covers `merge` properly.)

---

## Part 6 — Primitive variants and the boxing cost

**In plain words:** `Predicate<Integer>` works, but every `int` you pass has to be
wrapped in an `Integer` object first (Day 21, Part 4). The primitive versions skip the
wrapping: `IntPredicate` takes a plain `int`.

| Generic | Primitive-friendly | Method |
|---------|--------------------|--------|
| `Predicate<Integer>` | `IntPredicate`, `LongPredicate`, `DoublePredicate` | `test(int)` |
| `Function<T, Integer>` | **`ToIntFunction<T>`**, `ToLongFunction<T>`, `ToDoubleFunction<T>` | `applyAsInt(T)` |
| `Function<Integer, R>` | `IntFunction<R>` | `apply(int)` |
| `Supplier<Integer>` | `IntSupplier` (also `Long…`, `Double…`, `BooleanSupplier`) | `getAsInt()` |
| `BinaryOperator<Integer>` | `IntBinaryOperator`, `LongBinaryOperator`, … | `applyAsInt(int, int)` |
| `UnaryOperator<Integer>` | `IntUnaryOperator`, … | `applyAsInt(int)` |
| `Consumer<Integer>` | `IntConsumer`, … | `accept(int)` |

Note the method names change (`applyAsInt`, `getAsInt`) — a hint that a primitive is
involved.

How much does it matter? Two programs, honestly measured. The first, a simple loop
calling `Predicate<Integer>` vs `IntPredicate` 50 million times, gave **98 ms vs 73 ms**
— a small difference, because the JIT inlined the tiny lambda and removed most of the
boxing (the Day 21 `Generations` lesson: the JIT can hide costs in toy loops). The
second keeps the boxed value alive between calls, as real code does:

```java
import java.util.function.BinaryOperator;
import java.util.function.LongBinaryOperator;

public class PrimitiveCost {
    static long foldBoxed(int n, BinaryOperator<Long> op) {
        Long acc = 0L;
        for (int i = 0; i < n; i++) {
            acc = op.apply(acc, (long) i);
        }
        return acc;
    }

    static long foldPrimitive(int n, LongBinaryOperator op) {
        long acc = 0L;
        for (int i = 0; i < n; i++) {
            acc = op.applyAsLong(acc, i);
        }
        return acc;
    }

    public static void main(String[] args) {
        int n = 50_000_000;
        BinaryOperator<Long> boxedSum = (a, b) -> a + b;
        BinaryOperator<Long> boxedMax = (a, b) -> a > b ? a : b;
        LongBinaryOperator primSum = (a, b) -> a + b;
        LongBinaryOperator primMax = (a, b) -> a > b ? a : b;

        // warm up with several different lambdas so the call sites are not trivially inlined
        foldBoxed(1000, boxedSum);
        foldBoxed(1000, boxedMax);
        foldPrimitive(1000, primSum);
        foldPrimitive(1000, primMax);

        for (int round = 1; round <= 2; round++) {
            long t0 = System.nanoTime();
            long a = foldBoxed(n, boxedSum);
            long boxedMs = (System.nanoTime() - t0) / 1_000_000;

            t0 = System.nanoTime();
            long b = foldPrimitive(n, primSum);
            long primMs = (System.nanoTime() - t0) / 1_000_000;

            System.out.println("round " + round + ": same answer=" + (a == b)
                    + "  BinaryOperator<Long>=" + boxedMs + " ms  LongBinaryOperator=" + primMs + " ms");
        }
    }
}
```

```
round 1: same answer=true  BinaryOperator<Long>=447 ms  LongBinaryOperator=36 ms
round 2: same answer=true  BinaryOperator<Long>=269 ms  LongBinaryOperator=46 ms
```

Here the boxed version is roughly **6–12× slower** (numbers vary by run; the ratio is
the point): every step allocates a new `Long`, and nothing lets the JIT remove it.

The takeaway is calibrated: **don't switch to primitive interfaces everywhere** — in
ordinary code it doesn't matter and `Function<Integer, …>` is easier to read. **Do use
them in numeric hot paths and over large data** — and notice that Day 35's primitive
streams (`IntStream`, `mapToInt`) are exactly this idea applied to the whole pipeline.

---

## Part 7 — Composition in practice

**In plain words:** small named pieces, joined at the call site. It's the same
mindset as small methods, but you can store, pass and rebuild the joins.

You've built both halves today: a **validation rule** (`Predicate` + `and`/`negate`)
and a **transformation** (`Function` + `andThen`). A realistic combined shape — a
reusable "sanitise then validate then act" pipeline built from parts:

```
Function<String, String> sanitise = trim.andThen(lower);          // transform
Predicate<String> acceptable = notBlank.and(shortEnough);          // test
Consumer<String> save = repository::save;                          // act

input -> {
    String clean = sanitise.apply(input);
    if (acceptable.test(clean)) save.accept(clean);
}
```

Each piece is one line, has a name, and can be unit-tested alone — and swapped (a
different `sanitise` for a different tenant, say) without touching the others. That's
what "functional style" buys; not cleverness, but small, reusable, replaceable parts.

Honest limits: a chain of eight `andThen`s is *less* readable than a method with eight
named steps, and stack traces through composed lambdas are noisier. Compose when the
parts are reused or built dynamically. Otherwise a plain method is fine.

---

## Hands-on

1. **Required — predicates.** Run `PredicateDemo`. Add `Predicate<User> hasCompanyEmail =
   u -> u.email().endsWith("@example.com")` and a rule `canJoinBeta = ADULT.and(ACTIVE)
   .and(hasCompanyEmail.negate())`. Predict who passes, then check.
2. **Required — function pipeline.** Build `trim → lowercase → truncate(8)` from
   `FunctionDemo` using `andThen` only. Then add a fourth step that replaces spaces with
   `-`, and rebuild the same pipeline with `compose` to feel why `andThen` reads better.
3. **Required — prove laziness.** Run `SupplierDemo`. Write a method
   `String firstNonNull(String a, Supplier<String> fallback)` and call it so that the
   supplier's print statement is **not** executed. Then call it so that it is.
4. For each of these lambdas, name the interface *before* looking at the table:
   `() -> 42`, `n -> n * n`, `s -> s.isEmpty()`, `(a, b) -> a.compareTo(b)`,
   `x -> list.add(x)`, `s -> s.length() > 3 ? s : null`.
5. Use `list.removeIf(Predicate)` and `list.replaceAll(UnaryOperator)` on an
   `ArrayList<String>` — two methods that take these interfaces directly. Predict the
   result before running.
6. Use `Map<String,Integer>.forEach` with a `BiConsumer` to print each entry; then use
   `merge` with a `BinaryOperator` to count the letters in a sentence.
7. Run `PrimitiveCost` three times. Then change `foldBoxed` to use `Integer` for small
   values (`i % 100`) and watch what the `Integer` cache from Day 21 does to the gap.
8. Open `java.util.function.Predicate`'s source in VS Code (Ctrl-click). Find `and`,
   `negate` and `not`. Notice they're `default`/`static` methods — Day 29 explains how an
   interface can have them.

---

## Gotchas

- **`andThen` vs `compose`** — the order. If a pipeline gives the wrong answer, swap them
  first.
- **`andThen` on a `Predicate` doesn't exist** — Predicates combine with `and`/`or`.
  `Function` and `Consumer` chain with `andThen`.
- **A method reference has no methods to call on** — `String::isBlank.negate()` doesn't
  compile; use `Predicate.not(String::isBlank)` or assign to a `Predicate` first.
- **`Function<T, Boolean>` is not `Predicate<T>`.** `filter` needs a `Predicate`;
  they're different types even though the lambdas look the same.
- **Side effects in `Function`/`Predicate`** — especially inside `map`/`filter` on
  streams — are a classic bug source (the library may call them out of order or skip
  them). Day 34.
- **`Supplier` isn't cached.** Each `get()` recomputes.
- **Checked exceptions** can't escape any of these interfaces (none declare `throws`).
  Wrap them, or use `Callable` where that fits.
- **Raw lambdas are anonymous in stack traces** (`lambda$main$0`). A named predicate
  constant (`ADULT`) shows up in your code as a name — another reason to name reusable
  rules.
- **Primitive interfaces' method names differ** (`applyAsInt`, `getAsLong`, `test` stays
  `test`). IDE autocomplete helps.
- **Over-composing** is a readability trap. Compose rules and pipelines you reuse;
  write a plain method for one-offs.

---

## Recap

- Choose by the shape: `T → R` `Function`; `T → boolean` `Predicate`; `() → T`
  `Supplier`; `T → void` `Consumer`.
- `Bi…` = two inputs. `UnaryOperator<T>` is `Function<T,T>`; `BinaryOperator<T>` is
  `BiFunction<T,T,T>`.
- `andThen` (left-to-right) and `compose` (right-to-left) chain Functions; `and`, `or`,
  `negate` combine Predicates; `Consumer.andThen` does two things with the same input.
- A `Supplier` is a recipe: nothing runs until `get()`. That is the entire reason
  `orElseGet` and `debug(() -> …)` are lazy.
- Keep `Function`/`Predicate` free of side effects; side effects go in `Consumer`.
- Primitive variants avoid boxing; the gap is invisible in toy loops and 6–12× in a
  real fold. Use them in numeric hot paths.
- Composed functions are values: you can store, pass and build them at runtime.

Re-answer the four questions at the top.

---

## Say it out loud

1. "Name the four core functional interfaces and the shape of each."
2. "What's the difference between `andThen` and `compose`?"
3. "How would you build a validation rule from several smaller rules?"
4. "Why is `orElseGet(Supplier)` lazy and `orElse(value)` not?"
5. "What's the relationship between `UnaryOperator<T>` and `Function<T, T>`?"
6. "Why are there `IntPredicate` and friends, and when is it worth using them?"

---

## Q&A

_Follow-up questions from this lesson are appended here._
