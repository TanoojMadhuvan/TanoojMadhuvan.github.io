---
title: "Week 2: The single-cycle core comes together"
date: 2026-03-27
project: foundation-core
summary: "Wiring up the single-cycle datapath, getting per-module testbenches green, and a hardcoded path that's going to bite someone."
---
This week was `rtl/top.sv` and everything it instantiates: `imem`, `regfile`,
`immgen`, `control`, `alu_control`, `alu`, `dmem` — the classic single-cycle
datapath, one instruction fully through IF→ID→EX→MEM→WB every clock edge. The
register file is 32×64-bit (`regs[31:0]`, each 64 bits wide), with `x0` reads
hardwired to zero and writes to `x0` silently dropped
(`if (we && rd != 5'b0)`). Instruction memory is 1024 words (4KB), data
memory is 1024 doublewords (8KB) — plenty for anything I'm testing right now.

## Testing module by module

Before trusting the integrated `top.sv`, I wrote a directed Verilator
testbench per module: 5 tests for the ALU (add/sub/and/or plus the zero
flag), 5 for the main control unit (one per opcode class), 8 for ALU control
(cross-checking `alu_op`/`funct3`/`funct7[5]` combinations), 4 for the
register file (write/read, dual-port read, `x0` behavior, write-enable gating),
5 for the immediate generator (I-type positive/negative, S-type, B-type
positive/negative), and 3 for instruction memory fetch. That's 34 directed
test cases across 7 testbenches, all self-checking (`return fail ? 1 : 0`) —
nothing here is eyeballed.

<figure>
  <img src="/assets/images/foundation-core/control-unit-testbench.png" alt="control_tb.cpp directed testbench for the main control unit">
  <figcaption>The control unit testbench — one directed test per opcode class (R-type, I-arith, load, store, branch), each asserting the full control-signal tuple in one shot.</figcaption>
</figure>

One of the immgen test comments is honest about its own laziness —
`imem_tb.cpp` fetches a hardcoded `add x3, x1, x2` encoding with the comment
`// wait — wrong, but close enough for fetch test`, because the point of that
particular test was only "did `imem` return the right word at the right
address," not "is this a real program." Leaving it as-is since it does
exactly what it's testing.

## The mistake

`rtl/imem.sv` loads its contents like this:

```systemverilog
initial $readmemh("/home/tanoojkanike/riscv/hex.txt", mem);
```

That's a hardcoded absolute path, not a relative one. It works fine as long
as I'm running from that exact machine and directory, and breaks silently
(or loudly, with an empty instruction memory) anywhere else.

I only caught it when running from a different machine/directory — on the
machine and directory the path was hardcoded to, everything just worked, so
there was no error to notice in the first place.

## Open question

`immgen.sv` already decodes U-type (`lui`) and J-type (`jal`) immediates —
the case statement is right there — but `control.sv` has no case for either
opcode (`0110111` or `1101111`), so those instructions decode fine and then
go nowhere. Not fixing it yet; just noting the asymmetry so I remember it's
there when I do want `jal` for anything.

No Yosys/Verilator errors worth logging yet — synthesis hasn't happened, and
Verilator compiled clean for all 7 testbenches.

<figure>
  <img src="/assets/images/foundation-core/sum-1-to-10-trace.png" alt="Cycle-by-cycle trace of the sum-1-to-10 program running on the Verilated top.sv to completion">
  <figcaption>The sum-1..10 program from <code>INSTRUCTIONS.md</code>'s "Complete Example," traced cycle-by-cycle off the actual Verilated <code>top.sv</code> — 36 cycles, 1 instruction/cycle, no stalls, matching the C++ oracle exactly: <code>x2=55</code> (the sum), <code>x3=55</code> (loaded back from memory), <code>mem[0]=55</code>.</figcaption>
</figure>
