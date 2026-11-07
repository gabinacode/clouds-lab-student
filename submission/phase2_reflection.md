# Phase 2 — Reflection: where does parallelism stop helping?

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->480.07<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.09<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->449765.0<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->15892.0<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.436<!--/answer--> |

**The thread sweep.** Timings for 1, 2, 4 and 8 threads, in order, and the fastest count:

<!--answer:thread_sweep-->
1 thread: 2482.223 ms/step; 2 threads: 2215.067 ms/step; 4 threads: 2402.338 ms/step; 8 threads: 2525.244 ms/step. The fastest setting was 2 threads (2215.067 ms/step).
<!--/answer-->

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.**

<!--answer:vectorization_why-->
The naive implementation executes many small dot products inside Python loops, paying interpreter and function-call overhead repeatedly. The vectorized implementation expresses the same attention computation as large matrix operations dispatched to optimized native BLAS routines. This reduces Python overhead and takes advantage of SIMD instructions, contiguous memory access, and better cache use. The observed times were 480.07 ms versus 1.09 ms (reported speedup: 438.8x). The gain is not attributable to SIMD alone: moving work out of Python is also important.
<!--/answer-->

**Where the thread curve flattens, and why.**

<!--answer:threading_plateau-->
The best time occurred at 2 threads (2215.067 ms/step), compared with 2482.223 ms at 1 thread. Adding threads beyond 2 hurt performance: 2402.338 ms at 4 threads and 2525.244 ms at 8. PyTorch's intra-op workers must share the Colab CPU allocation, caches, memory bandwidth, and synchronization resources. When there are more runnable workers than the effective hardware can use profitably, scheduling and contention outweigh parallel work. The saved notebook does not record the runtime's exact vCPU allocation; this should be checked rather than inferred from the timings alone.
<!--/answer-->

**What made the GPU faster.**

<!--answer:gpu_why-->
Transformer training performs many dense matrix multiplications and attention operations with substantial parallel arithmetic. The GPU executes these operations concurrently on many processing units, whereas the CPU has far fewer such units. For the same 100 training steps, the CPU took 449765.0 ms and CUDA took 15892.0 ms, a speedup of approximately 28.3x. A tiny or largely sequential task, especially one with frequent CPU–GPU transfers, might not benefit: launch and transfer overhead can exceed computation time.
<!--/answer-->

## 3. The question

**Where does parallelism stop helping, and why?**

<!--answer:parallelism_limit-->
Parallelism stops helping when the additional computation it enables costs less time than the overhead needed to coordinate and feed it. In the attention experiment, replacing Python loops with native vectorized matrix operations reduced time from 480.07 to 1.09 ms: the original bottleneck was not just arithmetic, but interpreter overhead and inefficient execution of small operations. On the CPU, the best of the tested settings was 2 threads (2215.067 ms/step). More threads worsened the result (2525.244 ms at 8), consistent with contention for limited cores, bandwidth and synchronization overhead. On the GPU, 100 training steps fell from 449765.0 ms on CPU to 15892.0 ms on CUDA, because the model provides enough matrix arithmetic to exploit many parallel execution units. Even there, serial work, kernel launches and host-to-device transfers impose a limit, especially for small workloads. These observations illustrate Amdahl's law: once the parallel part accelerates, the remaining serial and coordination costs become proportionally more important.
<!--/answer-->

**What you would measure next.**

<!--answer:next_experiment-->
I would repeat the CPU/GPU benchmark across several batch sizes, including CPU-to-GPU transfer time, with multiple trials per size. This would reveal when GPU startup and transfer overhead outweigh the gain from parallel matrix arithmetic. If even very small batches retained the same 28.3x speedup, I would revise my expectation about where that crossover occurs.
<!--/answer-->
