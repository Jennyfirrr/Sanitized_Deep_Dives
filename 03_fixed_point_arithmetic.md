# Chapter 3: Fixed-Point Arithmetic

So I was sitting in a lecture one day, half paying attention honestly, and the professor brought up COBOL. A completely different professor in a different class had just been talking about the limitations of floating-point comparison, and those two things kind of collided in my head. I went home and looked up some COBOL examples and immediately the all-caps syntax hurt my eyes, like no thank you, but what caught my attention was how numbers are stored.

COBOL has fixed-point operations baked right into the language. You write something like `9(5)V99` and that means a variable that is 5 digits long, with a virtual decimal represented by the V, with 2 digits after the decimal. The key insight is that this lets all calculations happen in integers, which removes the variance that floating-point comparisons can introduce. For most things that variance isn't a huge deal, but when you're dealing with hundreds of thousands of buy/sell orders or moving real money around, it stacks up.

---

## The Core Idea

The trick is embarrassingly simple. You take a number like 101.55 and store it as 10155. That's it. It stays an integer, which means you get all the benefits of integer math, no rounding mode surprises, no denormal stalls, no IEEE 754 edge cases where `0.1 + 0.2 != 0.3`. Integer math is always faster than floating-point math, and more importantly, it's deterministic. Two machines running the same integer operations on the same inputs will always produce the exact same result. Floating-point can't make that guarantee because different compilers, different optimization levels, even different instruction reordering can change the least significant bits of a result.

When you actually need to show the number to a human, you just divide by your scale factor and add the decimal back in. But internally, everything stays as whole integers, and that's where the speed and determinism come from.

```cpp
// the basic idea
int64_t price = 10155;       // represents $101.55
int64_t qty = 250;           // represents 2.50 units

int64_t total = price * qty; // 2538750, which is $25387.50 at scale 10000
// (you have to track the scale factor yourself)
```

---

## Why Not Just Use Doubles

This is the thing that made it click for me. In a trading system, you're not just doing one calculation, you're doing millions of them, and they chain together. Every time you multiply two doubles, the result might be off by a tiny fraction. Multiply that result by something else, and the error compounds. Do this a few thousand times and suddenly your P&L calculation is off by real money.

But the bigger problem isn't accuracy, it's reproducibility. If you run a backtest and get a result, then run it again with the same data, you should get the same result. With floating-point that's not guaranteed if anything about the compilation or execution environment changes. With integer math it always is.

Also, and I thought this was really interesting when I learned it, the `imul` instruction on x86 has predictable, consistent latency. On modern Intel cores it's typically around 3 cycles. No rounding mode surprises, no denormal stalls. For signed integers the compiler picks `imul`, for unsigned it picks `mul`. That consistent latency is exactly what you want on a hot path where every tick matters.

---

## The Scaling Problem

The obvious issue with "just store it as an integer" is that you can hit the limits of your integer type pretty quickly. A regular `int32_t` maxes out at about 2.1 billion, which sounds like a lot until you realize that with 2 decimal places of precision, you're already limited to values under about 21 million. In crypto, where Bitcoin can be priced at tens of thousands of dollars and you need 8 decimal places for satoshi precision, you blow through `int32_t` almost immediately.

The standard approach in C++ is to wrap an `int64_t` in a struct, because `int64_t` gives you a much larger maximum value. And you pick your fractional precision based on what you need, like 16 bits of fractional precision gives you about 4 decimal places, 32 bits gives you about 9. The struct wrapper is important because it lets the compiler treat your fixed-point type differently from raw integers, so you don't accidentally add a fixed-point price to a raw integer count and get nonsense.

```cpp
// the naive approach (where I started)
struct FixedPoint {
    int64_t value;
};

FixedPoint from_double(double d) {
    return { (int64_t)(d * 65536.0) }; // 16 fractional bits
}

double to_double(FixedPoint fp) {
    return (double)fp.value / 65536.0;
}

FixedPoint add(FixedPoint a, FixedPoint b) {
    return { a.value + b.value }; // just integer addition
}
```

This works fine for simple stuff. Addition and subtraction are trivial because the scale factors are the same. Multiplication requires you to shift right after multiplying to get the scale factor back to where it should be. Division is the reverse. It's all just integer operations with some bookkeeping.

---

## When 64 Bits Isn't Enough

So here's where things got interesting for me. When you're building a trading engine that needs to handle both very large numbers (total portfolio value) and very small numbers (individual fee calculations at sub-cent precision), 64 bits starts feeling tight. You can do the math on what precision you lose at different price levels, and there are real scenarios where you'd get accumulation errors over a long trading session.

My solution, and I'm honestly still kind of proud of this, was to not pick a fixed width at all. I built an arbitrary-width fixed-point library where the width is a template parameter. You write `FPN<64>` and you get a 128-bit number with 64 fractional bits. Write `FPN<256>` and you get 512 bits total. The engine runs at `FPN<64>` which gives 4096-bit precision, which is absurd overkill, but it means I literally never have to worry about precision loss in any accounting path.

The representation is dead simple, just an array of `uint64_t` words in little-endian order, plus a sign bit stored separately.

```cpp
template <unsigned FRAC_BITS> struct FPN {
    static constexpr unsigned TOTAL_BITS = FRAC_BITS * 2;
    static constexpr unsigned N          = TOTAL_BITS / 64;

    uint64_t w[N];  // little-endian: w[0] = least significant
    int32_t sign;   // 0 = positive/zero, 1 = negative
};
```

The sign bit is stored separately instead of using two's complement because when you're doing multi-word arithmetic, having a separate sign makes the carry chains way simpler. You just do unsigned arithmetic on the magnitude and handle the sign at the boundaries. I was wrong about this at first actually, I tried to do two's complement across multiple words and the borrow propagation was a nightmare, so the separate sign was one of those "oh, that's why real libraries do it that way" moments.

---

## Making It Fast

The key thing that makes this not horrifically slow is that the word count is a compile-time template parameter. The compiler knows at compile time exactly how many loop iterations there are, so it fully unrolls every loop. An addition across 4 words becomes 4 inline add-with-carry sequences, no loop overhead, no branch prediction involved.

```cpp
// N-word add with carry chain
template <unsigned F>
inline void FPN_MagAddN(const uint64_t *a, const uint64_t *b, uint64_t *r) {
    uint64_t carry = 0;
    for (unsigned i = 0; i < FPN<F>::N; i++) {
        uint64_t t  = a[i] + b[i];
        uint64_t c1 = (t < a[i]);
        r[i]        = t + carry;
        uint64_t c2 = (r[i] < t);
        carry       = c1 | c2;
    }
}
```

The carry detection uses comparison, `(t < a[i])` is true when the addition wrapped around, which means there was a carry. No branches, the comparison produces a 0 or 1, and the OR combines carry sources. GCC compiles this into a chain of `add` and `adc` (add-with-carry) instructions, which is literally optimal because that's exactly what the hardware was designed to do.

Multiplication is the expensive one. Multiplying two N-word numbers requires N squared word multiplications, which is why I said 4096-bit precision is overkill. But even at that width, the hot-path operations (price comparison, addition for P&L tracking) are just comparisons and additions, not multiplications. The multiplications happen on the slow path where you're computing regression statistics and regime signals, and there you have the cycle budget for it.

---

## The Division Guard

One thing I learned the hard way is that you absolutely must guard division against zero denominators. `FPN_DivNoAssert` saturates to the maximum representable value when you divide by zero, which sounds like it would be safe but it's actually terrible because it silently produces extreme values that propagate through your calculations and make everything wrong in ways that are really hard to debug.

Every division in the engine now has an explicit zero check.

```cpp
if (FPN_IsZero(denominator)) return;  // or continue, or use fallback
```

Never rely on "it can't be zero in practice." Guard explicitly. I can't stress this enough.

---

## FPN-Only Accounting

This is a rule I set for myself after finding some bugs during an audit. Any code path that touches balance, P&L, fees, equity, or position pricing must use `FPN<F>` for all intermediate calculations. Doubles are only acceptable at system boundaries.

OK to use `double`:
- `FPN_ToDouble` for display, logging, CSV output, printf
- `FPN_FromDouble` at the exchange API boundary (Binance returns doubles)

Not OK:
- Computing equity in doubles and then converting back to FPN for a decision
- Multiplying price and quantity as doubles before converting the product to FPN

The product case is the sneaky one. If price is 67000.12345678 and quantity is 0.00123456, the double multiplication loses precision in the product that you can never recover by converting to FPN after the fact. Do the multiplication in FPN and the precision is preserved all the way through.

---

## Where This Lands in the Engine

The FPN library lives at `FixedPoint/FixedPointN.hpp` and it's used everywhere that math touches money.

| Pattern | Purpose |
|---------|---------|
| `FPN_Add`, `FPN_Sub` | Balance updates, P&L calculation |
| `FPN_Mul` | Fee computation, position sizing |
| `FPN_LessThan`, `FPN_GreaterThanOrEqual` | TP/SL exit comparisons on hot path |
| `FPN_FromDouble` | Exchange API boundary (incoming prices) |
| `FPN_ToDouble` | Display, logging, CSV output |
| `FPN_DivNoAssert` + zero guard | Regression statistics, ratio computation |

The entire hot path, the buy gate, the exit gate, the danger gradient, is all FPN arithmetic. No floats, no doubles, no rounding surprises. When the exit gate compares the current price against a take-profit target, it's doing multi-word unsigned comparison with a branchless carry chain. The result is deterministic regardless of what the compiler does with optimization flags, and that determinism is the whole point.

The journey from "store 101.55 as 10155" to an arbitrary-width template library with branchless carry chains is kind of the journey of this whole project in miniature. Start with a simple idea from a COBOL lecture, figure out why the simple version breaks, and keep iterating until you have something that actually works for real money.

---

*Next: [Chapter 4, Order Encoding and Bit Packing](04_order_encoding.md)*
