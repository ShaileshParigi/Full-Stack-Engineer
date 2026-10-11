# Day 29 — Default & Static Methods on Interfaces

_Why this matters:_ For three days you've been calling `.andThen(...)`, `.and(...)`,
`.negate()`, `.thenComparing(...)` and `list.forEach(...)` on *interfaces*. Until
Java 8 that was impossible — an interface could only list method signatures. This
one language change is what made streams, lambdas and a better `Map` possible without
breaking twenty years of existing code. It also introduced Java's only real
"multiple inheritance of behaviour", with a famous trap (the diamond problem) that
interviewers love. And in Spring (Phase 6) you'll meet interfaces like
`WebMvcConfigurer` that are *entirely* default methods.

---

## The one-paragraph version

An interface used to be a pure list of promises — "any class that implements me must
have these methods" — with no code in it. That created a trap: if you add one new
method to a popular interface, **every class in the world that implements it stops
working**, because none of them has the new method. Java 8 wanted to add `stream()` to
`Collection`, `forEach` to `Iterable` and `sort` to `List`; doing that the old way
would have broken every library. The fix: a **`default` method** is a method *with a
body* inside the interface, which implementing classes inherit automatically and may
override. A **`static` method** on an interface is a plain utility that belongs to the
interface itself (like `Comparator.comparing`). The price: a class can now inherit the
same-named default from two interfaces — the **diamond problem** — and Java has
three clear rules for who wins, plus an escape hatch (`Interface.super.method()`) when
you must choose yourself.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **abstract method** | A promise: "implementors must write this" | No body; the only kind interfaces had before Java 8 |
| **`default` method** | A method with a ready-made body that implementors get for free | Declared with `default` in an interface; inherited; may be overridden |
| **`static` method (interface)** | A utility that lives on the interface, called by its name | `Comparator.comparing(...)`; not inherited by implementors |
| **`private` method (interface)** | A helper shared by the interface's own defaults | Java 9; invisible outside the interface |
| **interface evolution** | Adding to an interface *after* people already implement it | What defaults make safe |
| **binary compatibility** | Old compiled code keeps working with a newer library | What breaks if a new abstract method is added |
| **`AbstractMethodError`** | The JVM saying "an object doesn't have a method it was promised" | A runtime error; the symptom of broken binary compatibility |
| **diamond problem** | A class inherits the same method from two parents and can't tell which to use | Two interfaces each supplying a default with the same signature |
| **`X.super.m()`** | "Call interface X's version of `m`, specifically" | The syntax for picking between inherited defaults |
| **override** | Replace an inherited method with your own | A class's method replaces the default |
| **template method** | A method that fixes the outline and calls abstract steps | Common good use of a default |
| **abstract class** | A partially-written class you extend (one only) | Can hold fields and constructors; an interface can't |
| **multiple inheritance** | Inheriting from more than one parent | Java: allowed for interfaces, not for classes |

---

## Before you read: four questions

1. Your library's interface `Greeter` has one method and 10,000 projects implement it.
   You add a second abstract method. What happens to those projects?
2. How is `Predicate.and(...)` possible when `Predicate` is an interface, not a class?
3. Class `C implements A, B`, and both `A` and `B` have a default `hello()`. What does
   the compiler say, and how do you fix it?
4. A superclass has `move()` and an interface has a default `move()`. A class extends
   one and implements the other, and overrides nothing. Which `move()` runs?

<details><summary>Answers (try first)</summary>

1. They break — they won't compile, and already-compiled ones fail at runtime with
   `AbstractMethodError`. (Part 1.)
2. `and` is a `default` method with a body, living in the interface. (Part 2.)
3. "Types A and B are incompatible; class C inherits unrelated defaults." Override
   `hello()` in `C` and choose with `A.super.hello()`. (Part 4.)
4. The class's. A class method always beats an interface default. (Part 5.)

</details>

---

## Part 1 — The problem: you can't add to an interface

**In plain words:** an interface is a contract. Thousands of classes have *signed* it.
If you add a new clause ("and you must also have a `farewell` method"), every
existing signature is suddenly incomplete. Authors of libraries therefore treated
interfaces as frozen the day they were published. That was fine for a decade, until
Java's designers wanted to add `stream()`, `forEach` and `removeIf` to the collection
interfaces — which every collection library on earth implements.

Seeing it break is more convincing than being told. This is a tiny "library" (the
`Greeter` interface) and a tiny "app" (a class that implements it), compiled
*separately*, like a real dependency. I ran these steps on your JDK:

```powershell
# Version 1 of the library: Greeter has one method. App is compiled against it.
#   interface Greeter { String greet(String name); }
javac -d out v1/Greeter.java app/EnglishGreeter.java app/Main.java
java -cp out Main
# Hello, alice
```

Now the library author ships **version 2 with a new abstract method**:

```
interface Greeter {
    String greet(String name);
    String farewell(String name);      // NEW, abstract
}
```

Three consequences, all real output:

```
# (a) The app, already compiled, is dropped onto the new library jar. It only ever
#     called greet(), so it still runs:
Hello, alice

# (b) ...but any code that calls the new method on the old class:
Hello, alice
Exception in thread "main" java.lang.AbstractMethodError: Receiver class EnglishGreeter
does not define or inherit an implementation of the resolved method
'abstract java.lang.String farewell(java.lang.String)' of interface Greeter.

# (c) And anyone who recompiles the app gets:
error: EnglishGreeter is not abstract and does not override abstract method
farewell(String) in Greeter
```

So adding one method to a published interface either **breaks the build** (c) or
**time-bombs the runtime** (b) — and (b) is the nasty one, because nothing complains
until the day that code path runs. This is the problem "interface evolution" means.

Now the same change as a **default method**:

```
interface Greeter {
    String greet(String name);
    default String farewell(String name) { return "Goodbye, " + name; }   // NEW, with a body
}
```

```
# The old, unrecompiled EnglishGreeter dropped onto the new library:
Hello, alice
Goodbye, alice
```

`EnglishGreeter` never heard of `farewell`, yet it works: it **inherited** the default.
That's the whole feature. Every JDK addition — `Collection.stream()`,
`Iterable.forEach`, `List.sort`, `Map.getOrDefault`, `Comparator.reversed()` — shipped
this way, which is why Java 7 code still ran on Java 8.

> **Say this in an interview:** "Default methods let you add behaviour to an interface
> without breaking existing implementors. Adding an abstract method breaks source and
> binary compatibility — old classes fail to compile or throw `AbstractMethodError` —
> but a default is inherited automatically. That's how Java 8 added `stream()` and
> `forEach` to the collection interfaces."

---

## Part 2 — `default` methods

**In plain words:** a `default` method is a method written *in the interface*, and
every class that implements the interface gets it for free. A class that wants
different behaviour just writes its own version, exactly like overriding a method of a
parent class. The default is a *fallback*, not a rule.

```java
public class DefaultBasics {
    interface Notifier {
        String channel();                       // abstract: every implementor must supply it

        default String format(String message) { // default: free for everyone, overridable
            return "[" + channel() + "] " + message;
        }

        default void send(String message) {
            System.out.println(format(message));
        }

        static Notifier of(String channel) {    // static: belongs to the interface itself
            return () -> channel;
        }
    }

    static class EmailNotifier implements Notifier {
        @Override public String channel() { return "email"; }
        // inherits both defaults unchanged
    }

    static class SmsNotifier implements Notifier {
        @Override public String channel() { return "sms"; }

        @Override
        public String format(String message) {   // overrides the default
            return message.length() > 10 ? message.substring(0, 10) + "..." : message;
        }
    }

    public static void main(String[] args) {
        Notifier email = new EmailNotifier();
        Notifier sms = new SmsNotifier();
        Notifier slack = Notifier.of("slack");

        email.send("server is on fire");
        sms.send("server is on fire");
        slack.send("server is on fire");
    }
}
```

```
[email] server is on fire
server is ...
[slack] server is on fire
```

This is today's first practice exercise (implement twice, override once). Read it
closely:

- `channel()` is the **one abstract method**, so `Notifier` is still a functional
  interface — that's why `Notifier.of("slack")` can return a lambda: `() -> channel`.
  *Default and static methods don't count* toward "exactly one abstract method"
  (Day 26).
- `EmailNotifier` writes one method and gets `format` and `send` for free.
- `SmsNotifier` overrides `format` — and **the inherited `send` automatically uses the
  override**, because `send` calls `format(...)` and `format` is resolved on the actual
  object. This is the **template method** pattern: the interface fixes the outline
  (`send` = format, then print) and lets implementors replace one step.
- A default can call the abstract methods (`channel()`), because by the time it runs
  there's a real object that implements them.

**Which defaults you've already used** (they all work exactly like the above):

| You wrote | Where it lives |
|-----------|----------------|
| `predicate.and(other)`, `.negate()`, `.or(...)` | default methods of `Predicate` |
| `function.andThen(g)`, `.compose(g)` | defaults of `Function` |
| `comparator.reversed()`, `.thenComparing(...)` | defaults of `Comparator` |
| `list.forEach(...)` | default of `Iterable` |
| `list.removeIf(...)`, `collection.stream()` | defaults of `Collection` |
| `list.sort(cmp)`, `list.replaceAll(...)` | defaults of `List` |
| `map.getOrDefault`, `computeIfAbsent`, `merge`, `forEach` | defaults of `Map` |

> **Say this in an interview:** "A default method is an interface method with a body;
> implementors inherit it and can override it. It's how `Predicate.and`,
> `Comparator.thenComparing` and `Collection.stream` exist on interfaces."

---

## Part 3 — `static` (and `private`) methods in interfaces

**In plain words:** before Java 8, utility methods that went with an interface lived in
a sidekick class — `Collection` had `Collections`, `Path` had `Paths`. A `static` method
on the interface lets the helper live right with it: `Comparator.comparing(...)` rather
than `Comparators.comparing(...)`. It's just a normal static method, called through the
**interface's name**.

Key differences from a `default` method:

| | `default` | `static` |
|---|-----------|----------|
| Belongs to | An *instance* (the implementing object) | The interface itself |
| Called as | `object.method()` | `InterfaceName.method()` |
| Inherited by implementors? | Yes | **No** |
| Can be overridden? | Yes | No |
| Can use `this` / abstract methods? | Yes | No |

The "not inherited" part surprises people. These are the real compiler errors for a
class `Impl implements Util` where `Util` has `static String tool()`:

```
impl.tool();     // error: cannot find symbol
Impl.tool();     // error: cannot find symbol
Util.tool();     // OK
```

Neither the object nor the implementing class has it. It's reachable **only** as
`Util.tool()`. (Deliberate: otherwise two interfaces with the same static name would
clash in a class that implements both.)

Typical uses: **factory methods** (`Notifier.of(...)`, `List.of(...)`, `Map.entry(...)`,
`Comparator.naturalOrder()`), **helpers** that build instances, and `Predicate.not(...)`
and `Predicate.isEqual(...)` you used on Day 27.

### `private` methods (Java 9)

Two defaults often share some code. Rather than copy-paste, or expose a helper to the
world, an interface can hold a **private** method:

```java
public class PrivateHelpers {
    interface Report {
        String title();
        int rows();

        default String summary() {
            return line("summary");
        }

        default String detail() {
            return line("detail") + " (" + rows() + " rows)";
        }

        // Java 9: private methods let defaults share code without exposing it
        private String line(String kind) {
            return kind.toUpperCase() + ": " + title();
        }
    }

    public static void main(String[] args) {
        Report r = new Report() {
            @Override public String title() { return "Q3 sales"; }
            @Override public int rows() { return 42; }
        };
        System.out.println(r.summary());
        System.out.println(r.detail());
    }
}
```

```
SUMMARY: Q3 sales
DETAIL: Q3 sales (42 rows)
```

`line(...)` is invisible to implementors and callers — an implementation detail of
`Report`'s defaults. (`private static` is also allowed.)

---

## Part 4 — The diamond problem

**In plain words:** you inherit a recipe card from your grandmother *and* one from your
other grandmother, both titled "Sunday soup", and they're different. When you cook
"Sunday soup", which do you follow? Guessing would be a bug waiting to happen, so Java
makes **you** decide and **say so in code**.

Java doesn't allow a class to extend two classes — that's partly to avoid this. But a
class can implement *many* interfaces, and now interfaces can carry code, so the
question arrives. When **two unrelated interfaces** give the same default, the class
that implements both *must* resolve it. If it doesn't, the compiler refuses:

```
interface A { default String hello() { return "A"; } }
interface B { default String hello() { return "B"; } }
class C implements A, B { }

// error: types A and B are incompatible;
//   class C inherits unrelated defaults for hello() from types A and B
```

The fix: override in `C`, and — if you want one (or both) of the originals — name it
with **`InterfaceName.super.method()`**:

```java
public class Diamond {
    interface Swimmer {
        default String move() { return "swimming"; }
    }

    interface Flyer {
        default String move() { return "flying"; }
    }

    // Duck inherits move() from BOTH. The compiler refuses to pick, so Duck must decide.
    static class Duck implements Swimmer, Flyer {
        @Override
        public String move() {
            return Swimmer.super.move() + " and " + Flyer.super.move();
        }
    }

    // Resolution rule 1: a CLASS method always beats an interface default
    static class Animal {
        public String move() { return "walking"; }
    }

    static class Dog extends Animal implements Swimmer {
        // no override: Animal.move() wins over Swimmer.move()
    }

    // Resolution rule 2: the MORE SPECIFIC interface wins
    interface Vehicle {
        default String describe() { return "a vehicle"; }
    }

    interface Car extends Vehicle {
        @Override default String describe() { return "a car"; }
    }

    static class Taxi implements Vehicle, Car {
    }

    public static void main(String[] args) {
        System.out.println("Duck:  " + new Duck().move());
        System.out.println("Dog:   " + new Dog().move() + "   (class wins)");
        System.out.println("Taxi:  " + new Taxi().describe() + "      (more specific interface wins, even though Vehicle is listed too)");
    }
}
```

```
Duck:  swimming and flying
Dog:   walking   (class wins)
Taxi:  a car      (more specific interface wins, even though Vehicle is listed too)
```

That's today's second practice item — the deliberate diamond, resolved.

### The three resolution rules

When a class inherits a method with the same signature from several places, Java
applies these **in order** (the first that gives an answer wins):

| # | Rule | Example in the demo |
|---|------|--------------------|
| 1 | **A class always wins.** A method declared in the class, or inherited from a *superclass*, beats any interface default | `Dog`: `Animal.move()` over `Swimmer.move()` |
| 2 | **The most specific interface wins.** If one interface extends the other, its default overrides the parent's | `Taxi`: `Car.describe()` over `Vehicle.describe()` |
| 3 | **Otherwise, you must decide.** Override and pick with `X.super.m()` (or write your own) | `Duck` |

Rule 1 is the reassuring one: **adding a default to an interface can never silently
change what an existing class does if the class (or its superclass) already defines
that method** — which is the point of defaults being a *safe* addition.

Two edge cases, so they don't surprise you:

- You can re-declare a method as **abstract** in a sub-interface to *cancel* an
  inherited default and force implementors to supply their own.
- A default method **cannot override `equals`, `hashCode` or `toString`**. The compiler
  says: *"default method equals in interface WithEquals overrides a member of
  java.lang.Object"*. A class always inherits `Object`'s, and rule 1 means those would
  always win anyway, so allowing it would be pointless.

> **Say this in an interview:** "If two interfaces provide the same default, the
> implementing class must override it and can choose with `A.super.m()`. Resolution
> prefers a class method over any interface default, then the more specific interface,
> and otherwise it's a compile error."

---

## Part 5 — What interfaces still can't do

Defaults give interfaces *behaviour*, not *state*:

| Can an interface have… | |
|------------------------|--|
| Abstract methods | Yes (the original purpose) |
| `default` / `static` / `private` methods | Yes |
| **Instance fields** (per-object state) | **No** — only constants, which are implicitly `public static final` |
| **Constructors** | No |
| `protected` members | No |
| Overriding `equals`/`hashCode`/`toString` as defaults | No (Part 4) |

The "constants" point catches people who try to put a counter in an interface:

```
interface WithField { int counter = 0; }
WithField.counter = 5;
// error: cannot assign a value to static final variable counter
```

Because there's no per-object state, a default method can only work through the
interface's other methods (`channel()`, `title()`…) — it *asks the implementor* for
whatever data it needs. That's a good discipline, not just a limitation.

### Abstract class or interface — now that interfaces have behaviour?

| | Interface | Abstract class |
|---|-----------|----------------|
| Per-object state (fields) | No | **Yes** |
| Constructors | No | **Yes** |
| A class can have | **Many** | **One** (single inheritance) |
| Method bodies | `default`/`static`/`private` only | Any kind, any visibility |
| Models | "can do X" — a capability or role | "is a kind of X" — a shared base |
| Adding methods later | Safe (via `default`) | Safe |
| Lambdas | **Yes** (if functional) | No |

Rules of thumb:

- **Default to an interface.** It lets an implementor extend another class, and it
  works with lambdas.
- Use an **abstract class** when the subtypes **share state** or need a **constructor**
  — a `BaseEntity` with `id` and `createdAt` fields, say.
- Combine them: interface for the contract, an `AbstractX` skeleton class with the
  common state and boilerplate (the JDK does this: `List` + `AbstractList`).

---

## Part 6 — Reading the JDK: every default and static method

Today's last practice item says: *read `Comparator`'s source and identify every
default and static method*. You can read the source in VS Code (Ctrl-click
`Comparator`) — or ask the JVM. This program lists them for five interfaces you've
used:

```java
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.util.Arrays;
import java.util.Collection;
import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.function.Predicate;

public class JdkDefaults {
    static void inspect(Class<?> type) {
        List<String> abstractOnes = Arrays.stream(type.getDeclaredMethods())
                .filter(m -> Modifier.isAbstract(m.getModifiers()))
                .map(Method::getName).distinct().sorted().toList();
        List<String> defaults = Arrays.stream(type.getDeclaredMethods())
                .filter(Method::isDefault)
                .map(Method::getName).distinct().sorted().toList();
        List<String> statics = Arrays.stream(type.getDeclaredMethods())
                .filter(m -> Modifier.isStatic(m.getModifiers()) && Modifier.isPublic(m.getModifiers()))
                .map(Method::getName).distinct().sorted().toList();
        System.out.println(type.getSimpleName());
        System.out.println("  abstract: " + abstractOnes);
        System.out.println("  default:  " + defaults);
        System.out.println("  static:   " + statics);
    }

    public static void main(String[] args) {
        inspect(Comparator.class);
        inspect(Predicate.class);
        inspect(Iterable.class);
        inspect(Collection.class);
        inspect(Map.class);
    }
}
```

```
Comparator
  abstract: [compare, equals]
  default:  [reversed, thenComparing, thenComparingDouble, thenComparingInt, thenComparingLong]
  static:   [comparing, comparingDouble, comparingInt, comparingLong, naturalOrder, nullsFirst, nullsLast, reverseOrder]
Predicate
  abstract: [test]
  default:  [and, negate, or]
  static:   [isEqual, not]
Iterable
  abstract: [iterator]
  default:  [forEach, spliterator]
  static:   []
Collection
  abstract: [add, addAll, clear, contains, containsAll, equals, hashCode, isEmpty, iterator, remove, removeAll, retainAll, size, toArray]
  default:  [parallelStream, removeIf, spliterator, stream, toArray]
  static:   []
Map
  abstract: [clear, containsKey, containsValue, entrySet, equals, get, hashCode, isEmpty, keySet, put, putAll, remove, size, values]
  default:  [compute, computeIfAbsent, computeIfPresent, forEach, getOrDefault, merge, putIfAbsent, remove, replace, replaceAll]
  static:   [copyOf, entry, of, ofEntries]
```

(That uses streams — Day 34 — to tidy the lists; ignore the mechanics, read the output.)

What it tells you:

- **`Comparator` is a functional interface** even though it lists `compare` *and*
  `equals` as abstract: `equals` is `Object`'s, so it doesn't count (Day 26). One real
  abstract method (`compare`), **5 defaults**, **8 statics** — the entire "Comparator
  toolkit" is defaults and statics around a single method. `comparing`, `naturalOrder`
  and `nullsFirst` are **factories**; `reversed` and `thenComparing` are **combinators**
  that return new comparators.
- **`Predicate`**: one abstract (`test`), three combinators, two factories — exactly the
  shape of every functional interface in `java.util.function`.
- **`Iterable.forEach`** (a default) is why *every* `List`, `Set` and `Queue` has
  `forEach` — implementors only wrote `iterator()`.
- **`Collection.stream()` and `removeIf`** are defaults: this is the change that
  motivated the whole feature.
- **`Map`**: ten defaults added in Java 8 — `getOrDefault`, `computeIfAbsent`,
  `merge`… — which Day 32 covers properly. Without defaults every `Map` implementation
  on earth would have had to be rewritten.
- The `Collection` list also shows `equals`/`hashCode` listed as abstract (they're
  redeclared there for documentation) — again, `Object`'s methods.

---

## Part 7 — Good design and code smells

**In plain words:** a default method is a tool for two specific jobs. Used for those,
it's excellent. Used as a way to get "mini multiple inheritance of code", it gets
messy fast.

**Good uses**

- **Evolving a published interface** — the original reason (Part 1).
- **Convenience built from the abstract core** — `Predicate.and` is just `test(a) &&
  test(b)`: derived behaviour that every implementor would otherwise copy.
- **Optional behaviour with a sane no-op default** — `HandlerInterceptor`'s three
  methods and, in Spring 5+, **`WebMvcConfigurer`** (every method is a default no-op;
  you override only what you need). It replaced the old `WebMvcConfigurerAdapter` base
  class, which burnt your one allowed superclass. You'll write one in Phase 6.
- **Template methods** — `Notifier.send` calling `format`.
- **Combinators** that return new instances — `Validator.and` from Day 28.

**Smells**

| Smell | Why it hurts |
|-------|--------------|
| **A default that depends on hidden assumptions about the implementor** | It can't see fields; if it works only for implementors that behave a certain way, it's a trap |
| **Big, stateful-feeling logic in an interface** | That's a class. Use an abstract class or a helper object |
| **Many interfaces with overlapping defaults** | You'll hit diamonds and rule 3 constantly |
| **A default that changes meaning of existing implementors** | Silent behaviour changes in a library update |
| **Defaults used to dodge writing a proper class hierarchy** | Hard to find where behaviour lives |
| **Calling `Interface.super.m()` everywhere** | Sign the abstractions are tangled |

A good test: **would a reasonable implementor ever want to override this?** If "no, ever"
and it doesn't depend on the implementor, make it a `static` helper. If "yes,
sometimes", a `default` is right. If "they must", leave it abstract.

---

## Hands-on

1. **Required — default, twice, override once.** Run `DefaultBasics`. Add a third
   implementor, `PushNotifier`, that overrides `send` (not `format`) to print a
   bell character before the message. Which methods does it still inherit?
2. **Required — the deliberate diamond.** In `Diamond`, delete the `Duck` override and
   read the real compiler error. Restore it with `Swimmer.super.move()` only. Then add
   a third interface `Walker` with its own default `move()` and make `Duck` implement
   all three — note you must now name three.
3. **Required — read `Comparator`.** Run `JdkDefaults`. Then open
   `java.util.Comparator` in VS Code and, for each of the 5 defaults and 8 statics, write
   one line saying whether it's a *factory* or a *combinator*.
4. Reproduce Part 1 yourself: build the v1 / v2 / v3 `Greeter` experiment in a scratch
   folder (PowerShell commands shown above) and get the `AbstractMethodError` with your
   own eyes.
5. Add `default Validator<T> or(Validator<T> other)` to Day 28's `Validator<T>`.
6. Prove rule 1: make a class that extends a class with `describe()` and implements an
   interface whose default `describe()` is different. Which one runs? Add an override and
   call `Iface.super.describe()` from it.
7. Try to give an interface a non-final field and a `default boolean equals(Object)`.
   Read both errors.
8. Find `WebMvcConfigurer` in the Spring docs (or your IDE once Phase 6 starts) and count
   its default methods. Why are *all* of them defaults?

---

## Gotchas

- **An implementing class doesn't inherit `static` interface methods.** Always
  `InterfaceName.method()`.
- **Defaults can be silently overridden** by a class method of the same signature — rule
  1. If a default "isn't being used", check the class hierarchy.
- **Diamonds appear when you upgrade a dependency.** A library adds `default
  describe()` to its interface and your class already implements another interface with
  the same default → your code stops compiling after a version bump. This is a real
  upgrade hazard.
- **`default` is not an access modifier.** A default method is implicitly `public`.
  Implementors must use `public` when overriding (or the compiler complains about
  weaker access).
- **Don't confuse the `default` keyword** with the `default:` branch of a `switch`.
- **Interface fields are constants** (`public static final`) whether or not you write
  it; "constants interfaces" (an interface just holding constants) are an anti-pattern.
- **Lambdas and defaults:** a lambda implementing a functional interface still gets
  all its defaults (`predicate.and(...)` works on a lambda).
- **You can't `@Override` a `static`** — statics aren't overridden.
- **Mocking:** Mockito mocks of an interface *don't run* the default methods unless you
  tell them to (`CALLS_REAL_METHODS`) — it matters on Day 39.
- **`default` methods and `synchronized`:** a default can't be `synchronized`; an
  interface has no instance state to lock on, which fits the "no state" theme.

---

## Recap

- Adding an abstract method to a published interface breaks implementors (compile error
  or runtime `AbstractMethodError`). A **`default` method** is inherited instead — the
  mechanism that let Java 8 add `stream()`, `forEach`, `sort`, `Map.merge`.
- `default` = per-instance, inherited, overridable. `static` = on the interface, called
  `Interface.m()`, **not** inherited. `private` (Java 9) = shared helper for defaults.
- Same default from two unrelated interfaces → compile error; resolve by overriding and
  calling `X.super.m()`.
- Resolution order: **class wins → most specific interface wins → you decide.**
- Interfaces have no instance state, constructors or overriding of `Object`'s methods.
- Interface for a *capability*, abstract class for shared *state*. Default to the
  interface.
- Good defaults: evolution, derived convenience, optional behaviour, template methods,
  combinators. Smell: logic that really wants state or a class.

Re-answer the four questions at the top.

---

## Say it out loud

1. "What problem did default methods solve in Java 8?"
2. "What's the difference between a `default` and a `static` method on an interface?"
3. "What is the diamond problem, and how does Java resolve it?"
4. "What are the resolution rules when a class inherits the same method from several
   places?"
5. "When would you use an abstract class rather than an interface?"
6. "Why can `Predicate` have `and()` and `negate()` when it's an interface?"

---

## Q&A

_Follow-up questions from this lesson are appended here._
