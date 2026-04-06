# Chapter 7: Lock-Free Concurrency

So I actually ran into concurrency problems before I knew what concurrency problems were. Back when I was working on FoxML_Core (the Python ML pipeline), I had a race condition where a model evaluation phase was trying to write to a file that hadn't been created yet, because another part of the pipeline was supposed to create it but hadn't finished. The file would get created after the model tried to write, so nothing actually got output. I fixed it with file locking and atomic writes, which is basically a mutex pattern, one thread grabs the lock, does its work, releases it, and everything else waits.

That worked fine for a training pipeline where latency isn't critical. But for a trading execution engine, having threads wait on each other is actually somehow worse than Java's garbage collector, and I didn't think that was possible. So this chapter is about how to share data between threads without any thread ever waiting.

---

## Why Mutexes Are Bad for Hot Paths

A mutex (mutual exclusion lock) works exactly like it sounds. One thread grabs the lock, does its thing, and releases it. Every other thread that wants to touch that data has to wait. The problem is that "wait" means the OS scheduler gets involved, and if the thread holding the lock gets preempted by the scheduler (context-switched out to run something else), then everything just sits there doing nothing, like me in my Java lectures.

In a trading system this is catastrophic. You're processing market data ticks that arrive every few milliseconds, and if your engine thread is blocked waiting on a mutex while the TUI thread is updating the display, you miss ticks. Missed ticks mean missed trading opportunities. The whole point of building a fast engine is wasted if you're going to let the operating system's scheduler decide when your threads get to run.

Lock-free programming says, what if nobody ever waits? What if instead of grabbing a lock, you just try to do the thing, and if someone else changed the data underneath you, you just try again?

---

## Compare and Swap (CAS)

The fundamental primitive that makes lock-free programming possible is `std::atomic` and specifically the Compare-And-Swap operation. In x86 assembly this is a single instruction called `CMPXCHG`. What it does, atomically, is.

"Look at this memory location. If it still contains the value I expect, replace it with my new value. If it changed, tell me what the current value is and I'll retry."

The important word is atomically. The CPU guarantees that between the comparison and the swap, nothing else can modify that memory location. It's not a software lock, it's a hardware guarantee at the cache-line level.

```cpp
#include <atomic>

std::atomic<int> counter{0};

void increment() {
    int expected = counter.load();
    while (!counter.compare_exchange_weak(expected, expected + 1)) {
        // CAS failed, someone else changed the value
        // "expected" is automatically updated to the current value
        // just loop and try again
    }
}
```

Walking through this, step 1 loads the current value into `expected`. Step 2 calls `compare_exchange_weak`, which says "is counter still equal to `expected`? If yes, swap it to `expected + 1`." If it succeeds, we're done. If it fails (another thread changed the value between our load and our CAS), `expected` gets automatically updated to whatever the current value actually is, and we loop and try again.

On x86, this compiles to a `lock cmpxchg` instruction. The `lock` prefix tells the CPU to lock the associated cache line for the duration of the instruction. No mutex, no syscall, no OS scheduler involvement. Just the CPU coordinating with other cores at the hardware level.

---

## Weak vs Strong CAS

There are two variants and the difference matters.

`compare_exchange_weak` can spuriously fail. Sometimes it just says "nope" for no real reason, which sounds terrible but it's actually the preferred option. On ARM architectures (not x86, but still), the underlying instruction is LL/SC (Load-Linked/Store-Conditional), which can fail due to cache-line eviction even if the value didn't change. On x86 this spurious failure basically doesn't happen, but the compiler can still optimize the weak version better because it doesn't have to guarantee success.

`compare_exchange_strong` guarantees that if it fails, the value actually changed. The cost is a higher instruction count because it might have to retry internally.

General rule, use `weak` in loops (where you're retrying anyway), use `strong` for one-shot attempts where you need to know if the value genuinely changed.

---

## Spin Loops

When CAS fails, you just try again. This is called a spin loop.

```cpp
while (!counter.compare_exchange_weak(expected, expected + 1)) {
    // spin: CAS failed, expected is updated, try again
}
```

For a hot path this is exactly what you want. No sleeping (sleeping means the OS scheduler gets involved), no yielding, no context switches. You just burn CPU cycles spinning until you succeed. If contention is low (which it should be in a well-designed system where threads touch different data most of the time), you'll succeed on the first or second try.

The trade-off is that if contention is high, you're wasting CPU cycles spinning. But that's a design problem, not a concurrency primitive problem. If your threads are contending heavily on the same atomic, you need to redesign your data flow so they're working on separate data most of the time.

---

## Memory Ordering

This is the part I still need to study more, but the basics are important enough to note. When you use atomics, you can specify memory ordering constraints that tell the compiler and CPU how much reordering is allowed.

`memory_order_seq_cst` (sequentially consistent) is the default and the safest. Every thread sees operations in the same order. It's also the slowest because it prevents all reordering.

`memory_order_acquire` and `memory_order_release` are a pair. A release store says "everything I wrote before this store is visible to anyone who does an acquire load of this location." An acquire load says "I can see everything that was written before the release store."

`memory_order_relaxed` provides no ordering guarantees at all. The atomic operation itself is still atomic (no torn reads/writes), but the compiler and CPU are free to reorder it relative to other operations.

For most trading engine use cases, `seq_cst` is fine because the performance difference is measured in nanoseconds and correctness is way more important than squeezing out the last few cycles from memory ordering. But knowing these options exist matters when you're trying to understand why a particular data sharing pattern works or doesn't work.

---

## Why This Matters for Trading

In a multi-threaded trading system, you have at minimum two threads that need to share data.

The **engine thread** processes ticks, runs the buy gate, manages positions, computes statistics. This is the hot path that absolutely cannot be blocked.

The **display thread** (TUI or GUI) reads the engine state to show prices, positions, P&L, statistics. This runs at a much lower frequency (maybe 10-60 times per second) and doesn't need real-time accuracy, just a recent-enough snapshot.

If you use a mutex to protect the shared state, the engine thread has to grab the lock every time it updates, and the display thread has to grab it to read. If they contend, the engine thread might stall waiting for the display thread to finish reading, which is completely unacceptable.

The lock-free solution is double-buffering. The engine writes to a snapshot buffer using regular (non-atomic) writes. When the snapshot is complete, it atomically swaps the "current" pointer. The display thread reads from whichever buffer the pointer currently indicates. There's no lock, no contention, and the worst case is that the display thread reads a slightly stale snapshot, which is perfectly fine for display purposes.

---

## The Cache-Line Connection

This ties back to the `alignas(64)` discussion from chapter 6. When you use a `lock cmpxchg` instruction, the CPU locks the cache line that contains the atomic variable. If other non-atomic data shares that cache line, the lock blocks access to all of it, even data that has nothing to do with the atomic operation.

The fix is to put atomic variables on their own cache line.

```cpp
struct alignas(64) AtomicState {
    std::atomic<int> current_buffer;
    // padding to fill the cache line
};
```

This way the lock only blocks the atomic itself, not unrelated data that happens to be nearby in memory. Combined with the false sharing prevention from chapter 6, you get threads that can operate independently without any accidental interference through the cache system.

---

## Where This Lands in the Engine

The engine uses a variant of the double-buffer pattern for sharing state between threads.

**TUI Snapshot** (DataStream/EngineTUI.hpp): On the slow path (every N ticks), the engine copies its entire state into a `TUISnapshot` struct. This is a bulk `memcpy`-style copy (actually field-by-field copy in `TUI_CopySnapshot()`). The TUI thread reads this snapshot for display. There's no lock because the TUI thread only reads between complete copies, and the engine thread only writes during the designated slow-path window.

**BacktestSnapshot** (Backtest/BacktestSnapshot.hpp): Same pattern but for the backtest suite. The backtest worker thread runs the simulation, and the GUI thread reads `BacktestSnapshot_Copy()` output for the dashboard panels. The copy is the synchronization, a complete snapshot at a point in time, not a live reference that could be modified mid-read.

The key insight is that you don't always need atomics for thread safety. If you can design your system so that writers and readers operate in clearly separated phases, a bulk copy is simpler and faster than fine-grained atomic operations. The engine writes 100% of the snapshot, then the reader reads 100% of the snapshot. No partial reads, no torn values, no need for per-field atomics.

This works because the snapshot is a value type, a complete copy of the state at a point in time. The reader gets a consistent view that might be a few ticks old, but it's internally consistent, which is all the display needs. The alternative, reading live state with per-field atomics, would be both slower and harder to reason about because you could see field A from tick 1000 and field B from tick 1003.

| Component | Thread safety approach |
|-----------|----------------------|
| TUISnapshot | Bulk copy on slow path, reader gets consistent snapshot |
| BacktestSnapshot | Same pattern, worker copies to GUI-visible struct |
| OrderPool bitmap | Single atomic word, CAS-based allocation in live engine |
| Config hot-reload | Bulk struct copy, no field-level synchronization needed |

The CAS loop pattern from this chapter shows up directly in the OrderPool for the live engine, where the bitmap needs to be updated atomically because fill detection and consumption can race. But the vast majority of the engine's thread safety comes from the simpler bulk-copy approach, which avoids the complexity of lock-free data structures entirely.

---

*Next: [Chapter 8, From Theory to Engine](08_from_theory_to_engine.md)*
