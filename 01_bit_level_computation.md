# Chapter 1: Bit-Level Computation

So I got bored one day and started looking into bitshifting, I'd heard that quant and HFT systems use bitwise operations heavily instead of passing full boolean values around and I wanted to understand why. What I found was that this stuff is way more interesting than Java (sorry not sorry), and it completely changed how I think about representing data.

This chapter covers the foundational bitwise operators, OR, XOR, AND, and shifts, and why they matter for building fast systems. Every one of these patterns shows up in the trading engine I eventually built, so if you're wondering "why should I care about playing with bits," stick around.

---

## OR, Setting Bits Without Touching Others

The OR operator (`|`) can directly manipulate a single bit without touching any of the others, so `x |= (1 << 3)` means set the 3rd bit from the right to 1, leave everything else alone. OR sets bits, AND clears bits, XOR toggles bits, that's basically the whole cheat sheet.

```cpp
int set_bit(int x, int position) {
    x |= (1 << position);
    return x;
}
```

This specific example basically functions as a +8 to whatever number you put in, because you're directly manipulating the 3rd bit from the right, so if you have 16 which is `00010000`, you OR it with `00001000` and get `00011000` which is 24. But if you try to use 8, nothing happens, if the bit is already a 1 that you're trying to set to 1, nothing actually changes. That matters because it means setting a flag that's already set is a no-op, you don't need to check first.

---

## XOR, Cancellation

Then I came across XOR and this one is really elegant. XOR returns 1 only when bits differ, which gives it this cool property where `a ^ a = 0` for any value, so pairs cancel each other out.

If you have an array like `{4, 1, 2, 1, 2}` and XOR everything together, the doubled 1s and 2s just cancel each other out, leaving the 4. You get to find a single number in an array of pairs in O(n) time with O(1) space, no hash maps, no sorting, nothing.

```cpp
int find_single(vector<int> &nums) {
    int result = 0;
    for (int n : nums) {
        result ^= n;
    }
    return result;
}
```

Walking through it, you get `0 ^ 4 = 4`, then `4 ^ 1 = 5`, then `5 ^ 2 = 7`, then `7 ^ 1 = 6` because the 1 cancels, then `6 ^ 2 = 4` because the 2 cancels. You're left with 4, the only unpaired element.

In the trading engine this shows up for change detection, `old_bitmap ^ new_bitmap` isolates exactly which positions changed state since the last tick.

---

## Shifts, Multiplication Without Multiplying

Shifting to the right is essentially division by 2^n per shift, and multiplication when shifted left, due to the way binary works. This isn't an optimization trick, it's literally how the CPU does power-of-2 arithmetic.

```
  1 << 3 = 8       (1 moved left 3 positions)
  16 >> 2 = 4      (16 divided by 4)
```

Shifts are also how you build masks, `(1 << n)` creates a mask with only bit n set, and `~(1 << n)` creates a mask with every bit *except* n set. These masks are the building blocks of basically everything that comes later.

---

## Power-of-2 Detection, n & (n - 1)

I read that `n & (n - 1) == 0` is heavily used in quant systems for fast power-of-2 checks. I was actually wrong about what it did at first (I was tired, had been studying for a Java exam), but here's how it actually works.

Powers of 2 have exactly one bit set, so 8 is `1000`. Subtracting 1 flips all the bits below it, so 7 is `0111`. AND-ing them together always produces zero, and this only works for powers of 2 because they're the only numbers with that single-bit property.

```
  8 & 7  = 1000 & 0111 = 0000   (power of 2)
  12 & 11 = 1100 & 1011 = 1000  (not a power of 2)
```

This shows up all over the engine for alignment checks and allocator sizing.

But the more useful thing is that `n & (n - 1)` clears the lowest set bit, which is how you walk through set bits in a bitmap without touching anything else:

```cpp
void walk_set_bits(uint32_t bitmap) {
    while (bitmap) {
        int lowest = __builtin_ctz(bitmap);  // index of lowest set bit
        printf("Bit %d is set\n", lowest);
        bitmap &= bitmap - 1;               // clear it, move to next
    }
}
```

This pattern, CTZ to find the lowest bit then clear it, is the exact loop the portfolio exit gate uses to process only active positions, skipping empty slots without any branch or comparison.

---

## Bitfields, 64 Flags in 8 Bytes

This is where it really clicked for me.

So I learned about bitfields while playing around with all this, where you use each index in a binary number to store a true or false value, and instead of reading them one by one you can just return the actual number and figure out which conditions are true and false in a single read cycle, because every number up to 255 has a unique binary representation. You can store 32 true/false values in the space of a single integer.

Like if you returned the integer 8 when reading a number, that means only the condition assigned to bit 3 is true, while every other condition is false. A single byte to represent 8 conditional statements nested within each other, that's wild honestly.

And this is why L1 cache matters so much. A `uint64_t` is 8 bytes, and the cache line is 64 bytes, so you can fit 8 complete trading states into a single cache line fetch, for a single nanosecond of operations. Compare that to using a struct of 32 separate bools, you're looking at 32 bytes minimum, scattered around memory, and you have to read each one individually. With bitfields you can control exactly where the data lives, for a fraction of the space and way faster access.

I found out later this is called a bitmasking dispatcher. I guess there's a name for everything once you figure it out yourself.

---

## Where This Lands in the Engine

Portfolio.hpp uses these patterns directly:

| Pattern | Purpose |
|---------|---------|
| `active_bitmap & ~(1 << idx)` | Clear position slot on exit |
| `__builtin_ctz(bitmap)` | Find next active position |
| `bitmap &= bitmap - 1` | Advance to next position |
| `__builtin_popcount(bitmap)` | Count active positions |
| `pool->bitmap & ~prev_bitmap` | Detect new fills |

Every tick the exit gate walks the active bitmap to check TP/SL on open positions, the buy gate uses bitmap operations to find free pool slots. These operations are single-cycle on modern CPUs, the bit-level thinking isn't premature optimization, it's just the natural representation of the problem.

---

*Next: [Chapter 2, Branchless Execution](02_branchless_execution.md)*
