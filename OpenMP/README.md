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
        ![](../../../blob/master/OpenMP/imgs/vs2.PNG?raw=true)
        ![](../../../blob/master/OpenMP/imgs/vs3.PNG?raw=true)


**Computation model**: multithreadind + shared-memory. Program in memory consists of stack, text, data and heap. Out of these 4 things,
threads only have their own stacks, the rest is shared for the purposes of fast contxt-switching. Communication happens through
shared variables.

![](../../../blob/master/OpenMP/imgs/openmp-model.PNG?raw=true)
![](../../../blob/master/OpenMP/imgs/fork-join-model.png?raw=true)

**Race condition**: results of execution change depensing on the specific interleaving.

Things to think about in HPC (high-preformance computing):
* **SMP** (Symmetric Multiprocessor, doesn't really exist enymore) vs **NUMA** (NonUniforM Address space).
Requirement for both: shared address space.
* The cache hierarchy

**False sharing**: (?)

**Loop-carry dependency**: iterations of a loop depend on previous iteration instead of depending only on the loop index.

**SIMP/SPMD**: (single instruction/program, multiple data). All threads execute the same code but parametrized with their
IDs and number of all threads.

## OpenMP Basics

Things to be executed in parallel (or just any code that uses omp pragmas should be in these blocks:
```c++
#pragma openmp parallel
{ // newline before the { is significant!
}
```

Most directives follow the pattern `#pragma omp ...` where what follows in the next line is either a single
statement or a block statement. Directives can be joined together for brevity: `#pragma omp parallel for schedule(static)`

Variables declared inside of the prarallel region are local to the thread, outside - global to all threads
(under thre hood sections are compiled to functions and then pthreads).

Parallel hello world

![](../../../blob/master/OpenMP/imgs/pi.png?raw=true)

**Round-robin allocation**:
```c++
for (int i = ID; i < n; i += NTHREADS) ...
```

`#include <omp.h>` is needed only if you want to use omp functions, and not only pragmas.

## Synchronization
* Barrier synchronization (high-level)
    * Critical `#pragma omp crirtical`
        * do very little in critical section
    * Atomic `#pragma omp atomic`
        * If hardware can do these operation atomically, it will, else compile to a critical section.
        * `x *= expr` (where `*` is any binop), `++x`, `--x`, `x++`, `x--` are supported
    * Barrier `#pragma omp barrier`
* Mutual exclusion (low-level)
    * flush
    * locks
        * simple
        * nested

## Work sharing (high-level)
* loops

    ```c++
    #pragma omp for
    for (int i = 0; i < n; i++) {
        // code without loop-carry dependencies
    }
    ```
    
    * Scheduling
    
        | Method | Comment |
        |--|--|
        | no annotation | compiler decides |
        | `schedule(static[, chunkSize])` | round-robin, with each pass doing chunkSize (1 bu default) iterations, best where each iteration will take approximately the same time |
        | `schedule(dynamic[, chunkSize])` | dynamic queue, each thread getting their task from it, better for cases where completion times of tasks cannot be determined upfront or vary greatly between instances |
        | `schedule(runtmie)` | specify at runtime through an environment variable |
        | `schedule(auto)` | gives the compiler permission to do what it wants |
        
    * Reductions
        ```c++
        int avg = 0;
        #pragma omp parallel for reduction(+:avg)
        for (int i = 0; i < n; i++)
            avg += a[i];
        avg /= n;
        ```
        `reduction(op: list-of-vars)` for op being associative/commutative and having well-defined neutral element creates a local (to the thread) copy of the vars, reduces them with op and then combines with the global value.
        
        | + | * | - | min | max | & | \| | ^ | && | \|\| |
        | --| -- | -- | -- | --- | -- | -- | -- | -- | -- |
        | 0 | 1 | 0 | +inf | -inf | ~0 | 0 | 0 | 1 | 0 |
                
        
* sections/section
    ```c++
    #pragma omp sections
    { // 3 threads will (hopefully) concurrenly execute these 3 parts
        #pragma omp section
        compute_x_direction();
        
        #pragma omp section
        compute_y_direction();
        
        #pragma omp section
        compute_z_direction();
    }
    ```
* single  `single` - any thread (but only one) will execute this section
* master `master` - like single, but done by the thread with `id` = 0
* task

## Implicit barriers
* after each worksharing loop (can be disabled with `nowait` directive, as in `#pragma omp for nowait`)
* after each parallel region (cannot be disabled)
* after each `single` block
* **no impled symchronization after `master` block**

## Tips
* Operate on local variables, then publish the data (better caching).
* During a parallel execution, let the threads have their separate results and combine these after joining (no sunchronization)
* **Always ask how many threads you actually got**, don't assume that your wish was fulfilled.
* Padding for avoiding unwanted cache-line sharing: `int results[n][16]` (assuming 64 bytes for cache line), access with `results[i][0]`

## OpenMP functions
| Name | Functions | Notes |
|--|--|--|
| omp_get_thread_num() | thread's id | \in **0..n-1**, where 0 - master |
| omp_get_num_threads() | number of threads currently running | outside of a parallel region it's **always 1** |
