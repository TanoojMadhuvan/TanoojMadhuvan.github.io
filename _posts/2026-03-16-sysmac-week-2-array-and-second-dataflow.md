---
title: "Week 2: From one PE to a 4x4 array — plus a second dataflow to compare against"
date: 2026-03-16
project: sysmac
tracks: [verification]
summary: "Wiring mac_pe into an NxN grid via generate, a golden model that actually earns its keep, a full 4x4 matmul, and adding output-stationary for a fair PPA comparison."
---
Three things this week: the array itself, proof it does a real matmul (not
just a single vector), and a second dataflow variant — because "I built a
systolic array" isn't the interesting finding on its own. The interesting
finding is comparing two dataflow choices against each other, and that needs
two real implementations, not one.

## `mac_array.sv`: NxN via `generate`

Wired via `generate`, `weight_in[(i*N+j)*DATA_W +: DATA_W]` loads directly
into PE(i,j) (parallel load, not a serial shift-in — simpler for v1),
`act_in[i*DATA_W +: DATA_W]` enters row `i` and flows rightward, `psum_in` at
row 0 is tied to zero and partial sums flow downward:

```systemverilog
for (i = 0; i < N; i++) begin : gen_row
    for (j = 0; j < N; j++) begin : gen_col
        mac_pe #(.DATA_W(DATA_W), .ACC_W(ACC_W)) pe (
            .clk(clk), .rst_n(rst_n),
            .weight_load(weight_load),
            .weight_in  (weight_in[(i*N+j)*DATA_W +: DATA_W]),
            .act_in     (act_wire[i][j]),
            .act_out    (act_wire[i][j+1]),
            .psum_in    (psum_wire[i][j]),
            .psum_out   (psum_wire[i+1][j]),
            .valid_in   (valid_wire[i][j]),
            .valid_out  (valid_wire[i][j+1])
        );
    end
end
```

## Toolchain wall #2: no array-typed ports, at all

Reached for `weight_in [N][N]` (unpacked) as the natural port shape. Hard
syntax error from Yosys (0.64) — confirmed with isolated test cases, not
assumed, that its Verilog frontend doesn't parse array-typed ports of *any*
kind, unpacked or packed-multi-dim alike. Only single flat packed vectors
work. So every array port here is a flat vector sliced with `+:`
part-selects inside the module, and the C++ testbenches pack/unpack it by
hand:

```cpp
// weight_in is a flat N*N*DATA_W-bit packed vector, not array-typed — see
// mac_array.sv's header comment. Verilator represents anything over 64 bits
// as a WData word array: word i holds all of row i's 4 bytes.
void set_weight(Vmac_array* dut, int i, int j, int value) {
    uint32_t w = dut->weight_in[i];
    w &= ~(0xFFu << (j * DATA_W));
    w |= (((uint32_t)value) & 0xFFu) << (j * DATA_W);
    dut->weight_in[i] = w;
}
```

This version is deliberately **unskewed**: hold one activation vector
steady, let the pipeline settle (worst case `~2*(N-1)+1` cycles for the
bottom-right PE), then read `psum_out`. That's a real simplification, flagged
in the RTL's header comment rather than hidden — continuous back-to-back
streaming needs skewed row timing instead, which is next week's problem, not
this week's.

## The golden model that actually earns its keep

`golden/mac_array_golden.py` computes `act @ weights` (single vector) and the
full `activations @ weights` matmul via `numpy` — the real oracle, at the
point where hand-checking a 4x4 array stops being practical:

```python
def golden_matmul(weights: np.ndarray, activations: np.ndarray) -> np.ndarray:
    """Full M x N matmul: activations is (M, N) — M rows, each fed through the
    array sequentially (same stationary weights, one settle per row in the
    unskewed version). Returns (M, N) result C = activations @ weights."""
    assert weights.shape[0] == weights.shape[1] == activations.shape[1]
    return activations @ weights
```

```
PASS identity weights, act=[2,3,5,7]: got [2, 3, 5, 7] expected [2, 3, 5, 7]
PASS general weights, act=[1,2,3,4]: got [9, 4, 7,10] expected [9, 4, 7, 10]
PASS full 4x4 matmul: got [[9,4,7,10],[2,1,2,1],[2,4,1,2],[3,3,3,3]] expected same
PASS streaming matmul: got [[15,5,11,17],[20,8,14,21],[18,17,5,11],[17,25,16,8]] expected same

4/4 golden checks passed
```

## Proving a *full* matmul, not just M=1

The first array testbench only proved a single activation vector through a
stationary weight matrix — that's really the M=1 special case, easy to miss
as "done" when it isn't. Test 3 feeds all 4 rows of an actual A matrix
through the *same* stationary weight load (no reload between rows) and
checks each output row against `golden_matmul`:

```
=== mac_array Testbench (N=4) ===
PASS identity weights, act=[2,3,5,7]: got [2,3,5,7] expected [2,3,5,7]
PASS general weights, act=[1,2,3,4]: got [9,4,7,10] expected [9,4,7,10]
PASS matmul row 0: got [9,4,7,10] expected [9,4,7,10]
PASS matmul row 1: got [2,1,2,1] expected [2,1,2,1]
PASS matmul row 2: got [2,4,1,2] expected [2,4,1,2]
PASS matmul row 3: got [3,3,3,3] expected [3,3,3,3]
PASS full 4x4 matmul (all rows)
```

## Adding output-stationary, sized for a fair comparison

The actual point of this project is a PPA *comparison*, which means the
second dataflow needs to exist before synthesis means anything. Added
`mac_pe_os.sv` / `mac_array_os.sv` — output-stationary, same PE count (4x4),
same workload (4x4 @ 4x4 matmul) as weight-stationary, so the eventual area/
timing/power numbers compare apples to apples.

The structural difference matters for what's coming next week: in
output-stationary, the accumulator stays local (becomes one output element)
while **both** activation and weight stream through the PE — neither operand
is held constant the way weight-stationary holds its weight. That has a
real consequence spelled out directly in the RTL's header comment, ahead of
actually hitting it:

> REQUIRES skewed input feeding to be correct — Row i's activation sequence
> must start i cycles later than row 0's, and column j's weight sequence
> must start j cycles later than column 0's. Without this stagger, only the
> diagonal PEs (i == j) end up multiplying correctly paired values — this
> isn't an optional optimization, unskewed feeding is simply incorrect for
> this dataflow.

Unlike `mac_array.sv`, there's no unskewed baseline for output-stationary at
all — `mac_array_os_tb.cpp` drives it with the skewed sequence from day one,
checked against the *exact same* A/weights/expected result as
weight-stationary's matmul test, so both dataflows are proven against one
shared oracle:

```
=== mac_array_os Testbench (N=4, K=4) ===
PASS full 4x4 matmul via output-stationary dataflow, one pass, 14 cycles
```

One pass, no settle-per-row reload needed — output-stationary produces the
*entire* output matrix in one streaming pass, which is a real structural
difference from weight-stationary's per-row settle. Whether that translates
into an actual speed advantage is a measured question, not an assumed one —
taken up directly in Week 4.

**Next:** stop measuring gate count as a PPA proxy and get real numbers —
Nangate45-mapped synthesis and OpenSTA timing/power for both variants.
