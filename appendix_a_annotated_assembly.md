# Appendix A: Annotated Assembly, The Journey to a Single Instruction

This appendix follows the progression of a single piece of functionality, order packing, from its initial implementation through multiple optimizations until the compiler reduced the entire thing to one instruction. This isn't hypothetical, this is the actual assembly output at each stage, annotated as I was learning.

---

## Stage 1: The Nested For Loop (24+ instructions)

My first attempt at packing 8 order IDs into a `uint64_t` used a nested for loop, iterating over 4 buy orders and 4 sell orders, shifting each one into position with OR:

```cpp
std::vector<int32_t>
build_order_book(const std::vector<int8_t> &potential_trades,
                 int32_t queued_orders) {
    int32_t order_bit_packed = 0;
    std::vector<int32_t> order_book_packed((queued_orders + 3) / 4);

    for (int j = 0; j < (int)order_book_packed.size(); j++) {
        order_bit_packed = 0;
        for (int i = 0; i < 4; i++) {
            order_bit_packed |= potential_trades[j * 4 + i] << (i * 8);
        }
        order_book_packed[j] = order_bit_packed;
    }
    return order_book_packed;
}
```

The compiler output for this was a mess. Two nested loops, heap allocation for the vector, `@PLT` calls to `memset` and `_Znwm` (that's `operator new`), and the inner loop had a `movsbl` (sign-extending byte load) plus `sall`/`orl` pair that ran 4 times per outer iteration. Looking at the jump targets you can see `.L8` is the inner loop and `.L9` is the outer:

```asm
.L8:
        movsbl  (%rdx), %eax
        addq    $1, %rdx
        sall    %cl, %eax        ; shift left by (i * 8)
        addl    $8, %ecx
        orl     %eax, %esi       ; OR into packed result
        cmpl    $32, %ecx
        jne     .L8              ; inner loop branch
        movl    %esi, (%rbx,%rdi)
        addq    $4, %rdi
        cmpq    %rdi, %r8
        jne     .L9              ; outer loop branch
```

This works but it's using a vector (heap allocation every call), has two branch points, and is doing redundant work. The compiler can't really help you when your data layout is fighting it.

---

## Stage 2: Struct-Based Packing (18 instructions, no loops)

I realized that if I organized the data into properly-sized structs, the compiler would have a much easier time. Two 4-byte structs (buy side and sell side) packed into an 8-byte struct:

```cpp
struct buy_side {
    uint8_t order0b, order1b, order2b, order3b;
};
static_assert(sizeof(buy_side) == 4, "struct must be 4 bytes");

struct sell_side {
    uint8_t order0s, order1s, order2s, order3s;
};
static_assert(sizeof(sell_side) == 4, "struct must be 4 bytes");

struct OrderPair {
    buy_side buys;
    sell_side sells;
};
static_assert(sizeof(OrderPair) == 8, "struct must be 8 bytes");

uint64_t order_packing_8byte(OrderPair pair) {
    uint64_t packed_orders = 0;
    packed_orders |= static_cast<uint64_t>(pair.buys.order0b);
    packed_orders |= static_cast<uint64_t>(pair.buys.order1b) << 8;
    packed_orders |= static_cast<uint64_t>(pair.buys.order2b) << 16;
    packed_orders |= static_cast<uint64_t>(pair.buys.order3b) << 24;
    packed_orders |= static_cast<uint64_t>(pair.sells.order0s) << 32;
    packed_orders |= static_cast<uint64_t>(pair.sells.order1s) << 40;
    packed_orders |= static_cast<uint64_t>(pair.sells.order2s) << 48;
    packed_orders |= static_cast<uint64_t>(pair.sells.order3s) << 56;
    return packed_orders;
}
```

The assembly dropped to 18 instructions with no loops at all. The compiler unrolled everything into straight-line shifts and ORs:

```asm
        movzbl  %dil, %eax       ; extract byte 0
        movq    %rdi, %rdx
        shrq    $8, %rdx
        movzbl  %dl, %edx        ; extract byte 1
        salq    $8, %rdx         ; shift into position
        orq     %rdx, %rax       ; OR together
        movq    %rdi, %rdx
        shrq    $16, %rdx
        movzbl  %dl, %edx        ; byte 2
        salq    $16, %rdx
        ; ... continues for all 8 bytes
```

It brings a tear to my eye to see myself growing so much through these, first I was using vectors like I actually wanted to code in Java, then I thought about using arrays, then I remembered struct padding and data alignment.

---

## Stage 3: The Revelation (1 instruction)

Then I looked at what was actually happening. The `OrderPair` struct is exactly 8 bytes. The function takes it by value. On x86_64, an 8-byte struct passed by value goes into `%rdi`, the first argument register. And the function returns a `uint64_t`, which goes into `%rax`, the return register.

The 34 lines of shift-and-OR code I wrote are just proving to the compiler that the byte layout of the struct IS the packed integer. The bytes are already in the right positions because that's how structs are laid out in memory. The compiler looked at all my work and went "yeah, I know":

```asm
.LFB9693:
        .cfi_startproc
        movq    %rdi, %rax
        ret
        .cfi_endproc
```

One `movq`. Move the argument register to the return register. That's it. 34 lines of C++ compiled down to a single instruction because the data layout already matched the desired output.

The compiler was really like, don't worry girl, I got you.

---

## Stage 4: The Risk Gate Gets the Same Treatment

I applied the same principle to the risk gate. The first version used de Bruijn multiplication to broadcast a byte across lanes:

```cpp
uint64_t build_risk_gate(risk_gate sides) {
    uint64_t buy = sides.buy_side_risk * 0x0101010101010101ULL;
    uint64_t sell = sides.sell_side_risk * 0x0101010101010101ULL;
    return (buy & 0x00000000FFFFFFFF) | (sell & 0xFFFFFFFF00000000);
}
```

That compiled to two `imul` instructions with a bunch of shifts and masks, like 12 instructions total:

```asm
        movabsq $72340172838076673, %rdx  ; 0x0101010101010101
        movl    %edi, %ecx
        movzbl  %dil, %eax
        movzbl  %ch, %edi
        imull   %edx, %eax                ; de Bruijn broadcast
        imulq   %rdx, %rdi                ; de Bruijn broadcast
        movabsq $-4294967296, %rdx
        andq    %rdx, %rdi
        orq     %rdi, %rax
        ret
```

Then I reorganized the struct layout so the data was already where it needed to be, and:

```asm
_Z15build_risk_gate9risk_gate:
.LFB9698:
        .cfi_startproc
        movq    %rdi, %rax
        ret
        .cfi_endproc
```

Annnnnnnnnddddddddd we won. Literally just a move quad instruction.

---

## The Lesson

The fastest code is the code you never write.

The progression went:
1. **Nested for loop with vectors** (24+ instructions, heap allocation, branches)
2. **Explicit shifts with structs** (18 instructions, no branches, no allocation)
3. **Data layout that matches the output** (1 instruction)

I didn't optimize the algorithm. I organized the data so there was no algorithm needed. The struct IS the packed integer, the compiler saw that, and removed all the transformation code because there was nothing to transform.

The `static_assert` lines are the key, they prove to both the compiler and the reader that the sizes are what you expect. If someone adds a field and breaks the layout, the build fails immediately instead of producing subtle bugs.

This lesson shows up everywhere in the trading engine. The `Portfolio` struct packs position data so the exit gate can walk it with minimal loads. The `BuySideGateConditions` struct is exactly the size the `BuyGate` function expects. The `TUISnapshot` is laid out for sequential reads by the display thread. Data layout is architecture.

Java would never let you do this, and I will die on that hill.

---

*Back to: [Table of Contents](README.md)*
