# Day 22 — Identity, equals/hashCode & String

_Why this matters:_ "Why is my `Set` holding duplicates?" "Why does `map.get(key)`
return `null` for a key I *know* I put in?" "Why did the entry vanish after I
updated the object?" Each of these is a broken `equals`/`hashCode` pair — and it
fails **silently**: no exception, just wrong answers. JPA entities (Phase 7),
cache keys (Phase 12) and every `HashMap` you'll ever use depend on getting this
right. The same day covers `String`, the most-used class in Java, and why a loop
of `+=` is a performance bug.

---

## The one-paragraph version

Yesterday you learned that a variable holds a *reference*. Today's question is:
when are two objects "the same"? Java gives you two answers. **Identity** (`==`):
are these two references pointing at the very same object? **Equality**
(`.equals`): do these two objects *mean* the same thing, like two separate
`Point(1, 2)` objects? By default `.equals` is just `==`; you override it to
say what "the same" means for your class. If you override `equals` you **must**
also override `hashCode`, because `HashMap` and `HashSet` use the hash code to
decide which bucket (storage slot) to look in *before* they ever call `equals`.
Two equal objects with different hash codes land in different buckets and never
meet, so the set stores both and the map can't find its own keys. Finally,
`String` objects can never change once created (**immutable**) — which makes them
safe to share and to use as keys, but means "modifying" one really creates a new
one, so building a long string with `+=` in a loop is quadratic.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **identity** | "Is this the very same object?" | Reference equality, tested with `==` |
| **equality** | "Do these mean the same thing?" | Logical equality, tested with `.equals()` |
| **contract** | The rules a method promises to obey, which other code relies on | Written in the Javadoc of `Object.equals` / `hashCode`; the compiler can't check it |
| **reflexive** | Everything equals itself | `x.equals(x)` is true |
| **symmetric** | If A equals B then B equals A | `x.equals(y) == y.equals(x)` |
| **transitive** | If A equals B and B equals C then A equals C | |
| **consistent** | Same answer every time (while nothing changed) | Repeated calls don't flip |
| **hash code** | A whole number summarising an object, used as a quick "which bucket?" guess | `int hashCode()`; equal objects **must** give equal hash codes |
| **bucket** | One slot in a hash table's array | `HashMap` picks `hash % table size` (roughly) |
| **hash collision** | Two different objects landing in the same bucket | Allowed and normal; resolved by calling `equals` within the bucket |
| **immutable** | Can't be changed after creation | `String`, wrappers, `LocalDate`, records (shallowly) |
| **string pool** | A shared set of one-copy-each string literals | Also called the intern table; literals are put in it by the JVM |
| **intern** | "Give me the pool's copy of this string" | `String.intern()` |
| **`StringBuilder`** | A string you're allowed to change | A mutable character buffer that grows as needed |

---

## Before you read: four questions

1. `new Point(1, 2)` twice, with no `equals` written. Is `p.equals(q)` true?
2. You override `equals` but forget `hashCode`, then add two equal objects to a
   `HashSet`. How many does it hold?
3. You use an object as a `HashMap` key, then change one of its fields. What
   happens to `map.get(thatObject)`?
4. Does `s.toUpperCase();` (result ignored) change `s`?

<details><summary>Answers (try first)</summary>

1. No — the default `equals` is `==`. (Part 1.)
2. Almost certainly two. (Part 3.)
3. It returns `null`, and the entry is still stuck in the map. (Part 4.)
4. No. Strings are immutable; `toUpperCase` returns a new one. (Part 5.)

</details>

---

## Part 1 — `==` vs `.equals()`

**In plain words:** two identical-looking twins are two people. `==` asks "is this
the same person?". `.equals()` asks "do these two look the same for our purposes?".
Out of the box, Java can't know what "look the same" means for *your* class, so
the default `.equals()` plays safe and just asks the `==` question. You teach it
better by overriding it.

```java
public class Identity {
    static class Point {
        final int x, y;
        Point(int x, int y) { this.x = x; this.y = y; }
    }

    public static void main(String[] args) {
        Point p = new Point(1, 2);
        Point q = new Point(1, 2);
        Point r = p;

        System.out.println("p == q       : " + (p == q));
        System.out.println("p.equals(q)  : " + p.equals(q));
        System.out.println("p == r       : " + (p == r));
        System.out.println("p.equals(r)  : " + p.equals(r));
        System.out.println("p.toString() : " + p);
    }
}
```

```
p == q       : false
p.equals(q)  : false
p == r       : true
p.equals(r)  : true
p.toString() : Identity$Point@5b2133b1
```

`p` and `q` hold the same coordinates but are two objects, so both tests say
"different". `r` is a second slip pointing at `p`'s object, so both say "same". The
last line shows the default `toString`: class name `@` hash code in hex (the hex
number varies run to run). That `5b2133b1` is `Object.hashCode()`, which by
default is derived from the object's identity — remember that for Part 3.

| | `==` | `.equals()` |
|---|---|---|
| On primitives | Compares values | Not available |
| On objects | Same object? | Whatever the class says; default = `==` |
| Can you change it? | Never | Yes, override it |

`String`, the wrappers (`Integer`…), `List`, `Set`, `Map`, `LocalDate` and records
all override `equals` to compare contents. Your own classes don't, unless you
write it (or use a record).

> **Say this in an interview:** "`==` tests reference identity; `equals` tests
> logical equality. The default `equals` is identity, so any class used as a
> value — in a collection, as a map key, in a test assertion — has to override it."

---

## Part 2 — The `equals` contract

**In plain words:** `equals` isn't just a method; it's a promise to the rest of
Java. `HashSet`, `List.contains`, `indexOf`, `assertEquals` all assume that "equal"
behaves like ordinary equality. If your `equals` breaks the rules, those
collections quietly misbehave.

The promise has five parts (from the `Object.equals` Javadoc):

| Rule | Meaning | Breaks if… |
|------|---------|------------|
| **Reflexive** | `x.equals(x)` is always true | You compare a field with `==` and it holds `NaN` |
| **Symmetric** | `x.equals(y)` ⇔ `y.equals(x)` | A subclass adds a field and `equals` is one-sided (below) |
| **Transitive** | `x=y` and `y=z` ⇒ `x=z` | Mixing subclass and superclass comparisons |
| **Consistent** | Same answer on repeated calls | `equals` depends on something that changes — a clock, a random, a mutable field |
| **Never equal to `null`** | `x.equals(null)` is false, no exception | You forget the null check (use `instanceof`/`getClass`) |

### Seeing symmetry break

The classic trap: a subclass adds a field, and `equals` is written with `instanceof`.

```java
public class Symmetry {
    static class Point {
        final int x, y;
        Point(int x, int y) { this.x = x; this.y = y; }

        @Override
        public boolean equals(Object o) {
            if (!(o instanceof Point)) return false;
            Point p = (Point) o;
            return x == p.x && y == p.y;
        }

        @Override
        public int hashCode() { return 31 * x + y; }
    }

    static class ColorPoint extends Point {
        final String color;
        ColorPoint(int x, int y, String color) { super(x, y); this.color = color; }

        @Override
        public boolean equals(Object o) {
            if (!(o instanceof ColorPoint)) return false;
            ColorPoint cp = (ColorPoint) o;
            return super.equals(o) && color.equals(cp.color);
        }
    }

    public static void main(String[] args) {
        Point p = new Point(1, 2);
        ColorPoint cp = new ColorPoint(1, 2, "red");
        System.out.println("p.equals(cp): " + p.equals(cp));
        System.out.println("cp.equals(p): " + cp.equals(p));
    }
}
```

```
p.equals(cp): true
cp.equals(p): false
```

`p` says "a `ColorPoint` at (1,2) is one of me" but `cp` says "a plain `Point` is
not one of me". Symmetry is broken. Put `p` in a list and ask `contains(cp)` → true;
put `cp` in a list and ask `contains(p)` → false. Same two objects, different
answers depending on the argument order.

There's no clean way to extend a class with a new field and keep `equals` correct
(Josh Bloch's *Effective Java* spends a whole item on it). The practical rules:

- Use **`getClass() != o.getClass()`** instead of `instanceof` if subclasses may
  exist, so a `Point` is never equal to a `ColorPoint`; or
- Make the class **`final`** (or a record, which is implicitly final) so there are
  no subclasses to worry about. Prefer this: **favour composition over adding fields
  in subclasses** — it comes back in Day 37.

> **Say this in an interview:** "`equals` must be reflexive, symmetric, transitive,
> consistent and false for null. Subclassing a value class and adding fields
> inevitably breaks symmetry or transitivity, so I make value classes final or use
> records."

---

## Part 3 — The `hashCode` contract, and what breaks without it

**In plain words:** a `HashMap` is a row of numbered boxes. To store a key, it
works out a box number from the key's hash code. To find a key later, it works out
the box number again and looks *only in that box*. Searching every box would
defeat the point — that's what makes lookup fast. So the system only works if
**the same key always produces the same box number**, and — crucially — if two keys
that `equals` says are the same also produce the same number. If they don't, the
second one goes to a different box and the map never looks there.

The contract (also from `Object`'s Javadoc):

1. **Equal objects must have equal hash codes.** If `a.equals(b)` then
   `a.hashCode() == b.hashCode()`.
2. **Consistent:** `hashCode` must return the same value each time, as long as the
   fields used in `equals` don't change.
3. *Not* required the other way: unequal objects *may* share a hash code (a
   **collision**). It only costs speed.

### Show the failure first

A class with `equals` but **no** `hashCode`:

```java
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

public class BrokenSet {
    static class Point {
        final int x, y;
        Point(int x, int y) { this.x = x; this.y = y; }

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (!(o instanceof Point)) return false;
            Point other = (Point) o;
            return x == other.x && y == other.y;
        }
        // no hashCode()!
    }

    public static void main(String[] args) {
        Point a = new Point(1, 2);
        Point b = new Point(1, 2);
        System.out.println("a.equals(b): " + a.equals(b));
        System.out.println("same hashCode: " + (a.hashCode() == b.hashCode()));

        Set<Point> set = new HashSet<>();
        set.add(a);
        set.add(b);
        System.out.println("set size (want 1): " + set.size());
        System.out.println("set.contains(new Point(1, 2)): " + set.contains(new Point(1, 2)));

        Map<Point, String> names = new HashMap<>();
        names.put(new Point(0, 0), "origin");
        System.out.println("names.get(new Point(0, 0)): " + names.get(new Point(0, 0)));
    }
}
```

```
a.equals(b): true
same hashCode: false
set size (want 1): 2
set.contains(new Point(1, 2)): false
names.get(new Point(0, 0)): null
```

Every line of output is wrong in a way a user would call a bug, and nothing threw.
`equals` says equal, but the inherited `hashCode` (identity-based, Part 1) gave
them different numbers, so they went to different boxes. The set kept both; `contains`
looked in a third box; the map couldn't find a key that is, by `equals`, identical.

(Strictly, two identity hash codes *could* collide by chance, so "almost certainly"
rather than "always" — don't build on that.)

What `HashMap` really does on `get(key)`:

```
1. h = key.hashCode()               -> which box?
2. look in box h (roughly h % tableSize)
3. for each entry in that box: if entry.hash == h AND key.equals(entry.key) -> found
```

`equals` is only ever consulted **inside the box that `hashCode` chose**. (Day 32
opens the box and shows what's inside.)

### The fix: both methods, from the same fields

```java
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Objects;
import java.util.Set;

public class FixedSet {
    static class Point {
        final int x, y;
        Point(int x, int y) { this.x = x; this.y = y; }

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            Point other = (Point) o;
            return x == other.x && y == other.y;
        }

        @Override
        public int hashCode() {
            return Objects.hash(x, y);
        }
    }

    record Coord(int x, int y) {}

    public static void main(String[] args) {
        Set<Point> set = new HashSet<>();
        set.add(new Point(1, 2));
        set.add(new Point(1, 2));
        System.out.println("set size: " + set.size());
        System.out.println("contains: " + set.contains(new Point(1, 2)));

        Map<Point, String> names = new HashMap<>();
        names.put(new Point(0, 0), "origin");
        System.out.println("get: " + names.get(new Point(0, 0)));

        System.out.println("record equals: " + new Coord(1, 2).equals(new Coord(1, 2)));
        System.out.println("record hashCodes equal: " + (new Coord(1, 2).hashCode() == new Coord(1, 2).hashCode()));
        System.out.println("record toString: " + new Coord(1, 2));
    }
}
```

```
set size: 1
contains: true
get: origin
record equals: true
record hashCodes equal: true
record toString: Coord[x=1, y=2]
```

The recipe — memorise it:

| Method | Recipe |
|--------|--------|
| `equals` | `this == o` fast path → null/class check → cast → compare **every significant field** (`==` for primitives, `Objects.equals(a, b)` for references so `null` is safe) |
| `hashCode` | `Objects.hash(f1, f2, …)` using **exactly the fields `equals` uses** |

Rules that follow:

- **Same fields in both.** If `equals` uses `x` and `y`, `hashCode` uses `x` and `y`.
  Extra fields in `hashCode` break rule 1; fewer are legal but give more collisions.
- A constant `hashCode` (`return 1;`) is *legal* — every object lands in one box,
  and the map degrades into a list. Correct but slow. Don't.
- **Records** (`record Coord(int x, int y)`, Day 36) generate `equals`, `hashCode`
  and `toString` from their components for free. For plain value classes in modern
  Java, reach for a record first.
- IDEs generate both methods (VS Code: Source Action → Generate `hashCode()` and
  `equals()`). Lombok's `@EqualsAndHashCode` does too — but with JPA entities,
  think before you use it (Phase 7).

> **Say this in an interview:** "Hash-based collections use `hashCode` to pick a
> bucket and then `equals` within it, so equal objects must have equal hash codes.
> If you override `equals` without `hashCode`, equal objects hash to different
> buckets and a `HashSet` holds duplicates and a `HashMap` can't find its keys. I
> derive both from the same fields, using `Objects.equals` and `Objects.hash`, or
> use a record."

---

## Part 4 — Mutable keys: the entry that vanishes

**In plain words:** the map decided which box your key goes in *at the moment you
put it in*, using its hash code then. If you later change the key's fields, its hash
code changes, but the key is still sitting in its old box. Now when you look it
up, the map calculates the *new* box number, looks there, and finds nothing. The
entry is still in the map, taking up space — you just can't reach it.

```java
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

public class LostKey {
    static class Key {
        String id;
        Key(String id) { this.id = id; }

        @Override
        public boolean equals(Object o) {
            return o instanceof Key k && Objects.equals(id, k.id);
        }

        @Override
        public int hashCode() {
            return Objects.hash(id);
        }
    }

    public static void main(String[] args) {
        Map<Key, String> map = new HashMap<>();
        Key key = new Key("alice");
        map.put(key, "stored");

        System.out.println("before mutation: " + map.get(key));

        key.id = "alicia";

        System.out.println("after mutation, same object:  " + map.get(key));
        System.out.println("lookup by old value:          " + map.get(new Key("alice")));
        System.out.println("lookup by new value:          " + map.get(new Key("alicia")));
        System.out.println("map.size():                   " + map.size());
        System.out.println("entry is still in there:      " + map.values());
    }
}
```

```
before mutation: stored
after mutation, same object:  null
lookup by old value:          null
lookup by new value:          null
map.size():                   1
entry is still in there:      [stored]
```

Even looking up with the **identical object** you inserted fails. Looking up by the
old value fails too: the box is right, but `equals` now says the stored key
("alicia") doesn't match "alice". The entry is a **leak** — Yesterday's Part 6, no
GC can help because the map still references it.

(`o instanceof Key k` is Java 16+ *pattern matching*, covered on Day 36; here it
means "if `o` is a `Key`, call it `k`".)

The rule: **keys in hash-based collections must be effectively immutable** — or at
least, never mutated while they're in the collection. This is why `String`,
`Integer`, `UUID` and records make good keys, and why a mutable JPA entity with a
database-generated ID is a famously bad `hashCode` source (its ID changes on first
save). Day 38 returns to immutability as a design tool.

> **Say this in an interview:** "A hash key's bucket is chosen at insert time from
> its hash code. If a field used in `hashCode` is mutated while it's in the map,
> lookups go to the wrong bucket and the entry becomes unreachable. Keys should be
> immutable."

---

## Part 5 — `String`: immutability, the pool, `+` in a loop

**In plain words:** a `String` is like a sentence carved in stone. You can't change
a letter; to "change" it you carve a new stone and point at that instead. This is a
feature: because nobody can alter a string behind your back, many parts of a program
can safely share one copy, and a string used as a `HashMap` key can never get into
the Part 4 trap.

### `s.toUpperCase();` does nothing to `s`

The most common beginner bug is ignoring the returned value. Every "modifying"
method (`toUpperCase`, `trim`, `replace`, `substring`, `concat`) returns a **new**
`String`.

### The string pool

Since strings never change, Java can keep **one copy of each literal** and reuse it.
Those copies live in the **string pool**. Run:

```java
public class StringPool {
    public static void main(String[] args) {
        String a = "hello";
        String b = "hello";
        String c = new String("hello");
        String d = "hel" + "lo";

        String half = "hel";
        String e = half + "lo";

        System.out.println("a == b          (two literals)       : " + (a == b));
        System.out.println("a == c          (new String)         : " + (a == c));
        System.out.println("a == d          (constant folded)    : " + (a == d));
        System.out.println("a == e          (built at runtime)   : " + (a == e));
        System.out.println("a == e.intern() (interned)           : " + (a == e.intern()));
        System.out.println("a.equals(e)                          : " + a.equals(e));

        String s = "java";
        s.toUpperCase();
        System.out.println("after s.toUpperCase() with result ignored: " + s);
        s = s.toUpperCase();
        System.out.println("after s = s.toUpperCase():                 " + s);
    }
}
```

```
a == b          (two literals)       : true
a == c          (new String)         : false
a == d          (constant folded)    : true
a == e          (built at runtime)   : false
a == e.intern() (interned)           : true
a.equals(e)                          : true
after s.toUpperCase() with result ignored: java
after s = s.toUpperCase():                 JAVA
```

Reading it:

- `a == b`: two identical literals are *one* pooled object.
- `a == c`: `new String(...)` always makes a fresh heap object, bypassing the pool.
- `a == d`: `"hel" + "lo"` is a **compile-time constant**; `javac` folds it into
  `"hello"` before the program runs, so it's the pooled one.
- `a == e`: `half + "lo"` is computed while running, producing a new object.
- `intern()` says "give me the pool's copy", so it matches `a`.

The takeaway is the same as the `Integer` cache: `==` on strings *sometimes works by
accident*. **Always compare strings with `.equals`.** (`"literal".equals(x)` is
also null-safe when `x` might be `null`.) Almost nobody calls `intern()` in
application code.

**Why `String` is immutable — the reasons:**

| Reason | What it buys |
|--------|--------------|
| **Safe to share** | The pool, caches, and many references to one object are harmless |
| **Safe as a hash key** | Its hash can be computed once and cached; it can never change under a `HashMap` |
| **Thread-safe for free** | Nothing to race on (Phase 5) |
| **Security** | A file path or hostname that was validated can't be changed after the check |

### Concatenation in a loop

Since each `+=` makes a **new** string, building a string one piece at a time copies
all the previous characters every time: 1 + 2 + 3 + … + n copies. That's **O(n²)** —
quadruple the input, 16× the work. `StringBuilder` is a mutable buffer that appends
in place, so it's O(n).

```java
public class Concat {
    public static void main(String[] args) {
        int n = 50_000;

        long t0 = System.nanoTime();
        String s = "";
        for (int i = 0; i < n; i++) s += "x";
        long plusMs = (System.nanoTime() - t0) / 1_000_000;

        t0 = System.nanoTime();
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < n; i++) sb.append("x");
        String built = sb.toString();
        long builderMs = (System.nanoTime() - t0) / 1_000_000;

        System.out.println("same result: " + s.equals(built) + " (" + built.length() + " chars)");
        System.out.println("+= in a loop:  " + plusMs + " ms");
        System.out.println("StringBuilder: " + builderMs + " ms");
    }
}
```

On this machine: 527 ms vs 7 ms, for 50,000 characters. Try `n = 200_000` and the
first number goes up by about 16×, not 4×.

Use `+` freely for a handful of pieces *outside* loops — the compiler handles that
well. Use `StringBuilder` when you build a string **across loop iterations**.
(`String.join(", ", list)` and `String.format` cover the common cases; Day 35's
`Collectors.joining` is the stream version.) `StringBuffer` is the old,
synchronized twin — don't reach for it.

> **Say this in an interview:** "`String` is immutable, so literals can be pooled
> and shared, hash codes cached, and strings used safely as map keys and across
> threads. `==` compares references and only works for pooled strings by accident,
> so I use `equals`. Concatenating with `+=` in a loop is quadratic because each
> step copies the whole string; I use `StringBuilder`."

---

## Hands-on

Run every block above. The required exercises are 1 and 2 (the plan's practice).

1. **Required — duplicates in a `HashSet`.** In `BrokenSet`, confirm the set holds
   two equal points. Add a correct `hashCode` (`Objects.hash(x, y)`) and confirm
   `size()` becomes 1, `contains` is true and `names.get` finds `"origin"`.
2. **Required — the vanishing key.** In `LostKey`, change `Key.id` to `final` — the
   assignment `key.id = "alicia"` no longer compiles. That's the fix in one
   keyword: you've made the failure impossible instead of unlikely.
3. In `Symmetry`, change both `equals` methods to compare
   `getClass() != o.getClass()` instead of `instanceof`. Is `p.equals(cp)` now
   symmetric? What does `Point` no longer consider equal?
4. Add a `toString()` to `Point` that prints `Point(1, 2)`. Notice how much more
   readable the failing output of `BrokenSet` would have been.
5. Print `new Point(1, 2).hashCode()` twice in `FixedSet`. Equal. Print it for
   `new Point(2, 1)`. Is it different? (It should be; `Objects.hash(1,2)` ≠
   `Objects.hash(2,1)` because order matters.)
6. `StringPool` — add `String f = "hello".toUpperCase().toLowerCase();` and predict
   `a == f`. Then run it.
7. `Concat` — raise `n` to `100_000` and `200_000` and watch the two columns scale.
   The `+=` numbers should grow roughly 4× per doubling, the builder's roughly 2×.
8. Search a codebase you work on for classes used as `HashMap` keys or in
   `HashSet`s. Do they all override both methods?

---

## Gotchas

- **Override both or neither.** An IDE warning "overrides equals but not hashCode"
  is not noise.
- **`equals(Object)`, not `equals(Point)`.** If you write `public boolean
  equals(Point p)` you've *overloaded*, not overridden; `HashSet` still calls the
  `Object` version. Always add **`@Override`** — the compiler then rejects the
  mistake.
- **Floating-point fields.** Compare `double`s with `Double.compare(a, b) == 0`, not
  `==` (NaN, -0.0). `Objects.hash` already handles this via boxing.
- **`Objects.hash` allocates** (varargs array + boxing) — fine for ordinary code, a
  measurable cost on a very hot path.
- **Array fields.** `Objects.equals(arr1, arr2)` compares references. Use
  `Arrays.equals` / `Arrays.hashCode` (or avoid arrays in value classes).
- **`hashCode` should not use mutable state** unless the object is never a key.
- **`new String("x")`** is almost always a mistake — it creates a pointless copy.
- **`switch` on a string and `.equals`** are safe; `==` on a string from user
  input, a file, or a request is a bug waiting.
- **`null` and `equals`.** `a.equals(b)` throws if `a` is `null`. `Objects.equals(a, b)`
  doesn't.

---

## Recap

- `==` = same object; `.equals` = same meaning (default is `==`; override it).
- `equals` must be reflexive, symmetric, transitive, consistent, and false for
  `null`. Subclasses adding fields break it → make value classes `final` / records.
- `hashCode` must agree with `equals`: equal ⇒ equal hash. Use the same fields in
  both; `Objects.hash` and `Objects.equals` do the work.
- Without `hashCode`, a `HashSet` keeps duplicates and a `HashMap` can't find its
  keys. Nothing throws.
- Don't mutate a key while it's in a hash collection — it becomes unreachable.
- `String` is immutable: pooled, shareable, safe keys; always compare with
  `.equals`; build in loops with `StringBuilder`.

Now re-answer the four questions at the top.

---

## Say it out loud

1. "What's the difference between `==` and `equals`, and what does the default
   `equals` do?"
2. "What's the `equals`/`hashCode` contract, and why does it exist?"
3. "I put two equal objects in a `HashSet` and it kept both. What went wrong?"
4. "Why are mutable objects dangerous as `HashMap` keys?"
5. "Why is `String` immutable, and what does that buy us?"
6. "Why is `s += x` in a loop a problem, and what would you use?"

---

## Q&A

_Follow-up questions from this lesson are appended here._
