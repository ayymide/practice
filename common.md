# Lesson — `src/common/` (the shared data layer)

My own reference for the `common` directory: every file, every struct, every
line, plus the C++ keywords I keep tripping over as a newcomer. The goal is to
be able to explain any single line here out loud without hand-waving.

---

## 0. The big picture — what is `common/` for?

`common/` is the **shared vocabulary** of the whole project. It holds no logic
that *runs* on its own; it holds the *type definitions* and *data* that every
other subsystem (`simulation`, `strategy`, `race_control`, `concurrency`)
agrees on. If two threads or two modules need to talk about "a driver", "a
telemetry frame", or "the leaderboard", the definition lives here so everyone
means the same thing.

Four files, and they build on each other in this order:

```
src/common/types.h        — the raw building-block structs & enums
src/common/season_data.h  — the actual 2025 F1 data (drivers, cars, track)
src/common/race_state.h   — global flags shared across threads (atomics)
src/common/leaderboard.h  — thread-safe sorted standings (shared_mutex)
```

`types.h` is the foundation. `season_data.h` fills those types with real
values. `race_state.h` and `leaderboard.h` are the two *concurrency-aware*
containers — they are where the threading concepts (atomics, memory ordering,
reader/writer locks) actually show up.

> These are all **header files** (`.h`). In C++ a header is text that gets
> pasted into every `.cpp` that `#include`s it, *before* the compiler runs.
> Because these files are almost entirely type definitions and tiny functions,
> they live entirely in headers — there is no matching `.cpp`.

---

## 1. Two C++ preliminaries you meet on line 1 of every file

### `#pragma once`

```cpp
#pragma once
```

Every file here starts with this. A header can end up `#include`d more than
once in the same compile (e.g. `leaderboard.h` includes `types.h`, and so does
`main.cpp`, which also includes `leaderboard.h` — so `types.h` arrives twice).
Defining the same struct twice is a compile error. `#pragma once` tells the
compiler "if you've already seen this file in this translation unit, skip it."

It is the modern one-line replacement for the old "include guard" pattern:

```cpp
#ifndef TYPES_H      // the old way — you'll see this in older code
#define TYPES_H
// ...
#endif
```

### `#include`

```cpp
#include <string>          // angle brackets  = standard/system library
#include "common/types.h"  // double quotes   = our own project files
```

`#include` literally copies the named file's text in at that point. Angle
brackets `<...>` search the system/standard include paths; double quotes
`"..."` search the project first. So `<string>` pulls in the standard library
string type; `"common/types.h"` pulls in our own header.

---

## 2. `types.h` — the building blocks

This file defines the plain data structures and the enums. No threading here
yet, just "what shape is the data".

### 2.1 `struct` vs `class` (the newcomer's first question)

```cpp
struct TelemetryFrame { ... };
```

`struct` and `class` in C++ are *almost the same thing*. The only difference:
in a `struct`, members are **public** by default; in a `class`, they are
**private** by default. Convention in this codebase (and most codebases):

- `struct` = a bag of data, everything public, no invariants to protect.
- `class` = something with behaviour and internal state to hide (see
  `Leaderboard` later, which is a `class`).

### 2.2 `TelemetryFrame` — one snapshot of one car

```cpp
struct TelemetryFrame {
    char driver_id[4] {};   // "VER\0" — fixed size avoids heap alloc on hot path
    int lap {0};
    int sector {1};  // 1 or 2 or 3
    float speed_kph {0.0f};
    float throttle {0.0f};
    float brake {0.0f};
    float tire_wear {0.0f};
    float fuel_kg {100.0f};  //remaining
    bool drs_active {false};
    bool in_pit {false};
    float gap_to_leader {0.0f};
};
```

This is the single most performance-sensitive struct in the project — one is
produced per car, per tick, 50 times a second. Line by line:

- **`char driver_id[4] {};`** — a fixed C-style array of 4 characters, e.g.
  `'V','E','R','\0'`. Why not `std::string`? A `std::string` usually allocates
  memory on the *heap*. This struct is created thousands of times a second on
  the "hot path", and heap allocation is slow. A fixed 4-byte array lives
  *inline* inside the struct with zero allocation. The 4th byte holds the
  `'\0'` (null terminator) so it can be treated as a C string.
  - The `{}` at the end is **default member initialisation** — it
    zero-initialises the array (all `'\0'`). Without it the array would contain
    unpredictable garbage.
- **`int lap {0};`** — the `{0}` is again default initialisation, using
  **brace/uniform initialisation** syntax. Every member here has a default so
  a freshly-made `TelemetryFrame` is always in a known state.
- **`float speed_kph {0.0f};`** — the `f` suffix on `0.0f` makes it a `float`
  literal (single precision) rather than a `double` (`0.0` with no suffix is a
  `double`). Matching the suffix to the member type avoids a silent
  narrowing conversion.
- **`bool drs_active {false};`** — DRS = the overtaking-aid rear wing flap.
  A plain boolean flag.

> **Why `float` and not `double`?** `float` is 4 bytes, `double` is 8. For
> telemetry we don't need 15 digits of precision, and smaller structs mean more
> fit in cache and copy faster. This is a deliberate size/speed choice.

### 2.3 The profile structs — `DriverProfile`, `CarProfile`, `TrackProfile`

These are the *slow-changing configuration* data (loaded once, rarely copied on
the hot path), so here `std::string` is fine.

```cpp
struct DriverProfile {
    std::string id;
    std::string name;
    std::string team;
    std::string color;
    float aggression;   // how hard they push
    float consistency;  // lap to lap variation
    float tire_mgmt;    // how gently they treat tires
    float risk_tolerance;   // likelihood of mistakes
};
```

The four `float`s are **behavioural knobs** in the range roughly 0.0–1.0 that
the simulation reads to decide how a driver behaves. Note these have *no*
`{...}` default — they'll be filled in explicitly by `season_data.h`.

```cpp
struct CarProfile {
    std::string team;
    float engine_power;     // top speed multiplier
    float aero_efficiency;  // cornering performance
    float cooling;          // tire temperature management
    float reliability;      // DNF probability inverse
};
```

Per-team car performance. "DNF probability inverse" = higher reliability means
*less* chance of a Did-Not-Finish breakdown.

```cpp
struct TrackProfile {
    std::string name;
    float length_km;
    int sectors {3};
    float tire_deg_factor {1.0f};   // multiplier on tire wear rate
    float fuel_consumption{2.5f};   // kg per km
};
```

Here some members *do* have defaults (`sectors {3}`, etc.) because a sensible
default track is 3 sectors with a neutral wear multiplier.

### 2.4 `enum class` — the strongly-typed enumeration

```cpp
enum class PenaltyState {
    NONE,
    PENDING,
    SERVING,
    SERVED
};

enum class WeatherState {
    DRY,
    DAMP,
    WET
};
```

An `enum` is a named set of integer constants. The **`class`** part
(`enum class`, also called a "scoped enum", C++11 onwards) matters a lot:

- **Scoped**: you must write `PenaltyState::PENDING`, not just `PENDING`. This
  prevents name clashes — both `PenaltyState` and another enum could have a
  `NONE` without colliding.
- **Strongly typed**: it will *not* silently convert to an `int`. A plain
  old `enum` lets you accidentally do `if (state == 2)` or mix two enums
  together; `enum class` forbids that, catching bugs at compile time.

Under the hood `NONE=0, PENDING=1, SERVING=2, SERVED=3`, but you should never
rely on the numbers — use the names.

### 2.5 `RaceControlEvent` — a nested enum and a timestamp

```cpp
struct RaceControlEvent {
    enum class Type {
        TRACK_LIMITS,
        PENALTY_ISSUED,
        PIT_ENTRY,
        PIT_EXIT,
        WEATHER_CHANGE,
        SAFETY_CAR,
        FASTEST_LAP,
        RADIO_MESSAGE
    };

    Type type;
    char driver_id[4] {};
    int lap {0};
    std::string message {};
    std::chrono::steady_clock::time_point timestamp {std::chrono::steady_clock::now()};
};
```

- The `enum class Type` is **nested inside** the struct, so its full name is
  `RaceControlEvent::Type::SAFETY_CAR`. Nesting keeps related things together
  and signals "this Type only makes sense in the context of an event".
- **`std::chrono::steady_clock::time_point`** — a point in time. `std::chrono`
  is the standard time library. `steady_clock` is a clock that only ever moves
  forward at a constant rate (unlike a wall clock, it can't jump backwards when
  the system time is adjusted) — exactly what you want for measuring durations.
- **`{std::chrono::steady_clock::now()}`** — the default value is "the moment
  this event is constructed". So an event auto-stamps itself with the current
  time unless you override it.

### 2.6 `DriverState` — the full per-driver record

```cpp
struct DriverState {
    DriverProfile profile;
    CarProfile car;
    TelemetryFrame latest_frame;

    // race bookkeeping
    PenaltyState penalty_state {PenaltyState::NONE};
    int penalty_warnings {0};
    int position {0};
    int pit_stops {0};
    float best_lap_ms {0.0f};

    // internal simulation by TelemetryGenerator
    float distance_in_lap {0.0f};
    bool in_pit {false};
    int pit_timer_ticks {0};
    bool has_completed_pit {false};
    std::chrono::steady_clock::time_point lap_start {std::chrono::steady_clock::now()};
};
```

This is the **complete live state of one driver** and the thing the
`Leaderboard` stores 20 of. It *composes* the smaller structs:

- `profile` + `car` = the static config (who they are, what they drive).
- `latest_frame` = the most recent telemetry snapshot.
- The "race bookkeeping" block = facts the whole system cares about
  (position, penalties, best lap).
- The "internal simulation" block = scratch variables only the
  `TelemetryGenerator` uses to advance the sim (how far into the lap, pit
  timing). Grouped and commented so a reader knows these are private-ish
  implementation details even though the struct exposes them.

`best_lap_ms` is a lap time in milliseconds; `0.0f` means "no lap set yet".

---

## 3. `season_data.h` — the real data, and `inline` / `constexpr`

This file holds the actual 2025 grid. The star keyword here is **`inline`**.

### 3.1 Why `inline` on these variables?

```cpp
inline const std::array<DriverProfile, 20> DRIVERS = {{ ... }};
inline const std::array<CarProfile, 10>    CARS    = {{ ... }};
inline const TrackProfile DEFAULT_TRACK = { ... };
```

Here is the problem `inline` solves. This is a **header**, so its text is copied
into *every* `.cpp` that includes it. Without `inline`, each `.cpp` would create
its *own* separate `DRIVERS` variable, and when the linker joins all the `.cpp`
files together it would see multiple definitions of the same global and error
with "duplicate symbol".

**`inline` on a variable (C++17)** tells the linker: "these are all the *same*
one variable — collapse the duplicates into a single shared definition." So one
`DRIVERS` array exists across the whole program, even though the definition
physically lives in a header.

> This is a *different* meaning of `inline` from the old one. Historically
> `inline` on a *function* was a hint to "paste the function body at the call
> site instead of doing a call" (a speed optimisation). Modern compilers
> mostly ignore that hint and decide for themselves. Today the *linker*
> meaning — "one shared definition allowed in multiple files" — is the reason
> we actually write `inline`. See `TIME_SCALE` etc. in
> `telemetry_generator.h` for the same trick on constants.

### 3.2 `const` and `std::array`

```cpp
inline const std::array<DriverProfile, 20> DRIVERS = {{ ... }};
```

- **`const`** = this data can never be modified after initialisation. The grid
  is fixed reference data, so making it `const` lets the compiler stop any
  accidental writes and lets it place the data in read-only memory.
- **`std::array<DriverProfile, 20>`** = a fixed-size array of exactly 20
  `DriverProfile`s. Unlike `std::vector` (which is resizable and heap-backed),
  `std::array` is a fixed compile-time size with no heap allocation — the
  right choice when the count is known and never changes (20 drivers, 10 cars).
- **The double braces `{{ ... }}`** look odd. A `std::array` is a struct that
  *wraps* a raw C array inside it, so the outer `{}` initialises the
  `std::array` and the inner `{}` initialises the C array it contains. You can
  often drop one pair, but writing both is the unambiguous, always-correct
  form.

### 3.3 The driver rows

```cpp
{"VER", "Max Verstappen", "Red Bull", "\033[34m", 0.85f, 0.95f, 0.80f, 0.70f},
```

Each row is a `DriverProfile` built by **aggregate initialisation** — the
values fill the struct's members *in declaration order*: `id`, `name`, `team`,
`color`, `aggression`, `consistency`, `tire_mgmt`, `risk_tolerance`.

- **`"\033[34m"`** — this is an **ANSI colour escape code**, not a normal
  string. `\033` is the octal escape for the ESC character (decimal 27); `[34m`
  tells the terminal "switch to blue text". Printing this before a driver's
  name colours their output in the terminal leaderboard. (`\033[0m` elsewhere
  resets back to default.) So each team gets its brand colour.
- The four floats are the behavioural knobs from `DriverProfile`. Verstappen:
  high consistency (0.95), high aggression (0.85) — as you'd expect.

### 3.4 The `car_for_driver` helper

```cpp
inline const CarProfile& car_for_driver(const DriverProfile& d) {
    for (const auto& car : CARS) {
        if (car.team == d.team) return car;
    }
    return CARS[9]; // fallback: Sauber (last place)
}
```

A small lookup: given a driver, find the car for their team. Newcomer-relevant
bits:

- **`inline` on a function** here is again the "allowed in a header, one shared
  definition" meaning — required because the function body is defined in a
  header included by many files.
- **Return type `const CarProfile&`** — the `&` means it returns a
  **reference** (an alias to the existing array element), *not a copy*. So no
  `CarProfile` is duplicated; the caller just borrows a view of the one in
  `CARS`. The `const` means the caller can't modify it through that reference.
- **`const DriverProfile& d`** — the parameter is taken *by const reference*
  too: we borrow the caller's driver without copying it, and promise not to
  change it. Passing big structs by `const&` instead of by value is the
  standard way to avoid needless copies.
- **`for (const auto& car : CARS)`** — a **range-based for loop**. `auto` asks
  the compiler to deduce the element type (`CarProfile`); `const auto&` means
  "each element, borrowed by const reference, no copy". Reads as
  "for each car in CARS".
- **`return CARS[9];`** — a defensive fallback so the function *always* returns
  something valid even if no team matched, instead of running off the end.

---

## 4. `race_state.h` — global flags across threads (`std::atomic`)

Now the concurrency starts. `RaceState` holds a handful of flags that **many
threads read and write at the same time**. The whole file is a lesson in
`std::atomic` and *memory ordering*.

### 4.1 Why `std::atomic` at all?

If two threads touch the same plain `bool` or `int` — one writing, one
reading — that is a **data race**, which in C++ is *undefined behaviour* (the
program is allowed to do anything). `std::atomic<T>` makes individual
reads/writes **indivisible**: a reader always sees either the old value or the
new value, never a half-written mess, and the compiler/CPU won't tear or
reorder the access in illegal ways.

```cpp
#include <atomic>
#include <limits>
```

`<atomic>` for `std::atomic`; `<limits>` for `std::numeric_limits` (used to get
"the largest possible float" as an initial fastest-lap time).

### 4.2 Memory ordering — the hard part, in plain terms

Every atomic operation in this file passes a `std::memory_order_*` argument.
This controls **how much other, non-atomic work is guaranteed visible** to
other threads around this atomic access. The header itself has an excellent
reference block; the short version:

- **`memory_order_relaxed`** — atomic, but *no* guarantee about the ordering of
  surrounding writes. Use only when the value alone matters and you don't care
  what else the other thread can see. (Here: `current_lap`, purely
  informational.)
- **`memory_order_release`** (on a store) — "publish everything I did before
  this." All my previous writes become visible to a thread that does an
  *acquire* load of the same atomic.
- **`memory_order_acquire`** (on a load) — "show me everything the writer did
  before their release." Pairs with a release store.
- **`memory_order_acq_rel`** — both at once, for read-modify-write operations
  (like compare-and-swap).
- **`memory_order_seq_cst`** — the strictest and slowest: one single global
  order all threads agree on. Rarely needed; not used here.

The mental model: a **release store** and a matching **acquire load** form a
one-way "everything before the store is visible after the load" bridge between
two threads.

### 4.3 `race_active` — the simplest release/acquire pair

```cpp
std::atomic<bool> race_active{false};

void start_race() { race_active.store(true,  std::memory_order_release); }
void end_race()   { race_active.store(false, std::memory_order_release); }
bool is_active()  { return race_active.load(std::memory_order_acquire);  }
```

- Written **once** at the start (true) and **once** at the end (false). Every
  thread loops on `is_active()` to know when to stop.
- **`.store(value, order)`** writes; **`.load(order)`** reads — the atomic API,
  not the `=`/read you'd use on a plain variable.
- Release on store + acquire on load means: when a worker thread first sees
  `race_active == true`, it is *also* guaranteed to see all the simulation
  setup the main thread did before calling `start_race()`. That's why it isn't
  `relaxed`.

### 4.4 `current_lap` — where `relaxed` is genuinely fine

```cpp
std::atomic<int> current_lap{1};

void set_lap(int lap)  { current_lap.store(lap, std::memory_order_relaxed); }
int  get_lap()   const { return current_lap.load(std::memory_order_relaxed); }
```

The lap number is just a display value. If a UI momentarily shows lap N-1
instead of N, nothing breaks. There's no other data whose visibility must be
tied to the lap number, so we use the cheapest ordering, `relaxed`.

- Note **`const`** on `get_lap() const` — this method promises not to modify
  the `RaceState` object, so it's safe to call on a `const RaceState`.

### 4.5 `safety_car` — one writer, many readers ("atomic broadcast")

```cpp
std::atomic<bool> safety_car{false};

void deploy_safety_car()  { safety_car.store(true,  std::memory_order_release); }
void retract_safety_car() { safety_car.store(false, std::memory_order_release); }
bool is_safety_car() const { return safety_car.load(std::memory_order_acquire); }
```

One race director writes it; all 20 driver threads read it every tick. This is
a lock-free broadcast: no mutex, just an atomic flag with release/acquire so
readers see the flag *and* any state the director set before raising it.

### 4.6 `try_claim_fastest_lap` — the compare-and-swap (CAS) loop

This is the trickiest and most interesting function in the file: several driver
threads might beat the fastest lap *at the same instant*, and exactly one must
win the claim without a lock.

```cpp
std::atomic<int>   fastest_lap_holder{-1};
std::atomic<float> fastest_lap_ms{std::numeric_limits<float>::max()};

bool try_claim_fastest_lap(int driver_index, float lap_ms) {
    float current_best = fastest_lap_ms.load(std::memory_order_acquire);
    while (lap_ms < current_best) {
        if (fastest_lap_ms.compare_exchange_weak(
                current_best, lap_ms,
                std::memory_order_acq_rel,
                std::memory_order_acquire))
        {
            fastest_lap_holder.store(driver_index, std::memory_order_release);
            return true;
        }
        // current_best was refreshed by the failed CAS; loop and re-check.
    }
    return false;
}
```

Walk through it:

- **`fastest_lap_ms` starts at `std::numeric_limits<float>::max()`** — the
  biggest possible float, so *any* real lap time beats it initially. `-1` in
  `fastest_lap_holder` means "nobody yet".
- **Load the current best** (acquire), then loop only while my lap is faster.
- **`compare_exchange_weak(expected, desired, ...)`** is CAS, the heart of
  lock-free programming. It atomically does: "*if* `fastest_lap_ms` still
  equals `current_best`, set it to `lap_ms` and return true; *otherwise* write
  the real current value back into `current_best` and return false."
  - If it **succeeds**, I won the race — I then publish my `driver_index` as
    the holder and return `true`.
  - If it **fails**, another thread got in first between my load and my CAS.
    But `current_best` has now been updated to *their* value, so the `while`
    condition re-checks whether I'm *still* faster than the new best, and
    either retries or gives up. No lock, no lost update.
- **`_weak`** may fail *spuriously* (fail even when the values matched) on some
  hardware — that's fine and cheaper *inside a loop* like this, because we just
  retry. (`compare_exchange_strong` is the version to use when you're *not*
  already looping.)
- The **two memory orders**: `acq_rel` on success (I both read the old value
  and publish my writes), `acquire` on failure (I only need to see the current
  winner's value).

### 4.7 `extern` — the last line of the file

```cpp
extern RaceState g_race_state;
```

This is the keyword the user specifically asked about, so in detail:

- A normal variable definition like `RaceState g_race_state;` **allocates
  storage** for the object. If that line were in a header included by many
  `.cpp` files, every one would allocate its own `g_race_state` → duplicate
  symbol linker error (the same problem `inline` solved for `DRIVERS`).
- **`extern`** means "**declaration only**, not a definition." It tells the
  compiler: "a variable called `g_race_state` of type `RaceState` exists
  *somewhere* in the program — trust me, don't allocate storage for it here."
- Then **exactly one** `.cpp` file must provide the actual definition:
  ```cpp
  RaceState g_race_state;   // in one .cpp, exactly once
  ```
  This is the classic way to share a single global across many files: `extern`
  declaration in the header, one real definition in a source file.

> As the comment in the file notes, that one definition is *not yet wired into*
> `main.cpp` — it's deliberately deferred, like the other not-yet-integrated
> components. So `g_race_state` is declared and ready but not actually
> instantiated across the program yet.
>
> `extern` vs `inline` for globals: `inline` (used for `DRIVERS`) lets the
> *definition itself* live in the header and merges duplicates. `extern` keeps
> the definition *out* of the header and points at one defined elsewhere. Both
> avoid the duplicate-symbol problem, by opposite strategies.

---

## 5. `leaderboard.h` — thread-safe standings (`std::shared_mutex`)

`RaceState` used lock-free atomics for tiny flags. The leaderboard stores a
*whole vector of 20 `DriverState`s*, which is too big to make a single atomic,
so it uses a **lock** instead — specifically a reader/writer lock.

### 5.1 The access pattern that motivates the design

- **One writer**: the `TelemetryGenerator` thread, updating the standings about
  once per lap (rare).
- **Many readers**: a future UI render loop and status queries, reading often
  (~10 times a second) but briefly.

### 5.2 `std::shared_mutex` — single writer, multiple readers (SWMR)

```cpp
#include <shared_mutex>
```

A plain `std::mutex` allows **exactly one** thread in at a time — even two
readers who aren't changing anything would have to queue. A
**`std::shared_mutex`** has two modes:

- **`std::unique_lock`** → *exclusive*: blocks everyone else (used by the
  writer).
- **`std::shared_lock`** → *shared*: any number of readers can hold it at once,
  in parallel; only a writer wanting the exclusive lock is blocked.

So multiple UI reads proceed simultaneously, and only the once-per-lap write
briefly stops the world. The header comment is honest that `shared_mutex` has
*more* overhead than a plain mutex under contention — it's the right trade here
because reads are frequent and short and writes are rare.

### 5.3 `class`, `public`, `private`

```cpp
class Leaderboard {
public:
    // ... the safe API ...
private:
    std::vector<DriverState>  standings_;
    mutable std::shared_mutex mutex_;
};
```

This is a **`class`** (not `struct`) precisely because it has an *invariant to
protect*: "the standings must never be touched without holding the mutex." So:

- **`public:`** — the API anyone may call (`update`, `snapshot`, `size`, …).
- **`private:`** — the raw data (`standings_`) and the lock (`mutex_`), hidden
  so no outside code can read the vector without going through a locking method.
- The trailing underscore `standings_`/`mutex_` is just this codebase's naming
  convention for "private member".

### 5.4 RAII locking — locks that release themselves

```cpp
void update(std::vector<DriverState> sorted_standings) {
    std::unique_lock lock{mutex_};
    standings_ = std::move(sorted_standings);
}
```

- **`std::unique_lock lock{mutex_};`** locks the mutex *when the object is
  created* and — crucially — **unlocks it automatically when `lock` goes out of
  scope** at the closing brace. This is **RAII** (Resource Acquisition Is
  Initialisation): tie a resource's lifetime to an object so you can't forget to
  release it, even if an exception is thrown. You never call `.unlock()` by
  hand.
- **`std::move(sorted_standings)`** — the parameter was taken *by value* (a copy
  the caller handed us). `std::move` lets us *steal* its internals into
  `standings_` rather than copying all 20 structs again. After a move the
  source is left empty, but it's a local about to be destroyed anyway. This is
  **move semantics**, a big C++11 idea: transfer ownership cheaply instead of
  deep-copying.

### 5.5 `const` methods + `mutable` — the subtle pairing

```cpp
std::vector<DriverState> snapshot() const {
    std::shared_lock lock{mutex_};
    return standings_;
}
```

- **`snapshot() const`** — the method promises not to change the logical
  contents of the `Leaderboard`. A reader *should* be a `const` operation.
- But locking `mutex_` *does* physically modify the mutex's internal state — so
  a `const` method normally couldn't lock it. That's what **`mutable`** on the
  member fixes:
  ```cpp
  mutable std::shared_mutex mutex_;
  ```
  `mutable` means "this member may be modified even inside a `const` method." It
  is the standard escape hatch for things like locks and caches that change for
  *implementation* reasons without changing the object's *logical* value.
- **`std::shared_lock`** (not `unique_lock`) takes the *shared* mode, so several
  `snapshot()` calls run in parallel.
- It **returns a full copy** of the vector. The point: the caller gets its own
  data and doesn't have to keep holding the lock while it uses it — the lock is
  released the instant `snapshot` returns. Short lock hold = good concurrency.

### 5.6 `size()` and `at_position()`

```cpp
std::size_t size() const {
    std::shared_lock lock{mutex_};
    return standings_.size();
}

DriverState at_position(int pos) const {
    std::shared_lock lock{mutex_};
    for (const auto& s : standings_) {
        if (s.position == pos) return s;
    }
    return {};
}
```

- **`std::size_t`** — the standard unsigned integer type used for sizes and
  counts (big enough to index any container). `.size()` returns it.
- **`at_position(int pos)`** looks up a driver by 1-indexed race position and
  returns a copy. **`return {};`** returns a *default-constructed* `DriverState`
  (all the `{0}` / `{false}` defaults from `types.h`) when nothing matches —
  a safe empty result instead of crashing on a bad position.

### 5.7 `fastest_lap_holder()`

```cpp
std::string fastest_lap_holder() const {
    std::shared_lock lock{mutex_};
    float best = std::numeric_limits<float>::max();
    std::string holder;
    for (const auto& s : standings_) {
        if (s.best_lap_ms > 0.0f && s.best_lap_ms < best) {
            best   = s.best_lap_ms;
            holder = s.profile.id;
        }
    }
    return holder;
}
```

A linear scan for the smallest positive `best_lap_ms`. Same pattern: start
`best` at the max possible float so any real time beats it; `> 0.0f` skips
drivers who haven't set a lap yet (recall `0.0f` means "no lap"). Returns the
winner's id, or an empty string if nobody has a lap.

> Note this duplicates *information* also tracked lock-free in
> `RaceState::fastest_lap_holder` — one is derived from the standings snapshot,
> the other is a live atomic. Different consumers, different mechanisms.

---

## 6. Cheat-sheet — every keyword the user asked about, in one place

| Keyword / symbol | What it means here |
| --- | --- |
| `#pragma once` | Include this header at most once per compile (dedup guard). |
| `#include <x>` / `"x"` | Paste in another file; `<>` = system, `""` = project. |
| `struct` | Data bag; members **public** by default. |
| `class` | Type with hidden state; members **private** by default. |
| `public:` / `private:` | Access sections — who may touch a member. |
| `enum class` | Scoped, strongly-typed enum (`Type::VALUE`, no int coercion). |
| `const` (variable) | Value can't change after init; may go in read-only memory. |
| `const` (method) | Method won't modify the object; callable on `const` objects. |
| `const T&` (param/return) | Borrow by reference, no copy, can't modify it. |
| `mutable` | This member *can* change even inside a `const` method (locks, caches). |
| `inline` (var/func) | May be defined in a header; linker merges the duplicates into one. |
| `extern` | Declaration only — the definition lives in exactly one `.cpp`. |
| `auto` | Compiler deduces the type (e.g. in range-for loops). |
| `{ }` init | Brace/uniform initialisation; `{}` alone = default/zero init. |
| `f` suffix | `float` literal (e.g. `0.5f`), vs `double` without it. |
| `std::array<T,N>` | Fixed-size, no-heap array; size known at compile time. |
| `std::vector<T>` | Growable, heap-backed array. |
| `std::string` | Heap-backed text (avoided on the hot path in favour of `char[4]`). |
| `std::move` | Transfer ownership instead of copying (move semantics). |
| `std::atomic<T>` | Indivisible reads/writes; no data race on that value. |
| `memory_order_*` | How much surrounding work is visible to other threads. |
| `compare_exchange_weak` | Lock-free CAS: swap only if the value is still what I expect. |
| `std::mutex` | One-at-a-time lock. |
| `std::shared_mutex` | Reader/writer lock: many `shared_lock`s **or** one `unique_lock`. |
| `unique_lock` / `shared_lock` | RAII lock guards (exclusive / shared) that auto-unlock. |
| `std::size_t` | Unsigned integer type for sizes and counts. |
| `std::chrono::steady_clock` | Monotonic clock for measuring elapsed time. |

---

## 7. One-paragraph summary to say out loud

> "`common/` is the shared data layer. `types.h` defines the plain building
> blocks — a hot-path `TelemetryFrame` that uses a fixed `char[4]` instead of a
> `std::string` to avoid heap allocation, the driver/car/track profiles, some
> `enum class` states, and `DriverState` which composes them into one driver's
> full record. `season_data.h` fills those types with the real 2025 grid in
> `inline const std::array`s — `inline` so the header can be included
> everywhere without duplicate-symbol errors. `race_state.h` holds global flags
> shared across threads using lock-free `std::atomic`s with explicit
> release/acquire memory ordering, including a compare-and-swap loop so one of
> many drivers claims the fastest lap without a lock, plus an `extern` global
> that one `.cpp` will define. `leaderboard.h` stores the full 20-driver
> standings behind a `std::shared_mutex` so many readers can snapshot in
> parallel while the once-per-lap writer takes an exclusive lock, using RAII
> lock guards and `mutable` so read methods can stay `const`."
