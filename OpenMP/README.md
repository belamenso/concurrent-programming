# OpenMP

Notes based on https://www.youtube.com/playlist?list=PLLX-Q6B8xqZ8n8bwjGdzBJ25X2utwnoEG
* slides: https://www.openmp.org/wp-content/uploads/Intro_To_OpenMP_Mattson.pdf
* exercises: http://www.openmp.org/wp-content/uploads/Mattson_OMP_exercises.zip

## Introduction

**Structured block**: 1+ statements, 1 entry point, 1 exit point. *How it's different from basic block?* 

It's hard not to have OpenMP in your compiler already. MS support is stalling for a decade now.

1. GCC
    ```sh
    gcc -fopenmp ...
    export OMP_NUM_THREADS=4
    ./a.out
    ```
2. Visual Studio
  
    ...


**Computation model**: multithreadind + shared-memory. Program in memory consists of stack, text, data and heap. Out of these 4 things,
threads only have their own stacks, the rest is shared for the purposes of fast contxt-switching. Communication happens through
shared variables.

**Race condition**: results of execution change depensing on the specific interleaving.

Things to think about in HPC (high-preformance computing):
* **SMP** (Symmetric Multiprocessor, doesn't really exist enymore) vs **NUMA** (NonUniforM Address space).
Requirement for both: shared address space.
* The cache hierarchy

**False sharing**: (?)

**SIMP/SPMD**: (single instruction/program, multiple data). All threads execute the same code but parametrized with their
IDs and number of all threads.

## OpenMP Basics

Things to be executed in parallel (or just any code that uses omp pragmas should be in these blocks:
```c++
#pragma openmp parallel
{ // newline before the { is significant!
}
```

Variables declared inside of the prarallel region are local to the thread, outside - global to all threads
(under thre hood sections are compiled to functions and then pthreads).

Parallel hello world
...

**Round-robin allocation**:
```c++
for (int i = ID; i < n; i += NTHREADS) ...
```

`#include <omp.h>` is needed only if you want to use omp functions, and not only pragmas.

### Synchronization
* Barrier synchronization (high-level)
    * Critical `#pragma omp crirtical`
    * Atomic `#pragma omp atomic`
    * Barrier `#pragma omp barrier`
* Mutual exclusion (low-level)
    * flush
    * locks
        * simple
        * nested

## Tips
* Operate on local variables, then publish the data (better caching).
* During a parallel execution, let the threads have their separate results and combine these after joining (no sunchronization)
* **Always ask how many threads you actually got**, don't assume that your wish was fulfilled.
* Padding for avoiding unwanted cache-line sharing: `int results[n][16]` (assuming 64 bytes for cache line), access with `results[i][0]`

## OpenMP functions
| Name | Functions | Notes |
|--|--|--|
| omp_get_thread_num() | thread's id | \in **0..n-1*, where 0 - master |
| omp_get_num_threads() | number of threads currently running | outside of a parallel region it's **always 1** |
