---
title: "Week 1: Scoping the ISA and setting up the toolchain"
date: 2026-03-20
project: foundation-core
summary: "Deciding what subset of RV64I to actually build first, and getting the assembler off the ground."
---
Starting this whole thing with a scoping question: how much of RV64I do I
actually need to prove out a working core end to end, versus how much am I
tempted to build because it's "supposed" to be there? I landed on a
deliberately narrow first cut: R-type `add`/`sub`/`and`/`or`, I-type `addi`,
`ld`/`sd` for memory (64-bit doublewords only, 8-byte aligned), and `beq`/`bne`
for control flow. Right now the goal is a complete
fetch→decode→execute→memory→writeback loop I actually trust, not ISA coverage.

Everything else is on the later list, not the focus right now:

- `jal` / `jalr` (jumps)
- `lui` (upper immediate)
- shift instructions
- the remaining ALU ops

## Toolchain

Settled on: `g++` for the assembler and a C++ reference simulator, Verilator
for per-module RTL testbenches, and Yosys for synthesis (generic + Nangate45
via OpenROAD-flow-scripts down the line). SystemC and UVM are on that same
later list — worth revisiting once the out-of-order work needs more rigorous
verification, but not a priority yet.

## The assembler

`assembler.cpp` is a two-pass assembler — first pass resolves label addresses,
second pass emits the actual 32-bit instruction words. It's 279 lines right
now: register parsing (`x0`–`x31` or ABI names like `a0`, `t0`, `s1`, via a
`REG_MAP` table), immediate parsing with explicit range checks
(`checkImmRange`) so a too-large immediate fails at assemble time with a line
number instead of silently truncating, and one `encode*` function per
instruction format (R/I/S/B).

<figure>
  <img src="/assets/images/foundation-core/risc-v-instruction-formats.png" alt="RISC-V instruction formats">
  <figcaption>Source: <em>The Hardware/Software Interface: RISC-V Edition</em>, David A. Patterson &amp; John L. Hennessy (University of California, Berkeley).</figcaption>
</figure>

The label resolution in the first pass is just a symbol table — an
`unordered_map<string, uint32_t>` from label name to byte address, built while
walking the source once to compute instruction addresses:

```cpp
std::unordered_map<std::string, uint32_t> labels;
```

<figure>
  <img src="/assets/images/foundation-core/assembler-first-pass.png" alt="First-pass label-resolution loop in assembler.cpp">
  <figcaption>The actual first-pass loop: walk the source once, record label addresses, and collect instructions — no encoding happens yet.</figcaption>
</figure>

`assembly.txt` right now is a small multiply-by-repeated-addition loop —
12 × 11 done as eleven additions, stored to address 0 when the counter hits
zero:

```
    addi x1, x0, 12
    addi x2, x0, 11
    addi x3, x0, 0
    addi x4, x0, 1

contmul:
    beq x2, x0, done
    add x3, x3, x1
    sub x2, x2, x4
    beq x0, x0, contmul

done:
    sd x3, 0(x0)
```

Each instruction is 4 bytes, and `contmul`/`done` are the 5th and 9th
instructions — so at addresses `0x10` and `0x20` — which is exactly what the
first pass's symbol table ends up holding:

| Label     | Address      |
|-----------|--------------|
| `contmul` | `0x00000010` |
| `done`    | `0x00000020` |

The second pass looks a branch's target label up in that table and computes
the PC-relative offset from there. Having the table as a concrete example
makes the two-pass split a lot less abstract than just saying "labels get
resolved first."

<figure>
  <img src="/assets/images/foundation-core/assembler-second-pass.png" alt="Second-pass instruction encoding in assembler.cpp for ld, sd, beq, and bne">
  <figcaption>The second pass encoding `ld`/`sd`/`beq`/`bne` — by now every label is a known address, so the branch case can compute and range-check its offset directly.</figcaption>
</figure>

The B-type encoder was the fussy one — RISC-V scrambles the branch-offset bits
across the instruction word in a specific order
(`imm[12|10:5] | rs2 | rs1 | funct3 | imm[4:1|11] | opcode`) instead of
keeping them contiguous, presumably so more of the immediate bits stay in the
same position as other formats for decoder reuse. Getting that bit-shuffle
right on the first try felt unlikely, so I leaned on `immgen_tb.cpp`-style
directed tests early rather than trusting the algebra.

No tool or setup errors worth logging yet — the friction so far has all been
in getting the B-type bit order right, covered above.

**Next:** get `assembler.exe` producing `hex.txt` from a real program
(`assembly.txt` has a small loop-and-store example already) and get that hex
loading into a testbench before writing a single line of datapath RTL.
