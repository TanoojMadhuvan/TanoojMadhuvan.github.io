---
title: "Week 1: Designing the processing element"
date: 2026-07-14
project: fpga-systems-lab
tracks: [systolic-array]
summary: "First pass at the systolic array's PE — MAC datapath and dataflow direction."
---
REPLACE_ME: replace with your actual write-up.

## Dataflow choice

REPLACE_ME: weight-stationary vs. output-stationary vs. input-stationary, and
why you picked one.

## SystemVerilog: single PE

```systemverilog
module pe (
    input  logic         clk,
    input  logic         reset,
    input  logic [15:0]  a_in,
    input  logic [15:0]  b_in,
    output logic [15:0]  a_out,
    output logic [15:0]  b_out,
    output logic [31:0]  acc
);
    always_ff @(posedge clk) begin
        if (reset) begin
            a_out <= '0;
            b_out <= '0;
            acc   <= '0;
        end else begin
            a_out <= a_in;
            b_out <= b_in;
            acc   <= acc + (a_in * b_in);
        end
    end
endmodule
```

## C++: golden model for a small NxN array

```cpp
std::vector<std::vector<int32_t>> matmul_reference(
    const std::vector<std::vector<int16_t>>& A,
    const std::vector<std::vector<int16_t>>& B) {
    size_t n = A.size(), m = B[0].size(), k = B.size();
    std::vector<std::vector<int32_t>> C(n, std::vector<int32_t>(m, 0));
    for (size_t i = 0; i < n; i++)
        for (size_t j = 0; j < m; j++)
            for (size_t p = 0; p < k; p++)
                C[i][j] += int32_t(A[i][p]) * int32_t(B[p][j]);
    return C;
}
```

## Next

REPLACE_ME: e.g. wire N×N PEs together and validate against the C++ golden
model on small matrices before scaling up.
