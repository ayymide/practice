# Lesson — `src/race_control/` (the race stewards)

My own reference for the `race_control` directory. This is the first directory
that is mostly *application logic* rather than pure infrastructure — it's the
"stewards' office" of the simulation: it watches the cars, decides when someone
broke a rule or the weather turned, and issues penalties. Crucially, it's where
everything from the two earlier lessons finally gets *used together*:

- from [`common.md`](common.md): `RaceControlEvent`, `PenaltyState`,
  `DriverState`, `std::shared_mutex`, `DRIVERS`.
- from [`concurrency.md`](concurrency.md): the `MpscQueue`, `std::atomic`,
  compare-and-swap (CAS), release/acquire memory ordering.

If those two words — *CAS* and *release/acquire* — aren't instantly clear, skim
Part A of [`concurrency.md`](concurrency.md) first; I'll lean on them here rather
than re-derive them.

---

## PART A — New concepts introduced in this directory

Four ideas show up here that we haven't met yet. Get these first and the code
reads easily.

### A.1 The producer/consumer pipeline these files form

This is the mental model for the whole directory. Three components *produce*
events; one *consumes* them, all through a single `MpscQueue<RaceControlEvent>`
(the Multi-Producer Single-Consumer queue from the concurrency lesson):

```
   PRODUCERS (push events)                 the pipe            CONSUMER (pops events)

   TrackLimitsMonitor  ──┐
   (driver ran wide)     │
                         ├──►  MpscQueue<RaceControlEvent>  ──►  PenaltyEnforcer
   WeatherSystem       ──┘         (from concurrency/)            .process_events()
   (rain started)                                                 (counts warnings,
                                                                    issues penalties)
```

So the queue you studied in the abstract last lesson has a concrete job now:
carry `TRACK_LIMITS` and `WEATHER_CHANGE` events from the monitors to the
enforcer. The `PenaltyEnforcer` is interesting because it's *both* a consumer
(it drains the queue) *and* a producer (it pushes new `PENALTY_ISSUED` events
back in) — more on the subtlety that creates later.

### A.2 Randomness — `std::mt19937` and distributions

Real cars don't run wide on a fixed schedule; it's *probabilistic*. C++ models
randomness in two separated pieces:

```cpp
std::mt19937 rng_{std::random_device{}()};
std::uniform_real_distribution<float> coin_{0.0f, 1.0f};
```

- **`std::mt19937`** is a **random number engine** — the "Mersenne Twister,"
  a well-known generator that produces a stream of pseudo-random bits. It's the
  *source* of randomness.
- **`std::random_device{}()`** — `std::random_device` is a source of genuine
  (hardware/OS) randomness. The `{}` constructs a temporary one and the trailing
  `()` *calls* it to produce one random number. That number is the **seed** —
  the starting point for the engine. Seeding from `random_device` means every
  run gets a different sequence (rather than the same "random" numbers each
  time).
- **`std::uniform_real_distribution<float> coin_{0.0f, 1.0f}`** is a
  **distribution** — it *shapes* the engine's raw bits into the range and shape
  you want. This one gives floats evenly spread between 0.0 and 1.0.
- **Using them together:** `coin_(rng_)` feeds the engine into the distribution
  and returns one float in [0, 1). The idiom `if (coin_(rng_) < prob)` is the
  standard "roll a weighted coin": it's true with probability `prob`. If `prob`
  is 0.004, this branch fires about 0.4% of the time.

> **Interview point:** engine vs distribution is a deliberate C++ split — one
> object *generates* entropy, another *maps* it to a useful shape. You can pair
> any distribution with any engine.

### A.3 References as members + the member initialiser list

Look at how every class here stores the queue:

```cpp
private:
    MpscQueue<RaceControlEvent>& events_;   // note the &
```

That `&` makes `events_` a **reference member** — the class does *not* own its
own queue; it holds a reference (an alias) to one that lives elsewhere and was
handed in at construction. This is **dependency injection**: the queue is shared
by all three components, so they each borrow the *same* one rather than each
making a private copy (which would defeat the point of a shared pipe).

A reference member forces a specific construction syntax:

```cpp
TrackLimitsMonitor::TrackLimitsMonitor(MpscQueue<RaceControlEvent>& event_queue)
    : events_(event_queue) {}
```

The **`: events_(event_queue)`** part is a **member initialiser list** — it
initialises members *before* the constructor body runs. It is **mandatory** for
references, because a reference must be bound to its target at the moment it's
created and can never be re-seated afterwards (you can't do `events_ = ...;` in
the body — there's nothing to assign to). The `{}` afterwards is an empty
constructor body: all the work was the binding.

### A.4 A tiny state machine (the penalty lifecycle)

The `PenaltyEnforcer` implements a **state machine**: each driver's penalty
moves through a fixed set of states in a fixed order, and only certain
transitions are legal. From the header comment:

```
   NONE ──(3rd warning)──► PENDING ──(enters pits)──► SERVING ──(pit done)──► SERVED
```

- **NONE** — clean, no penalty.
- **PENDING** — penalty issued, not yet served.
- **SERVING** — driver is in the pits serving it.
- **SERVED** — done.

These are the `enum class PenaltyState` values from `common/types.h`. The whole
enforcer is really "advance each driver through this state machine safely, even
if multiple threads try to advance the same driver at once" — which is exactly
what CAS is for.

### A.5 `std::string` vs `std::string_view`, and `std::memcpy`

Two string details appear:

- **`std::string_view`** (e.g. `driver_index(std::string_view id)`) is a
  **non-owning view** of a string — a pointer + length, no copy, no allocation.
  Use it for a read-only parameter you only *look at*. It happily accepts a
  `std::string`, a string literal, or the `char[4]` id without copying. It's the
  cheap way to say "I just need to read some characters."
- **`std::memcpy(dest, src, n)`** copies `n` raw bytes from `src` to `dest`. It's
  used here to stuff a driver id into the fixed `char driver_id[4]` field of an
  event — copying the 3 letters directly rather than going through string
  machinery, keeping the event cheap to build.

---

## PART B — The files

Three components, from simplest to most involved:

```
src/race_control/track_limits.h / .cpp     — a pure PRODUCER (probabilistic, lock-free push)
src/race_control/weather.h     / .cpp      — a PRODUCER with shared_mutex-protected state
src/race_control/penalty_enforcer.h / .cpp — the CONSUMER: CAS state machine, atomics
```

---

## 1. `track_limits.h` / `.cpp` — probabilistically catching drivers running wide

**Job:** each monitoring cycle, for each driver, roll a weighted coin to decide
if they exceeded track limits; if so, push a `TRACK_LIMITS` event. Pure producer
— it only ever *pushes* to the queue.

### 1.1 The class shape

```cpp
class TrackLimitsMonitor {
public:
    explicit TrackLimitsMonitor(MpscQueue<RaceControlEvent>& event_queue);
    void check(const std::vector<DriverState>& states, int current_lap);

private:
    MpscQueue<RaceControlEvent>& events_;
    std::mt19937                 rng_{std::random_device{}()};
    std::uniform_real_distribution<float> coin_{0.0f, 1.0f};

    static constexpr float BASE_RATE = 0.004f; // ~0.4% per sector crossing
};
```

- **`explicit`** on the constructor — as in the thread pool, stops accidental
  implicit conversion; you must construct it deliberately with a queue.
- **`events_`** is the reference member (A.3): the shared event pipe.
- **`rng_` / `coin_`** — the randomness pair from A.2.
- **`check(const std::vector<DriverState>& states, int current_lap)`** takes the
  current standings **by const reference** (borrow, no copy — 20 `DriverState`s
  is a lot to copy) and the lap number.
- **`static constexpr float BASE_RATE = 0.004f`** — a compile-time constant, one
  copy shared by the class, giving the baseline ~0.4% violation chance per
  sector crossing. Named instead of a magic number buried in the formula.

### 1.2 The constructor

```cpp
TrackLimitsMonitor::TrackLimitsMonitor(MpscQueue<RaceControlEvent>& event_queue)
    : events_(event_queue) {}
```

Nothing but the member initialiser list binding `events_` to the shared queue
(A.3). Empty body.

### 1.3 `check()` — the probability model

```cpp
void TrackLimitsMonitor::check(const std::vector<DriverState>& states, int current_lap) {
    for (const auto& state : states) {
        if (state.in_pit) continue; // can't violate track limits in pits

        const auto& frame = state.latest_frame;

        float speed_factor = frame.speed_kph / 280.0f;
        float prob = BASE_RATE
                   * state.profile.aggression
                   * (1.0f + frame.tire_wear)
                   * speed_factor;

        if (coin_(rng_) < prob) {
            RaceControlEvent ev;
            ev.type      = RaceControlEvent::Type::TRACK_LIMITS;
            std::memcpy(ev.driver_id, state.profile.id.data(), 3);
            ev.lap       = current_lap;
            ev.message   = state.profile.id + " exceeded track limits (S"
                         + std::to_string(frame.sector) + ")";
            ev.timestamp = std::chrono::steady_clock::now();
            events_.push(std::move(ev));
        }
    }
}
```

Walking through it:

- **`for (const auto& state : states)`** — range-based for over the drivers,
  each borrowed by const reference (no copy).
- **`if (state.in_pit) continue;`** — skip anyone in the pit lane; you can't run
  wide on a track you're not on. `continue` jumps to the next driver.
- **`const auto& frame = state.latest_frame;`** — a const-reference *alias* to
  the driver's latest telemetry, so the rest reads cleanly (`frame.speed_kph`
  rather than `state.latest_frame.speed_kph`). No copy.
- **The probability formula** — this is the domain logic. Violation chance goes
  *up* with:
  - `state.profile.aggression` — pushier drivers run wide more.
  - `(1.0f + frame.tire_wear)` — worn tyres = less grip = more mistakes. At 0
    wear this factor is 1.0 (no effect); at full wear it doubles the chance.
  - `speed_factor = speed_kph / 280.0f` — normalised to a typical F1 speed, so
    faster = riskier, and the factor is ~1.0 at racing speed.
  - all scaled by `BASE_RATE`. Multiplying independent factors is the standard
    way to combine "risk contributions."
- **`if (coin_(rng_) < prob)`** — the weighted-coin roll from A.2. Fires with
  probability `prob`.
- **Building the event** — set the `type`, copy the 3-letter id in with
  `std::memcpy` (A.5) into the `char[4]` field, stamp the lap, build a
  human-readable `message` (`std::to_string(frame.sector)` turns the sector int
  into text for concatenation), and timestamp it with the steady clock.
- **`events_.push(std::move(ev))`** — hand the event to the queue, **moving** it
  in (steal, don't copy — see move semantics in the concurrency lesson). This is
  the "P" in the pipeline diagram.

> Note: because `MpscQueue` is lock-free and multi-producer, you *could* run
> several `TrackLimitsMonitor`s on different threads and they'd all push safely.
> That's why the shared queue is MPSC and not SPSC.

---

## 2. `weather.h` / `.cpp` — mutable shared state behind a `shared_mutex`

**Job:** occasionally change the track weather (DRY → DAMP → WET → DRY), push a
`WEATHER_CHANGE` event when it does, and expose the current weather + a grip
multiplier that *all 20 driver updates read every tick*.

This is the counterpart to `Leaderboard` from the common lesson: one writer,
many frequent readers → **`std::shared_mutex`** (Single Writer, Multiple
Readers). The header comment spells out exactly why a plain mutex would be wrong:
the driver threads read weather at 50Hz × 20, and a plain mutex would force all
20 reads to queue one-by-one; a shared lock lets them all read in parallel.

### 2.1 The class shape

```cpp
class WeatherSystem {
public:
    explicit WeatherSystem(MpscQueue<RaceControlEvent>& event_queue);
    void update(int current_lap);
    WeatherState current() const;
    float grip_factor() const;

private:
    MpscQueue<RaceControlEvent>& events_;
    WeatherState                 state_{WeatherState::DRY};
    mutable std::shared_mutex    mutex_;
    std::mt19937                 rng_{std::random_device{}()};
    std::uniform_real_distribution<float> coin_{0.0f, 1.0f};
    int last_update_lap_{0};

    static constexpr float CHANGE_PROBABILITY = 0.15f; // per update call
};
```

- **`state_{WeatherState::DRY}`** — the shared mutable data this class protects.
  Starts dry.
- **`mutable std::shared_mutex mutex_`** — the reader/writer lock. **`mutable`**
  (covered in the common lesson) lets the `const` reader methods `current()` and
  `grip_factor()` still lock it — locking physically changes the mutex, but the
  object's *logical* value is unchanged, so those methods can stay `const`.
- **`last_update_lap_`** — remembers when weather last changed, to rate-limit
  changes.
- **`CHANGE_PROBABILITY = 0.15f`** — 15% chance of a change per eligible update.

### 2.2 `update()` — the writer path

```cpp
void WeatherSystem::update(int current_lap) {
    if (current_lap - last_update_lap_ < 5) return; // don't change too often
    last_update_lap_ = current_lap;

    if (coin_(rng_) > CHANGE_PROBABILITY) return; // no change this update

    WeatherState new_state;
    {
        std::shared_lock read{mutex_};
        switch (state_) {
            case WeatherState::DRY:  new_state = WeatherState::DAMP; break;
            case WeatherState::DAMP: new_state = WeatherState::WET;  break;
            case WeatherState::WET:  new_state = WeatherState::DRY;  break;
        }
    }

    {
        std::unique_lock write{mutex_}; // exclusive — blocks all readers
        state_ = new_state;
    }

    static const char* names[] = {"DRY", "DAMP", "WET"};
    RaceControlEvent ev;
    ev.type      = RaceControlEvent::Type::WEATHER_CHANGE;
    ev.lap       = current_lap;
    ev.message   = std::string("Weather: ") + names[static_cast<int>(new_state)];
    ev.timestamp = std::chrono::steady_clock::now();
    events_.push(std::move(ev));
}
```

Step by step:

- **Rate-limit:** `if (current_lap - last_update_lap_ < 5) return;` — at least 5
  laps between changes. Then record this lap as the last update.
- **Coin flip:** `if (coin_(rng_) > CHANGE_PROBABILITY) return;` — 85% of the
  time, no change; only ~15% proceed. (Note the direction: it *returns* when the
  roll is *above* the threshold, so it continues 15% of the time.)
- **Read the current state under a `shared_lock`** to decide the next one in the
  DRY→DAMP→WET→DRY cycle. The `{ }` scope means the shared lock is released as
  soon as we've computed `new_state`.
- **Write the new state under a `unique_lock`** (exclusive — blocks all readers
  for the brief instant of the assignment). Separate, minimal scope again.
- **Push a `WEATHER_CHANGE` event** so the UI/enforcer learn about it.
  `static_cast<int>(new_state)` converts the `enum class` to its integer index
  (0/1/2) to look up the name string — needed because `enum class` won't convert
  to int *implicitly* (that safety is the whole point of `enum class`), so we
  ask for it *explicitly* with `static_cast`.

> **The honest subtlety (worth mentioning in an interview):** dropping the read
> lock and then taking the write lock is a *check-then-act* gap. In this project
> there's only one weather-writer thread, so nothing changes `state_` in
> between and it's safe. If there were multiple writers, two could both read
> DRY, both compute DAMP, and you'd want either one lock held across the whole
> read-modify-write, or a CAS. It works here *because of the single-writer
> assumption* — which is exactly the sort of assumption an interviewer will
> probe, so know it's there.

### 2.3 `current()` and `grip_factor()` — the reader paths

```cpp
WeatherState WeatherSystem::current() const {
    std::shared_lock read{mutex_}; // concurrent reads allowed
    return state_;
}

float WeatherSystem::grip_factor() const {
    std::shared_lock read{mutex_};
    switch (state_) {
        case WeatherState::DRY:  return 1.00f;
        case WeatherState::DAMP: return 0.85f;
        case WeatherState::WET:  return 0.70f;
    }
    return 1.0f;
}
```

Both take a **`std::shared_lock`**, so any number of driver threads can call them
*simultaneously* — they only block while the rare writer holds the exclusive
lock. `grip_factor()` maps weather to a grip multiplier the physics uses (wetter
= less grip = slower). The trailing `return 1.0f` after the switch is a
safety net to satisfy the compiler that every path returns a value (and guards
against an unexpected enum value).

---

## 3. `penalty_enforcer.h` / `.cpp` — the consumer and its CAS state machine

**Job:** drain the queue, count each driver's track-limits warnings, and when a
driver hits 3, atomically move them NONE → PENDING and issue a penalty. Then
advance PENDING → SERVING → SERVED as they pit. This is the most concurrency-
heavy file in the directory — it's the payoff for the CAS material in the
concurrency lesson.

### 3.1 Per-driver atomic state, stored in arrays

```cpp
private:
    int driver_index(std::string_view id) const;

    MpscQueue<RaceControlEvent>& events_;

    std::array<std::atomic<int>,          20> warnings_{};
    std::array<std::atomic<PenaltyState>, 20> states_{};
    std::array<std::atomic<int>,          20> extra_pit_time_{};
```

Instead of a struct-per-driver, state is kept as **parallel arrays indexed by
driver number** (0–19, matching the `DRIVERS` order from `season_data.h`):

- **`warnings_[i]`** — how many track-limit warnings driver `i` has.
- **`states_[i]`** — driver `i`'s `PenaltyState`.
- **`extra_pit_time_[i]`** — extra pit ticks owed (the served time of the
  penalty).

Every element is a **`std::atomic`**, because these can be read and written by
different threads concurrently (the enforcer processing events, plus the
simulator calling `driver_entered_pits`). `std::atomic<PenaltyState>` works
because the enum is a small trivially-copyable value.

### 3.2 The constructor — initialising atomics

```cpp
PenaltyEnforcer::PenaltyEnforcer(MpscQueue<RaceControlEvent>& event_queue)
    : events_(event_queue)
{
    for (auto& s : states_)         s.store(PenaltyState::NONE, std::memory_order_relaxed);
    for (auto& w : warnings_)       w.store(0, std::memory_order_relaxed);
    for (auto& t : extra_pit_time_) t.store(0, std::memory_order_relaxed);
}
```

Binds the queue reference, then zeroes every atomic. **`relaxed`** ordering is
correct here because this runs in the constructor *before any other thread can
possibly touch the object* — there's no concurrency yet to order against, so we
use the cheapest option.

### 3.3 `driver_index()` — id → array index

```cpp
int PenaltyEnforcer::driver_index(std::string_view id) const {
    for (int i = 0; i < static_cast<int>(DRIVERS.size()); ++i) {
        if (DRIVERS[i].id == id) return i;
    }
    return -1;
}
```

A linear search mapping a driver id ("VER") to its array slot. Takes a
**`std::string_view`** (A.5) so callers can pass a `std::string`, a literal, or
the event's `char[4]` id with no copy. Returns `-1` if not found — every caller
checks `if (idx < 0)` and bails, the defensive pattern.

### 3.4 `process_events()` — the snapshot-first drain (a real bug avoided)

```cpp
void PenaltyEnforcer::process_events() {
    std::vector<RaceControlEvent> batch;
    RaceControlEvent ev;
    while (events_.pop(ev)) batch.push_back(std::move(ev));

    for (auto& e : batch) {
        if (e.type != RaceControlEvent::Type::TRACK_LIMITS) continue;
        ...
    }
}
```

The important design decision is in the comment: it **snapshots the whole queue
into a `batch` vector *first*, then processes the batch** — rather than a single
`while (events_.pop(ev)) { process(ev); }` loop.

Why does that matter? Because processing a `TRACK_LIMITS` event can itself
**push a new `PENALTY_ISSUED` event back onto the same queue** (the enforcer is
both consumer and producer, remember). If we drained and processed in one loop,
that freshly-pushed penalty event would be popped again a few iterations later
by the *same* loop, seen as "not TRACK_LIMITS," and silently `continue`d past —
the enforcer would **eat its own announcement** before any UI or test could see
it.

Snapshotting first means: anything pushed *during* processing stays in the queue
for the *next* call/consumer to handle, instead of being swallowed. This is a
genuinely subtle correctness point and a great thing to be able to explain.

### 3.5 The warning counter — `fetch_add` (why not CAS here?)

```cpp
int count = warnings_[idx].fetch_add(1, std::memory_order_relaxed) + 1;
```

- **`fetch_add(1)`** atomically adds 1 and returns the value *before* the add. So
  `+ 1` gives the new total. It's a single indivisible read-modify-write — no
  torn counts even if two threads increment at once.
- **Why `relaxed`?** Because, as the header comment says, we only care about the
  *final total*, not about ordering this increment relative to other memory. We
  don't need "the exact instant it crossed 3" to be synchronised with other
  writes — just "add one, tell me the count." That's the textbook use case for a
  relaxed atomic counter.
- **Why not CAS?** CAS is for when a decision depends on the *current* value
  ("change X to Y *only if* it's still Z"). A pure increment has no such
  condition, so `fetch_add` is simpler and cheaper. The *next* step is where the
  condition appears — and that's where CAS comes in.

### 3.6 The NONE → PENDING transition — CAS, and why exactly one winner

```cpp
if (count == 3) {
    PenaltyState expected = PenaltyState::NONE;
    if (states_[idx].compare_exchange_strong(
            expected, PenaltyState::PENDING,
            std::memory_order_acq_rel,
            std::memory_order_acquire))
    {
        // We won — issue the penalty.
        RaceControlEvent pen;
        pen.type      = RaceControlEvent::Type::PENALTY_ISSUED;
        std::memcpy(pen.driver_id, e.driver_id, 4);
        pen.lap       = e.lap;
        pen.message   = std::string(e.driver_id) + " - 5 second penalty (track limits)";
        pen.timestamp = std::chrono::steady_clock::now();
        events_.push(std::move(pen));

        extra_pit_time_[idx].store(3, std::memory_order_relaxed);
    }
    // If CAS failed: another thread already issued the penalty. Do nothing.
}
```

This is the heart of the file. On the 3rd warning we want to issue **exactly
one** penalty, even if two threads simultaneously observe `count == 3` for the
same driver.

**`compare_exchange_strong(expected, desired, ...)`** is CAS: atomically, *if*
`states_[idx]` currently equals `expected` (NONE), set it to `desired`
(PENDING) and return `true`; otherwise write the actual current value back into
`expected` and return `false`.

- The thread whose CAS **succeeds** flips NONE→PENDING and is the sole issuer:
  it pushes the `PENALTY_ISSUED` event and records the pit-time penalty.
- Any thread whose CAS **fails** sees the state was no longer NONE (someone beat
  it) and does nothing. **No double penalty** — this is the guarantee CAS buys
  that `fetch_add` couldn't.

Details:

- **`_strong` vs `_weak`:** we use `compare_exchange_strong` because we're *not*
  in a retry loop — this is a one-shot decision. (Recall from the concurrency
  lesson: `_weak` may fail spuriously and is only cheaper *inside a loop* where
  you'd retry anyway. A single-shot check wants `_strong`.)
- **Memory orders:** `acq_rel` on success (we both observe prior writes and
  publish our own — importantly, the penalty state must be visible to the
  simulator that later reads it), `acquire` on failure (we just need to see the
  winning value).
- **`std::memcpy(pen.driver_id, e.driver_id, 4)`** copies all 4 bytes of the id
  (including the `\0`) from the source event into the new one.
- **`extra_pit_time_[idx].store(3, relaxed)`** records the served-time cost; the
  comment notes this is a simplified stand-in for ~5 seconds of pit ticks.

### 3.7 The remaining transitions — same CAS pattern, guarded

```cpp
void PenaltyEnforcer::driver_entered_pits(const std::string& driver_id) {
    int idx = driver_index(driver_id);
    if (idx < 0) return;

    PenaltyState expected = PenaltyState::PENDING;
    states_[idx].compare_exchange_strong(
        expected, PenaltyState::SERVING,
        std::memory_order_acq_rel,
        std::memory_order_relaxed);
}

void PenaltyEnforcer::driver_exited_pits(const std::string& driver_id) {
    int idx = driver_index(driver_id);
    if (idx < 0) return;

    PenaltyState expected = PenaltyState::SERVING;
    states_[idx].compare_exchange_strong(
        expected, PenaltyState::SERVED,
        std::memory_order_acq_rel,
        std::memory_order_relaxed);
}
```

Same idea, enforcing the state machine's legal transitions:

- **PENDING → SERVING** only fires if the driver was actually PENDING. If they
  had no penalty (NONE), the CAS fails and nothing happens — entering the pits
  for a normal tyre stop doesn't invent a penalty. The CAS *is* the guard.
- **SERVING → SERVED** likewise only completes a penalty that was being served.
- The return values are ignored here on purpose: "transition if legal, else do
  nothing" is the whole intent, so there's nothing to branch on.
- **failure order is `relaxed`** here (vs `acquire` in 3.6) because on failure
  these callers do nothing at all — they don't read any state that would need
  to be synchronised, so the cheapest order suffices.

### 3.8 The read accessors

```cpp
PenaltyState PenaltyEnforcer::penalty_state(const std::string& driver_id) const {
    int idx = driver_index(driver_id);
    if (idx < 0) return PenaltyState::NONE;
    return states_[idx].load(std::memory_order_acquire);
}

int PenaltyEnforcer::warning_count(const std::string& driver_id) const {
    int idx = driver_index(driver_id);
    if (idx < 0) return 0;
    return warnings_[idx].load(std::memory_order_relaxed);
}
```

- **`penalty_state`** loads with **acquire** — it pairs with the `acq_rel`
  *release* side of the CAS transitions, so a reader that sees SERVED is also
  guaranteed to see everything the transitioning thread did before it.
- **`warning_count`** loads with **relaxed** — it mirrors the relaxed counter;
  it's an informational number with no ordering requirement.
- Both return a safe default for an unknown id, matching the `-1` guard pattern.

---

## PART C — The big picture and interview prep

### C.1 How the pieces fit the earlier lessons

This directory is the *integration test* of the whole project's design:

- It **consumes** `common/`'s data types (`RaceControlEvent`, `PenaltyState`,
  `DriverState`) and reuses `common/`'s `shared_mutex` pattern (`WeatherSystem`
  mirrors `Leaderboard`).
- It **consumes** `concurrency/`'s `MpscQueue` as the real event bus, and
  `concurrency/`'s atomics + CAS for the penalty state machine.
- It shows the *reason* those abstractions exist: many producers, one consumer,
  no locks on the hot path.

### C.2 The three "why did they choose X?" questions to have ready

| Decision | Why |
|---|---|
| **`MpscQueue`, not SPSC**, for events | Multiple producers (track limits + weather, and possibly more monitor threads) push; one consumer (enforcer) drains. |
| **`shared_mutex`** in `WeatherSystem` | One rare writer, 20 frequent readers per tick — shared reads run in parallel; a plain mutex would serialise them. |
| **`fetch_add` for warnings but CAS for state** | The counter just needs an atomic increment (no condition). The state change is conditional ("only NONE→PENDING, and only once"), which is exactly what CAS guarantees. |
| **Snapshot-then-process** in `process_events` | The consumer is also a producer; draining in one loop would let it swallow the `PENALTY_ISSUED` events it just pushed. |

### C.3 One paragraph to say out loud

> "`race_control` is the stewards' office and the place the whole architecture
> comes together. Three components form a producer/consumer pipeline over a
> single lock-free MPSC queue: `TrackLimitsMonitor` and `WeatherSystem` push
> `TRACK_LIMITS` and `WEATHER_CHANGE` events, and `PenaltyEnforcer` drains them.
> The monitors are probabilistic — a Mersenne Twister engine plus a uniform
> distribution roll a weighted coin, with violation chance scaled by aggression,
> tyre wear and speed. `WeatherSystem` guards its mutable state with a
> `shared_mutex` so all 20 driver reads run concurrently while the rare writer
> takes an exclusive lock. `PenaltyEnforcer` is the concurrency showcase: it
> keeps per-driver atomics, uses `fetch_add` for the warning counter because
> that's just an atomic increment, and uses compare-and-swap for the
> NONE→PENDING→SERVING→SERVED state machine so that even if two threads see the
> third warning at once, exactly one issues the penalty. And it snapshots the
> queue before processing, because it's also a producer and would otherwise eat
> the penalty events it pushes."
