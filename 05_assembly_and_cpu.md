# Chapter 5: Assembly and the CPU

So the first time I ran `g++ -S -O2` on one of my files and looked at the output, I kind of lost my mind. Like, you can see exactly what the compiler turns your code into. Every decision, every optimization, every shortcut, it's all right there. I went from being terrified of assembly to reading it for fun in about 48 hours, and honestly I think understanding assembly made me a better C++ programmer overnight, because now I could verify whether my "branchless" code was actually branchless, or whether I was just fooling myself.

This chapter covers x86_64 register layout, AT&T syntax (which is what GCC outputs on Linux), reading compiler output, and the key instructions you need to recognize when you're checking whether your hot path is actually branch-free.

---

## The Register File

The CPU registers are the fastest memory that exists. Even faster than L1 cache, because there's no fetch involved, the data is literally inside the CPU already. When you see something like `cmovl %eax, %ebx` in assembly, that's the CPU shifting values between registers with zero cache hit, zero memory bus transfer, just electrons moving around inside the chip.

On x86_64, the general-purpose registers come in different widths.

```
64-bit   32-bit   16-bit   8-bit
------   ------   ------   -----
%rax     %eax     %ax      %al      (accumulator, return values, arithmetic)
%rbx     %ebx     %bx      %bl      (base, callee-saved)
%rcx     %ecx     %cx      %cl      (counter, loop counts, shift amounts)
%rdx     %edx     %dx      %dl      (data, pairs with rax for mul/div)
%rsi     %esi     %si      %sil     (source index, 2nd function argument)
%rdi     %edi     %di      %dil     (destination index, 1st function argument)
%rsp     %esp     %sp      %spl     (stack pointer, ALWAYS current stack top)
%rbp     %ebp     %bp      %bpl     (base pointer, frame pointer)
%r8-%r15                             (extended registers, function args 3-6)
```

These registers have conventional purposes that matter when you're reading assembly. `%rax` is where return values go and where most arithmetic results land, so whenever a function returns something, it ends up in `%rax`. `%rcx` is the counter register, used for loop counts and shift amounts. `%rdx` is the data register that pairs with `%rax` for wide multiplication and division results.

The `%rdi` and `%rsi` registers are the first and second function arguments on Linux (System V calling convention). So when you see `movzbl %dil, %eax` at the start of a function, that's just loading the first argument into the accumulator.

Knowing why `%eax` keeps showing up as the return value, or why the loop counter lands in `%ecx`, makes assembly way more readable. It stops being random letters and starts being a story about data flow.

---

## AT&T Syntax

GCC on Linux outputs AT&T syntax, which has some quirks that confused me at first. The main ones.

The `$` prefix means a literal value. So `movl $1, %edx` means "put the number 1 into the %edx register." The `%` prefix means a register name. And the size suffixes tell you the operand width.

```
b = byte    (8-bit)
w = word    (16-bit)
l = long    (32-bit)
q = quad    (64-bit)
```

So `movl` is "move 32-bit", `movq` is "move 64-bit", `movsbl` is "move sign-extend byte to long" (takes an 8-bit value and sign-extends it to 32 bits), and `movzbl` is "move zero-extend byte to long" (same but fills with zeros, which is what you want for unsigned values).

The direction of operands in AT&T syntax is source, then destination. So `movl %eax, %ebx` means move FROM `%eax` TO `%ebx`. This is backwards from Intel syntax which does destination first, and it tripped me up for a while until I internalized it.

Labels like `.LFB3762` mean Local Function Begin, and `.LFE3762` means Local Function End. The `.cfi_startproc` and `.cfi_endproc` markers serve the same purpose. The `.cfi_def_cfa_offset` lines are the DWARF unwinder tracking the stack frame, the CPU needs to know where the stack is at all times so it can unwind properly if there's an exception. They look scary but they're just bookkeeping.

---

## The Instructions That Matter

For verifying branchless code, you only need to recognize a handful of instruction patterns.

**TEST and CMP**: These set flags without modifying the operands. `testl %esi, %edi` ANDs the two registers and sets flags based on the result, but throws away the AND result. It's used before conditional operations to set up the zero flag (ZF) or sign flag (SF).

**CMOV (conditional move)**: This is the gold standard for branchless code. The CPU calculates both possible values, and cmov just picks which one to keep at the last pipeline stage. No prediction, no flush. The variants tell you the condition.

```
cmove   / cmovz    - move if equal / zero
cmovne  / cmovnz   - move if not equal / not zero
cmovl              - move if less (signed)
cmovge             - move if greater or equal (signed)
cmova              - move if above (unsigned)
cmovb              - move if below (unsigned)
cmovs              - move if sign flag set (negative)
```

Important distinction: `cmovl` (conditional move if less) is completely different from `movl` (move 32-bit). The `l` in `cmovl` is a condition code meaning "less than", the `l` in `movl` is a size suffix meaning "long/32-bit." This confused me for longer than I'd like to admit.

**SETE/SETNE/SETL**: Set a byte register to 0 or 1 based on a flag condition. This is how the compiler materializes boolean results. When you write `return (a & b) == b`, the compiler emits `testl` followed by `sete %al`, which puts 0 or 1 into the low byte of the accumulator.

**LEA (load effective address)**: This instruction was originally designed for address computation, but the compiler uses it as a sneaky math unit. It has its own ALU separate from the main execution unit, so `lea eax, [ebx + ebx * 2]` computes `eax = ebx * 3` entirely within the register file without using `imul`. The scale factor is limited to 1, 2, 4, or 8 though, so you can only get odd multiples like 3 (`1 + 2`), 5 (`1 + 4`), and 9 (`1 + 8`). The compiler uses LEA to avoid calling `imul` which has higher latency, because shifting plus LEA is a single operation while `imul` is typically around 3 cycles.

**SAR/SHR**: Arithmetic shift right (preserves sign) vs logical shift right (fills with zeros). `sarl $2, %eax` is signed division by 4. This is how the branchless clamp from chapter 2 actually works in hardware, `x >> 31` compiles to a SAR that smears the sign bit.

**IMUL**: Signed integer multiplication. Comes in three flavors, one operand (full-width result in EDX:EAX), two operand (truncated result in destination), and three operand (src times immediate, stored in dest). On modern cores, `imul` has predictable, consistent latency around 3 cycles, which is why the compiler loves using it for strength reduction of loops into constant multiplies.

---

## Reading Compiler Output

The workflow is dead simple.

```bash
g++ -S -O2 program.cpp -o program.s
c++filt < program.s | less
```

The `-S` flag tells the compiler to output assembly instead of an object file. `-O2` is the optimization level, which is important because unoptimized assembly is basically useless for understanding what the hot path actually does. `c++filt` demangles the C++ name mangling so you can read function names instead of `_Z16build_order_bookRKSt6vectorIhSaIhEEj`.

When you're looking at the output, the things to check are.

**Good signs** (branchless):
- `test` / `and` / `cmov` sequences
- `sete` / `setne` / `setl` for boolean materialization
- No `jmp`, `je`, `jne` between your hot-path operations
- LEA for arithmetic instead of multiply

**Bad signs** (branching):
- `je` / `jne` / `jmp` in the middle of your logic
- Multiple label targets (`.L5:`, `.L8:`) with jumps between them
- `call` instructions to library functions (PLT calls)

**Acceptable jumps**: For loops with predictable iteration counts (like `i < 32`), the `jne` at the end of the loop is fine. Branch prediction gets these right 99.9% of the time because they always fall the same way except for the very last iteration. The dangerous jumps are the ones in data-dependent conditions where the predictor has to guess based on volatile data.

---

## A Real Example

Here's the kill switch function and what the compiler does with it.

```cpp
int32_t kill_switch(int32_t order_book_state, int32_t kill_mask) {
    return (order_book_state & kill_mask) == kill_mask;
}
```

Compiles to:

```asm
_Z11kill_switchii:
    notl    %edi          ; ~order_book_state
    xorl    %eax, %eax    ; clear return register
    testl   %esi, %edi    ; (~state) & mask, set ZF
    sete    %al           ; return 1 if ZF set (all mask bits present)
    ret
```

Four instructions. Zero branches. The compiler converted `(A & B) == B` to `(~A & B) == 0` because it maps directly to the TEST instruction, which does the AND and sets flags without storing the result. Then SETE materializes the boolean. This is as clean as it gets.

Compare that to a version with function calls in the body of an if statement, you'd get jump instructions, branch prediction involvement, and non-deterministic execution time. The ternary operator tends to produce CMOV more reliably than if/else with complex bodies, because the simpler the operation, the more willing the compiler is to keep it branchless.

---

## The Inline Assembly Discovery

I'm going to try to be calm about this but honestly when I found out you can write assembly directly inside C++ code, I was not calm. At all. I was the opposite of calm.

You can embed raw assembly instructions in your C++ using the `__asm__` keyword.

```cpp
__asm__ volatile("mfence" ::: "memory");
```

This specific instruction is a memory fence, it tells the CPU to complete all pending memory operations before continuing. You need this when you're using `__rdtsc()` (read timestamp counter) to measure cycle counts, because without the fence, the out-of-order execution engine might reorder your timing calls relative to the code you're trying to measure.

The `volatile` keyword tells the compiler "do not optimize this away, do not reorder this, just emit it exactly where I put it." The `"memory"` clobber tells the compiler that this instruction might read or write any memory location, so it shouldn't cache any memory values across this barrier.

I used this pattern to measure cycle counts for different functions.

```cpp
__asm__ volatile("mfence" ::: "memory");
uint64_t start = __rdtsc();
// ... code being measured ...
uint64_t end = __rdtsc();
__asm__ volatile("mfence" ::: "memory");

uint64_t cycles = end - start;
```

You'd never want to use inline assembly on a hot path in production, the compiler is smarter than you 99.9% of the time and you'd just be getting in its way. But for measurement and verification it's invaluable. And knowing that the bridge between C++ and the hardware is always there, that you can always drop down a level if you need to, that changed how I thought about the language. Java definitely doesn't let you do this.

---

## The Stack Frame Dance

One more pattern worth understanding because it shows up at the start and end of basically every function.

```asm
push    rbp           ; save the OLD base pointer
mov     rbp, rsp      ; set YOUR base pointer to current stack top
sub     rsp, N        ; make room for local variables
```

And the reverse on return.

```asm
mov     rsp, rbp      ; throw away local variables
pop     rbp           ; restore caller's base pointer
ret                   ; jump back to caller
```

Sometimes the compiler replaces the teardown with a single `leave` instruction that does the same thing. The reason this exists is that `%rbp` gives you a fixed reference point within the stack frame. Once you set `rbp = rsp` at the start, `%rbp` doesn't move for the entire function, even as you push and pop other stuff. Local variables are negative offsets from `%rbp`, like `[rbp - 8]` is your first local, `[rbp - 16]` is your second.

But here's the interesting thing, when you compile with `-O2`, the compiler often omits the frame pointer entirely. It uses `%rsp` directly for everything and frees up `%rbp` as an extra general-purpose register. The `.cfi_def_cfa_offset` DWARF annotations you see everywhere are the compiler telling the debugger "I moved the stack pointer, here's how to find things now" since there's no stable frame pointer to reference.

---

## Condition Code Suffixes

For completeness, here are the condition codes you'll see on jump and cmov instructions.

```
e  / z     equal / zero
ne / nz    not equal / not zero
g          greater (signed)
ge         greater or equal (signed)
l          less (signed)
le         less or equal (signed)
a          above (unsigned)
ae         above or equal (unsigned)
b          below (unsigned)
be         below or equal (unsigned)
s          sign flag set (negative)
ns         sign flag not set
```

The signed vs unsigned distinction matters in trading code. `g` and `l` respect the sign bit, while `a` and `b` treat everything as unsigned. For the FPN library where we separated the sign bit and do unsigned magnitude arithmetic, we want the unsigned variants.

The CPU flags themselves are.

```
ZF = zero flag       (result was zero)
CF = carry flag      (unsigned overflow)
SF = sign flag       (result was negative)
OF = overflow flag   (signed overflow)
```

After any arithmetic instruction, these flags are set automatically. That's why checking if an integer is negative is literally free on x86, the sign flag is already set from whatever the last operation was.

---

## Where This Lands in the Engine

Understanding assembly isn't about writing assembly. It's about being able to verify that the C++ code you wrote actually compiles to what you intended.

In the trading engine, every hot-path function was checked against its assembly output.

| Function | Expected pattern | Verified |
|----------|-----------------|----------|
| BuyGate | TEST/AND/CMOV, zero JMP | Yes, mask-and-zero, no branches |
| PositionExitGate | Comparison chain, no JE/JNE | Yes, branchless FPN comparisons |
| Danger gradient | Multiplication chain, no branches | Yes, proportional scaling |
| Kill switch halt | Single mask-and-zero | Yes, gates on `buying_halted` flag |

The `c++filt` workflow is also how I caught cases where the compiler decided a ternary was too complex and fell back to branching. When that happens you simplify the expression, re-check the assembly, and iterate until it's clean.

The ability to read assembly also helps you understand why certain data layouts are faster than others. When you see the compiler emitting `movzbl` to widen a byte to a register, followed by `salq` to shift it into position, followed by `orq` to merge it, and you know that chain can run on independent bytes simultaneously because of the CPU's out-of-order engine, you start designing your data structures to take advantage of that parallelism. That's the real payoff, not writing assembly, but thinking like the hardware.

---

*Next: [Chapter 6, Memory Architecture](06_memory_architecture.md)*
