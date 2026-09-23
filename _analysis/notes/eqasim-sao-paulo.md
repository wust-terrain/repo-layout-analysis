# eqasim-sao-paulo

- **Repo:** https://github.com/eqasim-org/eqasim-sao-paulo
- **Commit:** `26a6a46e397853b08fc0e848c115174f4baf0f74` (2021-11-01, wg `MANIFEST.tsv`)
- **Klon:** pełny, ale historia to **jeden commit** (cały kod wrzucony naraz, 75 plików, 17 863 linii; `git log` ma tylko „Update README.md”). Nie da się więc odtworzyć przebiegu adaptacji z historii git; wnioski o adaptacji wynikają z kodu i porównania z `eqasim-france`.

**Czym jest to repo.** Pipeline `synpp` (Python) generujący syntetyczną populację Regionu Metropolitalnego São Paulo z otwartych danych (spis Censo 2010, ankieta mobilności OD 2017 metra SP, OSM, szkoły, GTFS SPTrans/EMTU) i opcjonalnie budujący oraz uruchamiający scenariusz MATSim/eqasim. Jest to **starsza (ok. 2020–2021) kopia pipeline'u Île-de-France przerobiona pod dane brazylijskie** (Sallard, Balac, Hörl 2021, `README.md:29-30`). Traktujemy jako zapis historyczny: jak wyglądała adaptacja „rdzenia” eqasim do nie-francuskiej ankiety, zanim w `eqasim-france` pojawiły się przełącznik HTS i jawny kontrakt kolumn.

---

## Co wymagało adaptacji do lokalnych danych

Dla czytelności najpierw mapa stage'y. Legenda: **P** = przepisany od zera pod dane lokalne, **M** = rdzeń skopiowany z Francji i zmodyfikowany, **B** = skopiowany prawie bez zmian.

| Stage | Status | Co się zmieniło |
|---|---|---|
| `data.census.raw` / `.cleaned` | **P** | Wczytywanie SPSS `.sav` przez `pyreadstat` w kawałkach, filtr stanu SP (`V0001=='35'`), zmiana nazw kolumn `V*` na angielskie, imputacja braków, przeskalowanie wag po `dropna` (`data/census/raw.py:13-62`), odtwarzanie `household_id` z kolejności wierszy i `numberOfMembers` (`data/census/cleaned.py:67-80`). |
| `data.hts.cleaned` | **P + M** | Linie 1–323 to przepisany parser OD 2017 (`.dbf` przez `simpledbf`), mapy kodów trybów i celów, czyszczenie łańcuchów. Od linii 324 jest komentarz „From here everything as Paris” (`data/hts/cleaned.py:324`): dalej idzie kod z Francji (flagi `has_work_trip`, `commute_distance_*`, `is_passenger`). |
| `data.od.cleaned` | **P** | Francja brała macierze dojazdów do pracy/szkoły z INSEE, a SP buduje je **z samej ankiety** (ważona liczba osób home_zone→dest_zone) (`data/od/cleaned.py:20-52`). |
| `data.spatial.zones` | **P** | Strefy to áreas de ponderação spisu (`AP_2010_CH`) z shapefile CEM, CRS `EPSG:29183` (`data/spatial/zones.py:12-18`). |
| `data.opportunities.extract_roads_osm` | **P** | Francja brała BPE/SIRENE, a tu możliwości (work/shop/leisure/home) pochodzą z tagów OSM przez `osmium`. „Home” to centroidy dróg `residential`/`living_street` (`data/opportunities/extract_roads_osm.py:56-155`). Szkoły idą z osobnego CSV (`synthesis/destinations.py:32-43`). |
| `synthesis.population.sampled` | **B/M** | Ekspansja gospodarstw przez zaokrągloną wagę plus downsampling (`synthesis/population/sampled.py:11-44`). |
| `synthesis.population.matched` + `algo/hot_deck_matching.py` | **M** | Algorytm hot-deck z Francji. Zmieniły się tylko zmienne dopasowania: obowiązkowe `age_class, sex, binary_car_availability, employment`, preferowana lokalna `residence_area_index` (`synthesis/population/matched.py:25-41`). |
| `sociodemographics`, `trips`, `activities` | **B/M** | Generyczne. Lokalne reguły to np. `has_pt_subscription` dla <18 i ≥60 lat, `has_license` = dorosły z autem (`synthesis/population/sociodemographics.py:39-43`). |
| `spatial.by_person.primary_zones` | **B** | Losowanie strefy pracy/szkoły z macierzy OD metodą multinomialną (`primary_zones.py:27-55`). |
| `spatial.by_person.primary_locations` | **P** | Praca w tej samej strefie i edukacja przypisywane z rozkładów odległości z HTS, osobno per (wiek × płeć × `residence_area_index`) i osobno dla `car_passenger` (`primary_locations.py:126-205, 207-312, 400-427`). Strefa edukacji z OD jest de facto ignorowana. |
| `spatial.by_person.secondary.*` (`rda.py`, `problems.py`, `components.py`) | **B** | Ten sam algorytm co we Francji. `rda.py` różni się od dzisiejszej wersji w `eqasim-france` o ~61 linii diff na 324, `problems.py` o 22 na 91. Lokalny „tuning” to twarde przesunięcia rozkładów `pt=+0.2, walk=-0.2` (`secondary/locations.py:87-88`). |
| `matsim.scenario.*`, `matsim/writers.py` | **M** | Writer XML kopiowany. Atrybuty osób rozszerzone o `residenceAreaId`, `htsPersonId`, `censusPersonId` (`matsim/scenario/population.py:36-54`). |
| `matsim.scenario.supply.gtfs` | **P** | Scalanie dwóch GTFS (`gtfsmerger`) i ręczna naprawa formatu dat w `calendar*.txt` (`matsim/scenario/supply/gtfs.py:13-29`). |
| `matsim.simulation.prepare` | **M** | Wywołania generycznych klas `org.eqasim.core.*` przeplatane klasami SP z eqasim-java (`RunImputeInnerSPAttribute`, `RunAdaptConfig`) (`matsim/simulation/prepare.py:46-100`). |

**Styk lokalne dane ↔ rdzeń (interfejs).**
- **Zaobserwowane:** interfejs jest **niejawny**. Tworzą go nazwy kolumn i kształt krotek zwracanych przez dwa stage'e: `data.hts.cleaned` zwraca `(df_persons, df_trips)`, dwie ramki bez tabeli gospodarstw (`data/hts/cleaned.py:383`), a `data.census.cleaned` zwraca jedną ramkę osób z kolumnami `person_id, household_id, weight, zone_id, residence_area_index, age, sex, employment, binary_car_availability, household_size, household_income` (`data/census/cleaned.py:84-90`). Downstream stage'e indeksują `[0]`/`[1]` i wybierają kolumny po nazwie. Nie ma asercji schematu, listy wymaganych kolumn ani walidatora. Kategorie są stringami (`"male"`, `"yes"/"no"/"student"`, tryby `car/car_passenger/pt/walk/taxi`, cele `home/work/education/shop/leisure/other`), więc to te słowniki stanowią de facto kontrakt z rdzeniem i z eqasim-java. W `eqasim-france` ten sam styk jest dziś jawny: `data/hts/hts.py:223-262` definiuje `HOUSEHOLD_COLUMNS/PERSON_COLUMNS/TRIP_COLUMNS` plus `check()`, a `data/hts/selected.py` wybiera implementację ankiety przez `context.config("hts")` z `alias="hts"`.
- **Dlaczego:** *(wnioskowane)* adaptacja powstała przez skopiowanie i edycję, a nie przez wydzielenie rdzenia. Kontrakt istniał więc tylko w głowach autorów. Brak tabeli gospodarstw w HTS wymusił dopasowanie na poziomie osoby, nie gospodarstwa.
- **Czy dotyczy nas?** **tak.** KBR będzie naszym `data.hts.*`. Od początku potrzebujemy jawnego schematu (lista kolumn, dtypes, dozwolone kategorie, walidacja), analogicznego do dzisiejszego `hts.py:check()`, a nie do SP.

**Ankieta mobilności (odpowiednik KBR): wagi.**
- **Zaobserwowane:** wczytywane są waga osoby `FE_PESS` i waga podróży `FE_VIA` (`data/hts/cleaned.py:20`). Zachowana zostaje tylko waga osoby (`weight`, l. 104). `weight_trip` nie trafia do `KeepColummnsTrips` (l. 73-77), więc wagi podróży przepadają. Osoby usuwane za niespójne łańcuchy (nie zaczynają/kończą w domu, luki w łańcuchu, jedna podróż; l. 204-284) wypadają **bez przeważenia** pozostałych. W spisie przeważenie po `dropna` jest robione (`data/census/raw.py:47-51`). Hot-deck przyjmuje argument `source_weight` (`hot_deck_matching.py:28`), ale losowanie dawcy jest jednostajne (l. 76), więc wagi ankiety **nie wpływają na dopasowanie**. Wagi osoby są używane tylko w OD (`data/od/cleaned.py:40`) i w rozkładach odległości (`secondary/distance_distributions.py:32-75`).
- **Dlaczego:** *(wnioskowane)* uproszczenie przejęte z ówczesnej wersji francuskiej. Brak testów nie pokazał, że argument jest martwy.
- **Czy dotyczy nas?** **tak.** KBR ma wagi, a my porównujemy 2018 i 2024 z różnym podziałem modalnym. Każde odrzucenie rekordów musi mieć jawną politykę przeważenia, a każde miejsce użycia wag trzeba przetestować.

**Ankieta: tryby i cele.**
- **Zaobserwowane:** 17 kodów `MODOPRIN` jest mapowanych na `pt/car/car_passenger/taxi/ride_hailing/motorcycle/bike/walk/other` (`data/hts/cleaned.py:92-93`), a potem zwijanych: motocykl→`car`, ride-hailing→`taxi`, **rower→`walk`**, other→`walk` (l. 173-179). Cele: 11 kodów→`work/education/shopping/errand/leisure/home` (l. 80-90), dalej `shopping→shop`, `errand→other`, braki→`other` (l. 161-168). Do tego reguły lokalne: edukacja dla części dorosłych→`other` (l. 62, 157-159), praca poniżej 16 lat→`other` (l. 316-317), usunięte podróże home→home, work→work i education→education (l. 184-197). Zbiór trybów musi zgadzać się z tym, co zna Java (`osm.py:37-42` ustawia `car,car_passenger,taxi` jako tryby sieciowe).
- **Dlaczego:** *(wnioskowane)* model wyboru trybu w eqasim-java/SP obsługiwał tylko ten zestaw. Rower zniknął zapewne dlatego, że nie było dla niego parametrów modelu.
- **Czy dotyczy nas?** **tak.** Dla sterowania sygnalizacją rower i motocykl mogą mieć znaczenie. Mapowanie kodów KBR na zestaw trybów/celów powinno być tabelą danych (np. CSV/YAML w `docs/dane/`), wersjonowaną i opisaną, a nie literałami w kodzie. Decyzję „co zwijamy” trzeba zapisać w `docs/decyzje/`.

**Dopasowanie do stref i OD.**
- **Zaobserwowane:** ankieta ma współrzędne O/D i domu (`CO_O_X/Y`, `CO_D_X/Y`, `CO_DOM_X/Y`). Kod **nie używa własnych stref ankiety (`ZONA`)**, tylko robi `sjoin` punktów do stref spisowych AP (`data/hts/cleaned.py:120-126`; `data/od/cleaned.py:21-33`). Macierze work/education są liczone z HTS jako ważone przepływy home_zone→dest_zone, normalizowane per origin (`data/od/cleaned.py:40-75`). Strefy bez obserwacji dostają fallback wewnątrzstrefowy z komentarzem TODO (`data/od/cleaned.py:81-101`). Wywołanie `drop_duplicates(subset=["person_id"])` nie jest przypisane (l. 38, 47), więc osoba z kilkoma podróżami do pracy liczy się wielokrotnie *(obserwacja z kodu; wpływu nie mierzono)*. Lokalna zmienna przestrzenna `residence_area_index` (1 = region, 2 = miasto SP, 3 = centrum) pochodzi z dwóch **ręcznie przygotowanych** shapefile w `resources/` (`data/census/cleaned.py:31-46`; `docs/howto.md:236-239`) i jest liczona identycznie w spisie i HTS, co czyni ją wspólnym kluczem.
- **Dlaczego:** Brazylia nie ma odpowiednika francuskich macierzy dojazdów INSEE, więc jedynym źródłem OD była ankieta *(wnioskowane z README/howto: brak takiego źródła w liście danych)*.
- **Czy dotyczy nas?** **tak, bezpośrednio.** Mamy KBR (z O/D) **oraz** OD z sieci komórkowej. SP pokazuje wariant „OD z ankiety na strefach spisu”, a nasz stage `data.od` powinien mieć przełączalne źródło (KBR vs komórkowe vs fuzja) z tym samym wyjściem `(origin_id, destination_id, weight)`. Zmienna stratyfikująca wspólna dla spisu i ankiety (u nas np. rejon/osiedle) jest kluczowa dla hot-deck.

**Fork czy zależność?**
- **Zaobserwowane:** to **kopia (fork przez copy-paste), nie zależność.** Nie ma importu żadnego pakietu eqasim w Pythonie. Jedyna zależność „rdzeniowa” to biblioteka orkiestracji `synpp` (`environment.yml:19`, `requirements.txt:12`). Ślady pochodzenia z Francji: komentarz `data/hts/cleaned.py:324`, CRS francuski `EPSG:2154` w kontroli wiarygodności rozkładu PT (`matsim/scenario/supply/processed.py:77`) mimo że reszta używa `EPSG:29183`, nazwa `df_commune_zones` (`primary_locations.py:318`), prawie identyczne `rda.py`/`problems.py`/`writers.py`.
- **Dlaczego:** *(wnioskowane)* w 2020 nie było jeszcze w `eqasim-france` mechanizmu wymiennych ankiet. Najszybsza droga to było skopiować repo.
- **Czy dotyczy nas?** **częściowo.** Pokazuje koszt kopii: poprawki rdzenia z Francji nie docierają, a pozostałości (EPSG:2154) psują wynik po cichu. U nas rdzeń (np. hot-deck, lokalizacja drugorzędna) powinien być osobnym członkiem workspace'u, a dane lokalne adapterem.

---

### 1. Orkiestracja

**Stage'e `synpp` z listą celów w `config.yml`**
- **Zaobserwowane:** każdy moduł ma `configure/execute` (czasem `validate`). Zależności deklarowane są przez `context.stage(...)`, a cele wybierane listą `run:` w `config.yml:8-24` (większość zakomentowana, aktywne `matsim.runtime.eqasim` i `matsim.output`). Częściowe ponowne uruchomienie działa przez cache `synpp` w `working_directory`. Instrukcja mówi, że po odkomentowaniu `matsim.output` synpp „wie, które stage'e już działały” (`docs/howto.md:399-402`). `validate()` zwraca rozmiar pliku wejściowego jako sygnaturę unieważnienia (`data/census/raw.py:65-69`, `matsim/scenario/supply/gtfs.py:47-51`, `osm.py:230-234`). Zewnętrzne narzędzia (git/maven/java/pt2matsim) też są stage'ami (`matsim/runtime/*.py`), więc build JAR-a jest cache'owany jak dane.
- **Dlaczego:** sam synpp daje DAG i cache. Rozmiar pliku to tania heurystyka zmiany wejścia *(wnioskowane)*.
- **Czy dotyczy nas?** **częściowo.** Wzorzec jest dobry, ale SP go łamie (poniżej).

**Efekty uboczne poza cache: zapisy do `data_path`**
- **Zaobserwowane:** stage'e zapisują do katalogu danych surowych: `data/hts/cleaned.py:381-382` (`personsHTS.csv`, `tripsHTS.csv`), `synthesis/population/trips.py:21` (tu dodatkowo `data_path` nie jest zadeklarowany w `configure`), `data/opportunities/extract_roads_osm.py:156`, `matsim/scenario/supply/gtfs.py:18-29` (scalony GTFS rozpakowany i nadpisany w `data_path/gtfs`), `matsim/simulation/prepare.py:45` (przetransformowany shapefile do `data_path/Spatial`). Moduł `analysis/analysis.py:30-31` czyta potem `tripsHTS.csv` z `data_path`, czyli pośrednio zależy od efektu ubocznego, niewidocznego dla DAG-a.
- **Dlaczego:** *(wnioskowane)* wygoda przy debugowaniu i analizie. Brak dyscypliny „raw tylko do odczytu”.
- **Czy dotyczy nas?** **tak.** U nas `raw/` jest tylko do odczytu, a zapis tylko do `context.path()`/`interim/`. Warto to wymusić (uprawnienia FS albo test).

**Martwe/zepsute stage'e**
- **Zaobserwowane:** `synthesis/population/spatial/by_person/output.py:14-15` odwołuje się do stage'y bez prefiksu `synthesis.` (nie zadziała). `data/hts/output.py` i `data/out.py` to zrzuty CSV spoza głównego DAG-a.
- **Czy dotyczy nas?** **częściowo.** Bez CI takie rzeczy gniją. Warto mieć test „cały DAG się rozwiązuje”.

### 2. Konfiguracja i wiele scenariuszy

**Jeden `config.yml`, bez wariantów**
- **Zaobserwowane:** jeden plik z absolutnymi ścieżkami do NAS (`config.yml:6, 37-38, 52`), nazwami plików wejściowych jako kluczami (`shapefile_name`, `census_file`, `hts_file`, `osm_file`…; l. 40-46), `sampling_rate`, `random_seed`, `processes`, `java_memory`. Część parametrów ma domyślne wartości tylko w kodzie: `eqasim_version="1.2.0"` (`matsim/runtime/eqasim.py:13`), `pt2matsim_version="21.1"` (`pt2matsim.py:13`), `java_memory="50G"` (`java.py:6`). Część jest zaszyta mimo istnienia klucza: `osm.py:29` ma twardą ścieżkę `/nas/balacm/Data_SP/osm/sao_paulo.osm.gz` zamiast `osm_matsim_file`, walidacje sprawdzają stałe nazwy (`census/raw.py:66`, `osm.py:234`), `primary_locations.py:376, 427` zapisują wykresy pod `/home/asallard/...`. Kalibracyjne przesunięcia rozkładów są literałami (`secondary/locations.py:87-88`), liczba iteracji symulacji i wagi strategii też (`matsim/simulation/run.py:19-26`). README zapowiada użycie „dla innych regionów Brazylii” (`README.md:6-8`), ale nie ma mechanizmu wariantów. Porównanie: `eqasim-france` ma dziś `config_{lyon,nantes,toulouse,corsica,full}.yml` i przełącznik `hts`.
- **Dlaczego:** *(wnioskowane)* repo badawcze jednego zespołu na jednym serwerze.
- **Czy dotyczy nas?** **tak.** Scenariusze KBR 2018 vs 2024 to dokładnie „warianty”. Wszystkie parametry (także kalibracyjne i wersje narzędzi) muszą siedzieć w configu, a ścieżki pochodzić z `.env`.

### 3. Układ danych i duże pliki

**Dane zewnętrzne w jednym katalogu `data_path` o ustalonej strukturze; część wejść w repo**
- **Zaobserwowane:** `docs/howto.md:174-302` opisuje ręczne pobranie każdego źródła (Censo z rejestracją, OD 2017, Geofabrik plus wycięcie `osmosis` z `resources/sao_paulo.poly`, szkoły z ręczną poprawką wiersza 25881, shapefile CEM, GTFS) i docelowe ścieżki. Nie ma skryptu pobierania ani sum kontrolnych. Wielkość liter katalogów jest niespójna: howto podaje `data/Census`, `data/HTS`, `data/Spatial`, ale też „`data/spatial`” (l. 235), a kod używa `Census/`, `HTS/`, `Spatial/`, `osm/`, `gtfs/`. W git są `resources/`: shapefile stref (w tym ręcznie przygotowane `_city`, `_center`), `sao_paulo.poly` i **`emtu.zip` (54 MB, GTFS)** z ręcznie zmienioną datą `calendar.txt` (znacznik 2018-09-04 wobec 2017-06-13 pozostałych plików). `.gitignore` ma tylko `__pycache__` i `*.png`. Obok leży pusty plik-literówka `.gitgnore`. Wyjście: CSV/GPKG i XML MATSim do `output_path` (`synthesis/output.py`, `matsim/output.py:16-33`) plus `meta.json`. Gotowe scenariusze (1%) opublikowano na ETH polybox (`README.md:15-18`).
- **Dlaczego:** `resources/` zawiera pliki, których nie da się pobrać automatycznie albo które zostały wytworzone ręcznie (`docs/howto.md:236-239`).
- **Czy dotyczy nas?** **tak.** Artefakty ręczne (np. nasze podziały rejonów) powinny być w repo tylko jeśli są małe i licencyjnie czyste. Każde ręczne „poprawienie pliku źródłowego” (jak wiersz 25881 czy daty GTFS) powinno być krokiem w pipeline, nie instrukcją w README.

### 4. Proweniencja i wersjonowanie

**Minimalny `meta.json`**
- **Zaobserwowane:** `synthesis/output.py:139-146` zapisuje tylko `sampling_rate`, `random_seed` i czas utworzenia. Nie ma hasha commita, wersji wejść ani wersji eqasim-java. README mówi o „wersji 1.1” repo (`README.md:22`), ale nie ma tagów, changelogu ani `CITATION.cff` (cytowanie tylko w README). W `eqasim-france` są dziś `CHANGELOG.md`, `CITATION.cff`, `version.txt`.
- **Czy dotyczy nas?** **tak.** Artefakt popytu dla SUMO musi nieść hash kodu, wersję configu i identyfikatory wejść.

### 5. Źródło prawdy dla sieci
- Sieć MATSim z OSM przez pt2matsim `Osm2MultimodalNetwork` z konfiguracją XML wklejoną w kod (`matsim/scenario/supply/osm.py:12-217`), potem mapowanie GTFS (`processed.py`). Brak odniesień do innych narzędzi i spójności ID. Dla nas marginalne: nie dotyczy poza tym, że parametry `wayDefaultParams` są literałami.

### 6. Przekazanie MATSim → SUMO
- Nie dotyczy (brak SUMO; szukano `sumo`, `importPlans` w całym repo).

### 7. Kalibracja między narzędziami
- Nie dotyczy. Jedyna „kalibracja” to ręczne przesunięcia rozkładów odległości per tryb (`secondary/locations.py:87-88`) i porównania HTS vs synthetic w `analysis/` (wykresy, bez pętli zwrotnej).

### 8. Próbka / skalowanie
- `sampling_rate` stosowany po ekspansji wag spisu (`sampled.py:33-44`) i przekazywany do `RunGenerateConfig --sample-size` (`matsim/simulation/prepare.py:87-93`), który skaluje przepustowość po stronie Java. Publikowany scenariusz ma 1% (`README.md:17-18`). **Częściowo dotyczy**: ta sama konwencja „jeden parametr steruje populacją i przepustowością” jest nam potrzebna w eksporcie do SUMO.

### 9. Testy i determinizm

**Brak testów; determinizm częściowy i zepsuty**
- **Zaobserwowane:** nie ma katalogu testów, danych syntetycznych ani CI (szukano `test*`, `pytest`, `.github`, `Jenkinsfile`). `random_seed` jest użyty tylko w lokalizacji drugorzędnej (`secondary/locations.py:98-99`) i konfiguracji MATSim (`prepare.py:91`). Przy tym ziarna per paczka są liczone, ale w `process()` ignorowane, bo każdy proces seeduje się tym samym `context.config("random_seed")` (`secondary/locations.py:131-135`). Pozostałe losowania używają **globalnego, nieseedowanego** `np.random`: downsampling (`sampled.py:40`), hot-deck (`hot_deck_matching.py:62-64`, w procesach `mp.Pool`), rozrzut czasów odjazdu (`trips.py:49`), multinomial stref (`primary_zones.py:33, 50`), lokalizacje primary (`primary_locations.py:79, 95, 154, 164, 230, 285`). Wynik nie jest więc odtwarzalny, choć `meta.json` sugeruje, że jest. Jedyna „walidacja” to rozproszone `assert` (np. `data/hts/cleaned.py:355, 368`; `matched.py:60-61`) i `RunScenarioValidator` po stronie Java (`prepare.py:112-114`). Porównanie: `eqasim-france` ma dziś `tests/test_determinism.py`, `tests/testdata.py` (syntetyczne dane) i `test_pipeline.py`.
- **Dlaczego:** *(wnioskowane)* stan sprzed wprowadzenia testów determinizmu w rodzinie eqasim.
- **Czy dotyczy nas?** **tak.** To najmocniejszy argument, żeby od pierwszego dnia mieć (a) RNG przekazywany jawnie per stage, (b) test „dwa przebiegi → identyczny hash wyjścia” na małych syntetycznych danych KBR-like, bo prawdziwego KBR nie wolno commitować.

### 10. Dane zastrzeżone i licencje
- **Zaobserwowane:** repo na GPLv2 (`LICENSE:1-2`), bez licencji per katalog. `docs/howto.md:260-262` mówi, że GTFS EMTU nie jest otwarty i „tylko mieszkańcy Brazylii” mogą go pobrać, a jednocześnie `resources/emtu.zip` jest w repo. Spis wymaga rejestracji (`howto.md:186-188`). **Czy dotyczy nas?** tak, jako antywzorzec: dane o ograniczonej dystrybucji trafiły do publicznego repo bez adnotacji licencyjnej. U nas zastrzeżone dane idą wyłącznie przez `.env`/dysk, a `.gitignore` i pre-commit mają to blokować.

### 11. Runtime Python ↔ Java

**JAR budowany z nieprzypiętej gałęzi `develop` eqasim-java; wersja tylko w nazwie pliku**
- **Zaobserwowane:** stage `matsim.runtime.eqasim` robi `git clone --branch develop --depth 1` eqasim-java i `mvn -Pstandalone package`, po czym zwraca ścieżkę `eqasim-java/sao_paulo/target/sao_paulo-<eqasim_version>.jar` (`matsim/runtime/eqasim.py:26-41`). `eqasim_version` (domyślnie `1.2.0`, l. 13) **nie wybiera commitu**, służy tylko do zbudowania nazwy pliku. Wynik zależy więc od dnia uruchomienia i przestanie działać, gdy `develop` podbije wersję. pt2matsim jest natomiast przypięty tagiem `v21.1` i ma smoke test (`pt2matsim.py:30-46`). Java wywoływana przez `subprocess` z `-cp jar`, `-Xmx` z configu i `java.io.tmpdir` w cache stage'u (`matsim/runtime/java.py:8-60`). `validate` sprawdza obecność binarek i ostrzega, gdy Java ≠ 11 (`java.py:62-69`), a Maven 3 jest wymagany (`maven.py:35-40`). **Kod Java SP żyje w innym repo**: moduł `sao_paulo` w eqasim-java (`RunSimulation`, `RunAdaptConfig`, `preparation.RunImputeInnerSPAttribute`; `prepare.py:56-61, 97-100`, `run.py:19`, `README.md:18-20`). Atrybuty osób zapisywane w Pythonie (`residenceAreaId`, `hasPtSubscription`, `carAvailability`; `matsim/scenario/population.py:36-54`) muszą zgadzać się z tym, co czyta model wyboru trybu w Java, a nic tego nie sprawdza. Dla porównania `eqasim-france/matsim/runtime/eqasim.py:16-21, 79-83` ma dziś `eqasim_version` + `eqasim_commit`/`eqasim_tag` z wymogiem dokładnie jednego i sprawdza, czy JAR powstał.
- **Dlaczego:** *(wnioskowane)* w 2021 eqasim-java nie publikował artefaktów per region. Budowanie z gałęzi było najprostsze.
- **Czy dotyczy nas?** **tak.** Jeśli będziemy mieć własne klasy MATSim, JAR musi być przypięty do commitu/tagu (albo pobierany z rejestru Maven), a kontrakt atrybutów Python→Java opisany i testowany.

### 12. Pakietowanie

**Katalog skryptów, nie pakiet**
- **Zaobserwowane:** brak `setup.py`/`pyproject.toml`. Moduły to pakiety namespace (`data`, `synthesis`, `matsim`, `analysis`) importowane względem CWD, uruchamiane przez `python3 -m synpp` (`docs/howto.md:357-361`). Dwa rozjechane pliki zależności: `environment.yml` (pandas 1.0.3, synpp 1.2.2, geopandas 0.6.1) vs `requirements.txt` (pandas 0.25.3, synpp 1.2.1, geopandas 0.7.0). Nazwy katalogów `data`/`matsim` kolidują z popularnymi nazwami pakietów *(wnioskowane ryzyko)*. `eqasim-france` ma dziś `pyproject.toml` + `uv.lock`.
- **Czy dotyczy nas?** **tak.** U nas rdzeń syntezy jako biblioteka (członek workspace'u `uv`) i adaptery danych Wrocław jako osobny członek. Jeden plik locka, bez duplikowania list zależności.

### 13. Dokumentacja
- **Zaobserwowane:** `README.md` (cel, publikacje, linki do gotowych scenariuszy i klasy `RunSimulation`) oraz `docs/howto.md`: jedno miasto, instrukcja pobrania danych, struktura katalogów, uruchomienie. Nieaktualności: link do klonowania `github.com/eqasim-org/sao_paulo` (`howto.md:306-311`), numeracja źródeł skacze z 5) do 9) (l. 241), niespójna wielkość liter katalogów. Nie ma opisu schematu wyjść poza listą plików (`howto.md:368-381`), ani opisu mapowań kodów ankiety. W repo leży przypadkowy `docs/.Rhistory`. Walidacja wyniku jest w `analysis/` (skrypt + notebook porównujący łańcuchy aktywności, tryb×cel i rozkłady odległości HTS vs synthetic; `analysis/analysis.py:84-395`). Stage `analysis.analysis` czyta jednak pliki z efektów ubocznych (zob. 1).
- **Czy dotyczy nas?** **częściowo.** Dokument „skąd wziąć każde wejście i gdzie je położyć” jest dobry (odpowiada naszym `docs/dane/*.md`). Brakujący element to dokument mapowania kodów ankiety → kategorie modelu, który u nas powinien istnieć dla KBR 2018 i 2024 osobno.

---

## Najważniejsze dla nas

1. **Adaptacja do nowego kraju skupia się w 4 miejscach:** parser spisu, parser ankiety (kody trybów/celów, czyszczenie łańcuchów), źródło OD i źródło możliwości/lokalizacji (tu OSM zamiast rejestrów). Hot-deck, łańcuchy aktywności, lokalizacja drugorzędna (`rda.py`) i writery MATSim przetrwały prawie bez zmian. To wyznacza granicę „rdzeń vs adapter” w naszym monorepo.
2. **Interfejs rdzenia musi być jawny.** W SP był niejawny (krotka ramek + nazwy kolumn + stringi kategorii), co widać po pozostałościach francuskich (`EPSG:2154`) i martwym argumencie wag. Dzisiejszy `eqasim-france` ma `HOUSEHOLD/PERSON/TRIP_COLUMNS` + `check()` + wybór ankiety przez config i to jest wzór dla adaptera KBR.
3. **Wagi ankiety łatwo „gubią się” po drodze:** waga podróży odrzucona, osoby usuwane bez przeważenia, hot-deck ignoruje wagi. Dla porównania KBR 2018/2024 potrzebujemy testów, że wagi faktycznie działają.
4. **OD można budować z ankiety**, mapując współrzędne O/D do stref spisowych (a nie do stref ankiety). Nasz stage OD powinien mieć wymienne źródło (KBR / sieć komórkowa) z jednym formatem wyjścia.
5. **Determinizm i przypięcie wersji Java nie mogą być dodatkiem:** SP ma nieseedowane `np.random` w większości stage'y i buduje JAR z `develop`. `meta.json` z samym seedem daje fałszywe poczucie odtwarzalności.
6. **Raw tylko do odczytu:** SP zapisuje do `data_path` z kilku stage'y i analiza od tego zależy. Warto to wymusić technicznie.

## Nie znaleziono

- **Historii adaptacji w git:** repo ma jeden commit (`git log`), więc nie widać, co zmieniano stopniowo.
- **Testów, danych testowych, CI:** brak katalogów `tests/`, `.github/`, `Jenkinsfile` i plików `test_*` (listing `find` całego repo).
- **Walidacji schematu ramek / listy wymaganych kolumn:** grep `COLUMNS`, `check(`, `schema` w `data/`, `synthesis/`.
- **Pinningu eqasim-java do commitu/tagu:** `matsim/runtime/eqasim.py` (tylko `--branch develop`).
- **CHANGELOG, CITATION.cff, tagów wersji, hash commita w wynikach:** korzeń repo, `synthesis/output.py`.
- **Użycia wag podróży `FE_VIA` i stref ankiety `ZONA`/`ZONA_O`/`ZONA_D` do mapowania:** wczytywane w `data/hts/cleaned.py:19-27`, ale ta pierwsza wypada, a drugie są przekazywane dalej tylko jako kolumny, nie do dopasowania stref.
- **Kodu Java SP:** nie w tym repo, tylko linki do modułu `sao_paulo` w eqasim-java (`README.md:18-20`).
- **Integracji z SUMO / eksportu poza MATSim:** brak.
