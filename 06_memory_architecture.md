# Chapter 6: Memory Architecture

So it's 4am and I can't sleep, which means we're going over memory alignment, malloc, cache hierarchies, and allocators. This is the chapter where all the bit-level stuff from earlier starts to connect to how the hardware actually moves data around, and why the way you lay out your structs can matter more than how clever your algorithms are.

---

## Cache Hierarchy

The CPU doesn't talk directly to RAM for every operation, that would be absurdly slow. Instead there's a hierarchy of increasingly larger but slower caches between the registers and main memory.

```
Registers    ~bytes       0 cycles     (data is inside the CPU)
L1 cache     32-64 KB     ~3-4 cycles  (per core, this is where hot data lives)
L2 cache     256 KB-1 MB  ~10 cycles   (per core, warm data)
L3 cache     8-64 MB      ~30-50 cycles (shared across cores, last resort before RAM)
Main RAM     GBs          ~100+ cycles  (might as well be on the moon)
```

The stack gets allocated into these caches based on access patterns. The reason it stays hot in L1/L2 isn't because it's small (though it is), it's because the access pattern is so predictable. The stack grows top-down, you're always working near the top, so the same small region gets accessed over and over. The CPU tracks what gets used frequently and keeps it warm, like a cat sitting in your lap, it stays where the warmth is.

When the CPU fetches memory, it does it in cache-line sized chunks, typically 64 bytes. This means that when you read a single byte, you actually fetch the entire 64-byte line it lives on. This is both a blessing and a curse, and understanding it is pretty much the key to making fast programs.

---

## Memory Alignment and Struct Padding

Alignment means placing data in memory addresses that are multiples of the data type's size. A 4-byte `int` wants to live at addresses divisible by 4, an 8-byte `double` at multiples of 8. When data is misaligned and straddles two cache lines, the CPU has to do two fetches and stitch the result together, which introduces non-deterministic timing. On x86_64 it usually still works, it just costs extra cycles. On ARM and older architectures it can actually fault, which is kind of terrifying.

This is where struct layout becomes critical.

```cpp
struct Bad {
    char a;     // 1 byte + 7 padding
    double b;   // 8 bytes
    char c;     // 1 byte + 7 padding
};
// total: 24 bytes
```

The reason this is bad is that `double b` requires 8-byte alignment, so after the 1-byte `char a`, the compiler inserts 7 bytes of padding to push `b` to the next aligned address. Then `char c` gets 7 more bytes of padding to maintain the struct's overall alignment. You end up with 24 bytes when you only have 10 bytes of actual data, and it might straddle cache line boundaries.

```cpp
struct Good {
    double b;   // 8 bytes
    char a;     // 1 byte
    char c;     // 1 byte + 6 padding
};
// total: 16 bytes
```

By putting the largest member first and grouping the smaller members together, the chars can share padding and the total drops to 16 bytes. On a small scale this seems trivial, but over millions of these structs, it adds up, both in memory usage and in cache efficiency.

I was wrong about the explanation at first, I thought it was just about variable ordering, but it's actually about how the type determines the alignment requirement, and the alignment requirement determines the padding. The padding is a function of the relationship between adjacent member types, not just the storage layout in isolation. So I was like half right.

For explicit cache-line alignment.

```cpp
struct alignas(64) CacheLineAligned {
    double values[8]; // exactly one cache line
};
```

The `alignas(64)` guarantees that this struct starts at a cache-line boundary, so the whole block fits in exactly one cache line, no straddling. This matters a lot when different threads are touching different blocks, because without it you get false sharing.

---

## False Sharing

False sharing is sneaky. It happens when two threads write to different variables that happen to share the same cache line. Neither thread is touching the other's data, but because the cache operates at the line level, each write invalidates the other core's cached copy of that line. Both cores keep thrashing each other's caches, fighting over a line that conceptually has nothing to do with the other thread's work.

It doesn't cause incorrect results, just latency. And for deterministic behavior, unpredictable latency is almost as bad as wrong results.

The fix is padding or `alignas(64)` to ensure each thread's hot data lives on its own cache line. In the trading engine, this is exactly why the TUI snapshot double-buffer works the way it does, the engine thread and the display thread each touch separate memory regions that are aligned to avoid false sharing.

---

## Stack vs Heap

The stack is usually the "hot memory," and understanding why requires knowing how both allocation strategies work.

**Stack allocation** is literally just moving the stack pointer. `sub rsp, 64` in assembly means "I need 64 bytes of local variable space." That's one instruction. No searching free lists, no metadata headers, no fragmentation. Just pure pointer management. Freeing is the reverse, the function returns, the stack pointer moves back, done. `push` and `pop` are direct CPU instructions that write and read values at the stack pointer, and the CPU knows where `%rsp` is at all times.

This is why stack allocation is deterministic. The timing is always the same, one instruction for allocation, one for deallocation. There's nothing to search, nothing to coalesce, nothing that depends on allocation history.

**Heap allocation** via `malloc` is a different story. When you call `malloc(32)`, the allocator doesn't just give you 32 bytes. It stores a header right before your pointer containing the block size and whether it's free, so you're actually using 40-48 bytes. And `malloc` doesn't call the kernel every time either, that would be insanely slow. Instead it asks the OS for big chunks of memory upfront via `sbrk` or `mmap`, then carves them up as you request them. Like taking out a student loan and budgeting the payments.

The problem is that `malloc` timing is non-deterministic. Sometimes it's fast because there's a perfectly sized free block available. Sometimes it has to search the free list. Sometimes it has to call `mmap` to get more memory from the OS. You can't predict which case you'll hit, and on a hot path that uncertainty is unacceptable.

When you `free()` memory, the allocator puts it on a free list, and the next `malloc` call searches that list. Different strategies for managing free lists exist, first-fit, best-fit, and the buddy system. The buddy allocator splits and merges power-of-2 blocks, making allocation and deallocation O(log n) and predictable. That predictability is why it shows up in kernel memory management and real-time systems.

The takeaway, never call malloc on a hot path. Pre-allocate everything. Use memory pools. Recycle fixed-size blocks.

---

## The Memory Layout

In your process's virtual address space, code and static data live at the bottom. The heap starts right above that and grows upward toward higher addresses. The stack starts at the top and grows downward toward lower addresses. They grow toward each other, like the pointers in a two-pointer problem, `while (left < right)` where left is index 0 and right is the end. If they meet, well, you're going to have a very bad day. Stack overflows happen when the stack grows into the heap's territory.

The stack is typically limited to 1-8 MB depending on the OS, which is why you can't allocate huge arrays as local variables. The heap can grow much larger because `malloc` can always ask the OS for more pages via `mmap`.

```
High addresses:  [ STACK (grows DOWN) ]
                 [        ...         ]
                 [        ...         ]
                 [ HEAP (grows UP)    ]
                 [ Static data        ]
                 [ Code               ]
Low addresses:
```

---

## Vector Preallocation

A quick aside because this connects to the heap discussion. I had a leetcode problem where I couldn't figure out why the "optimal" solution was faster, and it turned out to be entirely about memory allocation strategy.

When you know the vector size ahead of time, `reserve()` allocates all the memory in one shot. When you use `push_back()`, the vector has to check if it has capacity, and if it doesn't, it allocates a new larger buffer, copies everything over, and frees the old one. That copy-and-move operation gets expensive when it happens repeatedly, especially for large vectors.

In a trading system context, this means building your data structures with known sizes at startup, not growing them dynamically during trading hours. Pre-allocate the order pool, pre-allocate the position array, pre-allocate the statistics buffers. The only allocation that should happen on the hot path is... nothing. Zero allocation. `sub rsp, N` for stack locals and that's it.

---

## Pool Allocators

OK so when I first saw "implement a pool allocator" as the next logical step, I kind of panicked because it sounded really hard. But then I actually did it and it was surprisingly straightforward.

A pool allocator pre-allocates a fixed block of memory and carves it into equal-sized slots. A bitmap tracks which slots are in use. Allocation finds the first free slot (one bit operation), marks it (one OR), and returns a pointer. Deallocation clears the bit (one AND). That's it.

```cpp
struct OrderPool {
    OrderPair *slots;
    uint64_t bitmap;
    uint32_t capacity;
};

void OrderPool_init(OrderPool *pool, uint32_t capacity) {
    pool->slots    = (OrderPair *)calloc(capacity, sizeof(OrderPair));
    pool->bitmap   = 0;
    pool->capacity = capacity;
}

OrderPair *OrderPool_Allocate(OrderPool *pool) {
    uint32_t index = __builtin_ctzll(~pool->bitmap);
    pool->bitmap |= (1ULL << index);
    return &pool->slots[index];
}

void OrderPool_Free(OrderPool *pool, OrderPair *slot_ptr) {
    uint32_t index = (uint32_t)(slot_ptr - pool->slots);
    pool->bitmap &= ~(1ULL << index);
}

uint32_t OrderPool_CountActive(const OrderPool *pool) {
    return __builtin_popcountll(pool->bitmap);
}
```

The `calloc` happens once at startup (heap allocation, but only during initialization). After that, every allocate and free is just bit manipulation on the bitmap integer, no `malloc`, no `free`, no searching, completely deterministic timing. The `__builtin_ctzll(~pool->bitmap)` finds the first zero bit (inverted to find the first free slot), `TZCNT` or `BSF` in hardware, which is a single instruction.

I was actually surprised how simple this was after all the worrying. And the insight that the bitmap IS the allocator, that tracking free slots is just bit manipulation on an integer, that connected everything from chapters 1 and 4 right back to memory management. Bits to track conditions, bits to track positions, bits to track memory slots, it's all the same pattern.

---

## The "34 Lines to 1 MOV" Moment

Remember from chapter 4 when I realized that organizing data into a properly-sized struct meant the compiler could just pass it through as a register? The order packing function, 34 lines of shift-and-OR operations, compiled down to `movq %rdi, %rax` followed by `ret`. One instruction.

This is the memory architecture payoff. When a struct is 8 bytes and fits in a single register, the compiler doesn't need to load from memory at all. The data arrives in `%rdi` (first argument register), gets moved to `%rax` (return value register), and that's it. No L1 hit, no cache miss, no bus transfer. The "fastest code is the code you never write" became real for me in that moment.

The lesson is that if you tell the compiler the truth about your data layout using structs and `static_assert`, it can make optimizations that no amount of clever bitwise tricks can match. The bit manipulation matters for the logic, but the struct layout matters for how fast that logic actually runs.

```cpp
struct OrderPair {
    buy_side buys;
    sell_side sells;
};
static_assert(sizeof(OrderPair) == 8, "struct must be 8 bytes");

// this function:
uint64_t order_packing_8byte(OrderPair pair) {
    uint64_t packed = 0;
    packed |= static_cast<uint64_t>(pair.buys.order0b);
    packed |= static_cast<uint64_t>(pair.buys.order1b) << 8;
    // ... 6 more lines ...
    return packed;
}

// compiles to:
//   movq  %rdi, %rax
//   ret
```

The compiler looks at the struct, sees it's already 8 bytes laid out exactly how the function arranges them, and realizes the function is an identity transformation. No work needed. No code generated. Just move the input to the output register and return.

---

## Buddy Allocators

The buddy allocator is worth understanding because it shows up in kernel memory management and it uses the same power-of-2 logic we've been working with throughout.

The idea is that you start with one big block of memory, say 1 MB. When someone requests 64 KB, you split the 1 MB block in half (two 512 KB buddies), split one of those in half (two 256 KB buddies), split again (two 128 KB buddies), and split once more (two 64 KB buddies). You hand out one of the 64 KB blocks and track which blocks are free at each level.

When the block is freed, you check if its buddy (the other half of the split) is also free. If so, you merge them back into the larger block. This coalescing prevents fragmentation because blocks always reunite with their buddy.

All the sizes are powers of 2, which means all the splitting and merging is just bit manipulation. Finding a block is O(log n), freeing is O(log n), and the sizes naturally align to cache-friendly boundaries.

---

## Where This Lands in the Engine

The trading engine enforces a strict separation between startup allocation and hot-path execution.

**Startup (heap allocation OK):**
- Pool allocator pre-allocates all position slots
- Rolling statistics buffers allocated once
- Configuration loaded, TUI initialized

**Hot path (zero allocation):**
- Position exit gate walks the bitmap, no memory allocation
- Buy gate computes masks on the stack
- Fill consumption uses pre-allocated pool slots
- Everything is stack locals or pre-allocated pool references

The actual engine components.

| Component | File | Pattern |
|-----------|------|---------|
| OrderPool | MemHeaders/PoolAllocator.hpp | Bitmap pool, `calloc` at init, bit ops at runtime |
| BuddyAllocator | MemHeaders/BuddyAllocator.hpp | Power-of-2 splitting, for larger allocation needs |
| PortfolioController | CoreFrameworks/PortfolioController.hpp | Stack-allocated state, zero heap on tick path |
| TUISnapshot | DataStream/EngineTUI.hpp | Pre-allocated snapshot struct, bulk copy on slow path |

The `alignas(64)` pattern shows up wherever data is shared between the engine thread and the TUI thread, preventing false sharing. And the struct sizing lesson, making sure structs fit register-friendly sizes, is why positions are sized the way they are and why the portfolio uses a flat array instead of a linked list.

The backtest suite is a different story, it's allowed to use dynamic allocation (malloc/realloc) for sample buffers because it's not a hot-path trading loop. But the live engine has a hard rule, no malloc, no realloc, no syscalls in the tick loop. This is a constraint, not a suggestion.

---

*Next: [Chapter 7, Lock-Free Concurrency](07_lock_free_concurrency.md)*
