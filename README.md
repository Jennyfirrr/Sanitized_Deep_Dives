# From Bits to Branchless: Building a Trading Engine from First Principles

*To my CS professors, Dr. Unan, and Dr. Johnstone <3*

---

This is a collection of notes I wrote while teaching myself low-latency systems programming. It started because I got bored in a Java lecture and wanted to understand why HFT firms care about bitwise operations. It ended with me building a live crypto trading engine from scratch in C++ in about three weeks.

These aren't polished tutorials written after the fact. They follow the actual learning path, including the parts where I was wrong about something and had to go back and figure out why. I think that's more useful than pretending I understood everything the first time.

The code examples all compile and the concepts all connect to a real system. If you want to see where this stuff actually landed, the engine is at [github.com/Jennyfirrr/FoxML_Trader](https://github.com/Jennyfirrr/FoxML_Trader).

---

## Chapters

1. **[Bit-Level Computation](01_bit_level_computation.md)** — OR, XOR, AND, shifts, bitfields, and why a single integer can replace 64 booleans
2. **[Branchless Execution](02_branchless_execution.md)** — branch prediction, pipeline flushes, two's complement, and why variance matters more than speed
3. **[Fixed-Point Arithmetic](03_fixed_point_arithmetic.md)** — COBOL-inspired determinism, the journey from "shift right 16" to a 4096-bit arbitrary-width library
4. **[Order Encoding and Bit Packing](04_order_encoding.md)** — packing orders into integers, de Bruijn multiplication, the Mycroft trick, and SWAR
5. **[Assembly and the CPU](05_assembly_and_cpu.md)** — x86_64 registers, reading compiler output, key instructions, and the moment I found out you can write ASM inline in C++
6. **[Memory Architecture](06_memory_architecture.md)** — cache hierarchy, alignment, pool allocators, and the moment 34 lines of code compiled to a single MOV instruction
7. **[Lock-Free Concurrency](07_lock_free_concurrency.md)** — CAS, atomics, spin loops, and sharing data between threads without anyone waiting
8. **[From Theory to Engine](08_from_theory_to_engine.md)** — how all of this came together into a live trading system

---

*~16,000 words. Written between 1am and 4am on various weeknights when I should have been doing my Java homework.*
