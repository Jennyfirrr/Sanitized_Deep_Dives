# Chapter 4: Order Encoding and Bit Packing

Alright so this is where things start getting really fun. I'd been playing with individual bits, branchless masks, and two's complement tricks, and then I started thinking about how you would actually encode real trading data using these techniques. Like if each bit can represent a condition, and you can pack conditions into integers, can you pack entire orders into integers? And the answer is yes, and it's way cooler than I expected.

This chapter covers packing multiple orders into single integers, the Mycroft trick for parallel byte comparison, de Bruijn multiplication, and how all of this connects to the risk gate and order pool patterns in the actual engine. There are a lot of hand-traced binary tables because honestly that's how I learned this stuff, and seeing the bits move is the only way it makes sense to me.

---

## Packing Orders into a Single Integer

The basic idea is that a `uint64_t` is 8 bytes, which means it has 8 lanes of 8 bits each. If your order IDs fit in a byte (0-255), you can pack 8 orders into a single 64-bit integer. The first approach I tried was the obvious one, a nested for loop that shifts each byte into position.

```cpp
for (int i = 0; i < 8; i++) {
    order_bit_packed |= potential_trades[i * 8 + j] << (j * 8);
}
```

This works, but when I looked at the assembly output I realized the compiler was doing something way smarter than my loops. For the risk gate build function where you're broadcasting the same byte into all 4 lanes of a `uint32_t`, the compiler converted my for loop into a single `imul` instruction with the magic number 65792, which is `0x10100`, which is `(1 << 8) + (1 << 16)`.

I was so excited when I figured out what that constant was. The compiler took both of my for loops and turned them into basically a two-instruction sequence. Two for loops, each iterating 4 times, reduced to like 6 cycles total. And this is where I first heard the term de Bruijn multiplication, although I was actually wrong about what de Bruijn means (I'll correct that in a sec).

---

## The IMUL Trick

So when the compiler sees you packing the same byte into multiple lanes, it can replace the shifting loop with a single multiply. Here's why it works.

Say you have a buy order ID of `0xAB` and you want to broadcast it into 4 bytes of a 32-bit integer, so you want `0xABABABAB`. Multiplying `0xAB` by `0x01010101` does exactly this. Each `0x01` in the multiplier places a copy of the source byte at that position. It's just how multiplication distributes across the digit positions, same idea as multiplying 12 by 1001001 in decimal to get copies of 12 spread out.

```
0xAB * 0x01010101 = 0xABABABAB
```

The compiler figured this out from my for loop, which honestly made me feel a little inadequate but also really impressed. It recognized that my loop was just broadcasting a byte and replaced the whole thing with a 3-cycle `imul` instruction. When I saw this in the assembly I started hand-tracing the multiplication to make sure I understood it, because understanding why the compiler makes the choices it makes is kind of the whole point of reading assembly output.

I should correct myself here because I initially called this de Bruijn multiplication and that's wrong. De Bruijn sequences are about using a multiplication as a perfect hash, which typically shows up in `ctz()` implementations for hardware that doesn't have native bit-scan instructions. The compiler optimization I was seeing is just standard strength reduction, replacing a loop with a constant multiply. Related concepts but not the same thing. I get stuff wrong sometimes while learning, sorry not sorry, I'd rather figure it out as I go than wait until I know everything before writing anything down.

---

## Binary Multiplication, by Hand

I traced this out because subtraction already tripped me up and I wanted to make sure I understood multiplication before moving on. In base-10 when you multiply 12 by 13, you're really doing `(12 * 3) + (12 * 10)`, splitting it into simpler operations. Binary multiplication works the same way but with powers of 2.

```
    0 0 0 0 0 1 0 1     (5)
  * 0 0 0 0 0 0 1 1     (3)
  ------------------

Bit 0 of multiplier is 1, so keep:  0 1 0 1
Bit 1 of multiplier is 1, so shift left 1:  1 0 1 0

Add them:
    1 0 1 0
  + 0 1 0 1
  ----------
    1 1 1 1 = 15
```

Each bit in the multiplier that's a 1 means you take the original number shifted left by that bit position, then add all the partial results together. Multiplying by a power of 2 is just a left shift, always. Multiplying by 7 is `(x << 3) - x`. Multiplying by 10 is `(x << 3) + (x << 1)`, because `8 + 2 = 10`. This is apparently called strength reduction, and it's related to Booth's multiplication algorithm, which is a refinement that handles signed integers and can reduce the number of additions by encoding runs of 1's.

This understanding matters because in the engine, the compiler does these strength reductions constantly. When you see `imul` with a constant in the assembly output, you can trace back to understand what mathematical operation the compiler decided to optimize, and whether the result is actually what you intended.

---

## Implicit Encoding

One trick I really liked was using the position within the integer to encode information without spending a bit on it. In a 64-bit packed order, bits 0-31 can be the sell side and bits 32-63 can be the buy side. You don't need a flag bit to say "this is a buy" because the position itself tells you that. This is called implicit encoding or schema-based development.

```cpp
// buy side: bits 32-63
for (int i = 0; i < 4; i++) {
    risk_gate_built |= risk_gate_buy << ((i * 8) + 32);
}

// sell side: bits 0-31
for (int i = 0; i < 4; i++) {
    risk_gate_built |= risk_gate_sell << (i * 8);
}
```

This saves you bits that would otherwise be wasted on metadata, and it lets you compare the buy and sell halves independently with a single mask operation. I ended up not using signed integers for any of this because sign bits are icky for order packing, the sign bit steals the 8th bit from your MSB lane and causes all kinds of sign-extension headaches when you shift. I learned this the hard way when my kill mask of 128 was blocking 100% of trades because `int8_t(128)` is actually -128, and shifting that left smears 1's across the entire register. Use `uint8_t` for order IDs. Always.

---

## The Mycroft Trick

OK this is probably my favorite thing in all of these notes. The Mycroft trick, also called hasZero or hasLess, was attributed to Alan Mycroft from 1987. It became famous through `strlen()` implementations in libc, where instead of checking one byte at a time for the null terminator, you load 4-8 bytes at once and use this trick to detect which word contains a zero byte.

Here's the function.

```cpp
uint32_t calc_laneMatchCount(uint32_t packed_order, uint32_t broadcast_mask) {
    uint32_t diff = packed_order ^ broadcast_mask;
    uint32_t matches = ((diff - 0x01010101) & ~diff & 0x80808080);
    return __builtin_popcount(matches);
}
```

Three operations and a popcount to check all 4 lanes simultaneously. Let me trace through the whole thing because this is the part that made my brain tingle.

Assume `packed_order = 0x03FF0A21` and `broadcast_mask = 0x03080A21`.

**Step 1: XOR to find differences**

```
-------------------------------------------------------------------
      lane 3    |    lane 2     |    lane 1     |     lane 0
-------------------------------------------------------------------
  0 0 0 0 0 0 1 1|1 1 1 1 1 1 1 1|0 0 0 0 1 0 1 0|0 0 1 0 0 0 0 1
  0 0 0 0 0 0 1 1|0 0 0 0 1 0 0 0|0 0 0 0 1 0 1 0|0 0 1 0 0 0 0 1  XOR
-------------------------------------------------------------------
  0 0 0 0 0 0 0 0|1 1 1 1 0 1 1 1|0 0 0 0 0 0 0 0|0 0 0 0 0 0 0 0
```

Lane 3, lane 1, and lane 0 all matched (diff = 0x00), lane 2 didn't (diff = 0xF7).

**Step 2: Subtract 0x01010101 (the trap)**

This is where it gets wacky. Binary subtraction with borrow propagation is not intuitive at all, and I spent a while figuring out the mental model for this.

The lane values before subtraction are: lane 3 = 0, lane 2 = 247, lane 1 = 0, lane 0 = 0.

When you subtract 1 from a lane that's 0, it can't do it, so it borrows from the next lane up. Lane 0 borrows from lane 1, but lane 1 is also 0, so lane 1 borrows from lane 2. Lane 2 has 247, so it can afford to give. Lane 3 is 0 and has nobody to borrow from, so it underflows to 255.

```
-------------------------------------------------------------------
      lane 3    |    lane 2     |    lane 1     |     lane 0
-------------------------------------------------------------------
  0 0 0 0 0 0 0 0|1 1 1 1 0 1 1 1|0 0 0 0 0 0 0 0|0 0 0 0 0 0 0 0
- 0 0 0 0 0 0 0 1|0 0 0 0 0 0 0 1|0 0 0 0 0 0 0 1|0 0 0 0 0 0 0 1
-------------------------------------------------------------------
  lane 3: overflow  lane 2: 245    lane 1: borrow   lane 0: borrow
-------------------------------------------------------------------
  1 1 1 1 1 1 1 1|1 1 1 1 0 1 0 1|1 1 1 1 1 1 1 0|1 1 1 1 1 1 1 1
```

Result: `0xFFF5FEFF`

An easier way to think about the borrow propagation, when you do `0xFF000000 - 0x00000001`, the 1 can't be subtracted from lane 0 (which is 0), so it propagates all the way up until it finds a lane with something to give. That top lane loses 1 (becomes 0xFE) and every lane below it fills to 0xFF, giving `0xFEFFFFFF`. It's like, the bit borrowed from the lane above makes the lane below become 255, because one bit in lane 1 is worth 256 in lane 0 terms.

**Step 3: AND with NOT diff (the guard)**

```
-------------------------------------------------------------------
      lane 3    |    lane 2     |    lane 1     |     lane 0
-------------------------------------------------------------------
  1 1 1 1 1 1 1 1|1 1 1 1 0 1 0 1|1 1 1 1 1 1 1 0|1 1 1 1 1 1 1 1
& 1 1 1 1 1 1 1 1|0 0 0 0 1 0 0 0|1 1 1 1 1 1 1 1|1 1 1 1 1 1 1 1
-------------------------------------------------------------------
  1 1 1 1 1 1 1 1|0 0 0 0 0 0 0 0|1 1 1 1 1 1 1 0|1 1 1 1 1 1 1 1
```

The `& ~diff` part is the guard. You only want to see bits that were originally 0 in the diff. If a byte was like 0x81, the subtraction would leave the MSB set, but `& ~diff` kills it because the original MSB was already 1.

**Step 4: AND with 0x80808080 (the filter)**

```
-------------------------------------------------------------------
      lane 3    |    lane 2     |    lane 1     |     lane 0
-------------------------------------------------------------------
  1 1 1 1 1 1 1 1|0 0 0 0 0 0 0 0|1 1 1 1 1 1 1 0|1 1 1 1 1 1 1 1
& 1 0 0 0 0 0 0 0|1 0 0 0 0 0 0 0|1 0 0 0 0 0 0 0|1 0 0 0 0 0 0 0
-------------------------------------------------------------------
  1 0 0 0 0 0 0 0|0 0 0 0 0 0 0 0|1 0 0 0 0 0 0 0|1 0 0 0 0 0 0 0
```

Result: `0x80008080`. Lane 2 is missing because it was the one that didn't match. The `0x80808080` mask ignores all the noise and only looks at the sign bit in each lane.

**Step 5: popcount**

`__builtin_popcount(0x80008080)` = 3 matching lanes.

And `POPCNT` isn't even a function call anymore, it's a single assembly instruction. There is dedicated silicon on the CPU just for counting active bits in a single clock cycle.

So to summarize how it works: the subtraction acts as a trap (if a lane is 0x00, it forces the MSB high), the `& ~diff` acts as a guard (only keeps bits that were originally 0), and the `0x80808080` acts as a filter (isolates one flag bit per lane). The whole thing normalizes all the bits except the sign bit to 0, then checks each lane's sign bit.

This is called a SWAR technique, which stands for SIMD Within A Register. You're treating a standard 64-bit register as a vector of 8-bit values and performing parallel operations without actually using the specialized SIMD hardware (AVX/SSE). It's clever and it's fast and I had way too much fun tracing through the binary math.

---

## Risk Gates vs Kill Switches

Working through all of this made me realize there's an important distinction between a risk gate and a kill switch that I was confusing earlier.

A **risk gate** checks individual orders against criteria. If some pass and some fail, you handle them differently. The Mycroft trick gives you per-lane granularity, you can see exactly which lanes matched and which didn't.

```cpp
// per-lane check: which orders pass the risk gate?
uint64_t breach = (risk_gate - packed_order) & 0x8080808080808080ULL;
// if breach is 0, every lane passed
// if not, the MSB tells you exactly which lanes failed
```

A **kill switch** is a hard stop. It closes out all current positions and completely blocks all trades. This should be separate from order-level validation because if something triggers a kill switch, there's a systemic issue at play, not just a few bad trades. Kill switch checks use the simpler `(packed & mask) == mask` pattern because you don't need per-lane detail, you just need a yes/no.

```cpp
int32_t kill_switch(int32_t state, int32_t kill_mask) {
    return (state & kill_mask) == kill_mask;
}
```

This compiles to just `notl`, `testl`, `sete`, three instructions, zero branches. The compiler actually converts `(A & B) == B` to `(~A & B) == 0` because it maps better to the TEST instruction. Neat optimization.

---

## The Struct Revelation

After all of this loop optimization and assembly analysis, I had what might be the most humbling moment of the whole learning process. All those for loops, all that shifting and OR-ing, all the de Bruijn multiplication analysis, and the real answer was just to use a struct.

```cpp
struct buy_side {
    uint8_t order0b, order1b, order2b, order3b;
};

struct sell_side {
    uint8_t order0s, order1s, order2s, order3s;
};

struct OrderPair {
    buy_side buys;
    sell_side sells;
};
static_assert(sizeof(OrderPair) == 8, "struct must be 8 bytes");
```

With `static_assert` proving the struct is exactly 8 bytes, the compiler knows the whole thing fits in a single register. The packing function compiles down to basically one instruction, `movq %rdi, %rax` then `ret`. The compiler looks at your struct, says "yep, this is 8 bytes arranged exactly how I'd arrange them in a register," and just passes it through. 34 lines of bit manipulation code, reduced to a single MOV instruction because you told the compiler the truth about your data layout.

That was the moment I realized that the fastest code is the code you never have to write. If you organize your data correctly, the compiler can see that everything is already where it needs to be, and it doesn't have to generate any transformation code at all. Java would never let you do this, and I will die on that hill.

---

## Where This Lands in the Engine

The order encoding experiments directly shaped two key engine components.

**OrderPool** (MemHeaders/PoolAllocator.hpp): Uses a `uint64_t` bitmap to track which slots are occupied. Allocation finds the first free slot with `__builtin_ctzll(~bitmap)`, marks it with an OR, deallocation clears it with an AND. No searching, no linked lists, just bit operations on a single integer.

**BuyGate** (CoreFrameworks/OrderGates.hpp): Gate conditions are computed as masks using the `-(condition)` two's complement trick from chapter 2, then combined with AND. If any gate fails, the price zeros out. The whole thing is the risk gate pattern from this chapter, scaled up, instead of checking order IDs against a risk threshold, it checks market conditions against trading criteria, same bit-level logic, different application.

| Pattern | Engine usage |
|---------|-------------|
| Byte packing into `uint64_t` | OrderPool slot storage |
| `__builtin_ctzll(~bitmap)` | Find first free pool slot |
| `bitmap &= ~(1ULL << index)` | Free a pool slot |
| `__builtin_popcountll(bitmap)` | Count active positions |
| `-(condition)` mask + AND | Buy gate condition zeroing |
| `static_assert(sizeof(...))` | Struct size guarantees everywhere |

The delta encoding pattern (XOR old state with new state to detect changes) also shows up in the portfolio controller for detecting which positions changed between ticks. And the implicit encoding idea, where position within the integer encodes meaning, is how the engine uses bitmap indices as position identifiers without storing separate ID fields.

---

*Next: [Chapter 5, Assembly and the CPU](05_assembly_and_cpu.md)*
