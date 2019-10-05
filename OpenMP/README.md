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

**leapfrog RNG**: (?), and also correct use of rngs in concurrent applications

**Loop-carry dependency**: iterations of a loop depend on previous iteration instead of depending only on the loop index.

**SIMP/SPMD**: (single instruction/program, multiple data). All threads execute the same code but parametrized with their
IDs and number of all threads.

## OpenMP Basics

Goals:
    1. Make parallelizing existing code easy
    1. Programs should have serial interpretation (if you ignore pragmas it works fine)
    1. It's about parallelism, not concurrency, concurrent constructs without serial interpretation are poorely supported (eg. through flushes)

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
        
        Optionally you can specity `atomic write`, `atomic read`, `atomic update`, `atomic capture` (? capture?) like
            ```c++
            #pragma omp atomic read
            my_copy = some_var;
            ```
    * Barrier `#pragma omp barrier` (no block of code needed)
* Mutual exclusion (low-level)
    * flush `#pragma omp flush` or `#pragma omp flush(x)` (no block of code needed)
        
        Makes your (thread's) view available to all the other threads (flushed RAM, registers, cache lines). **This is not a mechanism of global synchronization, only your view of your variables is make global, it doesn't force others' views to become global**.
        
        **Do not touch this. This is low-level (OpenMP is for parallel not concurrent)**
        
        **(?) flush sets** - some formalism associated with it
        
        Implicit flush (total):
            * enter/exit from a parallel region
            * enter/exit from a critical section
            * lock set/unset
            * on any barrier (when exactly?)
    * locks: `omp_init_lock()`, `omp_destroy_lock()`, `omp_set_lock()`,  `omp_unset_lock()`
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
    Loop's index will have private access by default (even in C-like style where you define `int i` before the loop). **Watch out for C-style inner loops defining their indices before parallel for - they're shared by default, very error prone** (inner `for (int j=...` is OK).
    
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
* task `task`, `taskwait`: an independent unit of work, high-level, query of unfinished tasks, dynamic scheduling
    Consists of
        * code to execute
        * data env (?)
        * internal control variables: **OpenMP ties internal state (things you access with omp_get...) to tasks, not threads!** (?), ICV - internal control variable
        
    **WARNING: tricky access patterns**
    
    ```c++
    node* p = head;
    while (p) {
        #pragma openmp task
        expensive_computation(p);
        p = p->next;
    }
    ```
    
     ```c++
    long fib(int n) {
        if (n < 2) return n;
        long x, y;
        #pragma omp task shared(x) // (!) without this, x would be private to the task
        x = fib(n-1);
        #pragma omp task shared(y) 
        y = fib(n-2);
        
        #pragma omp taskwait // important here, but not in thre previous example
        return x + y;
    }
    ```
    
    ```c++
    #pragma omp parallel single
    {
        for (int i = 0; i < n; i++) {
            #pragma omp task firstprivate(i) // (!) without it, i would be shared
            process(i);
        }
    }
    ```

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
* if you really want some fixed number of threads
    ```c++
    int nthreads;
    omp_set_dynamic(0);
    omp_set_num_threads(omp_num_procs()); // or any other fixed number
    #pragma omp parallel
    {
        #pragma omp single
        nthreads = omp_get_num_threads();
        
        if (not_ok(nthreads)) exit();
        else compute(omp_get_thread_num());
    }
    ```
* **Learn some parallel debugger!**
* "Computating is free and you pay for data movement"

## Global/local data

The most important rule:
    * stack is for private data
    * heap stores global data

* `shared(x,y)`: there variables are global
* `private(x,y)`: there variables are local to a thread, have no initial value, disappear with their thread
* `firstprivate(x,y)`: private, but copy their initial value from the globlal var of the same name
* `lastprivate(x,y)`: private but store their last value ((?), like from where i=n-1 in a parallel loop) in the global var of the same name
* `default( firstprivate | shared | none )`: which access applies to the variables I didn't mention with the above methods?
    * I didn't specify a default: `default(shared)`
    * `none`: **extremely useful, it gives you warning for every variable, great for debugging** (doesn't work in VisualStudio)
    * `default(private)` cannot be done in C (Fortran's version of OpenMP supports it)
* If you really need to do concurrent programming, remember about the flushes:
    ```c++
    #pragma omp parallel sections
    {
        #pragma omp section
        {
            fill_array(A);
            #pragma omp flush
            #pragma omp atomic write
                flag = 1;
            #pragma omp flush(flag)
        }
        #pragma omp section
        {
            while (1) {
                #pragma omp flush(flag)
                #pragma omp atomic read
                    _flag = flag;
                if (_flag) break;
            }
            #pragma omp flush
            process_array(A);
        }
    }
    ```

**threadprivate**: (?!)

## OpenMP functions
| Name | Functions | Notes |
|--|--|--|
| omp_get_thread_num() | thread's id | \in **0..n-1**, where 0 - master |
| omp_get_num_threads() | number of threads currently running | outside of a parallel region it's **always 1** |
| omp_set_num_threads() | Int | this is only a wish |
| omp_get_max_threads() | max possible threads | |
| omp_in_parallel() | am I in a parallel section right now? | |
| omp_get_dynamic() | dynamic mode? (yes => two parallel regions might get different number of threads) | |
| omp_set_dynamic() | | 0, 1 |
| omp_num_procs() | how many processors? | |


## OpenMP env variables
| Name | Domain | Functions |
|--|--|--|
| OMP_NUM_THREADS | Int| |
| OMP_STACKSIZE | | |
|OMP_WAIT_POLICY | ACTIVE\|PASSIVE | busy waiting or yielding? |
|OMP_PROC_BIND | TRUE\|FALSE | FALSE => threads can migrate away from the processor they started in |
