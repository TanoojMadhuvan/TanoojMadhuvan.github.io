---
title: "Week 1: Why KV-cache, and what I'm targeting"
date: 2026-07-21
project: memory-hierarchy-sim
tracks: [kv-cache]
summary: "Background reading on transformer inference bottlenecks and defining a concrete accelerator target."
---
REPLACE_ME: replace with your actual write-up.

## Background

REPLACE_ME: summarize why KV-cache bandwidth (not compute) is the bottleneck
in autoregressive decode, and cite what you read.

## Target

REPLACE_ME: define the concrete piece you're building — e.g. a memory-mapped
scratchpad peripheral with a custom addressing scheme for KV blocks, attached
to the core over its existing data-memory interface.

## Interface sketch

```systemverilog
module kv_cache_unit #(
    parameter ADDR_WIDTH = 16,
    parameter DATA_WIDTH = 32
) (
    input  logic                    clk,
    input  logic                    reset,
    input  logic [ADDR_WIDTH-1:0]   addr,
    input  logic [DATA_WIDTH-1:0]   wdata,
    input  logic                    we,
    output logic [DATA_WIDTH-1:0]   rdata
);
    // REPLACE_ME
endmodule
```

## Next

REPLACE_ME: e.g. build a C++ functional model of the access pattern before
writing any RTL.
