### Logowanie na miracle
1. ssh student
2. ssh login@miracle.tcs.uj.edu.pl

### Zajęcia
Tam są  `/usr/local/cuda/samples/{common,1_Utilities}`, skopiuj to.
Teraz wejdź w `1_Utilities/deviceQuery`, zrób `make`, uruchom -> info o systemie.

### Karta graficzna
Miracle ma 4, po 16 multiprocessors każda, każdy ma 128 rdzeni. SIMD. Wspólny ram 4GB.

To się liczy **blokami** (?). Cache, shared memory. Synchronizacja w bloku jest dość tania, między
blokami - nie.

### Programowanie
Piszesz w C z czymś, `*.cu`. `nvcc` compiler. `nvcc a.cu --std=c++11`. `-arch sm_50`.

Uwaga na `CUDA capability`, w deviceQuery - jaka wersja wspierana. Możesz kompilować dla niższych,
wtedy możesz korzystać z mniejszej liczby rzeczy, ale stare karty to uruchomią. `-arch sm_50` <=> `5.0`.

Funkcje, które mogą się wykonywać na karcie nazwyają się kernelami. Funckje w C, ale poprzedzone
`__global__ ret function_signature()`.
