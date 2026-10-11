# Day 21 — The JVM, Memory & References

_Why this matters:_ Almost every confusing Java bug in the next 280 days is a
memory-model bug in disguise. "I changed the object in the method but the caller
didn't see it." "Two `Integer`s are equal at 127 and not at 128." "The service
works for a week, then dies with `OutOfMemoryError`." "Every thread sees the same
list." All four are the same underlying picture — **where things live, and who
holds a pointer to them**. Phase 5 (concurrency) and Phase 6 (Spring) assume you
can see that picture without effort. Today you build it.

> **Long-ish day.** Required practice is hands-on 1, 2 and 3. If you're short on
> time, do Parts 1–4 and hands-on 1–5 today, and Parts 5–7 with hands-on 6–10
> next session.

---

## The one-paragraph version

You write `.java` files; `javac` turns them into **bytecode** (`.class` files), a
portable instruction set that no CPU understands directly. The **JVM** (Java
Virtual Machine) is the program that loads and runs that bytecode. While it runs,
it manages memory in two main places. The **stack** is a small scratchpad that
each thread gets for itself: it holds local variables and the chain of method
calls currently in progress. The **heap** is one big shared pool where every
object lives. A variable that holds an object doesn't hold the object — it holds a
**reference**, a pointer to it. That one fact explains "pass by value" (Java
copies the pointer, never the object), `==` vs `.equals` (same pointer vs same
contents), and the `Integer` cache surprise. Nobody frees objects by hand: a
**garbage collector** periodically finds objects that nothing can reach any more
and reclaims them. That makes leaks harder, not impossible — if something still
points at an object you're done with, it's never reclaimed. Run out of heap and
you get `OutOfMemoryError`; run out of stack (almost always runaway recursion) and
you get `StackOverflowError`.

---

## Words you'll meet today

| Term | In plain words | The precise version |
|------|----------------|---------------------|
| **source code** | The `.java` text you write | Input to `javac` |
| **bytecode** | Instructions for an imaginary computer, saved in `.class` files | The JVM's instruction set; same bytes run on Windows, Linux, Mac |
| **JVM** | The program that runs bytecode | Loads classes, verifies them, interprets/compiles them, manages memory and threads |
| **JRE** | The JVM plus the standard library — enough to *run* Java | Java Runtime Environment; since JDK 11 it is no longer shipped as a separate download |
| **JDK** | The JRE plus the tools to *build* Java | Adds `javac`, `jar`, `javap`, `jcmd`, …; this is what you installed (Adoptium JDK 21) |
| **JIT compiler** | The part that turns hot bytecode into real machine code while the program runs | Just-In-Time compiler; HotSpot's C1 and C2 |
| **stack** | A thread's private scratchpad for method calls and local variables | One per thread; grows and shrinks as methods are called and return |
| **stack frame** | One method call's slice of the stack | Holds that call's local variables, parameters and return address; pushed on call, popped on return |
| **heap** | The shared pool where objects live | One per JVM; every thread can reach it; the garbage collector manages it |
| **object** | A chunk of heap memory holding some fields | Created by `new` (or by boxing, string concatenation, lambdas capturing…) |
| **reference** | A variable's way of pointing at an object | A pointer-like value; `null` means "points at nothing" |
| **primitive** | A value stored directly in the variable | `int`, `long`, `double`, `boolean`, `char`, `byte`, `short`, `float` — 8 of them |
| **wrapper** | An object version of a primitive | `Integer`, `Long`, `Double`, … |
| **autoboxing / unboxing** | Java converting between `int` and `Integer` for you | The compiler inserts `Integer.valueOf(x)` and `x.intValue()` |
| **pass by value** | The method gets a *copy* of what you passed | For objects, the copy is of the *reference*, not of the object |
| **garbage collector (GC)** | The janitor that reclaims objects nobody can reach | Finds unreachable objects and frees their memory |
| **GC root** | A starting point the GC trusts is alive | Local variables on thread stacks, static fields, live threads |
| **reachable** | You can get to the object by following references from a root | Only reachable objects survive a collection |
| **generation** | A heap area for objects of a similar age | *Young* (new objects) and *old* (survivors) |
| **minor GC / major GC** | A quick cleanup of the young area / a slow cleanup of the old area | A "full GC" collects everything |
| **stop-the-world pause** | The moment your application threads freeze so the GC can work | Every GC has some; modern ones keep them to milliseconds |
| **memory leak** | Memory that is no longer useful but can't be reclaimed | Objects still reachable (so not garbage) but never used again |
| **`OutOfMemoryError`** | The heap (or another memory area) is full | An `Error`, not an `Exception` |
| **`StackOverflowError`** | One thread's stack is full | Usually infinite or very deep recursion |

---

## Before you read: four questions

Answer from gut feeling. You'll see them again at the end.

1. A method receives a `List<String>` and calls `list.add("x")`. Does the caller's
   list change? Now the method instead does `list = new ArrayList<>()` and adds
   to that. Does the caller's list change?
2. `Integer a = 127, b = 127;` — is `a == b` true? What about with 128?
3. Java has a garbage collector. So how can a Java program leak memory?
4. What's the difference between `OutOfMemoryError` and `StackOverflowError`?

<details><summary>Answers (try first)</summary>

1. First: yes. Second: no. (Part 3.)
2. 127: true. 128: false. (Part 4.)
3. By keeping a reference to objects it will never use again — a static list that
   only grows is the classic. (Part 6.)
4. Heap full (all threads share it) vs one thread's stack full (deep recursion).
   (Part 7.)

</details>

---

## Part 1 — What "Java" actually is: source → bytecode → running

**In plain words:** most languages are translated into your computer's own
language (machine code) in advance, so the result only runs on that kind of
computer. Java translates into an *in-between* language — bytecode — that no real
computer speaks. A program called the JVM, which *is* built for each kind of
computer, reads the in-between language and carries it out. Write once, and the
same `.class` file runs anywhere there's a JVM. The price is that there's always a
middleman running; the reward is that the middleman can watch your program run and
optimise the parts that matter.

```
Hello.java  --javac-->  Hello.class  --java-->  JVM loads, checks, runs it
 (source)               (bytecode)                (interprets, then JIT-compiles hot code)
```

**JDK vs JRE vs JVM** is a set of Russian dolls:

```
┌───────────────────────── JDK ─────────────────────────┐
│  javac, javap, jar, jcmd, jshell ...   (build & inspect) │
│  ┌──────────────────── JRE ───────────────────────┐   │
│  │  standard library (java.util, java.time ...)    │   │
│  │  ┌───────────────── JVM ─────────────────┐     │   │
│  │  │ class loader, interpreter, JIT, GC    │     │   │
│  │  └───────────────────────────────────────┘     │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

You installed a JDK. A production container normally ships only the runtime parts
(Phase 4 will make that concrete).

### Look at real bytecode

This class is complete — hit Run to see it print `5`:

```java
public class Hello {
    public static void main(String[] args) {
        int a = 2;
        int b = 3;
        System.out.println(a + b);
    }
}
```

`javap -c` disassembles the compiled class. It's a JDK tool, so run it in
PowerShell (Windows has the JDK; your WSL Ubuntu does not):

```powershell
# in a scratch folder containing Hello.java
javac Hello.java
javap -c Hello
```

The interesting part of the output, from your JDK 21:

```
 0: iconst_2          push the constant 2 onto the operand stack
 1: istore_1          pop it into local variable slot 1  (a)
 2: iconst_3
 3: istore_2          (b)
 4: getstatic System.out
 7: iload_1           push a
 8: iload_2           push b
 9: iadd              pop two ints, push their sum
10: invokevirtual PrintStream.println(int)
13: return
```

Two things to notice. The JVM is a **stack machine**: it works by pushing and
popping values rather than using named registers. And `a` and `b` have
**numbered slots in the current method's frame** — that frame is the "stack" of
Part 2.

`javap -v Hello` also prints `major version: 65`. That number *is* Java 21. A
class compiled by JDK 21 won't load on a Java 17 JVM ("unsupported class file
major version 65") — a real error you'll meet when a build tool and a runtime
disagree.

**Interpreted, then compiled.** The JVM starts by *interpreting* bytecode, which
is slow. It counts how often each method runs, and when one gets "hot" the
**JIT** compiles it to real machine code, using what it observed (which branch is
usually taken, which types actually show up) to optimise aggressively. This is why
Java benchmarks need a warm-up, and why a freshly started service is slower than
the same service ten minutes later.

> **Say this in an interview:** "`javac` compiles source to platform-independent
> bytecode; the JVM loads and verifies it, interprets it at first, and the JIT
> compiles hot methods to native code using runtime profiling. The JDK is the JRE
> plus build tools, and the JRE is the JVM plus the standard library."

---

## Part 2 — Stack vs heap

**In plain words:** picture a busy office. Each worker (thread) has a **desk** —
small, private, tidy. When a worker starts a task they put a fresh sheet on the
desk with that task's notes (local variables); when the task is done the sheet
goes in the bin. Nobody else touches your desk. Separately there's one huge shared
**warehouse** (the heap) where the bulky things live: the actual objects. Your
desk-sheet never holds a bulky object itself — it holds a **slip of paper with the
object's shelf location** on it.

That last sentence is the whole lesson. Keep it in your head.

| | Stack | Heap |
|---|---|---|
| **Holds** | Local variables, parameters, return addresses — one *frame* per method call | Every object, every array |
| **Who can see it** | Only its own thread | **All** threads |
| **Created/destroyed** | A frame is pushed on call and popped on return — automatic, instant | Objects are created by `new`; reclaimed by the GC at some later point |
| **Size** | Small: about 1 MB per thread by default (`-Xss`) | Large: up to a quarter of RAM by default (`-Xmx`) — about 1.9 GB on your machine |
| **Runs out →** | `StackOverflowError` | `OutOfMemoryError` |
| **Speed** | Very fast (just move a pointer) | Slower (allocate, track, collect) |

(There's a third area, **Metaspace**, holding class definitions and method
bytecode. You'll meet it only when someone reports `OutOfMemoryError: Metaspace`.)

```
void process() {
    int count = 3;                       // stack: the number 3, stored right here
    String name = "alice";               // stack: a reference -> heap: the String object
    List<String> items = new ArrayList<>();  // stack: a reference -> heap: the ArrayList
    items.add(name);                     // heap: the ArrayList now also points at the String
}
```

When `process()` returns, its frame is popped: `count`, `name` and `items` are
gone. The `ArrayList` and `String` are still in the heap, but nothing points at
them any more — they're now **garbage**, waiting for the collector (Part 5).

### Each thread has its own stack; all threads share the heap

This is the picture Phase 5 is built on. Run it — two threads run the same code;
`local` is private to each, `shared.value` is one object both can reach:

```java
public class ThreadStacks {
    static class Counter {
        int value;
    }

    public static void main(String[] args) throws Exception {
        Counter shared = new Counter();

        Runnable work = () -> {
            int local = 0;
            for (int i = 0; i < 1000; i++) {
                local++;
                shared.value++;
            }
            System.out.println(Thread.currentThread().getName()
                    + ": local=" + local
                    + "  shared.value=" + shared.value);
        };

        Thread t1 = new Thread(work, "worker-1");
        t1.start();
        t1.join();

        Thread t2 = new Thread(work, "worker-2");
        t2.start();
        t2.join();

        System.out.println("main sees shared.value=" + shared.value);
    }
}
```

Expected:

```
worker-1: local=1000  shared.value=1000
worker-2: local=1000  shared.value=2000
main sees shared.value=2000
```

`local` started again from 0 in worker-2 — it lives on that thread's own stack.
`shared.value` carried on from 1000 — it lives on the heap, and both threads hold a
reference to the same object.

> I made the threads run one after the other (`join()`) on purpose. Let them run
> *at the same time* and `shared.value++` goes wrong in a way that's a whole
> lesson of its own — a **race condition**, Phase 5. Today's point is only *where
> things live*.

> **Say this in an interview:** "Each thread has its own stack of frames holding
> locals and references; objects live on the heap, which all threads share. That's
> why local variables are inherently thread-safe and shared mutable objects aren't."

---

## Part 3 — References, and what "pass by value" really means

**In plain words:** the variable `list` is not the list. It's a slip of paper with
the list's address on it. Writing `List<String> other = list;` doesn't copy the
list — it copies the slip, so now two slips point at one list. When you pass `list`
to a method, Java photocopies the slip and hands the method the copy. The method
can walk to the address and rearrange the furniture (the caller will see that). But
if the method scribbles a *new address* on its photocopy, the caller's original
slip is untouched.

That's the entire "is Java pass-by-value or pass-by-reference?" debate. The honest
answer: **Java is always pass-by-value.** What gets copied, for an object, is the
reference.

Run it:

```java
import java.util.ArrayList;
import java.util.List;

public class PassByValue {
    static void mutate(List<String> list) {
        list.add("added by mutate");
    }

    static void reassign(List<String> list) {
        list = new ArrayList<>();
        list.add("added by reassign");
    }

    static void bump(int n) {
        n = n + 100;
    }

    public static void main(String[] args) {
        List<String> names = new ArrayList<>();
        names.add("original");

        mutate(names);
        System.out.println("after mutate:   " + names);

        reassign(names);
        System.out.println("after reassign: " + names);

        int count = 1;
        bump(count);
        System.out.println("after bump:     " + count);
    }
}
```

```
after mutate:   [original, added by mutate]
after reassign: [original, added by mutate]
after bump:     1
```

Draw what happens in `reassign`:

```
main's frame                       reassign's frame             heap
names ──────────────────────────────────────────────────────►  ArrayList A [original, added by mutate]
                                   list (copy of the slip) ──►  ArrayList A      ← at the moment of the call

   then:  list = new ArrayList<>()  rewrites only reassign's slip:

names ──────────────────────────────────────────────────────►  ArrayList A  (unchanged)
                                   list ───────────────────►  ArrayList B [added by reassign]   ← nobody else knows about B
```

When `reassign` returns, its frame is popped, nothing points at ArrayList B, and B
becomes garbage.

`bump` shows the same rule for primitives: the method got a copy of the number 1.
Changing the copy can't touch the caller's variable.

Three consequences you'll use daily:

- **`==` on references compares the slips, not the objects.** "Do these two point
  at the same object?" To ask "do these hold equal contents?" you need `.equals`
  (tomorrow, Day 22, derived from scratch).
- **`final` on a reference freezes the slip, not the object.** `final List<String>
  xs = new ArrayList<>(); xs.add("ok");` is legal; `xs = other;` is not.
- **A method that mutates its argument has a side effect the caller can't see in
  the signature.** That's why Day 38 pushes you toward immutable objects.

`null` is a slip with no address on it. Calling a method on it is the
`NullPointerException`: "you told me to walk to an address and there isn't one."

> **Say this in an interview:** "Java is strictly pass-by-value. For objects, the
> value being copied is the reference, so a method can mutate the object the caller
> sees, but reassigning its parameter only changes its own copy of the reference."

---

## Part 4 — Primitives vs wrappers, boxing, and the `Integer` cache

**In plain words:** the 8 primitive types (`int`, `long`, `double`, …) put the
actual number straight in the variable — no slip of paper, no warehouse trip. Fast
and small. But sometimes Java insists on an *object*: you can't put an `int` in a
`List`, because lists hold references. So each primitive has an object twin — `int`
↔ `Integer`, `long` ↔ `Long` — and Java converts between them automatically.
That automatic conversion is **autoboxing**, and "automatic" is exactly why its
costs are hidden.

```
List<Integer> xs = new ArrayList<>();
xs.add(5);          // you wrote an int; the compiler wrote xs.add(Integer.valueOf(5))
int y = xs.get(0);  // the compiler wrote xs.get(0).intValue()
```

| | `int` | `Integer` |
|---|---|---|
| Lives | Directly in the variable (stack, or inside an object's fields) | As an object on the heap; the variable holds a reference |
| Can be `null` | No | **Yes** — and unboxing a `null` throws `NullPointerException` |
| In generics (`List<…>`) | Not allowed | Required |
| Size | 4 bytes | ~16 bytes object + a reference to it |
| Compared with `==` | By value | By **reference** (see below) |

### The `Integer` cache surprise

Run this:

```java
public class IntegerCache {
    public static void main(String[] args) {
        Integer a = 127, b = 127;
        Integer c = 128, d = 128;
        System.out.println("127 == 127     : " + (a == b));
        System.out.println("128 == 128     : " + (c == d));
        System.out.println("128 equals 128 : " + c.equals(d));

        Integer e = Integer.valueOf(1000);
        Integer f = Integer.valueOf(1000);
        System.out.println("valueOf(1000) == valueOf(1000): " + (e == f));
    }
}
```

```
127 == 127     : true
128 == 128     : false
128 equals 128 : true
valueOf(1000) == valueOf(1000): false
```

Why: `Integer a = 127` compiles to `Integer.valueOf(127)`. Creating a fresh object for
every small number would be wasteful, so `Integer.valueOf` keeps a **cache** — one
pre-built `Integer` object for each value from **-128 to 127** — and hands out the
same object each time. So `a` and `b` are two slips pointing at the *same* cached
object: `==` is true. 128 is outside the cache, so each `valueOf(128)` makes a new
object; `c` and `d` are slips pointing at *different* objects with the same
contents: `==` is false, `.equals` is true.

The cache's upper bound can be raised with `-XX:AutoBoxCacheMax=…`, but that's
trivia — the lesson is **never compare wrappers with `==`**. (`Long`, `Short`,
`Byte` and `Character` have the same kind of cache.) This is a favourite
interview question because the bug it causes — a test with small numbers passes,
production with real IDs fails — is nasty.

### The hidden cost of boxing

Every boxed value outside the cache is a heap allocation. A loop that boxes
millions of times is allocating millions of objects for the GC to clean up. Run it
(your numbers will differ, the ratio is the point):

```java
public class BoxingCost {
    public static void main(String[] args) {
        int n = 30_000_000;

        long t0 = System.nanoTime();
        long primitiveSum = 0;
        for (int i = 0; i < n; i++) primitiveSum += i;
        long primitiveMs = (System.nanoTime() - t0) / 1_000_000;

        t0 = System.nanoTime();
        Long boxedSum = 0L;
        for (int i = 0; i < n; i++) boxedSum += i;
        long boxedMs = (System.nanoTime() - t0) / 1_000_000;

        System.out.println("same answer: " + (primitiveSum == boxedSum));
        System.out.println("primitive long: " + primitiveMs + " ms");
        System.out.println("boxed Long:     " + boxedMs + " ms");
    }
}
```

On this machine: 22 ms vs 208 ms — roughly 10×. The only difference is one word,
`Long` instead of `long`. Each `boxedSum += i` is: unbox, add, box a **new** `Long`
object, discard the old one.

When to care: tight loops, big numeric arrays (`int[]` is one flat block;
`List<Integer>` is a block of references to millions of separate objects), hot
paths. When not to: ordinary business code. Day 35 returns to this with primitive
streams (`IntStream`, `mapToInt`).

> **Say this in an interview:** "Autoboxing wraps a primitive in an object via
> `valueOf`. `Integer` caches -128 to 127, so `==` on small values accidentally
> works and on larger ones compares references and fails — always use `equals`.
> Boxing also allocates, so I avoid it in hot loops and prefer primitive types or
> primitive streams there."

---

## Part 5 — Garbage collection

**In plain words:** in C you must free every object yourself, and forgetting means
a leak and freeing twice means a crash. Java takes that job away: every so often
a **garbage collector** sweeps the warehouse and throws away objects that nobody
can possibly use again. How does it know? It starts from the things it trusts
(the slips on every thread's desk, plus static fields) and follows every slip to
every object, and every slip inside those objects, and so on. Anything it *can't*
reach by following slips is unreachable — and an unreachable object, by definition,
can never be used again, so it's safe to remove.

The precise version:

- **GC roots**: local variables and parameters on every live thread's stack,
  `static` fields, and a few JVM-internal references.
- **Reachable** = there's a chain of references from some root to the object.
- Everything else is garbage. Cycles don't save an object: two objects pointing at
  each other but unreachable from any root are both collected.

You can watch reachability happen. A `WeakReference` is a slip the GC is allowed to
ignore — it doesn't count as keeping the object alive — so it lets us ask "is the
object still there?":

```java
import java.lang.ref.WeakReference;

public class Reachability {
    public static void main(String[] args) throws Exception {
        byte[] data = new byte[1_000_000];
        WeakReference<byte[]> watcher = new WeakReference<>(data);

        System.gc();
        Thread.sleep(200);
        System.out.println("while 'data' still points at it: " + (watcher.get() != null ? "alive" : "collected"));

        data = null;
        System.gc();
        Thread.sleep(200);
        System.out.println("after data = null:               " + (watcher.get() != null ? "alive" : "collected"));
    }
}
```

```
while 'data' still points at it: alive
after data = null:               collected
```

Nothing about the object changed. The only difference is that the last *strong*
slip pointing at it was removed. (`System.gc()` is a polite request, not a command
— fine for a demo, never something to put in a service.)

### Generations: why the heap is split in two

Measured on real programs, **most objects die young**: a request object, a
temporary string, a loop's boxed numbers. A few live a long time: caches,
connection pools, singletons. Scanning the *whole* heap to find garbage would
waste time re-checking the long-lived ones. So the heap is split:

```
┌─────────── Young generation ───────────┐  ┌────── Old generation ──────┐
│ new objects are born here               │  │ objects that survived many │
│ (Eden + small "survivor" areas)         │─►│ young collections          │
│ collected OFTEN — and cheaply, because  │  │ collected RARELY           │
│ nearly everything in it is already dead │  │                            │
└─────────────────────────────────────────┘  └────────────────────────────┘
        "minor GC"                                  "major GC" / "full GC"
```

- **Minor GC**: cleans only the young area. Frequent, short. Survivors are copied to
  a survivor area; after surviving enough collections an object is **promoted** to
  old.
- **Major / full GC**: cleans the old area (a full GC, everything). Rarer and more
  expensive. If these start happening constantly, the app is in trouble.
- **Stop-the-world**: during parts of a collection, your application threads are
  paused. Modern collectors keep these to milliseconds, but they exist. A latency
  graph with periodic spikes is often GC.

Your JDK 21 uses **G1** by default (confirmed on your machine below). It divides
the heap into many equal regions rather than two big blocks, and tries to meet a
pause-time goal. Other collectors you'll hear of: **ZGC** (pauses of ~1 ms even on
huge heaps — `-XX:+UseZGC`), **Parallel** (throughput-oriented), **Serial** (tiny
heaps). Tuning them is a specialism; today you only need the vocabulary.

Watch minor collections happen:

```java
import java.lang.management.GarbageCollectorMXBean;
import java.lang.management.ManagementFactory;

public class Generations {
    static byte[] sink;

    static void report(String label) {
        System.out.println(label);
        for (GarbageCollectorMXBean gc : ManagementFactory.getGarbageCollectorMXBeans()) {
            System.out.println("  " + gc.getName() + ": " + gc.getCollectionCount() + " collections");
        }
    }

    public static void main(String[] args) {
        report("before:");

        long checksum = 0;
        for (int i = 0; i < 3_000_000; i++) {
            byte[] shortLived = new byte[1024];
            sink = shortLived;
            checksum += shortLived.length;
        }

        report("after allocating " + (checksum / 1_000_000) + " MB of short-lived garbage:");
    }
}
```

On this machine:

```
before:
  G1 Young Generation: 0 collections
  G1 Concurrent GC: 0 collections
  G1 Old Generation: 0 collections
after allocating 3072 MB of short-lived garbage:
  G1 Young Generation: 15 collections
  G1 Concurrent GC: 0 collections
  G1 Old Generation: 0 collections
```

3 GB of throwaway arrays went through a heap whose maximum is about 1.9 GB, using
only young collections — no old-generation work at all. That's the generational
idea working. (Your count will differ; it depends on heap size.) The `sink =`
line is deliberate — see Gotchas.

To see each collection as it happens, ask the JVM to log them (PowerShell):

```powershell
java -Xmx64m -Xlog:gc Generations
# [0.208s][info][gc] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 23M->0M(64M) 3.285ms
```

Read it as: heap went from 23 MB used to ~0 after the pause, out of 64 MB, and the
pause cost 3 ms. Everything that was in there was garbage.

> **Say this in an interview:** "The GC finds objects unreachable from the roots —
> thread stacks and static fields — and reclaims them. The heap is generational
> because most objects die young: minor collections of the young generation are
> frequent and cheap, and survivors are promoted to the old generation, which is
> collected rarely. G1 is the default; ZGC trades some throughput for very short
> pauses."

---

## Part 6 — Leaks in a garbage-collected language

**In plain words:** the GC throws away what nobody *can* reach. It has no idea
whether you *want* it any more. If you keep a slip pointing at something you'll
never look at again, the GC politely assumes you still care. A leak in Java isn't
"forgot to free" — it's "forgot to let go".

The classic shape: **a long-lived collection that only grows.** A `static` list,
map or cache is a GC root, so everything in it is reachable forever:

```java
import java.util.ArrayList;
import java.util.List;

public class Leak {
    static final List<byte[]> CACHE = new ArrayList<>();

    static long usedMb() {
        System.gc();
        Runtime rt = Runtime.getRuntime();
        return (rt.totalMemory() - rt.freeMemory()) / 1_000_000;
    }

    static void handleRequest(int id) {
        byte[] result = new byte[1_000_000];
        CACHE.add(result);
    }

    public static void main(String[] args) {
        System.out.println("baseline:                 " + usedMb() + " MB");

        for (int i = 0; i < 100; i++) handleRequest(i);
        System.out.println("after 100 'requests':     " + usedMb() + " MB   (all finished, none freed)");

        CACHE.clear();
        System.out.println("after CACHE.clear():      " + usedMb() + " MB");
    }
}
```

```
baseline:                 0 MB
after 100 'requests':     105 MB   (all finished, none freed)
after CACHE.clear():      0 MB
```

Every `handleRequest` call returned and its stack frame vanished, but the object it
created is still in `CACHE`, which hangs off a static field, which is a root. Not
one byte is reclaimable until something removes it.

Where this really happens:

| Leak | What keeps the reference |
|------|--------------------------|
| **Unbounded cache** | A `static Map` that you add to and never evict from — by far the most common |
| **Listeners / callbacks never removed** | The event source holds a list of them forever |
| **`ThreadLocal` in a thread pool** | Pool threads never die, so their `ThreadLocal` values never go (Phase 5) |
| **Per-request data stored in a long-lived object** | A singleton Spring bean with a `List` field that each request appends to |
| **Unclosed resources** | Streams, connections — usually leak *native* resources more than heap (Day 23) |
| **Class-loader leaks** | Redeploying an app in a server keeps the old classes alive — the Metaspace error |

How it shows up in production: memory climbs slowly, GC runs more and more often
and frees less each time, latency rises, and eventually `OutOfMemoryError`. The
shape on a memory graph is a sawtooth with a rising floor — each GC drops usage,
but never back to where it was.

Finding the culprit is a **heap dump**: a snapshot of every object and who points
at it. The two commands to remember:

```powershell
# take a snapshot of a running JVM (find its pid with: jcmd)
jcmd <pid> GC.heap_dump C:\temp\heap.hprof
# or have the JVM write one automatically at the moment it dies
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=C:\temp ...
```

Open the file in Eclipse MAT or VisualVM and look for the object type taking the
most memory, then follow "path to GC roots" to see what's holding it. In a
container, `-XX:+HeapDumpOnOutOfMemoryError` is worth setting on every service
before the incident, not after.

> **Say this in an interview:** "A GC only reclaims unreachable objects, so a leak
> is an object that's still reachable but no longer useful — typically an
> unbounded static cache, a listener never deregistered, or a ThreadLocal in a
> pooled thread. I'd confirm with a heap dump and follow the path to GC roots."

---

## Part 7 — `OutOfMemoryError` vs `StackOverflowError`

**In plain words:** two different containers, two different ways to overflow. The
**warehouse** (heap) is shared and huge; fill it with things you won't let go of,
and it runs out: `OutOfMemoryError`. Your **desk** (a thread's stack) is small and
private; start a task that starts a task that starts a task, forever, and the desk
overflows with sheets: `StackOverflowError`.

| | `StackOverflowError` | `OutOfMemoryError` |
|---|---|---|
| **Which memory** | One thread's stack (~1 MB) | The heap (or Metaspace, or native memory) |
| **Usual cause** | Recursion with no base case, or far too deep | A leak, a huge result set loaded into memory, a heap set too small for the load |
| **Affects** | The thread that overflowed | The whole JVM — any thread can be the one that trips it |
| **After it** | That thread unwinds; the rest of the JVM is usually fine | The JVM may be in a bad state; many teams kill and restart the process |
| **Fix** | Find the missing base case; turn deep recursion into a loop | Find the leak; stream instead of loading everything; raise `-Xmx` *last* |

Both are `Error`s, not `Exception`s — the JVM's way of saying "something is wrong
beyond what your code should handle". They're technically catchable, as the demos
do, but in real code you don't catch them and carry on.

**Show the stack failure** (recursion with no way out):

```java
public class Overflow {
    static int depth = 0;

    static void recurse() {
        depth++;
        recurse();
    }

    public static void main(String[] args) {
        try {
            recurse();
        } catch (StackOverflowError e) {
            System.out.println("StackOverflowError caught");
            System.out.println("depth reached > 1000: " + (depth > 1000));
            System.out.println("stack trace length: " + e.getStackTrace().length);
        }
        System.out.println("the JVM is still alive: only this thread's stack was exhausted");
    }
}
```

```
StackOverflowError caught
depth reached > 1000: true
stack trace length: 1024
the JVM is still alive: only this thread's stack was exhausted
```

Each call to `recurse()` pushed a frame; none ever returned; around 1 MB later
there was no room for another. The stack trace is capped at 1024 entries by the JVM
(`-XX:MaxJavaStackTraceDepth`) — the real depth is far greater, which is why
"read the bottom of the stack trace" doesn't always work with these. Real-world
version: a Jackson/JPA entity graph where `Order` has `Customer` and `Customer`
has `List<Order>`, and serialising it recurses forever. You'll meet this in Phase 7.

**Show the heap failure** — asking for an array bigger than the heap can ever hold:

```java
public class Oom {
    public static void main(String[] args) {
        try {
            long[] huge = new long[Integer.MAX_VALUE - 8];
            System.out.println("allocated " + huge.length);
        } catch (OutOfMemoryError e) {
            System.out.println("OutOfMemoryError: " + e.getMessage());
        }
        System.out.println("main carried on");
    }
}
```

```
OutOfMemoryError: Java heap space
main carried on
```

(That asks for ~17 GB in one piece. If you someday run this on a machine with more
than ~70 GB of RAM it will succeed — that's a property of the default heap size,
not of Java.)

The realistic version is the leak from Part 6, left to run until the heap is full.
Cap the heap at 16 MB so it happens in a blink (PowerShell, in the folder with a
`Fill.java` that loops `keep.add(new byte[1_000_000])`):

```powershell
java -Xmx16m Fill
# Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
#         at Fill.main(Fill.java:8)
```

The message after `OutOfMemoryError:` tells you which memory ran out:

| Message | Meaning |
|---------|---------|
| `Java heap space` | The heap is full — leak, or heap too small |
| `GC overhead limit exceeded` | The JVM is spending nearly all its time collecting and reclaiming almost nothing — a leak in its final stage |
| `Metaspace` | Too many/too large classes — a class-loader leak |
| `unable to create native thread` | The OS refused another thread — too many threads (Phase 5), not a heap problem |

> **Say this in an interview:** "`StackOverflowError` means one thread's stack
> overflowed, almost always unbounded recursion; the fix is a base case. 
> `OutOfMemoryError` means the heap — or Metaspace or native memory, depending on
> the message — is exhausted; I'd take a heap dump, find what's retaining the
> memory, and only raise `-Xmx` after ruling out a leak."

---

## Hands-on

Run each block above in the lab. The required ones are 1, 2 and 3.

1. **Required — `PassByValue`.** Run it. Then add a third method,
   `static void swap(StringBuilder a, StringBuilder b)` that tries to swap the two
   references, call it from `main`, and predict the output *before* running.
   Explain, in words and with a drawing of slips, why the caller's variables still
   point where they did.
2. **Required — `Overflow`.** Run it. Then change `recurse()` to take an `int`
   parameter and a `long` local variable and see whether `depth` at the point of
   failure goes up or down. Why would more locals change it? (Each frame is bigger,
   so fewer fit in the same ~1 MB.)
3. **Required — `IntegerCache`.** Run it, then explain 127 vs 128 in two sentences
   using the words *cache*, *reference* and `valueOf`. Then write
   `Long x = 127L, y = 127L;` and check `x == y`; and `Integer p = 127; Long q =
   127L;` and check `p.equals(q)`. (Second one: `false` — different types.)
4. `BoxingCost` — run it three times; confirm the ratio is stable even though the
   milliseconds aren't. Change `Long boxedSum` to `long boxedSum` and watch the gap
   vanish.
5. `ThreadStacks` — add a third thread. Predict the final `shared.value`.
6. `Reachability` — add a `String` that's a *strong* second reference
   (`byte[] backup = data;`) before `data = null`. Predict whether it's collected,
   then run it.
7. `Generations` — in PowerShell, run `java -Xmx64m -Xlog:gc Generations | more`.
   Count the young pauses and note the `->` numbers: used heap before and after.
8. `Leak` — change `CACHE` to a `Map<Integer, byte[]>` capped at 10 entries
   (evict the oldest when it grows past 10). Confirm memory stays flat. This is
   what every bounded cache does, and tomorrow's `LinkedHashMap` makes it a
   one-liner (Day 32).
9. `Oom` — change the array size to `Integer.MAX_VALUE` and note you get a
   *different* message (`Requested array size exceeds VM limit`). What's the
   difference between "too big for the heap" and "too big for the JVM at all"?
10. Find your heap limit and collector in PowerShell:
    `java -XX:+PrintFlagsFinal -version | Select-String "MaxHeapSize|UseG1GC "`.
    On this machine that shows about 1.97 billion bytes (1.9 GB) and G1 in use.

---

## Gotchas

- **`==` on objects is reference equality.** On strings, wrappers, anything.
  Habit: `.equals` for contents, `==` only for primitives, enums and genuine
  identity checks.
- **Unboxing a `null` is a `NullPointerException`.** `Integer count = map.get(k);
  int n = count;` blows up if the key is missing — and the stack trace points at
  an innocent-looking line with no method call on it.
- **`Integer` in a `switch` or arithmetic hides unboxing** and NPEs. If a field can
  be absent, think about why it's a wrapper.
- **`System.gc()` is a suggestion.** Don't call it in production code; it can
  trigger an expensive full collection. It's here only to make demos deterministic.
- **My first `Generations` attempt reported two collections for 3 GB of
  allocation.** The JIT noticed each 1 KB array never left the loop and
  *eliminated the allocation altogether* (escape analysis). Assigning to a static
  field made it "escape", so the allocations were real. Lesson: micro-benchmarks
  in Java lie unless you understand what the JIT is doing — which is why people
  use a proper harness (JMH) for performance claims.
- **Bigger heap isn't a fix for a leak.** It only delays the crash, and makes
  full-GC pauses longer.
- **Fields default, locals don't.** An object's fields get default values
  (`0`, `false`, `null`); a local variable must be assigned before use or the
  compiler refuses.
- **Heap size inside a container.** The JVM sizes its default heap from the
  *container's* memory limit, not the host's. You'll tune `-XX:MaxRAMPercentage`
  in Phase 4.

---

## Recap

- `javac` → bytecode → the JVM runs it (interpret first, JIT hot code). JDK ⊃ JRE ⊃ JVM.
- **Stack**: per thread, holds frames with locals and references; **heap**: shared,
  holds all objects. `StackOverflowError` = stack; `OutOfMemoryError` = heap
  (mostly).
- A variable holds a **reference**, not an object. Java passes **copies** of
  references: mutation is visible to the caller, reassignment isn't.
- Wrappers are objects: can be `null`, are compared by reference with `==`, cost an
  allocation. `Integer` caches -128..127.
- GC reclaims what's **unreachable from the roots**. Generational: minor GCs are
  cheap and frequent, major ones rare and costly.
- A leak is "still reachable, never used again" — unbounded static caches first.
  Diagnose with a heap dump and the path to GC roots.

Now answer the four questions at the top again, without scrolling.

---

## Say it out loud

Explain each to an imaginary colleague in about 30 seconds. If you stumble, that's
the part to re-read.

1. "Is Java pass-by-value or pass-by-reference?" — include what is copied.
2. "What's the difference between the stack and the heap, and which one does each
   thread get its own copy of?"
3. "Why does `Integer a = 128, b = 128; a == b` print false?"
4. "How can a memory leak happen in a language with a garbage collector?"
5. "What does it mean that the heap is generational, and why is it designed that way?"
6. "A service's memory use rises slowly for a week and then it crashes with
   `OutOfMemoryError`. What do you do?"

---

## Q&A

_Follow-up questions from this lesson are appended here._
