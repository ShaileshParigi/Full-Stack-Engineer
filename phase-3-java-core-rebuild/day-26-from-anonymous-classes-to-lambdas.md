# Day 26 — From Anonymous Classes to Lambdas

_Why this matters:_ Lambdas are the grammar of modern Java. Streams, `Optional`,
`CompletableFuture`, `@Async`, Spring's `RestClient`/`JdbcClient`, Kafka consumers,
test assertions — all of them are "here's a piece of code, run it for me". If a
lambda is just magic syntax to you, half the Java you read is a guess. If you know
what it actually *is*, it becomes readable. This is also where the Day 21 memory
model pays off: the "effectively final" rule that confuses everyone is a direct
consequence of the stack and heap.

---

## The one-paragraph version

Sometimes you don't want to *give* a method data — you want to give it **behaviour**:
"sort these, and here's how to compare two of them"; "when the response arrives, do
this". Java's only unit of code was a method on a class, so before Java 8 the way to
hand over a piece of behaviour was to write a whole **anonymous class** — five or six
lines of ceremony around one line of actual logic. A **lambda** is a short way to
write exactly that. It is **not** a free-floating function (Java has none): it's an
implementation of an interface that has **exactly one abstract method** — a
**functional interface**. The compiler works out which interface you mean from
where you put the lambda, which is how it can leave out almost everything. A lambda
may use variables from the method around it, but only ones that never change after
they're set (**effectively final**), because the lambda carries a *copy* of them —
the original lives on a stack frame that may be gone by the time the lambda runs.
Inside a lambda, `this` still means the object around it, unlike inside an anonymous
class.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **behaviour as a parameter** | Handing a method a *thing to do* rather than a value | Passing code as an argument |
| **anonymous inner class** | A class with no name, defined and instantiated in one go | `new Comparator<X>() { … }` — compiled to a real `Outer$1.class` |
| **boilerplate / ceremony** | Code that has to be there but says nothing | The `new`, `@Override`, method signature around a one-line body |
| **functional interface** | An interface with exactly one abstract method | e.g. `Runnable`, `Comparator<T>`; the only thing a lambda can be assigned to |
| **abstract method** | A method with a signature but no body — implementers must supply it | The "one thing" a functional interface asks for |
| **lambda expression** | A compact way to write the one method of a functional interface | `(params) -> body` |
| **method reference** | A lambda that just calls an existing method, with the call written as a name | `Person::getAge` (full treatment: Day 28) |
| **target type** | The interface a lambda is being assigned to or passed as | The compiler reads this to know what the lambda *is* |
| **type inference** | The compiler working out types so you don't write them | Lambda parameter types come from the target interface's method |
| **expression body** | A lambda whose body is a single expression; its value is the result | `x -> x + 1` |
| **block body** | A lambda whose body is `{ … }`; needs `return` to give back a value | `x -> { int y = x + 1; return y; }` |
| **capture** | A lambda using a variable from the surrounding method | "closing over" a variable |
| **effectively final** | A local variable that's never reassigned after its first assignment | Doesn't need to say `final`; the compiler checks |
| **`invokedynamic`** | The bytecode instruction that builds the lambda object at first use | Bootstraps via `LambdaMetafactory`; no `.class` file at compile time |
| **`this`** | The current object | In an anonymous class: the anonymous object. In a lambda: the *enclosing* object |

---

## Before you read: four questions

1. What's the one thing you must have in order to use a lambda? (Not a syntax rule —
   a *type* rule.)
2. Can a lambda change a local variable of the method it's written in? Why or why not?
3. Inside a lambda, what does `this` refer to?
4. When you compile a class containing an anonymous class *and* a lambda, which one
   gets its own `.class` file?

<details><summary>Answers (try first)</summary>

1. A functional interface to be the target — an interface with exactly one abstract
   method. (Part 3.)
2. No — captured locals must be effectively final. (Part 5.)
3. The enclosing object. (Part 6.)
4. Only the anonymous class. (Part 3.)

</details>

---

## Part 1 — The problem lambdas solve

**In plain words:** you're a chef (a method) who can sort ingredients but doesn't know
*how* the customer wants them ordered. You say: "tell me how to compare two items and
I'll do the rest." Fair enough — but in pre-Java-8 Java you couldn't just *tell* the
chef the rule. You had to **build and hand over a little object** whose only purpose
was to carry that rule. The rule was one line; the object around it was six.

A method that accepts behaviour (here `Collections.sort`) can't take a method as an
argument — only objects. So behaviour had to be wrapped in an object that implements
an interface with the method the receiver will call. When you only need that object
once, you write the class inline, without a name: an **anonymous inner class**.

Look at how much of this is ceremony:

```
Collections.sort(people, new Comparator<Person>() {          // ← ceremony
    @Override                                                 // ← ceremony
    public int compare(Person p1, Person p2) {                // ← ceremony (types repeated)
        return Integer.compare(p1.getAge(), p2.getAge());     // ← the ONLY real logic
    }                                                         // ← ceremony
});                                                           // ← ceremony
```

One line of meaning, six lines of syntax. Multiply by every callback, listener and
comparator in a codebase and you see why Java 8's biggest feature was removing it.

---

## Part 2 — The same sort, three ways

Run this — it's today's first practice exercise. All three sort the same list by age:

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.List;

public class SortThreeWays {
    static class Person {
        private final String name;
        private final int age;
        Person(String name, int age) { this.name = name; this.age = age; }
        String getName() { return name; }
        int getAge() { return age; }
        @Override public String toString() { return name + "(" + age + ")"; }
    }

    static List<Person> people() {
        List<Person> list = new ArrayList<>();
        list.add(new Person("Carol", 41));
        list.add(new Person("alice", 30));
        list.add(new Person("Bob", 25));
        return list;
    }

    public static void main(String[] args) {
        // 1. Anonymous inner class: the only way before Java 8
        List<Person> a = people();
        Collections.sort(a, new Comparator<Person>() {
            @Override
            public int compare(Person p1, Person p2) {
                return Integer.compare(p1.getAge(), p2.getAge());
            }
        });
        System.out.println("anonymous class: " + a);

        // 2. Lambda: same behaviour, the ceremony removed
        List<Person> b = people();
        Collections.sort(b, (p1, p2) -> Integer.compare(p1.getAge(), p2.getAge()));
        System.out.println("lambda:          " + b);

        // 3. Method reference: the lambda only calls other methods, so name them
        List<Person> c = people();
        c.sort(Comparator.comparingInt(Person::getAge));
        System.out.println("method ref:      " + c);

        // Sort by name, ignoring case - one line each
        List<Person> d = people();
        d.sort((p1, p2) -> p1.getName().compareToIgnoreCase(p2.getName()));
        System.out.println("by name (lambda): " + d);
    }
}
```

```
anonymous class: [Bob(25), alice(30), Carol(41)]
lambda:          [Bob(25), alice(30), Carol(41)]
method ref:      [Bob(25), alice(30), Carol(41)]
by name (lambda): [alice(30), Bob(25), Carol(41)]
```

What got removed at each step:

| Step | Written | What disappeared |
|------|---------|------------------|
| Anonymous class | `new Comparator<Person>() { @Override public int compare(Person p1, Person p2) { return … } }` | — |
| Lambda | `(p1, p2) -> Integer.compare(p1.getAge(), p2.getAge())` | `new Comparator<Person>()`, `@Override`, `public int compare`, the `Person` types, `{ return …; }` |
| Method reference | `Comparator.comparingInt(Person::getAge)` | The parameters entirely — the lambda was only "take `p`, call `p.getAge()`", so we just name the method |

Every piece that vanished was **information the compiler already had**: the interface
is `Comparator<Person>` because `sort` says so; the method is `compare` because that
interface has only one; the parameter types are `Person` because of the generic
argument. A lambda is "write only the bits the compiler can't guess". (The third
form, `comparingInt`, is a helper you'll meet properly on Day 33; method references
are Day 28.)

> **Say this in an interview:** "Before Java 8, passing behaviour meant an anonymous
> inner class — an object wrapping a single method. A lambda expresses the same thing
> concisely, with the compiler inferring the interface, method name and parameter
> types from the target type."

---

## Part 3 — What a lambda actually is

**In plain words:** many languages have functions as a basic building block. Java
doesn't — and didn't add them. A lambda in Java is **a shorthand for creating an
object**: an object that implements an interface, with your code as the body of that
interface's one method. "Passing a lambda" is really "passing an object that was
conveniently written inline". That's why it needs a target: *which* interface is the
object implementing?

### The one rule: functional interfaces

A **functional interface** has **exactly one abstract method**. A lambda can only be
used where such an interface is expected, because then there's no ambiguity about
which method the lambda's body belongs to.

Examples already in the JDK: `Runnable` (`run()`), `Comparator<T>` (`compare`),
`Callable<V>` (`call()`) and — next day — a whole package of them (`Function`,
`Predicate`, `Supplier`…).

```
@FunctionalInterface
interface Shout {
    String apply(String s);        // the one abstract method
}
```

`@FunctionalInterface` is **optional** — a safety label. It makes the *compiler*
reject the interface if it ever stops being functional (say, a colleague adds a second
abstract method and breaks every lambda that used it). It's like `@Override`: not
needed, strongly recommended.

What counts as "one abstract method":

| Doesn't count | Why |
|---------------|-----|
| `default` methods | They have a body (Day 29) |
| `static` methods | Same |
| `equals`, `hashCode`, `toString` redeclared in the interface | Every object already has them from `Object`; `Comparator` declares `equals` and is still functional |

Two abstract methods, and the compiler's real error message:

```
@FunctionalInterface
interface TwoMethods { void a(); void b(); }
// error: Unexpected @FunctionalInterface annotation
```

### A lambda has no type of its own

This is the point most people never get told. A lambda expression's type is
**whatever functional interface it's assigned to**. The same text can be many things:

```
Runnable r = () -> System.out.println("hi");     // a Runnable
Object o = () -> System.out.println("hi");       // error: Object is not a functional interface
```

The second is the real compiler message from your JDK: *"incompatible types: Object
is not a functional interface."* `Object` has no single abstract method, so there's
nothing for the lambda to be.

### What the compiler produces

An anonymous class is compiled to a **real class file**. A lambda is not. Prove it —
this program has one of each:

```java
import java.util.Comparator;

public class WhatIsALambda {
    public static void main(String[] args) {
        Runnable anon = new Runnable() {
            @Override public void run() { System.out.println("hello from the anonymous class"); }
        };
        Runnable lambda = () -> System.out.println("hello from the lambda");

        anon.run();
        lambda.run();

        System.out.println();
        System.out.println("anon   is a Runnable: " + (anon instanceof Runnable));
        System.out.println("lambda is a Runnable: " + (lambda instanceof Runnable));
        System.out.println("anon   class name:    " + anon.getClass().getName());
        System.out.println("lambda class name has 'Lambda' in it: " + lambda.getClass().getName().contains("Lambda"));
        System.out.println("lambda implements:    " + lambda.getClass().getInterfaces()[0].getName());

        Comparator<String> byLength = (x, y) -> Integer.compare(x.length(), y.length());
        System.out.println("byLength implements:  " + byLength.getClass().getInterfaces()[0].getName());
        System.out.println("compare(\"aa\", \"b\") = " + byLength.compare("aa", "b"));
    }
}
```

```
hello from the anonymous class
hello from the lambda

anon   is a Runnable: true
lambda is a Runnable: true
anon   class name:    WhatIsALambda$1
lambda class name has 'Lambda' in it: true
lambda implements:    java.lang.Runnable
byLength implements:  java.util.Comparator
compare("aa", "b") = 1
```

Both are genuinely `Runnable`s — a lambda really is an object implementing the
interface. But look at the files `javac` wrote for that source (I ran it):

```
WhatIsALambda$1.class     ← the anonymous class got a file
WhatIsALambda.class       ← and nothing at all for the lambda
```

The lambda's body is compiled into a **private method inside the enclosing class**
(named like `lambda$main$0`). At the point where the lambda is written, the
bytecode contains an **`invokedynamic`** instruction: at the first run, the JVM asks
a factory (`LambdaMetafactory`) to generate an object implementing `Runnable` whose
`run()` calls that private method. In `javap -c` it looks like:

```
invokedynamic #22, 0   // InvokeDynamic #1:run:(LThisDemo;)Ljava/lang/Runnable;
```

Why bother? Anonymous classes cost a class file each (a big app had thousands), a
class load at runtime, and tie the implementation to "a class" forever. With
`invokedynamic` the JVM can pick the best strategy and optimise it later. You'll see
the visible consequences in two places: **stack traces** contain frames named
`lambda$show$0`, and `getClass().getName()` shows a generated `…$$Lambda/0x…` name
(the digits vary run to run).

> **Say this in an interview:** "A lambda is an implementation of a functional
> interface — an interface with exactly one abstract method. It has no type of its own;
> the target type decides what it is. It's not compiled to an anonymous class: the body
> becomes a private method, and an `invokedynamic` call site creates the instance at
> runtime."

---

## Part 4 — Syntax forms and type inference

**In plain words:** a lambda is `parameters -> body`. Everything optional is
optional because the compiler can work it out. Learn it by writing the same idea at
different levels of ceremony.

```java
public class Syntax {
    @FunctionalInterface interface Action { void run(); }
    @FunctionalInterface interface Shout { String apply(String s); }
    @FunctionalInterface interface IntOp { int apply(int a, int b); }
    @FunctionalInterface interface DoubleOp { double apply(double a, double b); }
    @FunctionalInterface interface Check { boolean test(int n); }

    public static void main(String[] args) {
        // no parameters
        Action hello = () -> System.out.println("no params, expression body");

        // one parameter: parentheses optional
        Shout loud = s -> s.toUpperCase() + "!";
        Shout loud2 = (s) -> s.toUpperCase() + "!";

        // many parameters
        IntOp add = (a, b) -> a + b;

        // explicit parameter types (rarely needed)
        IntOp mul = (int a, int b) -> a * b;

        // block body: braces, and an explicit return for a value
        IntOp bigger = (a, b) -> {
            if (a > b) return a;
            return b;
        };

        // block body for a void method
        Action twoLines = () -> {
            System.out.println("block body, line 1");
            System.out.println("block body, line 2");
        };

        Check isEven = n -> n % 2 == 0;

        hello.run();
        System.out.println(loud.apply("hey") + " " + loud2.apply("ho"));
        System.out.println("add=" + add.apply(2, 3) + " mul=" + mul.apply(2, 3) + " bigger=" + bigger.apply(2, 3));
        twoLines.run();
        System.out.println("isEven(4)=" + isEven.test(4) + " isEven(7)=" + isEven.test(7));

        // Target typing: the SAME text means different things depending on the interface it's assigned to
        IntOp intPlus = (a, b) -> a + b;
        DoubleOp doublePlus = (a, b) -> a + b;
        System.out.println("same lambda text, int version:    " + intPlus.apply(1, 2));
        System.out.println("same lambda text, double version: " + doublePlus.apply(1.5, 2.5));
    }
}
```

```
no params, expression body
HEY! HO!
add=5 mul=6 bigger=3
block body, line 1
block body, line 2
isEven(4)=true isEven(7)=false
same lambda text, int version:    3
same lambda text, double version: 4.0
```

The forms:

| Form | Example | Notes |
|------|---------|-------|
| No parameters | `() -> doSomething()` | Parentheses are required |
| One parameter | `s -> s.trim()` or `(s) -> s.trim()` | Parentheses optional *only* without a type |
| Several | `(a, b) -> a + b` | Always parenthesised |
| Typed | `(int a, int b) -> a * b` | All typed or none; rarely needed |
| **Expression body** | `x -> x * 2` | The expression's value *is* the result. For a `void` method, the expression is just evaluated |
| **Block body** | `x -> { int y = x * 2; return y; }` | Braces. A value-returning method needs an explicit `return`; leaving it out is a compile error |

**Type inference — "target typing":** nowhere above did I write `a` and `b`'s type, yet
they're `int` in one and `double` in the other. The compiler reads the *target*
interface's method (`IntOp.apply(int, int)` or `DoubleOp.apply(double, double)`) and
gives the lambda's parameters those types. The same text `(a, b) -> a + b` is a
different lambda in each place. This is also why:

- **A lambda must have a target.** `var f = () -> 1;` fails with the real message
  *"cannot infer type for local variable f"* — `var` has nothing to learn the type from.
- **Overloaded methods can make a lambda ambiguous**, when two overloads accept
  different functional interfaces with the same shape. When the compiler can't pick,
  it says so; cast the lambda or give it a named variable of the type you mean.

A style guide you can adopt immediately: **keep lambdas short.** One expression is
ideal. If it needs several lines, give it a name — extract a method and use a method
reference (Day 28). A ten-line lambda inside a call is unreadable.

> **Say this in an interview:** "A lambda is `(params) -> body`; the parameter types
> are inferred from the target functional interface's method, so the same lambda text
> can mean different things in different contexts."

---

## Part 5 — Capturing variables: why "effectively final"

**In plain words:** a lambda can *use* the variables around it — `greeting`, a loop
counter, a config value. That's called **capturing**. But there's a hard rule: the
variable must never be changed after it's set. Why? Remember Day 21: a method's local
variables live on its **stack frame**, which is destroyed the moment the method
returns. A lambda, however, can be **kept** — stored in a field, handed to another
thread, run next week by a scheduler. So the lambda can't hold a *reference to the
variable* (the variable will be gone); it holds a **copy of its value**, made when
the lambda was created. If the original variable were allowed to change afterwards,
the lambda's copy and the original would silently disagree. Java prevents the whole
class of confusion by forbidding the change.

```
 makeGreeter("Hello") called              makeGreeter returns
 ┌───────────────────────────┐            ┌─────────────────────────┐
 │ stack frame               │            │ (frame is gone)         │
 │  greeting = "Hello"       │  ─ copy ─► │                         │
 │  suffix   = "!"           │            └─────────────────────────┘
 │  lambda ──────────────────┼──► heap: lambda object holds  "Hello", "!"
 └───────────────────────────┘                        ↑ still alive, can run later
```

"Effectively final" means: you didn't write `final`, but you also never reassign it
(no `x = …`, no `x++`). The compiler checks. Here are the three real errors, from
your JDK:

```
int count = 0;
Runnable r = () -> count++;
// error: local variables referenced from a lambda expression must be final or effectively final

for (int i = 0; i < 3; i++) {
    tasks.add(() -> System.out.println(i));
}
// error: local variables referenced from a lambda expression must be final or effectively final
//   (i changes on every loop iteration, so it is not effectively final)

int x = 1;
Runnable r2 = () -> System.out.println(x);
x = 2;
// error: ... (the later reassignment makes x not effectively final — even though the
//   lambda only reads it!)
```

The third one is the one that surprises people: the error appears on the *lambda's*
line, but the cause is the **later** assignment.

### What a lambda *can* change

The rule is about **local variables**. Everything that lives on the heap is fair game,
because the lambda holds a reference to the *object*, not a copy of a changing
variable:

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

public class Capture {
    @FunctionalInterface interface Greeter { String greet(String name); }

    static int staticCounter = 0;
    int fieldCounter = 0;

    static Greeter makeGreeter(String greeting) {
        String suffix = "!";
        return name -> greeting + ", " + name + suffix;
    }

    void bumpFieldFromLambda() {
        Runnable r = () -> fieldCounter++;
        r.run();
        r.run();
        System.out.println("fieldCounter after two runs: " + fieldCounter);
    }

    public static void main(String[] args) {
        Greeter g = makeGreeter("Hello");
        System.out.println(g.greet("alice") + "   <- makeGreeter() has already returned");

        Runnable bumpStatic = () -> staticCounter++;
        bumpStatic.run();
        bumpStatic.run();
        System.out.println("staticCounter after two runs: " + staticCounter);
        new Capture().bumpFieldFromLambda();

        AtomicInteger counter = new AtomicInteger();
        Runnable bumpAtomic = counter::incrementAndGet;
        bumpAtomic.run();
        bumpAtomic.run();
        bumpAtomic.run();
        System.out.println("AtomicInteger after three runs: " + counter.get());

        int[] box = {0};
        Runnable bumpBox = () -> box[0]++;
        bumpBox.run();
        bumpBox.run();
        System.out.println("int[] box after two runs: " + box[0]);

        List<Runnable> tasks = new ArrayList<>();
        for (int i = 0; i < 3; i++) {
            final int copy = i;
            tasks.add(() -> System.out.println("task sees copy=" + copy));
        }
        tasks.forEach(Runnable::run);
    }
}
```

```
Hello, alice!   <- makeGreeter() has already returned
staticCounter after two runs: 2
fieldCounter after two runs: 2
AtomicInteger after three runs: 3
int[] box after two runs: 2
task sees copy=0
task sees copy=1
task sees copy=2
```

Reading it:

- `makeGreeter` returned long before `g.greet(...)` ran, yet `greeting` and `suffix`
  still work — the lambda carries copies. That's the diagram above.
- **Fields and statics** can be changed freely: they live on the heap (instance) or in
  a class (static), not in the vanishing frame.
- **`AtomicInteger`** and a **one-element array** are the workarounds when you need a
  "mutable local" — the *variable* (`counter`, `box`) never changes, only the heap
  object it points at. It's legal, and it's a smell: if your lambda is accumulating
  state in a box, you're probably writing a loop in disguise (streams in Week 7 fix
  this with `reduce`/`collect`).
- **The loop fix** — copy the changing variable into a fresh `final` one *inside* the
  loop. Each iteration gets its own `copy`, so each lambda captures a different value.

**Thread-safety warning for Phase 5:** the `int[] box` trick is *not safe* if the
lambda runs on another thread while something else reads it (a race condition), which
is why `AtomicInteger` exists. Today's examples are single-threaded.

> **Say this in an interview:** "A lambda captures the *value* of local variables, not
> the variable, because the enclosing method's stack frame may be gone by the time the
> lambda runs. To keep the copy consistent, Java requires captured locals to be
> effectively final. Fields and statics aren't restricted since they live on the heap."

---

## Part 6 — `this`: lambda vs anonymous class

**In plain words:** an anonymous class is a brand new object, so inside it, "this" is
*that new object*. A lambda isn't a new kind of scope — its code is just a method of
the enclosing class, so "this" is *the object you were already inside*. Same keyword,
different meaning.

```java
public class ThisDemo {
    private final String label;

    ThisDemo(String label) { this.label = label; }

    @Override
    public String toString() { return "ThisDemo[" + label + "]"; }

    void show() {
        Runnable anon = new Runnable() {
            @Override public void run() {
                System.out.println("this inside anonymous class: " + this.getClass().getName());
            }
        };

        Runnable lambda = () -> System.out.println("this inside lambda:          " + this);

        anon.run();
        lambda.run();
    }

    public static void main(String[] args) {
        new ThisDemo("outer").show();
    }
}
```

```
this inside anonymous class: ThisDemo$1
this inside lambda:          ThisDemo[outer]
```

Inside the anonymous class, `this` is a `ThisDemo$1` — the anonymous object — so
`this.label` would not even compile (it has no `label`); you'd write
`ThisDemo.this.label`. Inside the lambda, `this` is the `ThisDemo` that called
`show()`, so `this.label` and bare `label` work directly. This is exactly what the
`javap` output showed: the lambda's body is a private instance method
`lambda$show$0()` *of `ThisDemo`*, so naturally `this` is the `ThisDemo`.

Practical consequences:

- A lambda can't refer to **itself** with `this` (it'd be the outer object). That's
  rarely needed.
- A variable declared inside a lambda can't **shadow** one of the enclosing method (an
  anonymous class's can) — another sign that a lambda is not its own scope.
- If you write `this::method` inside a lambda, `this` means the enclosing object —
  nice for Spring beans calling their own methods.

---

## Hands-on

1. **Required — three sorts.** Run `SortThreeWays`. Then sort the same list **by name
   length** with an anonymous `Comparator`, a lambda, and a method reference. Predict
   which of the three forms *needs* `Comparator.comparingInt` and why.
2. **Required — mutate a local.** Write `int total = 0;` followed by
   `list.forEach(x -> total += x);` and read the real error. Then fix it three ways:
   (a) `AtomicInteger`, (b) a one-element `int[]`, (c) a plain `for` loop. Which is the
   most readable? (Probably the loop — it's a legitimate answer.)
3. **Required — `this`.** Run `ThisDemo`. Then change the anonymous class to print
   `ThisDemo.this.label` and the lambda to print `label`. Both should print `outer`.
4. Run `javac -d out WhatIsALambda.java` (PowerShell, in the folder) and list `out`.
   Add a second anonymous class and a second lambda and recount the `.class` files.
5. Write your own `@FunctionalInterface interface Discount { double apply(double
   price); }` and a method `double checkout(double price, Discount d)`. Call it with
   three different lambdas (10% off, flat 5 off, none).
6. Add a second abstract method to `Discount`. Read the compiler message. Then make
   the second method `default` and watch the error go away.
7. Cause the "x = 2 after the lambda" error from Part 5 and notice which line the
   compiler blames.
8. In a stack trace from any Java app you have, find a frame like `lambda$…$0`. What
   does that tell you about where the code was written?

---

## Gotchas

- **A lambda isn't an object you can name the class of.** `lambda.getClass()` is a
  generated class; never compare or depend on it.
- **Lambdas can't throw checked exceptions** unless the interface's method declares
  them. `Runnable.run()` doesn't, so `() -> Files.readString(p)` won't compile. Wrap in
  try/catch inside the lambda. (Day 23's reason to prefer unchecked.)
- **`return` inside a lambda returns from the *lambda*,** not from the surrounding
  method. A common bug when porting a `for` loop body into a `forEach` lambda.
- **Long lambdas hurt.** If it doesn't fit comfortably on one or two lines, extract a
  method.
- **Overload ambiguity.** `executor.submit(() -> { … })` can match both `Runnable` and
  `Callable<T>` — the compiler picks one based on whether the body returns a value
  (Phase 5).
- **Capturing `this` captures the whole object.** A lambda that uses any instance
  field keeps the entire enclosing object alive — an easy memory-leak route (Day 21,
  Part 6) if the lambda is stored in a long-lived place.
- **Serialization of lambdas** exists but is fragile; don't.
- **Effectively final is a compiler check, not a runtime one.** Adding any `x = …`
  anywhere later breaks the capture, even after the lambda.
- **Shadowing:** `(x) -> …` where `x` is already a local in scope is an error — pick a
  different name.

---

## Recap

- Passing behaviour pre-Java-8 meant an anonymous inner class — mostly ceremony around
  one line.
- A **lambda** is a compact implementation of a **functional interface** (exactly one
  abstract method). It has no type of its own; the *target type* decides what it is.
- `@FunctionalInterface` is an optional compiler-checked label. `default`/`static`
  methods and `Object`'s methods don't count toward "one".
- Not an anonymous class underneath: the body becomes a private method and
  `invokedynamic` creates the object; no extra `.class` file.
- Syntax: `() ->`, `x ->`, `(a, b) ->`; expression body (value is the result) vs block
  body (needs `return`). Parameter types are inferred from the target.
- Captured locals must be **effectively final**, because the lambda holds a *copy* and
  the stack frame may be gone. Fields and statics are unrestricted.
- Inside a lambda, `this` is the **enclosing** object; in an anonymous class it's the
  anonymous object.

Re-answer the four questions at the top.

---

## Say it out loud

1. "What problem did lambdas solve, and what was the alternative before?"
2. "What is a functional interface? What does `@FunctionalInterface` do?"
3. "Is a lambda just an anonymous class? What's the difference?"
4. "How does the compiler know the parameter types of a lambda I wrote without any?"
5. "Why must variables captured by a lambda be effectively final?"
6. "What does `this` mean inside a lambda compared with inside an anonymous class?"

---

## Q&A

_Follow-up questions from this lesson are appended here._
