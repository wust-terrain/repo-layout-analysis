# ADR: Skalowalność i Handoff Multimodalny (MATSim → SUMO)

**Projekt:** Cyfrowy Bliźniak Wrocławia (`wroclaw-twin` / `wust-terrain`)
**Kontekst:** Konwersja popytu z modelu MATSim (próbka 10%) do mikrosymulacji SUMO (100% obciążenia) dla ewaluacji algorytmów sterowania ruchem (TSC).

## 1. Proweniencja Decyzji i Baza Referencyjna

Decyzje architektoniczne opierają się na dowodach zgromadzonych w repozytorium badawczym projektu (zob. `_analysis/comparison.md`). Poniższa tabela odwołuje się bezpośrednio do wyizolowanych wzorców w kodzie źródłowym analizowanych submodułów, spełniając rygor identyfikowalności (`plik:linia`).

| Repozytorium (Submoduł) | Strategia i Wykorzystanie | Decyzja Architektoniczna | Dowody źródłowe w submodule |
| :--- | :--- | :--- | :--- |
| **`matsim-berlin`** | Wstrzykiwanie ułamka do parametrów silnika i znakowanie plików "pieczęcią" (np. `10pct`). | **Zasada wejścia:** Konwerter odczytuje ułamek z nazwy pliku i weryfikuje z fizyczną liczbą agentów, zapobiegając błędom skali. | `OpenBerlinScenario.java:53, 73-86`<br>*(nadpisywanie nazw artefaktów z flagi CLI)* |
| **`dlr-tsc`** | Odrzucenie narzędzi wbudowanych. Klonowanie (*stochastic rounding*), rozpraszanie w czasie (jitter) i ustrukturyzowane UID. | **Baza konwertera:** Fundament architektury. Zapożyczamy mechanikę klonowania, rozbudowując ją o tryby niesamochodowe i determinizm. | `src/tapas_sumo_coupling/t2s.py:481-488`<br>*(klonowanie)* oraz `:527-537` *(jitter)* |
| **`sumo` (tools/import)**| Natywna flaga `--scale` mnożąca przepływy. | **Wzorzec odrzucony:** Brak możliwości własnego jittera przestrzennego, brak kontroli nad ziarnem losowości i filtrowaniem przed skalowaniem. | `tools/import/matsim/matsim_importPlans.py:42-76`<br>*(brak wbudowanej w konwerter opcji skali)* |
| **`single-tsc-baselines`** | Zasilanie docelowego środowiska badawczego płaską listą pojazdów z wyznaczonymi ścieżkami (odrzucenie hierarchii planów). | **Format wyjściowy:** Narzuca kontrakt pliku `.rou.xml` ze sztywnymi trasami, wymuszając pełny pre-routing offline przed uruchomieniem docelowej ewaluacji. | `Beijing_Beihuan/routes/low_density.rou.xml:37-40`<br>*(wymóg płaskiej listy `<vehicle>` z `<route edges>`)* |

## 2. Niezmienniki Kolejności Potoku i Kryteria Walidacji

Zapewnienie wydajności i poprawnej semantyki wymaga jednokierunkowej sekwencji przetwarzania. Dane lokalne (macierze OD, dane GPS) pozostają odizolowane w `wust-terrain` i nie przenikają do kodu konwertera.

1. **Krok 0: Wyznaczenie Trybu Dominującego:** Przed jakimkolwiek filtrowaniem określany jest główny tryb podróży agenta na podstawie najdłuższego dystansu w jego pełnej, dobowej historii z MATSim. Gwarantuje to spójną klasyfikację agenta, nawet jeśli jego najdłuższa podróż odbywa się poza strefą.
2. **Pruning (Filtrowanie przed klonowaniem):** Odrzucenie agentów omijających strefę testową. Trasy oryginalne służą wyłącznie do wycięcia współrzędnych O-D na przecięciu z granicą strefy i wyliczenia wirtualnego czasu wjazdu. 
3. **Klonowanie Probabilistyczne:** Aplikacja algorytmu *stochastic rounding* na poziomie wyznaczonego w Kroku 0 trybu dominującego. Wygenerowanie bezpiecznych UID z separatorem (np. `<id_osoby>~<id_klona>`).
4. **Rozpraszanie (Jitter):** Wyliczenie deterministycznego offsetu czasowego dla każdego klona oraz przestrzennych wektorów przesunięcia (Geo-Jitter).
5. **Geo-Map-Matching i Determinizm Startu:** Zrzut punktów O-D na topologię krawędzi SUMO przy użyciu przestrzennego indeksu `rtree`. Następnie deterministyczne wyliczenie wartości `departPos` i `departLane` (za pomocą unikalnej soli hasha), zastępujące niedeterministyczne flagi dynamiczne SUMO.
6. **Zamrożony Routing 1:1:** Wyznaczenie ścieżek przez iteracyjne rutery operujące na 100% obciążenia i eksport do `.rou.xml`.
7. **Rygorystyczna Walidacja Zautomatyzowana:** Scenariusz trafia do zautomatyzowanej bramki testowej i zostaje bezwzględnie **odrzucony**, jeśli wystąpi:
   * Stały deadlock na siatce dróg (brak ruchu na jakiejkolwiek krawędzi przez $> 300$ s).
   * Teleportacja lub insertion backlog przekraczający $1\%$ całkowitej wygenerowanej populacji.
   * Odchylenie profili popytu w oknach 1-godzinnych $> 5\%$ względem populacji bazowej. Wejściowa próbka referencyjna (10%) jest przepuszczana przez ten sam jitter przestrzenno-czasowy przed porównaniem, aby zapobiec fałszywym błędom ewaluacji na zboczach szczytów komunikacyjnych.

## 3. Strategia Multimodalna (Mode Router)

Skalowanie ruchu mieszanego wymusza rozgałęzienie logiki konwertera. 

| Tryb Podróży | Strategia Skalowania i Rozpraszania | Pre-Routing i Docelowy Eksport |
| :--- | :--- | :--- |
| **Samochody / Ciężarówki** | **Jitter Czasowy** ($\pm 450$ s). Akceptowany brak wariancji przestrzennej na poziomie punktu startowego. | `duaIterate` (poszukiwanie równowagi sieciowej, uwzględnienie zatorów). |
| **Rowery** | **Rozszerzony Jitter Czasowy** ($\pm 600$ s). Zwiększony bufor redukuje formowanie zbitych kolumn na wrocławskich śluzach. | `duaIterate` (równowaga z uwzględnieniem docelowej infrastruktury rowerowej). |
| **Piesi** | **Geo-Jitter + Jitter Czasowy**. Wymóg rozrzucenia punktów X/Y zapobiegający blokadom fizycznym. | `duarouter`. **Założenie dla TSC:** Jeśli parsery zewnętrznego środowiska badawczego nie obsługują tagu `<person>`, ruch pieszy funkcjonuje wyłącznie jako szum tła (wpływa na fizykę symulacji, ale jest ukryty przed samym algorytmem sterującym). |
| **Pasażerowie PT** | **Jitter Czasowy** ($\pm 450$ s). Klony wpływają bezpośrednio na czas wymiany na przystankach (*dwell time*). | `duarouter` (wymaga zintegrowanego modelu intermodalnego). |
| **Pojazdy PT** | **Bypass Skalowania (0%).** Fizyczne pojazdy PT (autobusy/tramwaje) omijają konwerter. | Ładowane do SUMO ze statycznych rozkładów GTFS Wrocławia w skali 1:1. |

## 4. Finalna Rekomendacja i Kompromisy Architektoniczne

Skalowanie realizujemy poprzez *stochastic rounding* wsparty zróżnicowanym jitterem czasowym i przestrzennym. Pełny determinizm losowań zamykamy w jednej stabilnej funkcji skrótu (np. blake2b), której wejściem są: ziarno, ID agenta, (opcjonalnie ID klona dla jittera) oraz unikalna sól osobna dla każdego rodzaju losowania. Gotowe trasy podlegają zamrożeniu po pre-routingu w SUMO.

**Zalety przyjętej architektury (Skalowalność):**
* **Niezależność implementacyjna:** Konwerter jest implementowany od zera na podstawie sprawdzonych koncepcji jako proces komunikujący się wyłącznie przez pliki. Izolacja procesowa pozwala bezpiecznie przetwarzać kod własny bez ryzyka naruszenia licencji (GPL/AGPL), z których korzystają narzędzia referencyjne.
* **Zrównoleglenie (Sharding):** Bezstanowość pozwala na dzielenie zbiorów, przetwarzanie na wielu rdzeniach i bezpieczne łączenie danych bez ryzyka kolizji.
* **Elastyczność Zasięgu:** Popyt obliczony dla jednej strefy jest spójny pod kątem punktów O-D i czasów z popytem dla innej strefy. Trasy wyliczone ostatecznie przez `duaIterate` będą się jednak różnić w zależności od geometrii badanej sieci.
* **Realizm Fizyki Mikroskopowej:** Geo-Jitter zapobiega fizycznym blokadom (nakładającym się pieszym), co jest niezbędne do utrzymania odsetka teleportowanych pojazdów poniżej 1%.

**Akceptowany Dług Techniczny (Wady):**
* **Systemowa stronniczość ewaluacji:** Trasy pojazdów z `duaIterate` optymalizują się pod bazowe, stałoczasowe programy sygnalizacji, co generuje stronniczy punkt odniesienia przy badaniu adaptacyjnych systemów TSC.
* **Kruchość determinizmu na poziomie ID:** Mechanika odtwarzalności sprzężona jest z identyfikatorami znakowymi. Jakakolwiek zmiana struktury nazywania agentów w modelu wyjściowym spowoduje kompletną zmianę wyników rozpraszania.
* **Ślepy routing pieszy:** `duarouter` wyznacza fizycznie najkrótsze ścieżki piesze, ignorując dynamiczne zatłoczenie chodników przy pełnym obciążeniu.
