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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->255.524<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.077<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->436562.9<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13552.4<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.5041<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2639.395 ms, 2 threads: 2005.675 ms, 4 threads: 2070.487 ms, 8 threads: 2477.889 ms. The fastest count was 2 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive implementation spends its time executing individual Python interpreter loops and scalar instructions, causing high instruction overhead and cache misses. In contrast, the vectorized version delegates dense matrix operations to optimized BLAS routines (utilizing AVX-512/SIMD instructions). The CPU hardware executes multiple data elements simultaneously within single vector registers, drastically reducing instruction dispatch overhead and maximizing memory throughput.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve flattens and performance starts hurting beyond 2 threads, with 8 threads lagging at 2477.889 ms compared to 2005.675 ms at 2 threads. On a 2-vCPU Colab instance, allocating more threads than physical cores forces aggressive context switching and thread contention. Furthermore, the overhead of synchronizing threads and managing memory cache coherence across virtual cores quickly exceeds the marginal parallelism gains for this small model size.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A transformer training step relies heavily on massive dense matrix multiplications (like Q @ Kᵀ and projections) that map perfectly onto the thousands of arithmetic logic units (ALUs) running in parallel on a GPU, achieving a 32.2x speedup (13552.4 ms vs 436562.9 ms). Conversely, highly sequential tasks with strict data dependencies or branching control flow (such as token-by-token autoregressive text generation or simple recursive algorithms) would not see this speed-up due to idle parallel units and high kernel-launch overheads.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism hits fundamental boundaries defined by hardware constraints and algorithmic structure. First, instruction-level vectorization is bounded by hardware execution width, yielding massive gains (down to 1.077 ms) until scalar bottlenecks or memory bandwidth limits take over. Second, intra-op thread parallelism is limited by coordination overhead; pushing beyond 2 threads causes contention that degrades performance (slowing from 2005.675 ms at 2 threads to 2477.889 ms at 8 threads on a constrained vCPU layout). Third, device scaling hits hardware limits when memory transfer and kernel-launch overheads outweigh compute benefits for small workloads, though dense blocks still thrive on GPUs (13552.4 ms vs 436562.9 ms on CPU). Ultimately, parallelism stops helping when the cost of synchronization, thread contention, and data movement exceeds the work saved by concurrent execution.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would measure batch-size scaling on the GPU to see where memory bandwidth saturation occurs. If increasing batch sizes failed to yield proportional throughput gains, it would prove that memory bandwidth—rather than compute capacity—is the ultimate bottleneck limiting transformer parallelism.
<!--/answer-->
