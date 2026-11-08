# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->258.243<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.243<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->459806.3<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13500.8<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4441<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2404.035 ms, 2 threads: 2167.593 ms, 4 threads: 2357.812 ms, 8 threads: 2657.842 ms. Fastest: 2 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
Both versions perform the same multiply-adds; the difference is the cost per operation. The naive version (258.2 ms) spends almost all its time in the Python interpreter: every scalar multiply-add of every dot product pays for bytecode dispatch, type checks and float-object allocation, one value at a time. The vectorized version (1.24 ms, 207.8× faster) makes one call into OpenBLAS (Haswell target), compiled code that uses AVX2/FMA SIMD instructions — 8 float32 multiply-adds per instruction — over cache-sized blocks, with no interpreter in the loop.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve bottoms out at 2 threads (2167.6 ms vs 2404.0 ms at 1 thread, only ~10% faster) and then gets worse: 2357.8 ms at 4 and 2657.8 ms at 8 — slower than a single thread. The Colab runtime has 2 vCPUs, which lscpu shows are 1 physical core × 2 hyperthreads: the two threads share one core's AVX/FMA units and caches, so 2 threads barely beat 1. Beyond 2, threads outnumber vCPUs: the OS context-switches them, every parallel op ends at a barrier waiting for the slowest (often descheduled) thread, and threads evict each other's cache lines — coordination costs more than it buys.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
The GPU was 34× faster (459806.3 ms on CPU vs 13500.8 ms on GPU for the same 100 steps). Training a transformer is mostly big matrix multiplications, which are millions of small multiplications that don't depend on each other. A GPU has thousands of small cores that can do them all at the same time, while the CPU did them on one core. A workload that would not speed up is one where each step needs the previous step's result, like my naive Python attention loop.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism stopped helping for a different reason in each experiment.
Vectorization worked very well (258.2 ms → 1.24 ms) because the work could be split into small identical pieces that the CPU's SIMD instructions handle many at a time.
Threads hit a hardware limit: Colab only has 2 vCPUs on one physical core, so 2 threads were only ~10% faster than 1 (2167.6 vs 2404.0 ms). They also hit a coordination limit: with 4 and 8 threads there were more threads than CPUs, so they spent time waiting for each other, and 8 threads (2657.8 ms) was slower than 1.
The GPU was 34× faster (13500.8 vs 459806.3 ms), not more, because some work cannot be split: Python still runs the training loop one step at a time, and data has to move between the GPU and the CPU every step.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would train on the GPU with bigger batches (32, 64, 128, 256) and measure the time per step. If the time barely goes up with bigger batches, the GPU was not fully used and 34× underestimates it. If the time grows with the batch size, the GPU was already working at full speed.
<!--/answer-->
