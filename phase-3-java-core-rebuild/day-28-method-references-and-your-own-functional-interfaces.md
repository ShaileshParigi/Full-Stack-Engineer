# Day 28 — Method References & Your Own Functional Interfaces

_Why this matters:_ Once you can read `.map(Person::getName)`, `.forEach(System.out::println)`
and `Collectors.toCollection(ArrayList::new)` without a pause, a huge fraction of modern
Java stops looking like code in a foreign language. The third kind of method reference
(`String::toLowerCase`, "unbound") is the one that trips almost everyone — it looks like
a static call and isn't. Today also covers the practical reality that bites in real
projects: **a lambda can't throw a checked exception**, and what to do about it — and
when to write your own functional interface instead of using `Function` or `Predicate`.

---

## The one-paragraph version

Often a lambda does nothing but **pass its arguments straight to an existing method**:
`s -> s.toLowerCase()`, `x -> System.out.println(x)`, `n -> new Person(n)`. A **method
reference** is a shorter way to say exactly that — "use *that method* as the lambda" —
by naming the method with `::` and dropping the parameters. There are four kinds,
depending on what's to the left of `::`: a **class + static method**
(`Integer::parseInt`), a **specific object + its method** (`System.out::println`), a
**class + instance method** where the first parameter becomes the object the method is
called on (`String::toLowerCase`), and a **class + `new`** (`ArrayList::new`). They
aren't a different feature from lambdas; each one is shorthand for a particular lambda,
and the compiler picks which by looking at the interface it has to fit. When the
standard interfaces (`Function`, `Predicate`…) don't give you the right name, the right
documentation, or the right `throws` clause, you write your own **`@FunctionalInterface`**
— which is also the first step in dealing with checked exceptions inside lambdas.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **method reference** | A lambda that only calls one existing method, written as just the method's name | `Type::method`, `object::method`, `Type::new` |
| **`::`** | "the method called …" | The method-reference operator |
| **receiver** | The object a method is called *on* | In `s.toLowerCase()`, `s` is the receiver |
| **static method reference** | `ClassName::staticMethod` — the class owns the method | `Integer::parseInt` ⇒ `s -> Integer.parseInt(s)` |
| **bound instance reference** | `object::method` — the receiver is a specific object, fixed when the reference is created | `System.out::println` ⇒ `x -> System.out.println(x)` |
| **unbound instance reference** | `ClassName::instanceMethod` — the receiver isn't fixed yet; it arrives as the **first parameter** | `String::toLowerCase` ⇒ `s -> s.toLowerCase()` |
| **constructor reference** | `ClassName::new` — calls a constructor | `Person::new` ⇒ `n -> new Person(n)` |
| **arity** | How many parameters | Decides which constructor/overload a reference picks |
| **overload** | Several methods with the same name, different parameters | Why some references are ambiguous |
| **checked exception** | An exception the compiler forces you to handle or declare | Day 23; `IOException` |
| **wrapper (adapter)** | A small helper that converts one thing to another | Here: converts a throwing function into a plain `Function` |
| **sneaky throw** | A trick that throws a checked exception without declaring it | Legal but hides the contract; mentioned, not taught |

---

## Before you read: four questions

1. `Function<String, String> f = String::toLowerCase;` — what is the *lambda* version?
   What plays the part of "the object toLowerCase is called on"?
2. `Comparator<String> c = String::compareToIgnoreCase;` — `compareToIgnoreCase` takes
   *one* argument, but a `Comparator` takes *two*. How does that compile?
3. What happens when you try `Function<String, String> f = s -> Files.readString(Path.of(s));`?
4. When would you write your own `@FunctionalInterface` instead of using `Predicate<T>`?

<details><summary>Answers (try first)</summary>

1. `s -> s.toLowerCase()`. The first (only) parameter is the receiver. (Part 2.)
2. The first parameter becomes the receiver and the second is the argument:
   `(a, b) -> a.compareToIgnoreCase(b)`. (Part 2.)
3. Compile error: `readString` throws `IOException`, and `Function.apply` doesn't allow
   it. (Part 6.)
4. For a domain name, better documentation, a checked exception, or an arity the JDK
   doesn't offer. (Part 5.)

</details>

---

## Part 1 — A lambda that only forwards

**In plain words:** look at these three lambdas. Each one takes its inputs and hands
them, unchanged, to an existing method. The lambda adds no logic — just plumbing. A
method reference says the same thing with less plumbing, so the reader sees *which
method* instead of re-reading the wiring.

```
s -> s.toLowerCase()          →   String::toLowerCase
x -> System.out.println(x)    →   System.out::println
n -> new Person(n)            →   Person::new
```

The test for "can this be a method reference?": **does the lambda body consist of
exactly one method call, whose arguments are exactly the lambda's parameters, in
order?** If you do anything else — `s -> s.toLowerCase().trim()`, `x -> x * 2`,
`s -> foo(s, 5)` — it stays a lambda.

---

## Part 2 — The four kinds

Run this; each pair is a reference and the lambda it stands for, and both give the same
result:

```java
import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;
import java.util.function.BiFunction;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.IntFunction;
import java.util.function.Supplier;

public class FourKinds {
    static class Person {
        final String name;
        Person(String name) { this.name = name; }
        @Override public String toString() { return "Person(" + name + ")"; }
    }

    public static void main(String[] args) {
        System.out.println("-- 1. STATIC: Integer::parseInt  (the class owns the method) --");
        Function<String, Integer> parseRef = Integer::parseInt;
        Function<String, Integer> parseLambda = s -> Integer.parseInt(s);
        System.out.println(parseRef.apply("42") + " " + parseLambda.apply("42"));

        System.out.println("-- 2. BOUND instance: System.out::println  (a specific object is fixed now) --");
        Consumer<String> printRef = System.out::println;
        Consumer<String> printLambda = s -> System.out.println(s);
        printRef.accept("via reference");
        printLambda.accept("via lambda");

        String greeting = "hello, ";
        Function<String, String> concatRef = greeting::concat;
        Function<String, String> concatLambda = s -> greeting.concat(s);
        System.out.println(concatRef.apply("world") + " | " + concatLambda.apply("world"));

        System.out.println("-- 3. UNBOUND instance: String::toLowerCase  (first parameter becomes the receiver) --");
        Function<String, String> lowerRef = String::toLowerCase;
        Function<String, String> lowerLambda = s -> s.toLowerCase();
        System.out.println(lowerRef.apply("ABC") + " " + lowerLambda.apply("ABC"));

        Comparator<String> cmpRef = String::compareToIgnoreCase;
        Comparator<String> cmpLambda = (a, b) -> a.compareToIgnoreCase(b);
        System.out.println("compareToIgnoreCase(\"a\", \"B\"): " + cmpRef.compare("a", "B") + " " + cmpLambda.compare("a", "B"));

        BiFunction<String, String, Boolean> startsRef = String::startsWith;
        BiFunction<String, String, Boolean> startsLambda = (s, prefix) -> s.startsWith(prefix);
        System.out.println("startsWith: " + startsRef.apply("java", "ja") + " " + startsLambda.apply("java", "ja"));

        System.out.println("-- 4. CONSTRUCTOR: ArrayList::new, Person::new, int[]::new --");
        Supplier<List<String>> newListRef = ArrayList::new;
        Supplier<List<String>> newListLambda = () -> new ArrayList<>();
        List<String> l = newListRef.get();
        l.add("x");
        System.out.println(l + " " + newListLambda.get());

        Function<String, Person> personRef = Person::new;
        Function<String, Person> personLambda = n -> new Person(n);
        System.out.println(personRef.apply("alice") + " " + personLambda.apply("bob"));

        IntFunction<int[]> arrayRef = int[]::new;
        System.out.println("int[]::new(3) -> length " + arrayRef.apply(3).length);
    }
}
```

```
-- 1. STATIC: Integer::parseInt  (the class owns the method) --
42 42
-- 2. BOUND instance: System.out::println  (a specific object is fixed now) --
via reference
via lambda
hello, world | hello, world
-- 3. UNBOUND instance: String::toLowerCase  (first parameter becomes the receiver) --
abc abc
compareToIgnoreCase("a", "B"): -1 -1
startsWith: true true
-- 4. CONSTRUCTOR: ArrayList::new, Person::new, int[]::new --
[x] []
Person(alice) Person(bob)
int[]::new(3) -> length 3
```

| # | Kind | Written | Lambda it stands for | What's left of `::` |
|---|------|---------|----------------------|---------------------|
| 1 | **Static** | `Integer::parseInt` | `s -> Integer.parseInt(s)` | A **class**; the method is `static` |
| 2 | **Bound instance** | `System.out::println`, `greeting::concat` | `x -> System.out.println(x)` | An **object / expression** |
| 3 | **Unbound instance** | `String::toLowerCase`, `String::compareToIgnoreCase` | `s -> s.toLowerCase()`, `(a, b) -> a.compareToIgnoreCase(b)` | A **class**; the method is an *instance* method |
| 4 | **Constructor** | `ArrayList::new`, `Person::new`, `int[]::new` | `() -> new ArrayList<>()`, `n -> new Person(n)` | A **class** + `new` |

### The confusing one: unbound instance

**In plain words:** `String::toLowerCase` looks like "call a static method named
`toLowerCase` on the class `String`". It isn't. `toLowerCase` isn't static — it only
makes sense *on a particular string*. So the reference has a gap in it: "a string to be
supplied later". The lambda it's used as supplies it: **the first parameter becomes the
object the method is called on; any remaining parameters become the method's
arguments.**

```
Type::instanceMethod      ≡      (first, rest...) -> first.instanceMethod(rest...)

Function<String,String>          String::toLowerCase           s       ->  s.toLowerCase()
Comparator<String> (2 params)    String::compareToIgnoreCase   (a, b)  ->  a.compareToIgnoreCase(b)
BiFunction<String,String,Bool>   String::startsWith            (s, p)  ->  s.startsWith(p)
```

That's the answer to question 2: `compareToIgnoreCase` takes one argument, `Comparator`
supplies two — the first is the receiver, the second is the argument.

**How to read any `X::m` — three questions:**

```
Is X a class name or an expression (object)?
  expression (a variable, System.out, this) ......... BOUND:  params go to m
  class name:
     is m a static method? ................ STATIC:  all params go to m
     is m an instance method? ............. UNBOUND: first param = receiver, rest go to m
     m is "new" ........................... CONSTRUCTOR: params go to the constructor
```

The compiler decides which meaning applies by looking at the **target interface's
parameters** — that's why the same-looking reference can mean different lambdas in
different places.

### When the reference is ambiguous

A class can have both a static and an instance method with the same name. Then the
compiler can't tell which you mean. A real one:

```
Function<Integer, String> c = Integer::toString;
// error: incompatible types: invalid method reference
//        reference to toString is ambiguous
```

`Integer` has a *static* `Integer.toString(int)` **and** an *instance*
`integer.toString()`. With one parameter, both fit: `i -> Integer.toString(i)` and
`i -> i.toString()`. The compiler refuses to guess. Fix: use the lambda (or
`String::valueOf`, which is unambiguous). This is the standard example of "a method
reference hid too much" — see Part 4.

> **Say this in an interview:** "A method reference is shorthand for a lambda that just
> calls an existing method. There are four kinds: static, bound instance (receiver fixed
> at creation), unbound instance (the first parameter is the receiver) and constructor.
> `String::toLowerCase` is `s -> s.toLowerCase()`."

---

## Part 3 — Bound references fix the receiver *now*

**In plain words:** with a lambda, `() -> counter.next()` looks up `counter` when the
lambda *runs*. With `counter::next`, the object is picked **at the moment the reference
is created** and carried along. Almost always the same thing — but not when the object is
`null`, or when the variable changes (it can't, per Day 26's effectively-final rule —
but a *field* or the result of an expression can).

```java
import java.util.function.Supplier;

public class Receiver {
    static class Counter {
        int n = 0;
        int next() { return ++n; }
    }

    public static void main(String[] args) {
        System.out.println("-- the bound receiver is evaluated ONCE, when the reference is created --");
        Counter first = new Counter();
        Supplier<Integer> ref = first::next;
        Supplier<Integer> lambda = () -> first.next();
        System.out.println("both use the same counter: " + ref.get() + ", " + lambda.get() + ", " + ref.get());

        System.out.println();
        System.out.println("-- null receiver: when does it blow up? --");
        String name = null;

        Supplier<Integer> lazyLambda = () -> name.length();
        System.out.println("lambda created fine (nothing has run yet)");
        try {
            lazyLambda.get();
        } catch (NullPointerException e) {
            System.out.println("lambda: NPE only when CALLED");
        }

        try {
            Supplier<Integer> eagerRef = name::length;
            System.out.println("reference created fine");
        } catch (NullPointerException e) {
            System.out.println("method reference: NPE already when CREATED");
        }
    }
}
```

```
-- the bound receiver is evaluated ONCE, when the reference is created --
both use the same counter: 1, 2, 3

-- null receiver: when does it blow up? --
lambda created fine (nothing has run yet)
lambda: NPE only when CALLED
method reference: NPE already when CREATED
```

The practical consequence: **`name::length` with `name == null` fails at the line that
*creates* the reference** — even if the `Supplier` is never called. A lambda defers the
failure until it runs. This is a real, if rare, source of "why did this blow up here?".
(Java deliberately evaluates a bound receiver immediately and checks it for `null`.)

Other bound forms: **`this::process`** (a method on the current object — very common in
Spring beans), and **`super::method`**.

---

## Part 4 — Clearer, or hides too much?

**In plain words:** a method reference removes the plumbing and keeps the *name*. That's
a win when the name says what's happening, and a loss when it doesn't.

**Use a reference when**

- the method's name makes the intent obvious: `.map(Person::getName)`,
  `.filter(String::isBlank)`, `.forEach(System.out::println)`;
- the lambda is pure forwarding — it'd be noise to write `p -> p.getName()`;
- it's a constructor/factory: `ArrayList::new`, `Order::new`.

**Stay with a lambda when**

| Situation | Why a lambda is better |
|-----------|------------------------|
| **Overloaded or ambiguous** (`Integer::toString`) | The compiler can't pick, or the reader can't tell which overload runs |
| **The method name is generic** (`Util::process`, `Helper::handle`) | The lambda shows *what's being passed* and where |
| **You want to see the arguments** | `(order, tax) -> price(order, tax)` documents the order of params |
| **A little logic is needed** | `s -> s.trim().toLowerCase()` is not forwarding |
| **Debugging** | A lambda gives you somewhere to put a breakpoint |
| **Mixed receiver** (`this::fn` vs `Type::fn` can be confused) | The lambda is explicit about which object |

A common reviewer rule: **if you have to pause to work out which of the four kinds it
is, use the lambda.** Neither form is "more functional"; the goal is the one that's
quicker to read.

> **Say this in an interview:** "I use method references when the lambda just forwards
> to a well-named method — `Person::getName`, `String::trim` — and keep a lambda when
> the method is overloaded or ambiguous, when I want the arguments visible, or when
> there's any logic."

---

## Part 5 — Writing your own functional interface

**In plain words:** `Function<String, String>` tells the reader *nothing* about what the
function is for. `Validator<String>` or `PriceCalculator` does. When a lambda is a real
concept in *your* program, give its type a name — it's almost free, and it makes
signatures read like English: `void register(Validator<User> rule)` instead of
`void register(Function<User, String> f)`.

**When to write your own, rather than reuse `Function`/`Predicate`:**

| Reason | Example |
|--------|---------|
| **A domain name** | `PriceCalculator`, `RetryPolicy`, `EventHandler<E>` |
| **Documentation** — the contract in the Javadoc | "returns `null` if valid, otherwise the reason" |
| **A checked exception** — the generic ones can't throw | `ThrowingSupplier<T> { T get() throws Exception; }` |
| **An arity or shape the JDK lacks** | Three inputs (`TriFunction`), `int,int → boolean` |
| **Extra default methods** that belong with it | `validator.and(other)` |
| **Overload-resolution control** | Distinct types stop `submit(() -> …)` from being ambiguous |

**When *not* to:** if `Function`, `Predicate`, `Supplier` etc. already express it and
there's no domain name worth giving, use the standard one — every Java developer knows
them, and they come with `andThen`, `and`, `negate` for free. Don't write
`StringToIntConverter` when `Function<String, Integer>` says it.

The rules are Day 26's: **exactly one abstract method**; annotate it
`@FunctionalInterface`; `default` and `static` methods are allowed.

```java
import java.io.IOException;
import java.util.List;
import java.util.function.Function;
import java.util.function.Predicate;

public class Validators {
    /** Checks a value and says why it was rejected. */
    @FunctionalInterface
    interface Validator<T> {
        /** @return null if valid, otherwise a human-readable reason */
        String validate(T value);

        default Validator<T> and(Validator<T> other) {
            return v -> {
                String problem = this.validate(v);
                return problem != null ? problem : other.validate(v);
            };
        }

        static <T> Validator<T> of(Predicate<T> test, String message) {
            return v -> test.test(v) ? null : message;
        }
    }

    /** Like Function, but allowed to throw a checked exception. */
    @FunctionalInterface
    interface ThrowingFunction<T, R> {
        R apply(T t) throws Exception;
    }

    static <T, R> Function<T, R> unchecked(ThrowingFunction<T, R> f) {
        return t -> {
            try {
                return f.apply(t);
            } catch (RuntimeException e) {
                throw e;
            } catch (Exception e) {
                throw new RuntimeException("wrapped: " + e.getMessage(), e);
            }
        };
    }

    static String load(String path) throws IOException {
        if (path.startsWith("missing")) throw new IOException("no such file: " + path);
        return "contents of " + path;
    }

    public static void main(String[] args) {
        Validator<String> notBlank = Validator.of(s -> !s.isBlank(), "must not be blank");
        Validator<String> shortEnough = Validator.of(s -> s.length() <= 5, "must be at most 5 characters");
        Validator<String> username = notBlank.and(shortEnough);

        for (String input : List.of("alice", "", "alexander")) {
            String problem = username.validate(input);
            System.out.println("'" + input + "' -> " + (problem == null ? "OK" : problem));
        }

        System.out.println();
        Function<String, String> loader = unchecked(Validators::load);
        System.out.println(loader.apply("notes.txt"));
        try {
            loader.apply("missing.txt");
        } catch (RuntimeException e) {
            System.out.println("caught: " + e.getMessage());
            System.out.println("cause:  " + e.getCause().getClass().getSimpleName());
        }

        System.out.println();
        List<String> paths = List.of("a.txt", "b.txt");
        paths.stream().map(unchecked(Validators::load)).forEach(System.out::println);
    }
}
```

```
'alice' -> OK
'' -> must not be blank
'alexander' -> must be at most 5 characters

contents of notes.txt
caught: wrapped: no such file: missing.txt
cause:  IOException

contents of a.txt
contents of b.txt
```

The first half is today's `Validator<T>` exercise. Notice:

- The interface is generic (`T`) and has **one abstract method** — a lambda fits.
- `default Validator<T> and(...)` gives it composition, like `Predicate.and` — but it's
  *yours*, and short-circuits by returning the first problem.
- `static Validator.of(predicate, message)` is a **factory** that adapts the standard
  `Predicate` into your type. You can build a custom interface *on top of* the
  standard ones.
- Its name and Javadoc ("null if valid, otherwise the reason") are the documentation
  `Predicate<T>` could never give.

(`default` and `static` methods inside interfaces are the subject of tomorrow, Day 29.)

---

## Part 6 — Checked exceptions inside lambdas

**In plain words:** `Function.apply(T)` is declared as *"returns R, throws nothing
checked"*. So whatever lambda you write for it can't throw a checked exception either.
If the method you need to call (`Files.readString`, a JDBC call, anything `throws
IOException`) does, the compiler stops you. This is the genuinely ugly corner of
Java 8, and the reason Day 23 recommended unchecked exceptions by default.

The real errors from your JDK:

```
static String load(String path) throws IOException { … }

Function<String, String> a = path -> load(path);
// error: unreported exception IOException; must be caught or declared to be thrown

Function<String, String> b = Errs::load;
// error: incompatible thrown types IOException in functional expression
```

Same cause, two phrasings — and the method reference doesn't escape it.

Your options, in order of preference:

| # | Option | What it looks like | Good for |
|---|--------|--------------------|----------|
| 1 | **Try/catch inside the lambda** | `path -> { try { return load(path); } catch (IOException e) { throw new UncheckedIOException(e); } }` | One-off, short |
| 2 | **Extract a method that handles it**, then reference it | `private String loadOrWrap(String p) { … }` → `this::loadOrWrap` | A lambda that grew, reused logic |
| 3 | **A wrapper that converts** a throwing function into a plain `Function` | `unchecked(Validators::load)` — the demo above | You do this in many places |
| 4 | **Your own throwing interface**, if the *consumer* of the lambda is yours too | `ThrowingFunction<T,R> { R apply(T) throws Exception; }` and the method that calls it handles the exception | APIs you control (retry helpers, transaction templates) |
| 5 | A library (jOOL's `Unchecked`, Vavr) | Same as 3, pre-written | If you already depend on one |
| 6 | "Sneaky throw" (a generics trick that smuggles a checked exception past the compiler) | — | **Avoid.** Callers can't know to catch it |

Read the wrapper in the demo (option 3) carefully, because it shows three good habits:

```
catch (RuntimeException e) { throw e; }          // don't double-wrap what's already unchecked
catch (Exception e) { throw new RuntimeException("wrapped: " + e.getMessage(), e); }   // keep the cause!
```

The second line passes `e` as the **cause** — Day 23's rule — so the real `IOException`
is still in the stack trace (`cause: IOException` in the output). `UncheckedIOException`
is the JDK's ready-made wrapper for `IOException` specifically; prefer it when it fits.

A design note: if you find yourself wrapping in every lambda, the question is whether
those APIs should have been unchecked in the first place. That's exactly the choice
Spring made with `DataAccessException`. And it's why the **hands-on 6** asks you to write
a method that *takes* a throwing interface instead of forcing callers to wrap.

> **Say this in an interview:** "The standard functional interfaces don't declare
> checked exceptions, so a lambda can't throw one. I catch and rethrow an unchecked
> exception *with the original as the cause*, extract the logic into a method, or wrap
> with a helper that adapts a throwing function — and I never swallow the exception."

---

## Hands-on

1. **Required — five conversions.** Convert each to a method reference, and for the
   *unbound* ones write the equivalent lambda back, naming the receiver:

   | Lambda | Reference? |
   |--------|-----------|
   | `s -> s.length()` | ? |
   | `(a, b) -> Integer.compare(a, b)` | ? |
   | `() -> new StringBuilder()` | ? |
   | `s -> System.err.println(s)` | ? |
   | `(s, t) -> s.concat(t)` | ? |

   <details><summary>Answers (try first)</summary>

   `String::length` (unbound; receiver = `s`) · `Integer::compare` (static) ·
   `StringBuilder::new` (constructor) · `System.err::println` (bound; receiver =
   `System.err`) · `String::concat` (unbound; receiver = `s`, argument = `t`).

   </details>

2. **Required — your own interface.** Run `Validators`. Add a `Validator<String>` for
   "contains an `@`", combine it with the others for an email field, and print the
   failing reason for `"hello"`. Then change `and` so it collects *all* the problems
   into a `List<String>` instead of stopping at the first — that needs a different
   return type, so you'll need to redesign the interface; notice how that's easier when
   the interface is yours.
3. **Required — the checked-exception error.** Write a method `static String read(String
   p) throws IOException`, try to use it as a `Function<String, String>` lambda and as
   `Main::read`, and read both errors. Then fix it with (a) a try/catch lambda, (b) the
   `unchecked` wrapper from this lesson.
4. Predict, then check, what these are as lambdas: `Integer::sum` (as
   `BinaryOperator<Integer>`), `String::valueOf` (as `Function<Object,String>`),
   `Math::abs` (as `UnaryOperator<Integer>`), `Objects::nonNull` (as `Predicate<String>`).
5. Run `Receiver`. Then change `name` into a *field* that's `null` when the reference is
   created and set later — does the reference still fail at creation? Why can't you do
   this with a local variable? (Hint: Day 26.)
6. Write `static <T> T retry(int times, ThrowingSupplier<T> op)` where
   `ThrowingSupplier<T> { T get() throws Exception; }`. Your method catches the
   exception, retries, and rethrows after the last failure — *callers never wrap*.
7. Reproduce the `Integer::toString` ambiguity, and fix it two ways.
8. Open `java.util.Comparator` in VS Code and find `comparing` and `thenComparing`.
   Rewrite yesterday's `SortThreeWays` to sort by age then by name using method
   references only. (This is Day 33's material arriving early — a good sign.)

---

## Gotchas

- **`Type::instanceMethod` isn't static.** If you read `String::toLowerCase` as a static
  call you'll never understand `Comparator<String> c = String::compareToIgnoreCase`.
- **Ambiguous references** — any class with both a static and an instance method of the
  same name and shape (`Integer::toString` is the classic) — write the lambda.
- **A bound receiver is evaluated immediately** and a `null` fails at creation.
- **Constructor references pick the constructor by the target's parameters:**
  `Function<String, Person>` uses `Person(String)`; `Supplier<Person>` would need
  `Person()`. The wrong shape gives an unhelpful error.
- **`this::method` captures `this`** — the whole object stays alive as long as the
  reference does (Day 21 leaks; Day 26 gotcha).
- **Don't wrap a checked exception without the cause.** `throw new RuntimeException()`
  loses the real error.
- **Don't put heavy logic inside the wrapper helper** — it's called on every element.
- **Generic custom interfaces are fine; over-engineering isn't.** A custom interface per
  lambda is clutter. One per *concept* is design.
- **Lambdas stay readable only while short.** If a lambda body needs a comment, extract
  a named method and reference that.
- **Method references and `var`:** `var f = String::trim;` doesn't compile — a
  reference needs a target type, same as a lambda.

---

## Recap

- A **method reference** (`::`) is shorthand for a lambda that only forwards to an
  existing method.
- Four kinds: **static** (`Integer::parseInt`), **bound** (`System.out::println` — the
  receiver is fixed when created), **unbound** (`String::toLowerCase` — the *first
  parameter is the receiver*), **constructor** (`ArrayList::new`).
- The compiler picks the meaning from the **target interface's parameters**; if both a
  static and instance method fit, it's ambiguous (`Integer::toString`).
- Prefer a reference when the name is clear; prefer a lambda when overloaded, when
  there's logic, or when the arguments help the reader.
- Write your own `@FunctionalInterface` for a **domain name**, **documentation**, a
  **checked exception**, or an **unusual arity** — otherwise use the standard ones.
- Lambdas can't throw checked exceptions. Catch-and-wrap *with the cause*, extract a
  method, adapt with a wrapper, or define a throwing interface for APIs you own.

Re-answer the four questions at the top.

---

## Say it out loud

1. "What is a method reference, and when can a lambda be turned into one?"
2. "Name the four kinds and give the equivalent lambda for each."
3. "Explain `String::compareToIgnoreCase` used as a `Comparator` — where do the two
   arguments go?"
4. "When is a lambda clearer than a method reference?"
5. "When would you write your own functional interface instead of `Function`?"
6. "How do you deal with a checked exception inside a lambda, and what must you not forget?"

---

## Q&A

_Follow-up questions from this lesson are appended here._
