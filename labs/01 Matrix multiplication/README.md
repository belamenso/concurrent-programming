### Labs
* 08:30
* wspolbiezne.tcs.uj.edu.pl
* CUDA server: miracle
* student server has 72 cores
* first month - CPU parallelism, then CUDA

### Tips
* Work on local variables, then update globals
* Smaller input size (s/int/short). `<cstdint>` -> `int32_t`, `uint16_t`...
* **Inverting a matrix, then using the classic multiplication with a local variable is way better than fiddling with the order of the for loops**
* cache-aware, nice memory access patterns
* write to file (better timing), time, check correctness
  ```bash
  ./generate <<< 2000 > in && time ./compute < in | md5sum
  ```
* compiling pthreads: `gcc -O3 -pthread`
* number of processors:
  * `/proc/cpuinfo`
  * `$ nproc`
  * `$ htop` (visually)
  * `thread::hardware_concurrency()`
* **(?) prawo wielkich liczb**

### Homework
**How to parallelize Floyd-Warshall?**
