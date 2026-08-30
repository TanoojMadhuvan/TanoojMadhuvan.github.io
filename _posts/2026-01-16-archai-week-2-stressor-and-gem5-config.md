---
title: "Week 2: The stressor workload, the gem5 config, and a first look at the dashboard"
date: 2026-01-16
project: archai
tracks: [ppa-analysis]
summary: "uarch_stressor.c gives gem5 something real to execute, uarch_spec.py wires a parameterized cache hierarchy around it, and presilicon_dashboard.py turns raw parameter ranges into something a human can actually steer."
---
Last week settled the architecture and cleared the gem5-build wall. This week
is about the two pieces that turn that scaffold into something that can
actually run a trial: a real workload for gem5 to execute, and a real,
parameterized gem5 config to execute it in.

## A workload worth stressing a cache hierarchy with

`uarch_stressor.c` is deliberately not one algorithm — it runs bubble sort,
merge sort, and quicksort back-to-back against copies of the *same* random
array, each timed independently:

```c
#define N 100   // Change this to scale the experiment

int main() {
    srand(time(NULL));
    int *original = malloc(N * sizeof(int));
    int *arr = malloc(N * sizeof(int));
    for (int i = 0; i < N; i++) original[i] = rand();

    clock_t start, end;

    copy_array(original, arr, N);
    start = clock();
    bubble_sort(arr, N);
    end = clock();
    printf("Bubble Sort Time: %.6f seconds\n", (double)(end - start) / CLOCKS_PER_SEC);

    copy_array(original, arr, N);
    start = clock();
    merge_sort(arr, 0, N - 1);
    end = clock();
    printf("Merge Sort Time: %.6f seconds\n", (double)(end - start) / CLOCKS_PER_SEC);

    copy_array(original, arr, N);
    start = clock();
    quick_sort(arr, 0, N - 1);
    end = clock();
    printf("Quick Sort Time: %.6f seconds\n", (double)(end - start) / CLOCKS_PER_SEC);
    // ...
}
```

Three sorts in one binary means three genuinely different access and control
patterns for the simulated core to execute: bubble sort's tight O(n²)
compare-swap sweep, merge sort's recursive divide-and-conquer with a
`malloc`/`free` pair on every merge call, quicksort's in-place partitioning.
That variety is the point — a single algorithm risks the cache-sensitivity
results being an artifact of that one access pattern rather than something
general. `N = 100` is a small, deliberately modest working set — small
enough to matter later (Week 3 leans on exactly how small it is). The binary
gets cross-compiled for the simulated target with
`aarch64-linux-gnu-gcc uarch_stressor.c -static -o microbench.arm`.

## `uarch_spec.py`: turning a JSON blob into a gem5 board

Every architectural knob this project can turn lives in `params.json`, and
`uarch_spec.py` is the translation layer from that JSON into gem5's actual
Python object model:

```python
with open(PARAM_FILE) as f:
    params = json.load(f)["vars"]

requires(isa_required=ISA.ARM)

cache_hierarchy = PrivateL1SharedL2CacheHierarchy(
    l1i_size=params["l1i_size"],
    l1i_assoc=params["l1i_assoc"],
    l1d_size=params["l1d_size"],
    l1d_assoc=params["l1d_assoc"],
    l2_size=params["l2_size"],
    l2_assoc=params["l2_assoc"],
)

memory = SingleChannelDDR3_1600(size=params["DDR_memory_size"])

processor = SimpleProcessor(
    cpu_type=CPUTypes.TIMING,
    isa=ISA.ARM,
    num_cores=params["num_cores"],
)

board = SimpleBoard(
    clk_freq="3GHz",
    processor=processor,
    memory=memory,
    cache_hierarchy=cache_hierarchy,
)

binary = CustomResource(local_path=str(Path(__file__).parent / "microbench.arm"))
board.set_se_binary_workload(binary)

simulator = Simulator(board=board)
simulator.run()
```

Nothing here is hardcoded: private per-core L1 instruction and data caches
feeding a shared L2, a single-channel DDR3-1600 memory system sized off
`DDR_memory_size`, a timing-accurate (not functional-only) CPU model so
cache and memory latency actually show up in the results, and the core count
itself pulled from the same JSON. Every parameter the dashboard will
eventually let a human (or Gemini) set flows through this exact same path —
there's no separate "manual mode" config drifting out of sync with the
automated one.

## The orchestrator's actual job

`main.py` is where the loop lives, and its job is narrower than "call an
LLM": load the current phase's hypothesis and parameter target, trigger a
gem5 run against that target via `uarch_spec.py`, pull real metrics back out
of gem5's own stats output (IPC, CPI, simulated time, memory usage — not
estimates), and feed those numbers back to Gemini to decide the *next*
trial's value.

The deliberate design choice: **interpolate across a phase's parameter range
over 10-20 trials, don't brute-force every representable value.** Sweeping
`l1i_size` from 1kB to 256kB one doubling at a time would already be
tractable by hand — the reason to hand this to an LLM at all is deciding
*where in the range results actually change*, and stopping a phase once
that's answered rather than exhausting the range for its own sake.

## Putting a human in the loop, not just a script

<figure>
  <img src="/assets/images/archai/archai-dashboard-parameter-ranges.png" alt="ARCHAI Streamlit dashboard's Pre-Experiment tab: a table of microarchitecture parameters (l1i_size, l1i_assoc, l1d_size, l1d_assoc, l2_size, l2_assoc, DDR_memory_size, num_cores) each with a checkbox to include it in the run and Min/Max range fields, showing l1i_size 1kB-256kB, l1i_assoc 1-5, l1d_assoc 3-7, num_cores 1-9, and a Deploy button top-right">
  <figcaption>The Pre-Experiment tab of <code>presilicon_dashboard.py</code>: every parameter Gemini is allowed to touch, with its own Min/Max range and an on/off checkbox, before a single trial runs.</figcaption>
</figure>

This is the actual control surface: which parameters are even in scope for
this run (unchecked ones — `l1d_size`, `l2_size`, `l2_assoc`,
`DDR_memory_size` here — stay fixed at their defaults), and the legal range
for each one that *is* in scope. Gemini doesn't get to invent a search space;
it gets to explore inside the one a human explicitly opened up, one **Change**
at a time, before ever pressing **Deploy**.

**Next:** press Deploy, and see what Gemini actually does with these
ranges — the phases it proposes on its own, and the first real trial results
to come back from gem5.
