# Week 6 Quiz — Java 8: Lambdas & Functional Programming (Days 26–30)

**How to take it.** Give yourself 75–90 minutes. For every question, answer **out
loud or on paper first**, then open the answer. For the *predict* questions, write
your prediction down, **then** press Run — a prediction made after seeing the output
doesn't count. Every runnable block is a complete program and needs no internet. Use
the lab or `javac`/`java` on JDK 21.

**Scoring.** One point per question, 32 in total. Half a point if you had the idea but
not the words. **26 or more (80%)** — you're ready for Week 7. This week is
load-bearing for streams: `filter`, `map`, `reduce`, `collect` and `groupingBy` are all
the Day 27 shapes plus the Day 28 method references plus the Day 29 defaults. Below 26,
each answer names the day to reread; reread it, then retake the questions you missed a
day later.

This quiz covers Days 26–30. (Week 5, Days 21–25, hasn't had a written quiz — ask if
you'd like it too.)

---

## Part A — Lambdas and capture (Day 26)

**A1.** A colleague says: "A lambda is just a nicer way to write an anonymous class."
Give **two concrete ways** that's wrong.

<details>
<summary>Answer</summary>

Any two of:

1. **No class file.** An anonymous class compiles to `Outer$1.class`. A lambda's body
   becomes a *private method* of the enclosing class, and an **`invokedynamic`**
   instruction creates the object at run time — compiling `WhatIsALambda` produced
   `WhatIsALambda$1.class` for the anonymous class and nothing for the lambda.
2. **`this` means different things.** In an anonymous class `this` is the anonymous
   object; in a lambda `this` is the **enclosing** object (the body is a method of the
   enclosing class).
3. **A lambda has no type of its own.** It's only ever "an implementation of whichever
   functional interface it's assigned to", and that interface must have **exactly one
   abstract method**. An anonymous class can implement an interface with many methods,
   or extend a class, and can hold its own fields.
4. **Scope.** A lambda isn't a new scope — it can't declare a local with the same name as
   one in the enclosing method (an anonymous class's can shadow).

→ *Day 26, Part 3 and Part 6*

</details>

**A2.** Which of these can be the **target of a lambda** (i.e. is usable as a
functional interface)? Say why for each.

```
interface P { boolean test(String s); }

interface Q {
    boolean test(String s);
    default Q negate() { return s -> !test(s); }
}

interface R { void a(); void b(); }

interface S extends P { boolean equals(Object o); }

interface T { }
```

<details>
<summary>Answer</summary>

- **P — yes.** Exactly one abstract method.
- **Q — yes.** `default` (and `static`) methods don't count; there's still one abstract
  method (`test`). Its own default even returns a lambda of type `Q`.
- **R — no.** Two abstract methods: a lambda can't say which one it implements. Adding
  `@FunctionalInterface` to it would be a compile error ("Unexpected @FunctionalInterface
  annotation").
- **S — yes.** It inherits `test`, and `equals(Object)` redeclared in an interface
  doesn't count — it's one of `Object`'s public methods every object already has (that's
  also why `Comparator` is functional despite listing `equals`).
- **T — no.** Zero abstract methods; there's nothing for the lambda's body to implement.

→ *Day 26, Part 3*

</details>

**A3.** *Predict, then Run.* Write down the four output lines.

```java
public class CaptureQuiz {
    interface Counter { int next(); }

    static int total = 0;

    static Counter make(int start) {
        int[] box = {start};
        return () -> box[0]++;
    }

    public static void main(String[] args) {
        Counter a = make(10);
        Counter b = make(100);
        System.out.println(a.next() + " " + a.next() + " " + b.next());
        System.out.println(a.next() + " " + b.next());

        Runnable r = () -> total += 5;
        r.run();
        r.run();
        System.out.println("total=" + total);

        String prefix = "id-";
        java.util.function.Function<Integer, String> f = n -> prefix + n;
        System.out.println(f.apply(7));
    }
}
```

<details>
<summary>Answer</summary>

```
10 11 100
12 101
total=10
id-7
```

Each call to `make` creates its **own** `box` on the heap, and each returned lambda
captured a reference to *its* box — so `a` and `b` count independently, and `make`'s
stack frame is long gone while they keep working. `box[0]++` is the lambda mutating a
**heap object** (legal), not reassigning a local variable (illegal). `total` is a
**static field** — also on the heap/class, so `total += 5` is fine. `prefix` is
effectively final so `f` can capture a copy of it. → *Day 26, Part 5*

</details>

**A4.** Which line does `javac` report as the error, and why? Then give **two fixes**.

```
String label = "n";
int count = 0;
Runnable r = () -> System.out.println(label + count);   // line 3
count++;                                                  // line 4
```

<details>
<summary>Answer</summary>

It reports **line 3** — inside the lambda: *"local variables referenced from a lambda
expression must be final or effectively final"* — even though the lambda only **reads**
`count`. The cause is the **later** `count++` on line 4, which means `count` is not
effectively final. (`label` is fine.)

**Why the rule exists:** locals live on the method's **stack frame** (Day 21), which is
destroyed when the method returns; the lambda may live on and run later, possibly on
another thread. So the lambda holds a **copy of the value**. If the original could
change afterwards the copy and the original would silently disagree, so Java forbids the
change.

**Fixes:** (1) take a snapshot — `final int snapshot = count; Runnable r = () ->
System.out.println(label + snapshot);` — then `count++` is free to continue; (2) if you
genuinely need shared mutable state, use a heap object: an `AtomicInteger`, or a
one-element array (a smell, and not thread-safe unless Atomic); (3) don't mutate in the
first place — compute the final value, then create the lambda. → *Day 26, Part 5*

</details>

**A5.** *Predict, then Run.* Write down the three output lines. (Remember what each
`this` refers to.)

```java
public class ThisQuiz {
    private final String name;

    ThisQuiz(String name) { this.name = name; }

    @Override
    public String toString() { return "ThisQuiz(" + name + ")"; }

    Runnable makeLambda() {
        return () -> System.out.println("lambda: " + this);
    }

    Runnable makeAnon() {
        return new Runnable() {
            private final String name = "inner";

            @Override
            public void run() {
                System.out.println("anon: " + name + " / " + ThisQuiz.this.name);
            }

            @Override
            public String toString() { return "anon-object"; }
        };
    }

    public static void main(String[] args) {
        ThisQuiz q = new ThisQuiz("outer");
        q.makeLambda().run();
        q.makeAnon().run();
        System.out.println("the anon itself: " + q.makeAnon());
    }
}
```

<details>
<summary>Answer</summary>

```
lambda: ThisQuiz(outer)
anon: inner / outer
the anon itself: anon-object
```

In the lambda, `this` is the **enclosing** `ThisQuiz`, so printing it uses
`ThisQuiz.toString()`. The anonymous class is a **separate object** with its own `name`
field that *shadows* the outer one: bare `name` is `"inner"`, and reaching the outer
field needs `ThisQuiz.this.name`. A lambda can't shadow like that — it has no scope of
its own. → *Day 26, Part 6*

</details>

---

## Part B — The functional interface toolkit (Day 27)

**B1.** Name the standard functional interface for each lambda (the `java.util.function`
type, with its type arguments where obvious):

1. `() -> UUID.randomUUID()`
2. `u -> u.age() >= 18`
3. `msg -> log.info(msg)`
4. `s -> s.trim()`
5. `s -> s.length()`
6. `(k, v) -> System.out.println(k + "=" + v)`

<details>
<summary>Answer</summary>

1. **`Supplier<UUID>`** — nothing in, a value out.
2. **`Predicate<User>`** — one in, boolean out.
3. **`Consumer<String>`** — one in, nothing out (a side effect).
4. **`UnaryOperator<String>`** (which *is a* `Function<String, String>`) — same type in
   and out.
5. **`Function<String, Integer>`** — or **`ToIntFunction<String>`** to avoid boxing the
   result.
6. **`BiConsumer<K, V>`** — two in, nothing out (it's exactly what `Map.forEach` takes).

Rule: look at **the shape of the arrow**, not the name. → *Day 27, Part 0*

</details>

**B2.** *Predict, then Run.* Write down the three lines, including **the order** of the
bracketed trace text.

```java
import java.util.function.Function;

public class ComposeQuiz {
    public static void main(String[] args) {
        Function<Integer, Integer> addTwo = n -> { System.out.print("[+2 on " + n + "] "); return n + 2; };
        Function<Integer, Integer> square = n -> { System.out.print("[sq on " + n + "] "); return n * n; };

        System.out.println(addTwo.andThen(square).apply(3));
        System.out.println(addTwo.compose(square).apply(3));
        System.out.println(Function.<Integer>identity().andThen(square).andThen(addTwo).apply(4));
    }
}
```

<details>
<summary>Answer</summary>

```
[+2 on 3] [sq on 5] 25
[sq on 3] [+2 on 9] 11
[sq on 4] [+2 on 16] 18
```

`f.andThen(g)` runs **`f` first, then `g`** (left to right): add 2 → 5, then square →
25. `f.compose(g)` runs **`g` first, then `f`** (maths order `f(g(x))`): square 3 → 9,
then add 2 → 11. The third chain starts from `identity()` (does nothing), then squares,
then adds two. The trace prints before the result because `println` evaluates its
argument first. → *Day 27, Part 1*

</details>

**B3.** *Predict, then Run.* For each line, the result **and** how many predicates ran.

```java
import java.util.function.Predicate;

public class ShortCircuitQuiz {
    public static void main(String[] args) {
        int[] calls = {0};
        Predicate<String> notEmpty = s -> { calls[0]++; return !s.isEmpty(); };
        Predicate<String> shortWord = s -> { calls[0]++; return s.length() < 5; };
        Predicate<String> rule = notEmpty.and(shortWord);

        for (String w : new String[] {"", "tree", "forest"}) {
            calls[0] = 0;
            boolean ok = rule.test(w);
            System.out.println("'" + w + "' -> " + ok + " (predicates run: " + calls[0] + ")");
        }

        calls[0] = 0;
        boolean either = shortWord.or(notEmpty).test("ab");
        System.out.println("either('ab') -> " + either + " (predicates run: " + calls[0] + ")");
    }
}
```

<details>
<summary>Answer</summary>

```
'' -> false (predicates run: 1)
'tree' -> true (predicates run: 2)
'forest' -> false (predicates run: 2)
either('ab') -> true (predicates run: 1)
```

`and` and `or` **short-circuit** like `&&` and `||`. With `""`, `notEmpty` is false, so
`and` never calls `shortWord`. With `"forest"` both run (`notEmpty` true, `shortWord`
false). For `or`, `shortWord("ab")` is already true so `notEmpty` is skipped. Practical
consequence: put the **cheap or most-likely-to-fail** rule first; and never rely on a
predicate's side effects happening. → *Day 27, Part 2*

</details>

**B4.** *Predict, then Run.* Which `computing …` lines appear under each number, and
how many times?

```java
import java.util.Optional;
import java.util.function.Supplier;

public class LazyQuiz {
    static String fallback(String label) {
        System.out.println("  computing " + label);
        return label;
    }

    public static void main(String[] args) {
        Optional<String> some = Optional.of("have");
        Optional<String> none = Optional.empty();

        System.out.println("1");
        some.orElse(fallback("A"));
        System.out.println("2");
        some.orElseGet(() -> fallback("B"));
        System.out.println("3");
        none.orElseGet(() -> fallback("C"));
        System.out.println("4");
        none.or(() -> Optional.of(fallback("D")));
        System.out.println("5");
        Supplier<String> s = () -> fallback("E");
        System.out.println("6");
        s.get();
        s.get();
    }
}
```

<details>
<summary>Answer</summary>

```
1
  computing A
2
3
  computing C
4
  computing D
5
6
  computing E
  computing E
```

- **1:** `orElse(fallback("A"))` — Java evaluates the *argument* before calling `orElse`,
  so A is computed **even though the value is present**. The classic bug.
- **2:** `orElseGet` takes a `Supplier` (a recipe) and the `Optional` has a value, so
  the recipe is never run.
- **3:** the `Optional` is empty, so the supplier runs once.
- **4:** `or(Supplier<Optional>)` — empty, so it runs.
- **5:** *creating* a `Supplier` runs **nothing**.
- **6:** every `get()` re-runs the body — a `Supplier` is a recipe, not a cached value.

→ *Day 27, Part 3; Day 25, Part 3*

</details>

**B5.** *Write the code.* Using only `Predicate`s combined with `and`/`or`/`negate`,
build a `Predicate<String> validSlug` meaning: **not blank, at most 20 characters, and
contains no spaces**. Then build `Predicate<String> invalidSlug` from it without writing
a new lambda. Then build a `Function<String, String> normalise` that trims, lowercases,
and replaces spaces with `-`, **composed with `andThen`, not nested**.

<details>
<summary>Answer</summary>

```java
import java.util.function.Function;
import java.util.function.Predicate;

public class SlugQuiz {
    public static void main(String[] args) {
        Predicate<String> notBlank = s -> !s.isBlank();
        Predicate<String> shortEnough = s -> s.length() <= 20;
        Predicate<String> hasSpace = s -> s.contains(" ");

        Predicate<String> validSlug = notBlank.and(shortEnough).and(hasSpace.negate());
        Predicate<String> invalidSlug = validSlug.negate();

        Function<String, String> trim = String::trim;
        Function<String, String> lower = String::toLowerCase;
        Function<String, String> dashes = s -> s.replace(' ', '-');
        Function<String, String> normalise = trim.andThen(lower).andThen(dashes);

        String raw = "  Hello Big World ";
        String slug = normalise.apply(raw);
        System.out.println("'" + slug + "' valid=" + validSlug.test(slug));
        System.out.println("'" + raw + "' invalid=" + invalidSlug.test(raw));
    }
}
```

```
'hello-big-world' valid=true
'  Hello Big World ' invalid=true
```

Notes: `Predicate.not(hasSpace)` would also do. `String::trim` and `String::toLowerCase`
are **unbound** method references (first parameter = receiver). The three small
functions are each named, testable, and reusable. Ordering the cheapest check first
(`notBlank`) is a free optimisation. → *Day 27, Parts 1–2 and 7*

</details>

**B6.** Explain in your own words: **(a)** how `UnaryOperator<T>` relates to
`Function<T, T>`; **(b)** why `IntPredicate` exists when `Predicate<Integer>` works; and
**(c)** whether you should switch every `Function<Integer, …>` in your code to a
primitive interface.

<details>
<summary>Answer</summary>

**(a)** `UnaryOperator<T> extends Function<T, T>` — it *is* a function whose input and
output types are the same; it exists for readability and for signatures like
`List.replaceAll(UnaryOperator)`. (`BinaryOperator<T>` likewise extends
`BiFunction<T, T, T>`.) A `UnaryOperator` can be assigned to a `Function<T, T>`.

**(b)** `Predicate<Integer>` forces every `int` to be **boxed** into an `Integer`
object. `IntPredicate` takes a plain `int` — no allocation. The method names change
(`applyAsInt`, `getAsInt`) as a hint.

**(c)** **No.** In a toy loop the JIT inlined away most of the boxing (98 ms vs 73 ms
for 50 million calls). In a fold that keeps a boxed `Long` alive between calls it was
**6–12× slower**. So: use the primitive interfaces in **numeric hot paths and over large
data** (and Day 35's `IntStream` is this idea for whole pipelines); elsewhere prefer the
more readable generic ones. → *Day 27, Parts 5–6*

</details>

---

## Part C — Method references and your own interfaces (Day 28)

**C1.** Turn each lambda into a method reference and name the **kind** (static / bound /
unbound / constructor):

1. `s -> s.isEmpty()`
2. `() -> new HashMap<String, Integer>()`
3. `x -> Math.abs(x)`
4. `msg -> logger.info(msg)` (where `logger` is an existing variable)
5. `(a, b) -> a.compareTo(b)` (for `String`s)

<details>
<summary>Answer</summary>

1. **`String::isEmpty`** — *unbound*: the first parameter `s` becomes the receiver.
2. **`HashMap::new`** — *constructor* (the compiler infers the type arguments from the
   target).
3. **`Math::abs`** — *static*: all parameters go to the method.
4. **`logger::info`** — *bound*: a specific, already-existing object is the receiver,
   fixed when the reference is created.
5. **`String::compareTo`** — *unbound* with two parameters: first = receiver, second =
   argument. This is the one that looks impossible until you apply the rule.

→ *Day 28, Part 2*

</details>

**C2.** Write the **equivalent lambda** for each reference, given the target type:

1. `BiFunction<String, Integer, Character> f = String::charAt;`
2. `Function<String, Person> g = Person::new;`
3. `BiFunction<String, Integer, Person> h = Person::new;` (assume `Person(String, int)`)
4. `IntFunction<String[]> k = String[]::new;`

<details>
<summary>Answer</summary>

1. `(s, i) -> s.charAt(i)` — unbound: `s` is the receiver, `i` is the argument.
2. `n -> new Person(n)` — the **`Person(String)`** constructor.
3. `(name, age) -> new Person(name, age)` — the compiler picks the **two-parameter**
   constructor because the *target interface* has two parameters.
4. `size -> new String[size]` — an array constructor reference.

The target's parameter list decides which overload/constructor a reference means.
→ *Day 28, Part 2*

</details>

**C3.** *Predict, then Run.* Five lines. (Think about **when** `sb.append("b")` runs, and
what `sb` *is*.)

```java
import java.util.function.Supplier;

public class RefQuiz {
    public static void main(String[] args) {
        StringBuilder sb = new StringBuilder("a");

        Supplier<String> viaRef = sb.append("b")::toString;
        Supplier<String> viaLambda = () -> sb.append("c").toString();

        System.out.println(viaRef.get());
        System.out.println(viaRef.get());
        System.out.println(viaLambda.get());
        System.out.println(viaLambda.get());
        System.out.println(viaRef.get());
    }
}
```

<details>
<summary>Answer</summary>

```
ab
ab
abc
abcc
abcc
```

For a **bound** reference, the receiver expression (`sb.append("b")`) is evaluated
**once, when the reference is created** — so `"b"` was appended exactly once, before any
`get()`. That expression returns `sb` itself, so the reference is bound to the one
`StringBuilder`; each `viaRef.get()` just calls `sb.toString()` at that moment (hence
`ab`, `ab`, then — after the lambda has appended twice — `abcc`). The lambda re-runs its
whole body each time, appending `"c"` each call. Two lessons: bound receivers are
evaluated eagerly (a `null` receiver fails at *creation*), and because the receiver is a
**mutable** object, what you see depends on its state at call time. → *Day 28, Part 3*

</details>

**C4.** `Function<Integer, String> c = Integer::toString;` fails to compile. What does
the error say, why does it happen, and how do you fix it?

<details>
<summary>Answer</summary>

*"incompatible types: invalid method reference — reference to toString is ambiguous."*

`Integer` has a **static** `Integer.toString(int)` **and** an **instance**
`toString()`. With one parameter both are valid readings: `i -> Integer.toString(i)`
(static) and `i -> i.toString()` (unbound instance). The compiler refuses to guess.

**Fix:** write the lambda you mean (`i -> i.toString()`), or use the unambiguous
`String::valueOf`. It's the standard example of when a method reference *hides too
much*. → *Day 28, Part 2 and Part 4*

</details>

**C5.** This does not compile:

```
Function<String, String> read = path -> Files.readString(Path.of(path));
```

Explain the error, give **three ways** to deal with it, and name **the one thing you
must not forget** when you wrap an exception.

<details>
<summary>Answer</summary>

`Files.readString` declares `throws IOException` (a **checked** exception), but
`Function.apply` declares no checked exceptions, so the lambda can't throw it: *"unreported
exception IOException; must be caught or declared to be thrown."* (A method reference
version fails with *"incompatible thrown types IOException in functional expression."*)

**Ways to deal with it:**
1. **Try/catch inside the lambda**, rethrowing `UncheckedIOException` (the JDK's wrapper
   for exactly this).
2. **Extract a method** that does the try/catch and reference it (`this::readOrWrap`).
3. **A wrapper helper** `unchecked(ThrowingFunction<T,R>)` that converts a throwing
   function into a plain `Function` — or define your own `ThrowingFunction`
   interface *if you own the code that calls it* and let that code handle the exception.

**Do not forget the cause:** `throw new UncheckedIOException(e)` / `new
RuntimeException("…", e)` — pass the original exception, so the real error survives in
the `Caused by:` chain (Day 23). `throw new RuntimeException()` loses it; an empty catch
swallows it. → *Day 28, Part 6*

</details>

**C6.** Give **three reasons** to write your own `@FunctionalInterface` instead of using
`Function`/`Predicate`, and one situation where you should **not**.

<details>
<summary>Answer</summary>

Any three of: **a domain name** (`PriceCalculator`, `Validator<T>` read better in
signatures than `Function<Order, Double>`); **documentation** — a Javadoc contract
("returns `null` if valid, otherwise the reason"); a **checked exception**
(`ThrowingSupplier<T> { T get() throws Exception; }`); an **arity or shape the JDK
lacks** (three inputs, `TriFunction`); **extra `default` methods** that belong with it
(`validator.and(other)`); **overload control** (distinct types stop `submit(() -> …)`
being ambiguous).

**Don't** when a standard one says it already and there's no domain name worth giving —
everyone knows `Predicate`, it comes with `and`/`or`/`negate` for free, and
`StringToIntConverter` instead of `Function<String, Integer>` is just clutter. One
custom interface per *concept*, not per lambda. → *Day 28, Part 5*

</details>

---

## Part D — Default and static methods (Day 29)

**D1.** A library publishes `interface Greeter { String greet(String n); }`; 10,000
projects implement it. The author adds `String farewell(String n);` (abstract) in the
next release. **(a)** What happens to projects that recompile? **(b)** What happens to
projects that don't, but call `farewell`? **(c)** How does a `default` method change
this, and what real Java 8 changes needed it?

<details>
<summary>Answer</summary>

**(a)** They **fail to compile**: *"EnglishGreeter is not abstract and does not override
abstract method farewell(String) in Greeter."*

**(b)** Already-compiled classes still load, but the call throws
**`java.lang.AbstractMethodError`** at run time — the nasty case, because nothing
complains until that code path runs. (Classes that never call `farewell` keep working.)

**(c)** With `default String farewell(String n) { return "Goodbye, " + n; }`, every
existing implementor **inherits** the method automatically — source *and* binary
compatible. That's how Java 8 could add `Collection.stream()`, `Iterable.forEach`,
`List.sort`, `Map.computeIfAbsent`/`merge` and `Comparator.reversed()` without breaking
every collection library in existence. → *Day 29, Part 1*

</details>

**D2.** *Predict, then Run.* Which `who()` does each class get? (Careful: one of them
compiles *without* any override even though two interfaces are involved.)

```java
public class ResolveQuiz {
    interface Base { default String who() { return "Base"; } }
    interface Left extends Base { default String who() { return "Left"; } }
    interface Right extends Base { }

    static class Parent { public String who() { return "Parent"; } }

    static class A implements Left, Right { }
    static class C extends Parent implements Left { }
    static class D implements Left {
        @Override
        public String who() { return "D>" + Left.super.who(); }
    }

    public static void main(String[] args) {
        System.out.println(new A().who());
        System.out.println(new C().who());
        System.out.println(new D().who());
    }
}
```

<details>
<summary>Answer</summary>

```
Left
Parent
D>Left
```

- **A:** `Right` adds nothing, so it just inherits `Base.who()`. But `Left.who()`
  **overrides** `Base.who()`, and "the most specific interface wins" — so there is no
  conflict and `Left` wins without any override in `A`.
- **C:** the **class wins**. `Parent.who()` (inherited from the superclass) beats any
  interface default, with no override needed.
- **D:** the class overrides and explicitly calls `Left.super.who()` to reuse the default.

The three rules, in order: **class wins → most specific interface wins → otherwise you
must decide.** → *Day 29, Part 4*

</details>

**D3.** Which of these compile? For each failure, give the reason.

```
(a) interface X { default boolean equals(Object o) { return true; } }

(b) interface X { int count = 0; default void inc() { count++; } }

(c) interface X { static String id() { return "x"; } }
    class Y implements X { }
    ... new Y().id();

(d) interface X {
        private String helper() { return "h"; }
        default String a() { return helper(); }
    }
```

<details>
<summary>Answer</summary>

- **(a) No.** *"default method equals in interface X overrides a member of
  java.lang.Object."* Every class already inherits `Object`'s `equals`/`hashCode`/`toString`
  and a class method always beats a default, so it would be meaningless.
- **(b) No.** Interface fields are implicitly `public static final` — constants.
  `count++` fails: *"cannot assign a value to static final variable count."* Interfaces
  have no per-object state.
- **(c) No.** *"cannot find symbol."* **Static interface methods are not inherited**:
  neither `new Y().id()` nor `Y.id()` works; the only call is `X.id()`.
- **(d) Yes.** `private` interface methods (Java 9) let several defaults share code
  without exposing it.

→ *Day 29, Parts 3–5*

</details>

**D4.** `interface A { default String ping() { return "A"; } }` and `interface B { default
String ping() { return "B"; } }`. Class `C implements A, B { }`. **(a)** What does the
compiler say? **(b)** Write `C` so that `new C().ping()` returns `"A+B"`.

<details>
<summary>Answer</summary>

**(a)** *"types A and B are incompatible; class C inherits unrelated defaults for ping()
from types A and B."* (The diamond problem — Java refuses to guess.)

**(b)**
```java
public class DiamondQuiz {
    interface A { default String ping() { return "A"; } }
    interface B { default String ping() { return "B"; } }

    static class C implements A, B {
        @Override
        public String ping() {
            return A.super.ping() + "+" + B.super.ping();
        }
    }

    public static void main(String[] args) {
        System.out.println(new C().ping());
    }
}
```
```
A+B
```
`InterfaceName.super.method()` selects a specific inherited default. → *Day 29, Part 4*

</details>

**D5.** Interface or abstract class? Pick one for each and justify in a sentence.

1. A `Notifier` capability that many unrelated classes need to offer, some of which
   already extend other classes.
2. A `BaseEntity` with `id` and `createdAt` fields and a constructor, shared by 20
   persistent classes.
3. A callback you want callers to pass as a lambda.

<details>
<summary>Answer</summary>

1. **Interface** — a class can implement many but extend only one, so an interface
   doesn't use up the one superclass slot; defaults can supply convenience behaviour.
2. **Abstract class** — it needs **instance state** (fields) and a **constructor**;
   interfaces can't have either.
3. **Interface** (a functional one) — only interfaces can be lambda targets.

Heuristic: *default to an interface* (a "can do X" capability); reach for an abstract
class when subtypes share **state**. They combine well: interface for the contract, an
`AbstractX` skeleton for the shared fields (`List` + `AbstractList`). → *Day 29, Part 5*

</details>

---

## Part E — `java.time` (Day 30)

**E1.** Which type would you use for each?

1. When an order was placed (stored in the database, shown to users worldwide)
2. A customer's date of birth
3. A dentist appointment "9:00 in Berlin, 3 March next year"
4. A request timeout of 30 seconds
5. "Subscription renews every month"
6. A shop's opening time, "09:30"

<details>
<summary>Answer</summary>

1. **`Instant`** (UTC) — Postgres `timestamptz`; convert to the viewer's zone for display.
2. **`LocalDate`** — a calendar fact with no time and no zone (`date` in Postgres).
3. **`LocalDateTime` + a `ZoneId`** (two fields) — *not* an `Instant`, because zone
   rules (DST) can be changed by governments; "9:00 Berlin" must stay 9:00 Berlin.
4. **`Duration`** — an exact amount of time (`Duration.ofSeconds(30)`).
5. **`Period`** (`Period.ofMonths(1)`) — a calendar amount; a month isn't a fixed
   number of seconds.
6. **`LocalTime`** — a clock-face time with no date.

→ *Day 30, Part 8*

</details>

**E2.** *Predict, then Run.* Write down all seven lines.

```java
import java.time.LocalDate;
import java.time.Period;
import java.time.temporal.ChronoUnit;

public class DateQuiz {
    public static void main(String[] args) {
        LocalDate d = LocalDate.of(2026, 1, 31);
        d.plusMonths(1);
        LocalDate a = d.plusMonths(1);
        LocalDate b = a.plusMonths(1);
        LocalDate c = d.plusMonths(2);

        System.out.println(d);
        System.out.println(a);
        System.out.println(b);
        System.out.println(c);
        System.out.println(ChronoUnit.DAYS.between(d, c));
        System.out.println(Period.between(d, c));
        System.out.println(d.plusDays(30).getDayOfWeek());
    }
}
```

<details>
<summary>Answer</summary>

```
2026-01-31
2026-02-28
2026-03-28
2026-03-31
59
P2M
MONDAY
```

`d.plusMonths(1);` with the result ignored changes nothing — `LocalDate` is **immutable**,
so `d` stays Jan 31. `a` = Jan 31 + 1 month is **clamped to Feb 28** (the last valid
day). `b` = `a` + 1 month = Mar **28** — month arithmetic doesn't "remember" the 31st.
`c` = `d` + 2 months directly = Mar 31 (always add from the original). Jan 31 → Mar 31 is
28 + 31 = **59 days**; as a calendar `Period` it's exactly **2 months**. Jan 31 2026 is a
Saturday, and +30 days is Mar 2 → **Monday**. → *Day 30, Part 2*

</details>

**E3.** *Predict, then Run.* The clocks in London go **back** at 02:00 BST on **25 October
2026**. Write down the four lines. (An offset of zero prints as `Z`.)

```java
import java.time.Duration;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.time.ZonedDateTime;

public class ZoneQuiz {
    public static void main(String[] args) {
        ZoneId london = ZoneId.of("Europe/London");
        ZonedDateTime saturdayNoon = ZonedDateTime.of(2026, 10, 24, 12, 0, 0, 0, london);

        ZonedDateTime viaPeriod = saturdayNoon.plusDays(1);
        ZonedDateTime viaDuration = saturdayNoon.plus(Duration.ofHours(24));

        System.out.println(viaPeriod);
        System.out.println(viaDuration);
        System.out.println(Duration.between(saturdayNoon, viaPeriod).toHours() + " hours elapsed to the same wall time");
        System.out.println(LocalDateTime.of(2026, 10, 25, 1, 30).atZone(london));
    }
}
```

<details>
<summary>Answer</summary>

```
2026-10-25T12:00Z[Europe/London]
2026-10-25T11:00Z[Europe/London]
25 hours elapsed to the same wall time
2026-10-25T01:30+01:00[Europe/London]
```

`plusDays(1)` is a **calendar** step — "the same time tomorrow" — so it lands on 12:00,
but because the clocks went back that night, **25 real hours** elapsed.
`plus(Duration.ofHours(24))` is exactly 24 hours' worth of seconds, so it arrives at **11:00** on
the wall. This is the mirror image of the spring-forward case (23 hours) from the
lesson. The last line: **01:30 on 25 October happens twice** in London (once at +01:00
BST, once at +00:00 GMT); `atZone` picks the **earlier** offset (+01:00) by default.
`withLaterOffsetAtOverlap()` selects the other one. A `LocalDateTime` alone cannot tell
them apart. → *Day 30, Parts 5–6*

</details>

**E4.** A bug report: *"Invoices dated 29 December 2025 show the year 2026, but invoices
dated 27 December 2025 are fine. It only happens in the last days of December."* The code is
`DateTimeFormatter.ofPattern("YYYY-MM-dd")`. Diagnose it, fix it, and say why the tests
passed all year.

<details>
<summary>Answer</summary>

**`YYYY` is the *week-based* year, not the calendar year.** In the US locale weeks start
on Sunday and week 1 of a year is the week containing 1 January; 28–31 December 2025 fall
in the week (Sunday 28 Dec – Saturday 3 Jan) that contains 1 January 2026, so `YYYY`
prints **2026** — while 27 December is still in the last week of 2025. For most of the year
`YYYY` and `yyyy` agree, which is why every test with an ordinary date passed — it only
breaks in the last (and sometimes first) days of a year, and the exact days depend on the
locale. **Fix:** `yyyy-MM-dd` (or, better for machine output, the built-in ISO formatter
`DateTimeFormatter.ISO_LOCAL_DATE` / just `date.toString()`).

Related letter traps: `MM` is month but `mm` is minutes (`HH:MM` prints the month where
the minutes should be), and `DD` is day-of-year. → *Day 30, Part 7*

</details>

**E5.** Explain the difference between `Duration` and `Period` using the London example in
E3. Which would you use for **"remind me in 24 hours"** and which for **"same time
tomorrow"**? Which for a cache TTL and which for "30 days from signup"?

<details>
<summary>Answer</summary>

A **`Duration`** is an **exact** length of time (seconds/nanoseconds); one hour is always
3,600 seconds and "24 hours" is always 86,400 seconds. A **`Period`** is a **calendar**
amount — years, months, days — and a calendar day isn't a fixed number of seconds: across
the October clock change it was **25** hours, across the March one **23**.

- "Remind me in 24 hours" → **`Duration.ofHours(24)`** (exact elapsed time).
- "Same time tomorrow" → **`Period.ofDays(1)`** / `plusDays(1)` on a **`ZonedDateTime`**.
- Cache TTL → **`Duration`** (exact).
- "30 days from signup" → **`Period.ofDays(30)`** on a date (a calendar rule, applied to a
  `LocalDate`).

Remember `Duration` can't be added to a `LocalDate` (no hours there). → *Day 30, Part 5*

</details>

**E6.** **(a)** Which Postgres types correspond to `Instant` and to `LocalDateTime`?
**(b)** Does `timestamptz` store the time zone it was written in? **(c)** State the
"store UTC" rule in one sentence.

<details>
<summary>Answer</summary>

**(a)** **`timestamptz` ↔ `Instant`**; **`timestamp` (without time zone) ↔
`LocalDateTime`** — the one to be wary of.

**(b)** **No.** Despite the name, it stores a UTC instant; the input is converted to UTC
and the output is rendered in the *session's* zone. If you need the original zone (a
future local appointment), store it in a separate column.

**(c)** Store and transmit moments as **UTC instants** (`Instant` / `timestamptz` /
ISO-8601 with `Z`), and convert to a time zone — the **user's**, not the server's — only
when displaying them. (Run servers with `TZ=UTC`, and don't depend on
`ZoneId.systemDefault()`.) → *Day 30, Part 8; Phase 7*

</details>

**E7.** *Write the code.* Write `static boolean isOverdue(LocalDate due, Clock clock)` and
prove it with three "todays" **without changing your computer's date**. Why is a `Clock`
parameter better than calling `LocalDate.now()` inside the method?

<details>
<summary>Answer</summary>

```java
import java.time.Clock;
import java.time.Instant;
import java.time.LocalDate;
import java.time.ZoneId;

public class OverdueQuiz {
    static boolean isOverdue(LocalDate due, Clock clock) {
        return LocalDate.now(clock).isAfter(due);
    }

    static Clock at(String isoInstant) {
        return Clock.fixed(Instant.parse(isoInstant), ZoneId.of("UTC"));
    }

    public static void main(String[] args) {
        LocalDate due = LocalDate.of(2026, 10, 15);
        System.out.println("before due : " + isOverdue(due, at("2026-10-14T12:00:00Z")));
        System.out.println("on due day : " + isOverdue(due, at("2026-10-15T23:59:00Z")));
        System.out.println("day after  : " + isOverdue(due, at("2026-10-16T00:00:00Z")));
    }
}
```
```
before due : false
on due day : false
day after  : true
```

`LocalDate.now()` is different every day and depends on the machine's default zone, so a
test that calls it can pass today and fail tomorrow, and can't exercise "the day after".
Passing a `Clock` makes time an **input**: production uses `Clock.systemUTC()`, tests use
`Clock.fixed(...)`. It's the technique Days 39–40 use for loan due dates. → *Day 30,
Part 4*

</details>

---

## Part F — Putting the week together

**F1.** *Bug hunt.* This code has **five** problems from this week. Find each, say which
day taught it, and how to fix it.

```
class ReminderService {
    private final UserRepository repo;
    private final Notifier notifier;

    void sendAll(List<String> userIds) {
        int sent = 0;
        userIds.forEach(id -> {                                   // (1)
            notifier.send(id);
            sent++;
        });
        log.info("sent " + sent);
    }

    User findOrCreate(String id) {
        return repo.find(id).orElse(repo.save(new User(id)));     // (2)
    }

    String stamp(LocalDate date) {
        return date.format(DateTimeFormatter.ofPattern("YYYY-MM-dd"));   // (3)
    }

    LocalDateTime dueAt() {
        return LocalDateTime.now().plusHours(24);                 // (4)
    }

    Function<String, Long> sizeOf = p -> Files.size(Path.of(p));  // (5)
}
```

<details>
<summary>Answer</summary>

1. **`sent++` on a captured local** — compile error (*must be final or effectively
   final*). Day 26. Fix: don't count inside the lambda — use a `for` loop, or an
   `AtomicInteger`, or compute `userIds.size()` if that's what you want.
2. **`orElse(repo.save(...))` always saves** — the argument is evaluated whether or not the
   user exists, creating a user (a side effect) every time. Days 25 and 27. Fix:
   `orElseGet(() -> repo.save(new User(id)))`.
3. **`YYYY` is the week-year** — wrong year in the last/first days of December/January.
   Day 30. Fix: `yyyy-MM-dd`, or `ISO_LOCAL_DATE`.
4. **`LocalDateTime.now()` for a due moment** — no zone, depends on the server's default
   zone, can't be tested, and is wrong across DST. Day 30. Fix: store an **`Instant`**
   (`clock.instant().plus(Duration.ofHours(24))`), with an injected `Clock`.
5. **Checked exception in a lambda** — `Files.size` throws `IOException`, which
   `Function` can't. Day 28. Fix: catch and rethrow `UncheckedIOException(e)` (with the
   cause), or use a wrapper.

Bonus smell: `ReminderService` holds a `Function` in a field written as a lambda with
logic — name it as a method and use a method reference. → *Days 25–30*

</details>

**F2.** *Predict, then Run.* This combines the week: a fixed `Clock`, `Function`,
`Predicate`, `Optional`, method references and `Duration`. Write down the three lines.

```java
import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneOffset;
import java.util.Map;
import java.util.Optional;
import java.util.function.Function;
import java.util.function.Predicate;

public class SynthQuiz {
    record Order(String id, Instant placedAt) {}

    static final Map<String, Order> ORDERS = Map.of(
            "A", new Order("A", Instant.parse("2026-10-09T10:00:00Z")),
            "B", new Order("B", Instant.parse("2026-10-12T08:30:00Z")));

    static Optional<Order> find(String id) {
        return Optional.ofNullable(ORDERS.get(id));
    }

    public static void main(String[] args) {
        Clock clock = Clock.fixed(Instant.parse("2026-10-12T10:00:00Z"), ZoneOffset.UTC);

        Function<Order, Duration> age = o -> Duration.between(o.placedAt(), clock.instant());
        Predicate<Order> stale = o -> age.apply(o).toHours() >= 48;

        for (String id : new String[] {"A", "B", "Z"}) {
            String line = find(id)
                    .map(o -> o.id() + ": " + age.apply(o) + " stale=" + stale.test(o))
                    .orElseGet(() -> id + ": not found");
            System.out.println(line);
        }
    }
}
```

<details>
<summary>Answer</summary>

```
A: PT72H stale=true
B: PT1H30M stale=false
Z: not found
```

Order A was placed three days before the fixed "now" → `PT72H` (a `Duration` prints in
hours when over 24h — `Duration` has no days), ≥ 48 hours so stale. B is 1 h 30 min old.
`Z` is missing, so `find` returns `Optional.empty()`, `map` is skipped, and `orElseGet`
supplies the fallback — lazily. Because the `Clock` is fixed, this output is identical on
any machine on any day: that's the testability point of Day 30. → *Days 25, 27, 30*

</details>

**F3.** Without notes, explain each in **30 seconds** to a colleague who knows Java 7:
**(a)** what a lambda is, **(b)** why default methods exist, **(c)** the difference
between `Instant` and `LocalDateTime`. Then compare with the model answers.

<details>
<summary>Model answers</summary>

**(a)** "A lambda is a short way to write an object that implements an interface with a
single abstract method — a functional interface. The compiler infers which interface from
the context, so I only write the parameters and the body. It can use variables from the
surrounding method, but only ones that never change, because it carries a copy; and
inside it `this` still means the enclosing object."

**(b)** "Adding an abstract method to an interface breaks every class that implements it.
Default methods are methods with a body inside the interface, so implementors inherit them
automatically. That's how Java 8 added `stream()`, `forEach` and `Comparator.reversed()`
without breaking existing code. If two interfaces supply the same default, the class has to
resolve it with `X.super.method()`; otherwise class beats interface, and the more specific
interface wins."

**(c)** "An `Instant` is a point on the timeline — the same moment for everyone, in UTC.
A `LocalDateTime` is just a date and a clock reading with no zone, so it doesn't identify a
moment; the same value means different times in different places and can describe times that
don't exist or happen twice across daylight saving. I store `Instant`s, convert to a
`ZonedDateTime` for display, and use `Local…` types only for calendar concepts like
birthdays."

</details>

---

## What your score says

| Score | Meaning | What to do |
|-------|---------|------------|
| **26–32** | Ready for Week 7 | Move on; skim the Day 27 shape table before Day 34 |
| **20–25** | Mostly there | Reread the days of the questions you missed (each answer names them), retake those a day later |
| **Below 20** | The foundation isn't set yet | Redo Days 26–27 first — everything in Week 7 stands on lambdas and the function shapes — then 28–30 |

**Topic map of the misses:** A → Day 26 · B → Day 27 · C → Day 28 · D → Day 29 · E → Day 30.
If you miss mostly **B**, spend an evening on the "name the shape" drill (B1) with ten new
lambdas of your own; that single skill is what makes `stream.map(...)` readable.
