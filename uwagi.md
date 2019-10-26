## Uwagi

* Polak chciał ładne tabelki z czasami i przyspieszeniami w raporcie.
* Mierzysz czasy tylko raz dla każdego kodu, chyba rozsądniej jest policzyć średnią razem z odchyleniem
(napisałem do tego skrypt w Pythonie, który zakłada że po każdej iteracji pętli `while (z--)` na `stderr` wypisujesz
czas jaki upłynął w sekundach)
* Co do `fast_4.cpp`: wybór `(static, 12)` wydaje mi się dziwny.
[Tutaj](https://software.intel.com/en-us/articles/openmp-loop-scheduling) pisze, że domyślnie to wielkość pracy / liczba wątków
i wydaje mi się że ten parametr oznacza liczbę iteracji pracy którą wątek bierze z kupy rzeczy do roboty i wykonuje na raz. Chyba
większa liczba ma sens? `(static, 12)` daje mi około 4.2 sekundy, `(static)`, `(static 1000)` dają mi od 3.0 do 3.4 sekund.
Większe jak i mniejsze liczby wydają się spowalniać.
* Jeśli chodzi o laptopa, 4 wątki w `fast_4.cpp` oferowały najmniejsze czasy, to w sumie zgadza się z wynikami innych.
* Na zajęciach padła propozycja przyspieszenia tego przez bardziej równomierny rozkład większych kawałków pracy między wątki,
na przykład rozrysowanie na kartce jak przebiegają zależności w tablicy dynamicznej i liczenie tego w jakiejś nietrywialnej
kolejności, gdzie iteracje są równej wielkości. W problemie plecakowym nie widzę zbytnio takiej możliwości, ponieważ
  * i tak operujesz tylko na ostatnich dwóch lub jednym wierszu,
  * iteracje wewnętrznej pętli są równej wielkości,
  * `D[j - w[i]]` to skok który uniemożliwia np. pokrywanie prostokątnej tablicy kwadratami równej wielkości, jak
Polak proponował w oliwkach.
* Nie sprawdzałem niczego na studencie ponieważ wszystko liczy mi się tam po kilka minut, nie wiem w sumie dlaczego
wszystko co tam odpalę ma tak niski priorytet.

**Ciekawostka**: nasze rozwiązania zadania B z asdów są prawie identyczne, ale moje o 2 sekundy wolniejsze.
Zastanawiałem się dlaczego i okazało się, że to jest ta różnica:
```c++
for (int i = 0; i < n; i++) {
    for (int j = B; j >= 0; j--) {
        if (w[i] <= j) { // vs (0 <= j - w[i])
            D[j] = max(D[j], c[i] + D[j - w[i]]);
        }
    }
}
```
