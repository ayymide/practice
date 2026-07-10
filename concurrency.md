# Lesson — `src/concurrency/` (the plumbing that lets threads talk)

My own reference for the `concurrency` directory. This one is the hardest in the
project, so I'm going to build the ideas up from *zero* — what a thread even is,
why sharing data between them is dangerous, and every concept the code leans on —
before dissecting each file line by line. Goal: I can be grilled on any line and
explain *why* it is the way it is.

---

## PART A — Foundations (read this before the files)

### A.1 What is a thread, and why do we want more than one?

A **process** is your running program. Inside it, a **thread** is one
independent line of execution — one "worker" stepping through code. By default
you have one thread (`main`). You can start more, and they all run *at the same
time* on different CPU cores.

Why bother? In this project, twenty simulated F1 cars need updating 50 times a
second, a UI needs drawing, strategy needs calculating, race-control events need
processing — all *simultaneously*. One thread doing everything in turn would
stutter. Multiple threads let the work happen in parallel.

The catch: threads in the same process **share the same memory**. That sharing
is the source of every hard bug in this directory.

### A.2 The core problem: the data race

Imagine two threads both running `counter = counter + 1` on a shared `counter`.
That one line is really *three* steps for the CPU:

```
1. read counter from memory into a register   (say it reads 5)
2. add 1 in the register                       (register now 6)
3. write the register back to memory           (writes 6)
```

Now interleave two threads:

```
Thread A: read 5
Thread B: read 5      <-- B read before A wrote
Thread A: add -> 6
Thread B: add -> 6
Thread A: write 6
Thread B: write 6     <-- should be 7! one increment vanished
```

Two increments happened but the counter only went up by one. This is a **data
race**: two threads touching the same memory at the same time, at least one
writing, with no coordination. In C++ a data race is **undefined behaviour** —
the standard says the program may do *literally anything* (crash, wrong answer,
work on Tuesdays only). We must prevent it.

There are two families of tools to prevent it, and this directory uses both:

1. **Locks** (mutexes) — "only one thread in this room at a time." Simple,
   general, but threads have to *wait*. Used by `ThreadPool`.
2. **Lock-free atomics** — carefully designed so threads never block each other,
   using special indivisible CPU instructions. Faster, far harder to get right.
   Used by `SpscQueue` and `MpscQueue`.

### A.3 `std::atomic<T>` — the indivisible variable

An `std::atomic<T>` is a variable whose reads and writes happen **all-at-once**
(atomically) — they can never be split the way the `counter` example was split.
A reader always sees either the fully-old or fully-new value, never a torn
half-written mess. It also provides special combined operations:

- **`.load(order)`** — atomically read the value.
- **`.store(value, order)`** — atomically write the value.
- **`.exchange(value, order)`** — atomically write a new value *and* return the
  old one, in one indivisible step. (Used heavily in `MpscQueue`.)
- **`.compare_exchange_weak/strong(...)`** — the "compare-and-swap" (CAS)
  seen in `race_state.h`.

Every one takes a `memory_order` argument. That's the next, and hardest, idea.

### A.4 Memory ordering — the concept that trips everyone up

Here is the surprise: **modern CPUs and compilers reorder your instructions.**
For speed, they'll run independent operations out of order, as long as *your own
single thread* can't tell the difference. On one thread this is invisible and
harmless. Across threads it is a nightmare, because another thread *can* see the
reordering.

Example. Thread A does:

```cpp
data = 42;              // (1) write the payload
ready = true;           // (2) raise a flag
```

Thread B does:

```cpp
while (!ready) {}       // (3) spin until the flag is up
use(data);              // (4) read the payload
```

You'd expect B to see `data == 42`. But the CPU is allowed to reorder A's (1)
and (2) — they look independent to A — so B might see `ready == true` while
`data` is *still the old value*. Broken.

**Memory ordering is how we forbid the harmful reorderings**, and *only* those,
so we don't pay for more synchronisation than we need. The two you must know
cold:

- **`memory_order_release`** on a **store** = "a one-way gate closing behind me.
  Everything I wrote *before* this store is guaranteed done and visible to
  anyone who later reads this atomic with *acquire*." (Put it on step 2, the
  flag-raise.)
- **`memory_order_acquire`** on a **load** = "a one-way gate opening in front of
  me. Once I read the value a *release* store published, I'm also guaranteed to
  see everything that writer did before it." (Put it on step 3, the flag-check.)

Together they form a **release/acquire pair** — a synchronisation "bridge"
between exactly two threads through one atomic variable:

```
Thread A                         Thread B
--------                         --------
data = 42;                       while(!ready.load(acquire)) {}
ready.store(true, release);  ─────►  (now data==42 is guaranteed visible)
use(data);   // safe: 42
```

The mantra: **release publishes, acquire subscribes.** Whenever you see a
release store in this code, hunt for the matching acquire load, and vice versa —
they always come in pairs.

Two more orders appear in the code:

- **`memory_order_relaxed`** — atomic (no torn reads) but **no ordering
  guarantee at all**. Use only when the value is self-contained and you don't
  care what else becomes visible. Cheapest.
- **`memory_order_acq_rel`** — for read-modify-write ops like `exchange`: it is
  an acquire *and* a release at the same time (subscribe to prior writes, and
  publish mine).

> **Interview soundbite:** "A data race is undefined behaviour. Atomics fix the
> torn-read problem; memory ordering fixes the *reordering/visibility* problem.
> Release/acquire is a one-way publish/subscribe bridge between two threads —
> everything before a release store is visible after the matching acquire load."

### A.5 Cache lines and "false sharing" (needed for `SpscQueue`)

CPUs don't fetch memory one byte at a time. They pull it in chunks called
**cache lines**, almost always **64 bytes**. A whole 64-byte line is the unit
the hardware tracks for "who owns this memory right now".

Here's the trap. Suppose two variables `x` and `y` sit next to each other in the
same 64-byte line, and Thread A writes `x` constantly while Thread B writes `y`
constantly. Even though they touch *different variables*, the hardware sees them
touching the *same cache line*, so it forces the line to bounce back and forth
between the two cores' caches, over and over. Each bounce costs ~100 cycles.
This is **false sharing** — the variables aren't logically shared, but they
share a cache line, so performance tanks as if they were.

The fix: **push the two hot variables onto separate cache lines** so each core
owns its own line. In C++ that's `alignas(64)` — "start this member on a fresh
64-byte boundary." You'll see exactly this in `SpscQueue`.

### A.6 What a ring buffer is (needed for `SpscQueue`)

A **ring buffer** (circular buffer) is a fixed-size array you treat as if its
end wraps around to its beginning — like seats around a round table. Two
indices track it:

- **`tail`** = where the producer writes next.
- **`head`** = where the consumer reads next.

```
index:   0   1   2   3   4   5   6   7      (Capacity = 8)
       [ . ][ A ][ B ][ C ][ . ][ . ][ . ][ . ]
             ^head                 ^tail
             (read here)           (write here)
```

The producer writes at `tail` then advances `tail`; the consumer reads at `head`
then advances `head`. When an index passes the end of the array it **wraps back
to 0**. `head == tail` means empty; if advancing `tail` would land on `head`,
it's full. No memory is ever allocated after construction — it's the same array
reused forever, which is exactly what you want on a hot path.

### A.7 Templates — `template<typename T>` (needed for all three files)

A **template** is a recipe for generating code, parameterised by *type*. Instead
of writing a separate queue class for `TelemetryFrame`, another for `int`,
another for `RaceControlEvent`, you write one:

```cpp
template<typename T>
class MpscQueue { ... T data ... };
```

`T` is a placeholder. When you write `MpscQueue<TelemetryFrame>`, the compiler
stamps out a real class with every `T` replaced by `TelemetryFrame`. `typename T`
just means "T is some type to be filled in later."

**Key practical consequence:** template code must be **fully visible to every
file that uses it**, so templates live *entirely in headers* (no `.cpp`). That's
why the queues are header-only, and why even `ThreadPool::submit` — a template
method — is defined in the header even though the rest of `ThreadPool` is in a
`.cpp`.

### A.8 Move semantics recap — `std::move` and `T&&`

Copying a big object is expensive. **Moving** transfers its guts (e.g. the
pointer to its heap data) to a new object and leaves the old one empty — cheap.

- `std::move(x)` doesn't move anything by itself; it just *casts* `x` to say
  "you may steal from me."
- `T&&` (double ampersand) is an **rvalue reference** — a parameter that binds
  to temporaries/movable things, signalling "I can take ownership of this."

You'll see both when items are pushed into the queues (steal the item in rather
than copy it) and in the thread pool's perfect forwarding.

---

## PART B — The files

Three components, each solving a different producer/consumer shape:

```
src/concurrency/spsc_queue.h   — 1 producer → 1 consumer, lock-free, bounded ring
src/concurrency/mpsc_queue.h   — N producers → 1 consumer, lock-free, unbounded list
src/concurrency/thread_pool.h  — many submitters → N workers, mutex + condition_variable
src/concurrency/thread_pool.cpp
```

The pattern to notice: **the more producers you have, and the more you care
about blocking, the more machinery you need.** SPSC is the simplest and fastest;
the thread pool is the most general but uses a lock.

---

## 1. `spsc_queue.h` — Single-Producer, Single-Consumer ring buffer

**Used for:** the telemetry generator (1 thread) feeding the display consumer
(1 thread). Exactly one writer, exactly one reader — the simplest possible
sharing, and it can be done *completely lock-free*.

### 1.1 The class header and its template

```cpp
template<typename T, std::size_t Capacity>
class SpscQueue {
```

Two template parameters: the element type `T`, and a compile-time size
`Capacity` (a *number*, not a type — a "non-type template parameter"). Because
`Capacity` is known at compile time, the buffer can be a fixed `std::array` with
zero heap allocation.

### 1.2 `static_assert` — checking rules at compile time

```cpp
static_assert(Capacity >= 2, "Capacity must be at least 2");
static_assert((Capacity & (Capacity - 1)) == 0,
              "Capacity must be a power of 2 (enables bitmask instead of modulo)");
```

A **`static_assert`** is a check the *compiler* runs; if it fails, the program
won't compile and you get that message. No runtime cost. Here it enforces two
things about `Capacity`:

- At least 2 (a 1-slot ring can't distinguish full from empty here).
- A **power of two** (2, 4, 8, 16, …). The trick `(Capacity & (Capacity - 1)) == 0`
  is the classic power-of-two test. A power of two in binary is a single 1 bit
  (`1000`); subtracting 1 flips it to all lower bits (`0111`); AND-ing them gives
  0. For any non-power-of-two it's non-zero. Why require it? See the mask next.

### 1.3 The bitmask instead of modulo

```cpp
static constexpr std::size_t MASK = Capacity - 1;
```

To wrap an index around a ring you normally do `index % Capacity` (modulo).
Division/modulo is comparatively slow. **If `Capacity` is a power of two**, then
`index % Capacity` gives the *exact same result* as `index & (Capacity - 1)` — a
single-cycle bitwise AND. That `Capacity - 1` is the `MASK`. For `Capacity = 8`,
`MASK = 7 = 0b111`, and `& 7` keeps only the low 3 bits, i.e. wraps 0–7. That's
the entire reason the power-of-two rule exists.

- **`static constexpr`**: `constexpr` = computed at compile time (a true
  constant); `static` = one shared copy for the class, not one per object.

### 1.4 The cache-line layout — the clever bit

```cpp
// ── Producer-owned cache line ──
alignas(64) std::atomic<std::size_t> tail_{0};
std::size_t head_cache_{0};

// ── Consumer-owned cache line ──
alignas(64) std::atomic<std::size_t> head_{0};
std::size_t tail_cache_{0};

std::array<T, Capacity> buffer_{};
```

This layout is deliberately engineered against false sharing (Part A.5):

- **`tail_`** is written only by the producer; **`head_`** only by the consumer.
  If they shared a cache line, every push and every pop would bounce that line
  between the two cores. **`alignas(64)`** forces each onto its *own* 64-byte
  cache line, so the producer's writes to `tail_` never invalidate the
  consumer's line and vice versa.
- **`head_cache_` / `tail_cache_`** are the real performance trick. Each side
  keeps a *private, non-atomic, possibly-stale copy* of the other side's index:
  - Producer owns `tail_` and caches the consumer's head in `head_cache_`.
  - Consumer owns `head_` and caches the producer's tail in `tail_cache_`.
  - In steady state (queue neither full nor empty) the producer can tell there's
    space *just by looking at its own cached copy* — it never has to read the
    consumer's atomic `head_` across the cache-line boundary. Cross-thread
    atomic traffic drops to nearly zero. It only refreshes the cache on the rare
    occasion the cache says "might be full/empty".

### 1.5 Deleting copy — `= delete`

```cpp
SpscQueue() = default;
SpscQueue(const SpscQueue&) = delete;
SpscQueue& operator=(const SpscQueue&) = delete;
```

- **`= default`** — "compiler, generate the ordinary default constructor for me."
- **`= delete`** — "make this operation a compile error if anyone tries it."
  Here it bans **copying** the queue. Copying a live concurrent queue makes no
  sense (which thread owns the copy? the atomics can't be meaningfully copied),
  so we forbid it at compile time. The two deleted lines are the **copy
  constructor** and the **copy assignment operator**.

### 1.6 `empty()` and `size_approx()`

```cpp
bool empty() const {
    return head_.load(std::memory_order_relaxed) ==
           tail_.load(std::memory_order_relaxed);
}
```

`head == tail` means empty. These use `relaxed` because they're only advisory
snapshots — by the time you act on the answer it may have changed anyway, so
there's no ordering to guarantee.

```cpp
std::size_t size_approx() const {
    const auto t = tail_.load(std::memory_order_relaxed);
    const auto h = head_.load(std::memory_order_relaxed);
    return (t - h) & MASK;
}
```

`(t - h) & MASK` is the number of items, with the `& MASK` handling the wrap-
around case where `tail` has wrapped past 0 but `head` hasn't yet. "Approx"
because in a running system the true size shifts moment to moment.

### 1.7 `push()` — the producer side, step by step

```cpp
bool push(const T& item) {
    const auto tail = tail_.load(std::memory_order_relaxed);
    const auto next = (tail + 1) & MASK;

    // Fast path: head_cache_ says there is space — no cross-thread read.
    if (next == head_cache_) {
        head_cache_ = head_.load(std::memory_order_acquire);
        if (next == head_cache_) return false; // truly full
    }

    buffer_[tail] = item;
    tail_.store(next, std::memory_order_release);
    return true;
}
```

Line by line:

1. **`tail_.load(relaxed)`** — read our *own* write index. `relaxed` is fine
   because only the producer writes `tail_`; there's no cross-thread ordering to
   establish on this read.
2. **`next = (tail + 1) & MASK`** — where `tail` will point *after* this push,
   wrapped around the ring with the mask.
3. **`if (next == head_cache_)`** — would advancing land on where we last knew
   the consumer to be? If so the ring *might* be full. This checks the cheap
   local cache first — the **fast path** does no cross-thread read at all.
4. If it might be full, **refresh** `head_cache_` from the real `head_` with an
   **acquire** load, then re-check. If it's *still* equal, the queue is genuinely
   full → `return false` (caller decides whether to drop or retry).
5. **`buffer_[tail] = item;`** — write the payload into the slot.
6. **`tail_.store(next, release)`** — publish the new tail. This **release**
   store is the critical one: it guarantees the `buffer_[tail] = item` write in
   step 5 is visible *before* the consumer can see the advanced tail. Without
   release, the consumer could see the new tail and read a slot that isn't
   written yet — the exact bug from Part A.4.

> **The release/acquire pair here:** producer's `tail_.store(release)` (step 6)
> pairs with the consumer's `tail_.load(acquire)` in `pop()`. That bridge is
> what makes the payload safely visible.

The second overload:

```cpp
bool push(T&& item) {
    ...
    buffer_[tail] = std::move(item);
    ...
}
```

Identical logic but takes `T&& item` (an rvalue reference) and uses
`std::move` to **steal** the item into the slot instead of copying it. Cheaper
when the caller passes a temporary. Having both overloads means callers get a
copy when they need to keep their object and a move when they don't.

### 1.8 `pop()` — the consumer side, the mirror image

```cpp
bool pop(T& out) {
    const auto head = head_.load(std::memory_order_relaxed);

    // Fast path: tail_cache_ says there is data — no cross-thread read.
    if (head == tail_cache_) {
        tail_cache_ = tail_.load(std::memory_order_acquire);
        if (head == tail_cache_) return false; // truly empty
    }

    out = buffer_[head];
    head_.store((head + 1) & MASK, std::memory_order_release);
    return true;
}
```

Exactly the producer logic mirrored:

1. Read our own `head_` (`relaxed` — only the consumer writes it).
2. `head == tail_cache_`? Cache says maybe empty. Fast path avoids the
   cross-thread read.
3. If maybe-empty, refresh `tail_cache_` from real `tail_` with **acquire**
   (this is the load that pairs with the producer's release store — now the
   payload the producer wrote is guaranteed visible). Re-check; if still equal,
   genuinely empty → `false`.
4. **`out = buffer_[head];`** — copy the item out to the caller's variable.
5. **`head_.store((head+1) & MASK, release)`** — publish that we've consumed the
   slot. This **release** pairs with the producer's `head_.load(acquire)` in
   `push()`, telling the producer "this slot is free again."

**Why is SPSC lock-free and correct with just these atomics?** Because there's
exactly *one* writer of `tail_`/`buffer_` slots (the producer) and exactly *one*
writer of `head_` (the consumer). No two threads ever write the same variable,
so there's no contended update to protect — only *visibility ordering* between
the two, which release/acquire handles. That single-writer-per-variable property
is the whole reason SPSC needs no CAS and no lock.

---

## 2. `mpsc_queue.h` — Multi-Producer, Single-Consumer linked queue

**Used for:** race-control events (track limits, weather, safety car) from *many*
producer threads into *one* UI consumer. Now multiple threads push at once, so
`tail_`-style "single writer" reasoning breaks — we need something that lets
producers *race safely*. This is a lock-free **linked list** (unbounded — grows
by allocating nodes, so it never reports "full").

### 2.1 The node and the two ends

```cpp
template<typename T>
class MpscQueue {
    struct Node {
        T data {};
        std::atomic<Node*> next {nullptr};
    };
    std::atomic<Node*> head_;  // insertion point; producers race to update it
    Node* tail_;               // consumption point; only the consumer touches it
```

- A **`Node`** holds one `data` item and an atomic **`next`** pointer to the
  following node. `Node*` is "pointer to a Node." `nullptr` = points at nothing.
- **`head_` is atomic** because *many producers* update it concurrently — it's
  the contended end.
- **`tail_` is a plain (non-atomic) pointer** because *only the single consumer*
  ever reads or writes it. No contention → no atomic needed. This is a
  deliberate optimisation, and the comment says exactly why.

> Naming caution: in *this* implementation `head_` is where new items are
> *inserted* (the producer end) and `tail_` is where items are *removed* (the
> consumer end) — the opposite of the SPSC naming. Don't let the names trip you;
> read them as "insertion end" and "consumption end."

### 2.2 The sentinel node — why start with an empty node

```cpp
MpscQueue() {
    Node* sentinel = new Node{};
    head_.store(sentinel, std::memory_order_relaxed);
    tail_ = sentinel;
}
```

The constructor creates one **sentinel** (dummy) node and points *both* ends at
it. A sentinel is a permanent placeholder that carries no real data; it exists so
the queue is **never truly "empty" of nodes**, even when it has no items. This
removes a whole class of edge cases — producers and the consumer never have to
special-case "the queue is completely empty and has no node to attach to." The
list always has at least the sentinel.

- **`new Node{}`** allocates a node on the **heap** and returns a pointer to it.
  (Heap = long-lived memory you manage yourself; must be freed with `delete`.)
- `relaxed` on the initial store is fine — no other thread exists yet during
  construction.

### 2.3 `push()` — the lock-free insert, and why `exchange` is the magic

```cpp
void push(T value) {
    Node* new_node = new Node{std::move(value)};

    // atomically swap head_ with new_node; prev_head is the old head
    Node* prev_head = head_.exchange(new_node, std::memory_order_acq_rel);

    // link the old head to the new node
    prev_head->next.store(new_node, std::memory_order_release);
}
```

This is the heart of the lock-free multi-producer design. The problem: ten
threads all want to add a node at the same instant. How do we let them without a
lock and without losing any?

The answer is **`exchange`** — an atomic read-modify-write that, *in one
indivisible step*, sets `head_` to `new_node` **and** hands back whatever `head_`
was before. Because it's indivisible, if ten threads call it at once, the
hardware serialises them: each gets a *distinct* `prev_head`, and they chain up
in some order with none lost or duplicated. No two threads can both think they
grabbed the same predecessor.

Step by step:

1. **`new Node{std::move(value)}`** — allocate the node, moving the caller's
   value into it (steal, don't copy).
2. **`head_.exchange(new_node, acq_rel)`** — atomically make `new_node` the new
   head and get the previous head back in `prev_head`. **`acq_rel`** because this
   is a read-modify-write: *acquire* so we see other producers' prior pushes,
   *release* so our node-creation write is published.
3. **`prev_head->next.store(new_node, release)`** — link the old head forward to
   our new node, so the list stays connected. **release** so that when the
   consumer follows `next` with an *acquire* load, it sees a fully-constructed
   node.

> **The subtle window:** between step 2 and step 3 the new node is the head but
> its predecessor's `next` doesn't point to it *yet*. For that instant the
> consumer can't reach the new node — it looks momentarily empty. That's not a
> bug; it's handled in `pop()` below (returns "nothing yet"), and the item
> appears as soon as step 3 lands.

### 2.4 `pop()` — the single consumer, and how the sentinel advances

```cpp
bool pop(T& out) {
    Node* next = tail_->next.load(std::memory_order_acquire);
    if (!next) return false;  // empty, or a push is mid-flight (between exchange and store)

    out = std::move(next->data);
    Node* old_tail = tail_;
    tail_ = next;          // next becomes the new sentinel
    delete old_tail;
    return true;
}
```

Only one thread ever runs this, so it needs no CAS. The mechanism:

1. **`tail_->next.load(acquire)`** — look at the node *after* the current
   sentinel. **acquire** pairs with the producer's `release` store on `next`, so
   if we see the link we also see the node's fully-written `data`.
2. **`if (!next) return false;`** — `next` is null means either the queue truly
   has no items, *or* a producer is mid-push (did its `exchange` but not yet its
   `next.store`). Either way there's nothing safe to hand out right now → return
   false.
3. **`out = std::move(next->data);`** — move the item out of that node into the
   caller's variable.
4. **The sentinel trick:** the *old* `tail_` (the previous sentinel, already
   consumed) is discarded, and the node we just read becomes the *new* sentinel.
   So the consumed node's slot lives on as the new dummy; the queue always keeps
   exactly one sentinel. `delete old_tail;` frees the retired node so we don't
   leak memory.

### 2.5 Destructor — draining and cleaning up

```cpp
~MpscQueue() {
    T ignored;
    while (pop(ignored)) {}   // drain everything, deleting each node
    delete tail_;             // tail_ is now the final sentinel
}
```

The **destructor** (`~ClassName`) runs when the object dies. It pops until empty
(each `pop` deletes a node) and then deletes the last remaining sentinel. Since
every `new` must be matched by a `delete`, this prevents a **memory leak**.

### 2.6 `empty()`

```cpp
bool empty() const {
    return tail_->next.load(std::memory_order_acquire) == nullptr;
}
```

Empty iff the sentinel has no `next`. Same acquire logic as `pop`.

> **Why can this be lock-free with many writers when SPSC couldn't handle it?**
> Because the one contended operation — claiming your spot in the list — is done
> by a *single atomic `exchange`*, which the hardware guarantees is indivisible.
> SPSC avoided contention by having one writer per variable; MPSC *embraces*
> contention but funnels it through one atomic instruction that can't be
> interrupted. That's the classic lock-free move: reduce the race to one atomic
> read-modify-write.

---

## 3. `thread_pool.h` / `.cpp` — the general worker pool (locks + condition variable)

**Used for:** strategy analysis, weather updates, race-control processing — jobs
you want run "somewhere, soon" without spawning a fresh thread each time.

A **thread pool** creates a fixed set of worker threads *once*, then feeds them a
queue of tasks. Creating threads is expensive; reusing a handful is cheap. This
one is *not* lock-free — it uses a `std::mutex` and a `std::condition_variable`,
because the general "many submitters, many workers, sleep when idle" shape is
exactly what those tools are designed for.

### 3.1 The members

```cpp
std::vector<std::thread>          workers_;   // the worker threads
std::deque<std::function<void()>> tasks_;     // FIFO queue of jobs to run
std::mutex                        mutex_;      // protects tasks_ and stop_
std::condition_variable           cv_;         // lets idle workers sleep/wake
bool                              stop_{false};// "time to shut down" flag
```

- **`std::vector<std::thread>`** — the owned worker threads.
- **`std::deque<std::function<void()>>`** — the task queue. A
  **`std::function<void()>`** is a type that can hold *any* callable taking no
  args and returning nothing (a lambda, function pointer, etc.) — a "job." A
  `deque` is a double-ended queue, giving cheap push-back / pop-front for FIFO
  order.
- **`std::mutex mutex_`** — the lock guarding shared state (`tasks_`, `stop_`).
- **`std::condition_variable cv_`** — explained in 3.3; it's how workers *sleep*
  when there's no work instead of burning CPU.
- **`stop_`** — set true on shutdown.

### 3.2 The mutex — mutual exclusion

A **`std::mutex`** ("mutual exclusion") is the "one thread in the room" lock.
A thread **locks** it before touching shared data and **unlocks** after; while
locked, any other thread trying to lock **waits**. This serialises access so no
data race can occur on `tasks_`.

In this code the mutex is always taken via RAII guards:

```cpp
std::unique_lock lock{mutex_};   // locks now, unlocks automatically at scope end
```

**`std::unique_lock`** locks on construction and unlocks when it goes out of
scope (the closing `}`), even if an exception is thrown. You never call
`unlock()` by hand — that's the RAII pattern (Resource Acquisition Is
Initialisation) protecting you from forgetting.

### 3.3 The condition variable — sleeping until work arrives

Here's the problem a mutex alone doesn't solve: when the queue is empty, what
should a worker do? *Spinning* (`while (tasks_.empty()) {}`) burns 100% CPU for
nothing. We want the worker to **sleep** and be **woken** the instant a task
arrives. That's precisely what a **`std::condition_variable`** does.

Look at the worker loop (`thread_pool.cpp`):

```cpp
void ThreadPool::worker_loop() {
    while (true) {
        std::function<void()> task;
        {
            std::unique_lock lock{mutex_};
            cv_.wait(lock, [this] {
                return stop_ || !tasks_.empty();
            });

            if (stop_ && tasks_.empty()) {
                return; // clean exit
            }

            task = std::move(tasks_.front());
            tasks_.pop_front();
        }
        task();   // run OUTSIDE the lock
    }
}
```

The star line is **`cv_.wait(lock, predicate)`**. It does three things atomically
so no wake-up is ever missed:

1. If the predicate `stop_ || !tasks_.empty()` is already true, carry on
   immediately.
2. Otherwise, **release the mutex and put this thread to sleep** — zero CPU used.
3. When another thread calls `cv_.notify_one()`/`notify_all()`, the sleeper
   **wakes, re-acquires the mutex, and re-checks the predicate.**

**Why the predicate (the lambda)?** Two reasons, both classic interview points:

- **Spurious wake-ups:** a condition variable is allowed to wake a thread for *no
  reason*. Re-checking the predicate on wake means a spurious wake just goes back
  to sleep instead of proceeding wrongly.
- **The "lost wake-up" race:** checking the condition and going to sleep must be
  one atomic step relative to a notifier setting the condition. `wait(lock, pred)`
  guarantees that, so a task added *just before* a worker sleeps isn't missed.

Then:

- **`if (stop_ && tasks_.empty()) return;`** — we only exit once we're both
  shutting down *and* the queue is drained, so no submitted task is dropped on
  shutdown.
- **`task = std::move(tasks_.front()); tasks_.pop_front();`** — take the front
  job (move, don't copy) and remove it. FIFO order.
- The task runs **outside the `{ }` scope**, i.e. after the lock is released. If
  we ran it while holding `mutex_`, no other worker could grab the next task —
  we'd have accidentally serialised everything. Releasing first lets all workers
  run tasks in parallel. This "hold the lock only to touch the queue, never while
  doing real work" rule is a key thread-pool design point.

### 3.4 Construction and destruction (`thread_pool.cpp`)

```cpp
ThreadPool::ThreadPool(std::size_t num_threads) {
    workers_.reserve(num_threads);
    for (std::size_t i = 0; i < num_threads; ++i) {
        workers_.emplace_back([this] { worker_loop(); });
    }
}
```

- **`explicit`** on the constructor (in the header) stops accidental implicit
  conversions like `ThreadPool p = 4;` — you must write `ThreadPool p{4};`,
  making intent clear.
- **`workers_.reserve(num_threads)`** pre-allocates the vector's storage so it
  doesn't reallocate mid-loop.
- **`emplace_back([this]{ worker_loop(); })`** constructs a `std::thread` that
  immediately starts running the lambda, which calls `worker_loop()`. `[this]`
  **captures** the pool pointer so the lambda can call the member function. So
  the constructor launches all N workers, and they all block in `cv_.wait` until
  work arrives.

```cpp
ThreadPool::~ThreadPool() {
    {
        std::unique_lock lock{mutex_};
        stop_ = true;
    }
    cv_.notify_all();          // wake every worker so they can see stop_
    for (auto& t : workers_) {
        t.join();
    }
}
```

Shutdown sequence — important to get right:

1. Lock, set `stop_ = true`, unlock. (Setting it under the lock ensures workers
   see it correctly ordered with the queue.)
2. **`cv_.notify_all()`** wakes *every* sleeping worker (not just one) so they
   all re-check the predicate, see `stop_`, drain remaining tasks, and return.
3. **`t.join()`** on each worker **waits for that thread to actually finish**
   before the destructor completes. This is essential: if the `ThreadPool` object
   were destroyed while threads were still running against its `mutex_`/`tasks_`,
   they'd touch freed memory → crash. `join()` guarantees a clean, orderly
   shutdown.

### 3.5 `submit()` — the template method, futures, and perfect forwarding

This is the densest function in the directory. It lets a caller hand in *any*
callable with *any* arguments and get back a **`std::future`** that will
eventually hold the result.

```cpp
template <typename F, typename... Args>
auto submit(F&& f, Args&&... args)
    -> std::future<std::invoke_result_t<F, Args...>>;
```

Decoding the signature:

- **`template <typename F, typename... Args>`** — `F` is the callable's type; the
  **`typename... Args`** is a **variadic template** — "zero or more type
  parameters." The `...` means "a pack." This is what lets `submit` accept any
  number of arguments of any types.
- **`F&& f, Args&&... args`** — these `&&` here are **forwarding references** (not
  plain rvalue refs): combined with template deduction they can bind to *either*
  lvalues or rvalues and remember which. `args...` expands the pack.
- **`auto ... -> std::future<...>`** is **trailing return type** syntax. `auto`
  up front says "the real return type is written after the arrow," which is
  needed because the return type depends on the template parameters.
- **`std::invoke_result_t<F, Args...>`** = "the type you'd get from calling `f`
  with those args." So if you submit a function returning `double`, you get back
  a `std::future<double>`.

The body:

```cpp
using ReturnType = std::invoke_result_t<F, Args...>;

auto task = std::make_shared<std::packaged_task<ReturnType()>>(
    [f = std::forward<F>(f), ...args = std::forward<Args>(args)]() mutable {
        return f(std::forward<Args>(args)...);
    }
);

std::future<ReturnType> result = task->get_future();

{
    std::unique_lock lock{mutex_};
    if (stop_) {
        throw std::runtime_error("submit() called on a stopped ThreadPool");
    }
    tasks_.emplace_back([task]() { (*task)(); });
}

cv_.notify_one();
return result;
```

Piece by piece:

- **`std::packaged_task<ReturnType()>`** wraps a callable so that when it runs,
  its return value is automatically stored where an associated **`std::future`**
  can retrieve it. Think of it as a task with a built-in "result mailbox."
- **`std::future<ReturnType>`** is the caller's handle to that mailbox. The
  caller can later call `result.get()`, which **blocks until the task has run on
  a worker and then returns its value** (or rethrows its exception). This is how
  you get a result *back* from work done on another thread — the whole point of
  the pool being usable for computations, not just fire-and-forget.
- **`std::make_shared<...>`** puts the packaged_task on the heap behind a
  **`shared_ptr`** (a reference-counted smart pointer that auto-deletes when the
  last owner goes away). Needed because `packaged_task` **cannot be copied**, but
  the lambda we put in the queue *is* copied around — capturing a cheap-to-copy
  `shared_ptr` to it sidesteps that.
- **The inner lambda `[f = std::forward<F>(f), ...args = ...] mutable`** captures
  the callable and each argument *by value into the lambda* using
  **`std::forward`** — this is **perfect forwarding**: it preserves whether each
  argument was an lvalue or rvalue so moves stay moves and copies stay copies. It
  bundles the "call `f(args...)`" into a zero-argument callable. `mutable` lets
  the lambda modify its captured copies (needed to move out of them).
- **`task->get_future()`** grabs the future *before* the task is queued, so we
  can hand it back to the caller.
- The `{ ... }` block locks the mutex, checks `stop_` (throwing if the pool is
  already shut down — you can't queue onto a dead pool), then
  **`tasks_.emplace_back([task]{ (*task)(); })`** pushes a plain `void()` wrapper
  that, when a worker runs it, invokes the packaged_task (which runs `f`, stores
  the result, and unblocks the future). The wrapper captures the `shared_ptr` so
  the task stays alive until it runs.
- **`cv_.notify_one()`** wakes *one* sleeping worker to pick it up (one task →
  one worker needed, so `notify_one`, not `notify_all`).
- **`return result;`** hands the future to the caller.

> **Why is `submit` in the header when the rest of `ThreadPool` is in the
> `.cpp`?** Because it's a **template** (Part A.7): the compiler must see its
> full body to stamp out a version for each `F`/`Args` combination a caller uses.
> Non-template methods (`ThreadPool(...)`, `~ThreadPool()`, `worker_loop()`) have
> fixed signatures and live in the `.cpp`.

---

## PART C — The big picture, and interview-ready comparisons

### C.1 Why three different tools? Match the shape to the job.

| | `SpscQueue` | `MpscQueue` | `ThreadPool` |
|---|---|---|---|
| Producers | 1 | many | many (submitters) |
| Consumers | 1 | 1 | many (workers) |
| Blocking? | lock-free, never blocks | lock-free, never blocks | uses mutex; workers *sleep* when idle |
| Storage | fixed ring (`std::array`) | linked list (grows) | `deque` of tasks |
| Bounded? | yes (can be "full") | no (unbounded) | no |
| Sync mechanism | release/acquire on 2 indices | atomic `exchange` + release/acquire | mutex + condition_variable |
| Cost when contended | ~none (no contention by design) | one atomic RMW per push | lock contention |
| Best for | steady 1:1 hot pipeline (telemetry) | bursty many→one events | arbitrary jobs needing results back |

### C.2 The single sentence for each concept

- **Data race** — two threads, same memory, ≥1 writing, no sync → undefined
  behaviour.
- **Atomic** — reads/writes that can't be torn; the smallest sync building block.
- **Release/acquire** — a one-way publish/subscribe bridge between two threads
  through one atomic; everything before a release store is visible after the
  matching acquire load.
- **Lock-free** — threads coordinate via atomics and never block each other;
  progress doesn't depend on any thread being scheduled.
- **False sharing** — unrelated variables on the same 64-byte cache line bounce
  between cores; fixed with `alignas(64)`.
- **Ring buffer** — fixed array with wrap-around head/tail; power-of-two size
  lets `& MASK` replace modulo.
- **Sentinel node** — a permanent dummy node so a linked queue is never
  node-empty, killing edge cases.
- **`exchange`** — atomic "swap in new, hand me the old" in one step; lets many
  producers link into a list without a lock.
- **Mutex** — one-thread-at-a-time lock for shared data.
- **Condition variable** — lets threads sleep until a condition holds; always
  wait with a predicate to survive spurious and lost wake-ups.
- **`future`/`packaged_task`** — the mechanism to get a *result* back from work
  run on another thread.
- **Template** — a code recipe parameterised by type; lives in headers.
- **Perfect forwarding** (`T&&` + `std::forward`) — pass arguments on through a
  wrapper preserving lvalue/rvalue-ness so moves stay moves.

### C.3 One paragraph to say out loud

> "The `concurrency` directory has three producer/consumer tools, each matched to
> a sharing shape. `SpscQueue` is a lock-free bounded ring buffer for the one-
> writer-one-reader telemetry pipeline: it needs no locks because each index has
> a single writer, so only release/acquire *visibility* between them matters, and
> it uses `alignas(64)` plus cached copies of the other side's index to avoid
> false sharing and cross-core traffic. `MpscQueue` handles many producers into
> one consumer with a lock-free linked list: producers claim their slot with a
> single atomic `exchange` on the head, and a sentinel node removes empty-queue
> edge cases. `ThreadPool` is the general case — a mutex plus a condition
> variable let a fixed set of worker threads sleep when idle and wake on new
> work, and `submit` uses variadic templates, perfect forwarding, a
> `packaged_task` and a `std::future` so callers can run arbitrary jobs on the
> pool and retrieve the result. The theme: the more concurrency you allow, the
> more machinery you need — lock-free for the fixed hot paths, a lock for the
> flexible general one."
