# Day 24 — Generics & Type Erasure

_Why this matters:_ Every collection, every `Optional`, every Spring repository
(`JpaRepository<User, Long>`), every `CompletableFuture<T>` and every stream you'll
write is generic. You've been *using* generics since your first `List<String>`;
today you learn what they are, what the compiler really does with them, and why a
few things Java "should" let you do (`new T[]`, `instanceof List<String>`, passing
a `List<String>` where a `List<Object>` is wanted) are flatly refused. The
wildcard rules (`? extends`, `? super`) are the part people memorise and don't
understand. By the end you'll be able to derive them.

---

## The one-paragraph version

Before generics, a `List` held `Object`s, so you had to cast on the way out, and
nothing stopped you putting a number in a list of names — you found out at
runtime, far from the mistake, with a `ClassCastException`. **Generics** let you
write `List<String>` so the *compiler* checks, before the program ever runs, that
only strings go in and that what comes out is already a string. A class or method
becomes generic by declaring a **type parameter** (`T`, a placeholder for "some
type to be chosen later"). To keep old code working, the designers implemented
generics by **type erasure**: after compilation the type arguments are *gone* —
`List<String>` and `List<Integer>` are both just `List` at runtime, and the
compiler quietly inserts the casts. That one decision explains almost every
restriction: you can't create `new T[]`, can't ask `instanceof List<String>`, and
a `List<String>` is **not** a `List<Object>`. Wildcards (`? extends X`, `? super X`)
are how you express "any list of something related to X" while keeping the compiler
honest. The mnemonic is **PECS**: *Producer Extends, Consumer Super*.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **generic type** | A class or interface with a blank for a type, filled in by the user | `class Box<T>`, `List<E>` |
| **type parameter** | The blank itself | The `T` in `Box<T>` — a name standing for a type |
| **type argument** | What you fill the blank with | The `String` in `Box<String>` |
| **generic method** | A method with its own blank, independent of its class | `static <T> T firstOf(List<T> l)` |
| **diamond `<>`** | "You can work out the type argument yourself" | `new ArrayList<>()` — inferred from the left side |
| **bounded type parameter** | A blank that only accepts certain types | `<T extends Comparable<T>>` — "T must be comparable to itself" |
| **type safety** | The compiler guarantees you can't mix up types | Violations are compile errors, not runtime ones |
| **type erasure** | The compiler removes the type arguments after checking | In bytecode `List<String>` is `List`; `T` becomes its bound (or `Object`) |
| **reifiable type** | A type whose full information exists at runtime | `String`, `List` (raw), `int[]`; **not** `List<String>` or `T` |
| **raw type** | A generic type used with no argument at all | `List` instead of `List<String>`: opts out of the checking |
| **unchecked warning** | The compiler saying "I can't prove this is safe" | `-Xlint:unchecked`; usually a real hazard |
| **heap pollution** | A variable whose type claims one thing but holds another | A `List<String>` that contains an `Integer` |
| **invariant** | `List<String>` and `List<Object>` are unrelated | Generics aren't covariant, even if `String` is an `Object` |
| **covariant** | If `Dog` is an `Animal`, then `Dog[]` is an `Animal[]` | Arrays are (a design mistake — see Part 5) |
| **wildcard** | "Some unknown type" | `?`, `? extends X`, `? super X` |
| **PECS** | Producer Extends, Consumer Super | When a collection *gives you* items use `extends`; when it *receives* items use `super` |

---

## Before you read: four questions

1. You get a `List` (no `<…>`) from some old code and write
   `String s = (String) list.get(2);`. What can go wrong, and *when* will you find out?
2. Does `new ArrayList<String>().getClass() == new ArrayList<Integer>().getClass()`?
3. A method takes `List<Object>`. You hold a `List<String>`. Can you pass it?
4. A method needs to *read* numbers from a list and add them up. It needs to
   accept `List<Integer>` and `List<Double>`. What parameter type do you write?

<details><summary>Answers (try first)</summary>

1. The element may not be a `String` → `ClassCastException` at runtime, at that line,
   not where the wrong item was added. Part 1.
2. `true` — erasure. Part 4.
3. No — `List<String>` isn't a `List<Object>`. Part 5.
4. `List<? extends Number>`. Part 6.

</details>

---

## Part 1 — Why generics exist: the bug they remove

**In plain words:** imagine a shared storage cupboard with no labels. Anyone can put
anything in, and when you take something out you have to *assume* what it is. Put a
label on the cupboard ("shoes only") and the person who tries to put a saucepan in
is stopped at the door — not three weeks later when someone reaches in for shoes
and grabs a saucepan.

Here is life before Java 5. `List` held `Object`; you cast on the way out:

```java
import java.util.ArrayList;
import java.util.List;

public class BeforeGenerics {
    @SuppressWarnings({"rawtypes", "unchecked"})
    static List loadNames() {
        List names = new ArrayList();
        names.add("alice");
        names.add("bob");
        names.add(42);
        return names;
    }

    @SuppressWarnings("rawtypes")
    public static void main(String[] args) {
        List names = loadNames();
        System.out.println("loaded " + names.size() + " names, no complaints so far");

        try {
            for (Object o : names) {
                String name = (String) o;
                System.out.println(name.toUpperCase());
            }
        } catch (ClassCastException e) {
            System.out.println("ClassCastException: " + e.getMessage());
        }
    }
}
```

```
loaded 3 names, no complaints so far
ALICE
BOB
ClassCastException: class java.lang.Integer cannot be cast to class java.lang.String (...)
```

The `42` was added in `loadNames()`. The crash happens in a completely different
method, after two successful iterations — possibly in a different class, on a
different day, in production. The stack trace points at the **cast**, not at the
code that put the bad value in.

With a type argument the same mistake can't be written:

```
List<String> names = new ArrayList<>();
names.add(42);          // compile error: incompatible types: int cannot be converted to String
String s = names.get(0); // no cast needed
```

Two gains: **the error moves from runtime to compile time**, and **the casts
disappear**. That's the whole point of generics. Everything else today is detail
about how.

> **Say this in an interview:** "Generics give compile-time type safety — the
> compiler rejects putting the wrong type into a collection and removes the need for
> casts — so a whole class of `ClassCastException`s becomes a compile error."

---

## Part 2 — Generic classes, methods and bounds

**In plain words:** a generic class is a template with a blank for the type. `Box<T>`
says "a box that holds *one thing of some type T*". You choose `T` when you use it:
`Box<String>` is a box of strings. The compiler then makes sure every use is
consistent: what you put in is a `T`, what you take out is a `T`.

```java
import java.util.ArrayList;
import java.util.List;

public class Basics {
    static class Box<T> {
        private T value;
        Box(T value) { this.value = value; }
        T get() { return value; }
        void set(T value) { this.value = value; }
    }

    static class Pair<A, B> {
        final A first;
        final B second;
        Pair(A first, B second) { this.first = first; this.second = second; }
        @Override public String toString() { return "(" + first + ", " + second + ")"; }
    }

    static <T> T firstOf(List<T> list) {
        return list.get(0);
    }

    static <T extends Comparable<T>> T max(List<T> list) {
        T best = list.get(0);
        for (T item : list) {
            if (item.compareTo(best) > 0) best = item;
        }
        return best;
    }

    public static void main(String[] args) {
        Box<String> name = new Box<>("alice");
        String s = name.get();
        System.out.println("Box<String> holds " + s);

        Pair<String, Integer> p = new Pair<>("age", 30);
        System.out.println("Pair: " + p);

        List<String> names = new ArrayList<>(List.of("bob", "alice", "carol"));
        System.out.println("firstOf: " + firstOf(names));
        System.out.println("max of strings: " + max(names));
        System.out.println("max of ints:    " + max(List.of(3, 9, 4)));
    }
}
```

```
Box<String> holds alice
Pair: (age, 30)
firstOf: bob
max of strings: carol
max of ints:    9
```

Walk through each piece:

- **`class Box<T>`** — declares a type parameter. Inside the class, `T` is used like
  any type. Common letter conventions: `T` (type), `E` (element), `K`/`V` (key/value),
  `R` (result), `ID`. They're just names.
- **`new Box<>("alice")`** — the **diamond** `<>` tells the compiler to infer
  `String` from the left side.
- **`static <T> T firstOf(List<T> list)`** — a **generic method**: the `<T>` before
  the return type declares a blank belonging to *the method*. The compiler infers `T`
  at each call from the argument (here `String`).
- **`<T extends Comparable<T>>`** — a **bound**. Without it, `max` couldn't call
  `compareTo`, because `T` could be anything. The bound says "T must be a type that
  can compare itself to another T" — `String` and `Integer` qualify; a class with no
  ordering doesn't, and the compiler rejects `max(listOfThatClass)`. Notice `extends`
  here means "is a subtype of" even for interfaces. Multiple bounds: `<T extends
  Number & Comparable<T>>`.

### Your own: a bounded `Repository<T, ID>` (today's practice)

Spring Data's `JpaRepository<T, ID>` is exactly this shape. Build the miniature
version:

```java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;

public class RepoDemo {
    interface Entity<ID> {
        ID getId();
    }

    interface Repository<T extends Entity<ID>, ID> {
        void save(T entity);
        Optional<T> findById(ID id);
        List<T> findAll();
    }

    static class InMemoryRepository<T extends Entity<ID>, ID> implements Repository<T, ID> {
        private final Map<ID, T> store = new HashMap<>();

        @Override public void save(T entity) { store.put(entity.getId(), entity); }
        @Override public Optional<T> findById(ID id) { return Optional.ofNullable(store.get(id)); }
        @Override public List<T> findAll() { return new ArrayList<>(store.values()); }
    }

    record User(Long id, String name) implements Entity<Long> {
        @Override public Long getId() { return id; }
    }

    record Product(String sku, double price) implements Entity<String> {
        @Override public String getId() { return sku; }
    }

    public static void main(String[] args) {
        Repository<User, Long> users = new InMemoryRepository<>();
        users.save(new User(1L, "alice"));
        users.save(new User(2L, "bob"));

        Repository<Product, String> products = new InMemoryRepository<>();
        products.save(new Product("A-100", 9.99));

        System.out.println("user 1:    " + users.findById(1L).map(User::name).orElse("none"));
        System.out.println("user 9:    " + users.findById(9L).map(User::name).orElse("none"));
        System.out.println("product:   " + products.findById("A-100").map(Product::price).orElse(-1.0));
        System.out.println("all users: " + users.findAll().size());
        // users.save(new Product("X", 1.0));   // would not compile: a Product is not a User
    }
}
```

```
user 1:    alice
user 9:    none
product:   9.99
all users: 2
```

What the bound `T extends Entity<ID>` buys: inside `InMemoryRepository`, the
compiler knows every `T` has `getId()` returning an `ID`, so
`store.put(entity.getId(), entity)` compiles. Without the bound `T` would be a plain
`Object` with no `getId()`. And the two parameters are **linked**: a
`Repository<User, Long>` only accepts a `User` whose ID type is `Long`;
`Repository<User, String>` wouldn't compile because `User` isn't an `Entity<String>`.
(The `record` syntax and `Optional.map` are Days 25 and 36 — read them as "a compact
class" and "transform the value if present".)

> **Say this in an interview:** "A type parameter is a placeholder the compiler
> resolves per use; a bound like `T extends Comparable<T>` restricts what's allowed
> and lets me call the bound's methods on `T`. Spring Data's `Repository<T, ID>` is a
> generic interface in exactly this way."

---

## Part 3 — Type erasure: what the JVM actually sees

**In plain words:** generics are a **compile-time-only** feature. The compiler uses
the type arguments to check your code, then *erases* them — like a teacher who
checks your workings and then hands back only the final answers. By the time the
program runs, `List<String>` and `List<Integer>` have both become plain `List`, and
the labels are gone. Where you took something out and the code needs it as a
`String`, the compiler has silently inserted a cast. Why? When generics arrived in
Java 5 there were millions of lines of pre-generic code and compiled libraries;
erasure meant old and new code could run together. The price is that generic
information is not available at runtime.

Don't take my word for it — ask the runtime:

```java
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;

public class Erasure {
    static class Box<T> {
        T value;
        T get() { return value; }
        void set(T v) { value = v; }
    }

    static class BoundedBox<T extends Number> {
        T value;
        T get() { return value; }
    }

    public static void main(String[] args) throws Exception {
        List<String> strings = new ArrayList<>();
        List<Integer> ints = new ArrayList<>();
        System.out.println("same runtime class: " + (strings.getClass() == ints.getClass()));
        System.out.println("class name:         " + strings.getClass().getName());

        Method get = Box.class.getDeclaredMethod("get");
        System.out.println("Box<T>.get() returns at runtime:            " + get.getReturnType().getSimpleName());
        Method set = Box.class.getDeclaredMethod("set", Object.class);
        System.out.println("Box<T>.set parameter at runtime:            " + set.getParameterTypes()[0].getSimpleName());
        Method bget = BoundedBox.class.getDeclaredMethod("get");
        System.out.println("BoundedBox<T extends Number>.get() returns: " + bget.getReturnType().getSimpleName());
    }
}
```

```
same runtime class: true
class name:         java.util.ArrayList
Box<T>.get() returns at runtime:            Object
Box<T>.set parameter at runtime:            Object
BoundedBox<T extends Number>.get() returns: Number
```

The erasure rules:

| In source | In bytecode |
|-----------|-------------|
| `List<String>`, `List<Integer>` | `List` |
| `T` | `Object` |
| `T extends Number` | `Number` (the leftmost bound) |
| `String s = list.get(0)` | `String s = (String) list.get(0)` — a cast the compiler inserted |
| `Box<String>` | `Box` |

Your source code never shows the casts, but they're there. This is why the next
section's trick works.

### Heap pollution: a cast in the wrong place

Since the checking is done at compile time only, anything that sidesteps the
compiler — a raw type, an unchecked cast — can put a wrong object into a typed
list, and the JVM can't object, because at runtime it's just a `List`:

```java
import java.util.ArrayList;
import java.util.List;

public class HeapPollution {
    @SuppressWarnings({"rawtypes", "unchecked"})
    static void sneak(List raw) {
        raw.add(42);
    }

    public static void main(String[] args) {
        List<String> names = new ArrayList<>();
        names.add("alice");

        sneak(names);
        System.out.println("size is " + names.size() + " - the add succeeded, nothing complained");

        try {
            for (String n : names) {
                System.out.println(n.length());
            }
        } catch (ClassCastException e) {
            System.out.println("ClassCastException far from the cause: " + e.getMessage());
        }
    }
}
```

```
size is 2 - the add succeeded, nothing complained
5
ClassCastException far from the cause: class java.lang.Integer cannot be cast to class java.lang.String (...)
```

That's the pre-generics bug, *back again*, via a back door. The `ClassCastException`
fires on `n.length()` — on the cast the compiler inserted when reading from
`names`. This is **heap pollution**: a `List<String>` that contains an `Integer`.
The wrong place for the exception, a long way from the cause. You can't *prevent* it
without discipline: don't use raw types, and don't ignore unchecked warnings.

### What erasure forbids

Real compiler messages from your JDK 21:

| You write | The compiler says | Why |
|-----------|-------------------|-----|
| `new T[3]` | `error: generic array creation` | At runtime `T` is `Object`; the JVM wouldn't know what array type to create |
| `x instanceof List<String>` | `error: Object cannot be safely cast to List<String>` | At runtime there's no `<String>` to test. `x instanceof List<?>` is fine |
| `new T()` | cannot instantiate | `T` has no runtime identity |
| `T.class` | cannot select from a type variable | same |
| `static T field` | non-static type variable cannot be referenced from a static context | `T` belongs to an instance |
| `catch (MyEx<T> e)` | generic class may not extend Throwable | exceptions are matched at runtime |
| overloading `m(List<String>)` and `m(List<Integer>)` | name clash: same erasure | both erase to `m(List)` |

Workarounds you'll actually see: pass a `Class<T>` token (`Class<User> type`) and call
`type.cast(x)` or `type.getDeclaredConstructor().newInstance()`; or create an
`Object[]` and cast it with `@SuppressWarnings("unchecked")` (what `ArrayList` does
internally); or use a `List<T>` instead of an array. This is also why frameworks that
need a generic type at runtime — Jackson's `TypeReference<List<User>>`, Spring's
`ParameterizedTypeReference` — make you subclass something: an *anonymous subclass
records its generic superclass* in the class file, and reflection can recover it.
You'll meet that in Phase 6.

> **Say this in an interview:** "Java implements generics by type erasure: type
> arguments exist only at compile time, so `List<String>` and `List<Integer>` are the
> same class at runtime and the compiler inserts casts. That's why you can't do
> `new T[]`, `instanceof List<String>` or overload on type arguments, and why raw
> types can pollute a typed collection."

---

## Part 4 — `List<String>` is not a `List<Object>`

**In plain words:** it sounds obvious — a string *is* an object, so a list of strings
*is* a list of objects, right? No, and the reason is the whole motive for wildcards.
If Java let you treat your list of strings as a list of objects, you could then put
an `Integer` into it through the "list of objects" view — and now your "list of
strings" holds an integer. The rule that stops this: generics are **invariant**.
`Box<Dog>` is not a `Box<Animal>`, even though `Dog` is an `Animal`.

```
List<String>  s = new ArrayList<>();
List<Object>  o = s;     // error: incompatible types: List<String> cannot be converted to List<Object>
```

If that compiled:

```
o.add(42);               // legal: o is a List<Object>
String first = s.get(0); // ClassCastException: s is "a List<String>" yet holds 42
```

### Arrays made the opposite choice — and it was a mistake

Arrays *are* covariant: `String[]` **is** an `Object[]`. The consequence is exactly
the hazard above, caught only at runtime:

```java
import java.util.ArrayList;
import java.util.List;

public class ArrayCovariance {
    public static void main(String[] args) {
        String[] strings = new String[2];
        Object[] objects = strings;
        try {
            objects[0] = 42;
        } catch (ArrayStoreException e) {
            System.out.println("arrays: caught at RUNTIME -> ArrayStoreException: " + e.getMessage());
        }

        List<String> list = new ArrayList<>();
        // List<Object> objs = list;      // does not compile: incompatible types
        System.out.println("generics: the same mistake is rejected at COMPILE time");
    }
}
```

```
arrays: caught at RUNTIME -> ArrayStoreException: java.lang.Integer
generics: the same mistake is rejected at COMPILE time
```

Arrays predate generics, so they keep a runtime type tag and check each store.
Generics, designed later and subject to erasure, pushed the check to compile time.
This is the deeper reason `new T[]` is illegal: the two systems disagree about when
checking happens. **Prefer `List<T>` over `T[]`.**

---

## Part 5 — Wildcards and PECS

**In plain words:** invariance is safe but too strict. A method that only needs to
*read* numbers out of a list and add them up doesn't care whether it was given a
`List<Integer>` or a `List<Double>` — but both are unrelated to `List<Number>`, so
`sum(List<Number>)` rejects them. A **wildcard** says "a list of *some* type related
to this one — I don't know exactly which". Having given up knowing the exact type,
the compiler restricts what you can do, and *that restriction is what keeps it safe*.

Three flavours:

| Syntax | Reads as | You can… | You can't… |
|--------|----------|----------|------------|
| `List<?>` | list of something | read items **as `Object`** | add anything (except `null`) |
| `List<? extends Number>` | list of Number *or a subtype* | **read** items as `Number` | **add** anything (except `null`) |
| `List<? super Integer>` | list of Integer *or a supertype* | **add** `Integer`s | read items as anything but `Object` |

### Why `? extends` can read but not write

`List<? extends Number>` might secretly be a `List<Integer>`, a `List<Double>` or a
`List<BigDecimal>`. Whatever it is, anything in it **is a `Number`** — so reading as
`Number` is always safe. But if you try to `add(5)`: is it safe to put an `Integer`
into it? Only if the list is really a `List<Integer>` — and it might be a
`List<Double>`. The compiler can't know, so it refuses:

```
error: incompatible types: int cannot be converted to CAP#1
```

(`CAP#1` is the compiler's name for "that unknown type, captured".)

### Why `? super` can write but not read

`List<? super Integer>` might be a `List<Integer>`, `List<Number>` or `List<Object>`.
Whichever it is, it can certainly **hold an `Integer`** — an `Integer` is an `Integer`,
a `Number` and an `Object` — so adding is always safe. But reading: what comes out
might be an `Integer`, a `Number`, or any `Object` — all you can promise is `Object`.

### PECS

Put it together:

- If the collection **P**roduces items you read → **E**xtends: `List<? extends T>`
- If the collection **C**onsumes items you write → **S**uper: `List<? super T>`

Run the three cases and the plan's copy exercise:

```java
import java.util.ArrayList;
import java.util.List;

public class Wildcards {
    static double sum(List<? extends Number> numbers) {
        double total = 0;
        for (Number n : numbers) total += n.doubleValue();
        return total;
    }

    static void fillWithIntegers(List<? super Integer> sink) {
        for (int i = 1; i <= 3; i++) sink.add(i);
    }

    static <T> void copy(List<? super T> dest, List<? extends T> src) {
        for (T item : src) dest.add(item);
    }

    static void printAll(List<?> anything) {
        for (Object o : anything) System.out.print(o + " ");
        System.out.println();
    }

    public static void main(String[] args) {
        List<Integer> ints = List.of(1, 2, 3);
        List<Double> doubles = List.of(1.5, 2.5);
        System.out.println("sum(ints)    = " + sum(ints));
        System.out.println("sum(doubles) = " + sum(doubles));

        List<Number> numbers = new ArrayList<>();
        List<Object> objects = new ArrayList<>();
        fillWithIntegers(numbers);
        fillWithIntegers(objects);
        System.out.println("numbers = " + numbers + ", objects = " + objects);

        List<Number> dest = new ArrayList<>();
        copy(dest, ints);
        copy(dest, doubles);
        System.out.println("copied into List<Number>: " + dest);

        printAll(List.of("a", "b"));

        List<? extends Number> readOnly = ints;
        Number first = readOnly.get(0);
        System.out.println("read from ? extends: " + first);
        // readOnly.add(5);   // does not compile - see below
    }
}
```

```
sum(ints)    = 6.0
sum(doubles) = 4.0
numbers = [1, 2, 3], objects = [1, 2, 3]
copied into List<Number>: [1, 2, 3, 1.5, 2.5]
a b 
read from ? extends: 1
```

Why `copy` needs *both* wildcards, which is the heart of the exercise:

```
static <T> void copy(List<? super T> dest, List<? extends T> src)
```

- `src` **produces** `T`s for us to read → `? extends T`. It can be a `List<Integer>`
  or `List<Double>` when `T` is `Number`.
- `dest` **consumes** `T`s we write → `? super T`. It can be a `List<Number>` or
  `List<Object>`.
- With plain `copy(List<T> dest, List<T> src)`, copying the `List<Integer>` into a
  `List<Number>` wouldn't compile, because `List<Integer>` and `List<Number>` are
  unrelated. The wildcards are exactly what makes that — an extremely reasonable
  thing to want — legal.

You'll see PECS in the JDK itself: `Collections.copy(List<? super T> dest, List<?
extends T> src)`, `Collection.addAll(Collection<? extends E> c)`, and
`Comparator<? super T>` all over the streams API (Days 33–35). When you hit
`Predicate<? super T>` in a signature next week, you now know what it means: "I'll
*consume* `T`s (I only call `test` with them), so a predicate on any supertype of `T`
is fine."

Rules of thumb:

- Pure input (you only read) → `? extends`.
- Pure output (you only write) → `? super`.
- Both read and write → no wildcard; use the exact `T`.
- Don't use wildcards in **return types** — it forces callers to deal with them.
- `List<?>` is the right way to say "a list of anything, I only need `Object`" — and
  it's what to write instead of the raw `List`.

> **Say this in an interview:** "Generics are invariant, so `List<Integer>` isn't a
> `List<Number>`. Wildcards restore flexibility: `? extends T` is a producer I can read
> from but not add to, `? super T` is a consumer I can add to but only read as
> `Object`. PECS — Producer Extends, Consumer Super — tells me which one to pick."

---

## Part 6 — Raw types and unchecked warnings

**In plain words:** a **raw type** is a generic class written with the blanks left
empty — `List` instead of `List<String>`. It's a leftover so that pre-Java-5 code
still compiles, and it means "switch generics off for this variable". The compiler
then can't protect you, so it prints an **unchecked warning** instead of an error:
"I can't prove this is safe." That warning is the compiler telling you where the
Part 3 trap is.

Break type safety with a raw type — the plan's last exercise:

```java
import java.util.ArrayList;
import java.util.List;

public class RawBreak {
    @SuppressWarnings({"rawtypes", "unchecked"})
    public static void main(String[] args) {
        List<Integer> numbers = new ArrayList<>();
        numbers.add(1);

        List raw = numbers;
        raw.add("not a number");

        System.out.println("list now: " + numbers);
        int total = 0;
        try {
            for (int n : numbers) total += n;
        } catch (ClassCastException e) {
            System.out.println("blew up on iteration: " + e.getMessage());
        }
    }
}
```

```
list now: [1, not a number]
blew up on iteration: class java.lang.String cannot be cast to class java.lang.Integer (...)
```

The `@SuppressWarnings` line is what *hid* the compiler's warnings so the demo is
quiet. Remove it and run `javac -Xlint:all RawBreak.java` (PowerShell) to see the
warnings: `found raw type: List`, and `unchecked call to add(E) as a member of the
raw type List`. **In real code, never silence these without a reason you can
state.** `@SuppressWarnings("unchecked")` is acceptable only in a small, commented
spot where *you* can prove the cast is safe (for example inside a generic container
that manages its own array).

Rules:

- **Never use raw types in new code.** Write `List<?>` if you truly don't care.
- **Treat unchecked warnings as bugs** until proven otherwise.
- Legacy methods returning raw types: assign to a properly-typed variable *and
  validate* (`for (Object o : raw) names.add((String) o)`).

> **Say this in an interview:** "A raw type opts out of generic checking, so it
> yields unchecked warnings and allows heap pollution — a `ClassCastException` far
> from the real mistake. I don't use raw types and I treat unchecked warnings as bugs
> unless I can prove the cast safe."

---

## Hands-on

Run every block. The plan's required three are 1, 2 and 3.

1. **Required — bounded repository.** In `RepoDemo`, add a method
   `List<T> findAllById(Iterable<ID> ids)` to the interface and implement it. Then
   uncomment `users.save(new Product("X", 1.0));` and read the compiler's error.
   Explain in one sentence why the compiler can catch it.
2. **Required — copy with both wildcards.** In `Wildcards`, change `copy`'s signature
   to `copy(List<T> dest, List<T> src)` and try `copy(dest, ints)` where `dest` is a
   `List<Number>`. Read the error. Restore the wildcards and explain why each is
   needed in your own words.
3. **Required — break it with a raw type.** Run `RawBreak` and `HeapPollution`.
   Then remove `@SuppressWarnings` and compile with `javac -Xlint:all` (PowerShell).
   Write down the two warning messages and which one predicts the later crash.
4. Write `static <T extends Comparable<T>> T min(List<T> list)`. Call it with
   `List<String>`, `List<Integer>`, and a list of a class that doesn't implement
   `Comparable`. Read the third error.
5. Try each forbidden thing from the table in Part 3 (`new T[3]`, `instanceof
   List<String>`, overloading on `List<String>`/`List<Integer>`) in a scratch file and
   match each real error message to its explanation.
6. Write `static void addNumbers(List<? super Integer> sink)` and call it with a
   `List<Number>`, `List<Object>` and `List<Integer>`; then try a `List<Double>` and
   read the error.
7. Write `static <T> void swapFirstTwo(List<T> list)`. Then try writing it with
   `List<?>`. Why does the second not compile? (Try; the answer is "capture" — you
   can't write back to an unknown type.)
8. Open the source of `java.util.Collections.copy` and `ArrayList.addAll` (in VS Code:
   Ctrl-click `addAll`). Find the wildcards.

---

## Gotchas

- **Primitives can't be type arguments.** `List<int>` is illegal; use `List<Integer>`
  (boxing, Day 21). This is why primitive streams (`IntStream`) exist.
- **Generic ≠ safe at runtime.** `Collections.unmodifiableList` and `List.of` return
  lists whose type argument you must still trust.
- **`List<? extends Number>` is read-only for practical purposes** — and `List.of`
  results are immutable anyway. Don't confuse the two.
- **Diamond with anonymous classes** works since Java 9; with lambdas, inference
  usually just works.
- **Static methods can't use the class's `T`.** A static method needs its own
  `<T>`.
- **`Class<T>` tokens** are how you smuggle runtime type info past erasure.
- **`T extends Comparable<T>` vs `Comparable<? super T>`:** the most precise bound on
  `max` is `<T extends Comparable<? super T>>` (so a `java.sql.Timestamp`, which
  compares via its parent `Date`, works). `Collections.max` uses it. Day 33 returns
  to this.
- **Inference messages are ugly.** When a generics error mentions `CAP#1`,
  `inference variable T has incompatible bounds` or `equality constraints`, read it
  as "the types you gave don't line up". Add an explicit type witness
  (`Collections.<String>emptyList()`) or a local variable to guide the compiler.
- **Bridge methods.** When a class implements `Comparable<Foo>`, the compiler
  generates a hidden `compareTo(Object)` that casts and calls yours. You'll see
  `bridge`/`synthetic` methods in stack traces and reflection output.

---

## Recap

- Generics move type errors from runtime (`ClassCastException`) to compile time and
  remove casts.
- Type parameters make classes and methods generic; bounds (`extends`) restrict
  them and expose the bound's methods.
- **Erasure:** type arguments exist only at compile time. `List<String>` is `List` at
  runtime, `T` is `Object` (or its bound), casts are inserted. So no `new T[]`, no
  `instanceof List<String>`, no overloading by type argument.
- Generics are **invariant**: `List<String>` is not a `List<Object>`. Arrays are
  covariant and pay for it with `ArrayStoreException`.
- Wildcards: `? extends T` — read, can't add; `? super T` — add, can't read (except as
  `Object`). **PECS.**
- Raw types disable checking and allow heap pollution. Never use them; treat
  unchecked warnings as bugs.

Re-answer the four questions at the top.

---

## Say it out loud

1. "Why do generics exist? What bug do they prevent?"
2. "What is type erasure, and what restrictions does it cause?"
3. "Why isn't a `List<String>` a `List<Object>`, when a `String[]` is an `Object[]`?"
4. "Explain PECS with an example."
5. "What's the difference between `List<?>`, `List<Object>` and the raw `List`?"
6. "What does `<T extends Comparable<T>>` mean, and why would you need it?"

---

## Q&A

_Follow-up questions from this lesson are appended here._
