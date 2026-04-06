# Chapter 2: Branchless Execution

We're expanding on bitwise operators with branchless programming today, I may go back and make notes for all the operators before I do this, I don't know, I'm seeing where the rabbit hole takes me.

The core idea is that using if/else statements in latency critical systems is bad, because the CPU essentially tries to predict which way the branch will fall, and if it guesses wrong you're down 15-20 clock cycles. For most things that isn't a big deal, but when something as minor as a few nanoseconds can cost you money, this is critical. The reason this happens is because if the predictor is wrong, the entire pipeline has to flush, and essentially rebuild these operations until it gets it right.

I also found out that branching exposes security vulnerabilities through those flush cycles, stuff like the Spectre and Meltdown exploits, but cybersecurity isn't really my thing so I'm just noting it here in case anyone finds it interesting.

---

## Branchless Clamp

The traditional way to clamp a value between 0 and 255 uses two if statements:

```cpp
int clamp(int x) {
    if (x < 0) return 0;
    if (x > 255) return 255;
    return x;
}
```

Thankfully, someone smarter than me laid the groundwork to do this with pure bit manipulation:

```cpp
int clamp_branchless(int x) {
    x &= ~(x >> 31);   // if x is negative, x >> 31 is all 1s, ~all 1s is all 0s
    x -= 255;
    x &= (x >> 31);    // same trick again for the upper bound
    x += 255;
    return x;
}
```

This works because of two's complement and sign extension. In a 32-bit system, a positive 5 is `00000000_00000000_00000000_00000101`. To get the negative value, your CPU inverts all the bits and adds 1, so you end up with `11111111_11111111_11111111_11111011` which represents -5. The very first bit is the sign bit, and because it's a 1, the number is negative.

When you tell the computer to do `x >> 31`, if the number is positive the new empty slots on the left fill with 0s, and if it's negative they fill with 1s. So for -5, you shift right 31 times and end up with `11111111_11111111_11111111_11111111` which is `0xFFFFFFFF`. For any signed integer, this essentially takes the sign bit and smears it across the entire register. That smearing is what enables the whole branchless technique.

Walking through it with x = -5: the shift gives you all 1s, the NOT flips to all 0s, then `-5 & 0 = 0`, clamped. With x = 300: the shift gives you all 0s, NOT flips to all 1s, `300 & 0xFFFFFFFF = 300` passes through, then `300 - 255 = 45`, shift again, 45 is positive so the mask is 0, `45 & 0 = 0`, then `0 + 255 = 255`, clamped. With x = 100: passes the first gate, `100 - 255 = -155`, shift gives all 1s, `-155 & 0xFFFFFFFF = -155`, then `-155 + 255 = 100`. The original value survives unchanged.

Literally just directly manipulating bits and removing the need for if statements.

---

## Two's Complement, Why It Exists

Two's complement was created to solve the problem of representing negative numbers on the same hardware adder that handles positive numbers. The goal was getting `-5 + 5 = 0` to work on the same circuit paths.

At first the idea was just using the leftmost bit as a negative/positive flag, like `10000101` for -5. But then you end up with a negative AND a positive zero, which apparently just completely breaks math.

So engineers decided to flip all the bits and add 1. Using 8-bit as an example because I don't want to write this out 32 times:

```
 5 = 00000101
~5 = 11111010  (flip all bits)
+1 = 11111011  (this is -5)
```

When you add them together:

```
  00000101   ( 5)
+ 11111011   (-5)
----------
 100000000
```

That leading 1 has nowhere to go, it just falls off the edge. You're left with `00000000`. Two's complement is designed so that when this overflow happens, it always produces 0. This removes the need for software to know or care if a number is negative or positive, because it just takes advantage of the limitations of how much information can be stored in a finite space within memory.

It's honestly crazy when you realize that all programming languages essentially compile down to operation chains that just do this faster than I fall asleep in a Java lecture.

---

## Branchless Min

Here's a branchless minimum of two values, this is apparently the industry standard approach:

```cpp
int32_t smallest(int32_t a, int32_t b) {
    int32_t mask = -(a < b);
    return b ^ ((a ^ b) & mask);
}
```

The `-(a < b)` part works because `a < b` returns either 0 or 1. Negating gives you either `0x00000000` or `0xFFFFFFFF`. Then `a ^ b` captures the bits where a and b differ. AND-ing with the mask either keeps those differences (selecting a) or zeroes them out (keeping b). Then XOR-ing with b applies or doesn't apply the swap.

This is safer than the shift approach because it avoids signed overflow edge cases, you don't get to play with your bits as much (please laugh at this), but it's correct.

Walking through it with a = 8, b = 16: `mask = -(8 < 16) = -(1) = 0xFFFFFFFF`. `8 ^ 16 = 24`, `24 & 0xFFFFFFFF = 24`, `16 ^ 24 = 8`. Returns 8, the smaller value.

With a = 16, b = 8: `mask = -(16 < 8) = -(0) = 0x00000000`. `16 ^ 8 = 24`, `24 & 0 = 0`, `8 ^ 0 = 8`. Returns 8 again.

---

## Variance, Not Speed

This is the thing I got wrong at first. The argument for branchless programming isn't that it's faster, because often it isn't. When your input data is predictable, branch prediction can actually be faster because the compiler is smarter than you in 99.9% of cases.

But stocks aren't predictable.

So you need operations that never use the branch prediction functionality of the CPU, because while faster is good, the pipeline flush is like slamming into a wall. The variance is huge. With branching, execution time might bounce between 2ns and 20ns depending on whether the predictor guessed right. With branchless code, it might be a consistent 5ns every single time.

And that consistency is the whole point. If you know that it will always take a certain amount of time to execute a cycle, you can plan and structure your entire execution harness around that. It's like in video games, where 30 ping 80% of the time and 300 ping the other 20% is way worse than a consistent 100ms, because you can adjust to the delay when it's predictable.

CPUs are essentially state prediction machines. They love consistency.

---

## The Kill Switch

A practical application: implementing a kill switch using the sign bit.

My first instinct was if/else or case switching because that's what I was used to, but just using a mask to clear bits is better, because nothing else triggers after the check. The simplest version uses the leftmost bit as the kill switch, because checking the sign of an integer is literally free on x86. After any arithmetic operation the sign flag is set automatically, so checking if an `int32_t` is negative is just a single SAR instruction:

```cpp
int32_t apply_kill_switch(int8_t packed_state) {
    return packed_state >> 7;
}
```

That's it. If bit 7 (the sign bit) is set, this returns -1 (all 1s). If it's clear, it returns 0. One instruction, zero branches, fully deterministic.

The more general pattern for checking any combination of bits as a kill condition:

```cpp
int32_t check_kill(uint8_t packed_state, uint8_t kill_mask) {
    return (packed_state & kill_mask) == kill_mask;
}
```

This is one AND plus one CMP, two instructions total, zero branches. You can check any combination of conditions by setting different bits in the kill mask, and the cycle count is completely deterministic.

---

## Bit State Extraction

To pull individual states out of a packed integer, you mask and shift:

```cpp
int32_t extract_bit(int32_t packed, int bit_index) {
    return (packed >> bit_index) & 1;
}
```

So assuming you start with bit 7 in a number like 255, bit 7 represents 128. You AND with the mask to isolate it, then shift right to normalize it to either 1 or 0:

```
  11111111 = 255
& 10000000 = (1 << 7)
----------
  10000000 = 128

  10000000 >> 7 = 00000001 = 1
```

Without the normalization you'd get 128 back instead of 1, which isn't ideal for further processing. Playing with your bits (still funny) requires attention to these details.

---

## Compiler Behavior

One insight I picked up is that it's not about how many if statements you have, but how complex what's inside them is. The simpler the body, the more likely the compiler will emit a cmov (conditional move, which is branchless) instead of a branch. The ternary operator tends to produce cmov more reliably than if/else with function calls:

```cpp
// more likely to compile branchless (cmov)
int32_t result = (state == 1) ? value_a : value_b;

// more likely to compile as a branch (JMP)
if (state == 1) {
    do_complex_function();
}
```

The cmov instruction was introduced in the Pentium Pro era and it's kind of the gold standard for this, the CPU actually calculates both possible values and the cmov just picks which one to keep at the last pipeline stage. No prediction, no flush.

You can check what the compiler actually generates with:
```bash
g++ -S -O2 program.cpp -o program.s
c++filt < program.s | less
```

If you see TEST/AND/CMOV in the output, you're golden. If you see JE/JNE/JMP, the code is branching and you probably need to simplify it.

---

## Where This Lands in the Engine

The entire hot path of the trading engine is branchless by design:

- **BuyGate** (OrderGates.hpp): gate conditions computed as masks, then AND-ed together. No branches to decide if a trade passes, the math produces a zero price if any gate fails.
- **PositionExitGate** (Portfolio.hpp): TP/SL comparisons use branchless FPN operations, exit records are written using mask-select patterns.
- **Danger gradient** (PortfolioController.hpp): proportional crash protection computed every tick with branchless multiplication, the danger score scales the gate price toward zero without any conditional logic.
- **Kill switch**: uses the same mask-and-zero pattern described above, `buying_halted` is checked via a branchless gate that zeros `buy_conds` when active.

The variance argument is exactly why this matters in practice. During volatile market moves, the CPU is processing unpredictable price patterns. Branch prediction would be wrong constantly. The branchless hot path ensures consistent latency regardless of market conditions.

---

*Next: [Chapter 3, Fixed-Point Arithmetic](03_fixed_point_arithmetic.md)*
