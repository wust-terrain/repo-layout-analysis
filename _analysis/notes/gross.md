# gross (GROSS — German Rail Open-Source SUMO Scenario)

- **Repo:** https://github.com/ethz-coss/GROSS
- **Commit:** `d1134079021eab78fb40fb1e30580809d396f00e` (MANIFEST.tsv: `full`, `ok`, 2026-05-16); jeden commit w historii („Initial Pipeline Commit”, Juri Penell, 2026-05-16).
- **Klon:** pełny (nie częściowy).

**Czym jest to repo.** Pipeline (ETH Zürich, COSS), który z OSM (Geofabrik) i GTFS (DELFI, całe Niemcy) buduje scenariusz **kolejowy** dla SUMO: sieć tylko torowa, rozkłady z GTFS jako pojazdy z przystankami. Siedem numerowanych skryptów bash (`scripts/stage_1..7_*.sh`) orkiestrowanych przez `run.sh`, który sam siebie uruchamia ponownie w kontenerze Docker. Ciężkie parsowanie w C++ (`bin/main`, `bin/routing`), reszta w Pythonie (luźne skrypty, nie pakiet), SUMO `netconvert`/`duarouter`/`sumo` jako narzędzia zewnętrzne. Artykuł na SUMO Conference 2026 (`README.md:53-69`, URL wciąż `PLACEHOLDER`). Brak drogowego popytu — „popyt” to rozkład GTFS. Dla nas to wzorzec „numerowane skrypty + `--skip-to`” i przykład trzymania ID OSM przez netconvert, a nie źródło rozwiązań dla MATSim.

---

### 1. Orkiestracja

**Numerowane skrypty etapów, liniowy łańcuch w `run.sh`**
- **Zaobserwowane:** `run.sh:109-115` wywołuje po kolei `scripts/stage_1_prepare.sh` … `stage_7_simulate.sh` przez `_run_stage <idx> <script> "$@"`; każdy etap dostaje *te same* argumenty CLI. Każdy skrypt robi `source scripts/paths.sh "$@"` i `make_dirs` (np. `stage_5_route.sh:34-35`), więc ścieżki wszystkich plików pośrednich są wyliczane deterministycznie z `<name>` (`scripts/paths.sh:44-121`). Kontrakt między etapami = stałe nazwy plików w katalogu scenariusza, opisane w nagłówkach skryptów („Inputs (from previous stage)” / „Outputs”, np. `stage_2_parse.sh:11-21`, `stage_6_assemble.sh:13-24`).
- **Dlaczego:** prosty, czytelny przepływ bez frameworka; każdy etap daje się też odpalić ręcznie jako `scripts/stage_N_*.sh <name> …` *(wnioskowane z nagłówków „Usage”)*.
- **Czy dotyczy nas?** częściowo — pokazuje, że wzorzec „wspólny `paths.sh` + etapy” jest tani, ale też jego granice (poniżej).

**`--skip-to N` to czysty licznik — zero detekcji nieaktualności**
- **Zaobserwowane:** `run.sh:72-89` wycina `--skip-to` z argumentów; `_run_stage` (`run.sh:91-99`) wykonuje etap tylko gdy `idx >= SKIP_TO`, inaczej drukuje „(skipping stage …)”. Nie ma sprawdzenia, czy wyjścia wcześniejszych etapów istnieją, ani czy pochodzą z tych samych wejść/kodu/konfiguracji. Częściowe re-run zakłada więc, że katalog `data/scenarios/<name>/` zawiera kompletne, spójne wyjścia etapów < N z *poprzedniego* przebiegu. Niektóre etapy mają ręczne guardy istnienia (`stage_5_route.sh:39-49`, `stage_6_assemble.sh:34-58`, `stage_7_simulate.sh:16-19`), ale tylko „plik istnieje”, nie „plik aktualny”. Komunikaty błędów są nieaktualne po przenumerowaniu etapów („Run stage_4_net.sh first” w `stage_5_route.sh:41,47`; nagłówek `stage_3_net.sh:3,9` mówi „stage_4_net.sh”, echo „Stage 4” w `stage_3_net.sh:45,61`) — dwa etapy przedstawiają się jako „Stage 4”.
- **Dlaczego:** przy danych całych Niemiec etapy są drogie; `--skip-to` pozwala iterować nad późnym etapem *(wnioskowane)*.
- **Czy dotyczy nas?** tak — to jest dokładnie antywzorzec, którego chcemy uniknąć przy wersjonowanym artefakcie popytu: bez fingerprintu wejść `--skip-to` po zmianie configu po cichu miesza stare i nowe pliki.

**Cache wewnątrz etapów: „istnieje ⇒ pomiń”, z flagami wymuszenia**
- **Zaobserwowane:** Stage 1 pomija pobieranie, split `osmium` i unzip, jeśli pliki istnieją, chyba że `--update` (`src/file_preparation/filePreparation.py:17-19,119-122,131-135,171-179`; sentinel `stops.txt` dla GTFS). Stage 3: `src/net/net.py:769-791` pomija `osmium tags-filter`, `osmconvert` i `netconvert`, jeśli wyjścia istnieją i mają nagłówek gzip (`_is_valid_gz`, `net.py:45-51` — jedyna „walidacja”, chroni przed uciętym zapisem), chyba że `--force`. Za to `correctPermissions` i edge weights liczone są zawsze (`net.py:794-826`), a Stage 2/4/6 zawsze nadpisują.
- **Dlaczego:** drogie kroki zewnętrzne (download, netconvert na całych Niemczech) cache'owane, tanie liczone od nowa *(wnioskowane)*.
- **Czy dotyczy nas?** częściowo — sprawdzanie magic bytes gzip to tani trik na przerwane zapisy, ale cache bez klucza (hash wejścia/opcji) oznacza, że zmiana `config/convert.netccfg` nie unieważnia `intermediary.net.xml.gz` bez `--force`.

**Przekazywanie flag: wszystkie etapy dostają wszystko, każdy wyłuskuje swoje**
- **Zaobserwowane:** `paths.sh:26-35` ignoruje nieznane flagi (`*) shift`); etapy same parsują `--date` (`stage_2_parse.sh:26-45`), `--maps` (`stage_4_match.sh:23-31`), `--force`/`--database` (`stage_3_net.sh:29-40`), `--skip-dua` (`stage_5_route.sh:28-32`). `--update` i `ROUTING_SEED`, `EDGE_LENGTHS_SOURCE`, `BASE_DIR` idą przez zmienne środowiskowe (`stage_1_prepare.sh:41`, `stage_6_assemble.sh:63,114`, `paths.sh:44`), ale `docker run` w `run.sh:57-63` przekazuje tylko `GROSS_IN_CONTAINER=1` — więc przy standardowym wejściu przez `run.sh` te zmienne nie docierają do kontenera *(wnioskowane z kodu, nie uruchamiane)*.
- **Dlaczego:** brak centralnego parsera; każdy etap rozwijany niezależnie *(wnioskowane)*.
- **Czy dotyczy nas?** tak (jako przestroga) — opcje przebiegu rozsiane po argv i env, bez jednego obiektu konfiguracji, są niezapisywane i łatwo gubione na granicy kontenera.

**Budowa C++ przy każdym uruchomieniu**
- **Zaobserwowane:** bez `--skip-build` `run.sh:102-106` robi `make clean && make -j` przy każdym przebiegu; `Makefile` ma cele plikowe (`Makefile:12`, `bin/main` z obiektów), ale `clean` je kasuje. `--skip-build` nie wpływa na budowę obrazu Docker (ta jest sterowana mtime `Dockerfile`, `run.sh:25-37`).
- **Czy dotyczy nas?** nie (nie mamy C++), ale uwaga na nasz JAR MATSim: budowa przy każdym runie = brak przypięcia artefaktu.

### 2. Konfiguracja i wiele scenariuszy

**Scenariusz = nazwa + flagi CLI; brak pliku scenariusza**
- **Zaobserwowane:** „miasto/wariant” to `<name>` + `--country`/`--region` (nazwy Geofabrik) lub `--osm-source`/`--gtfs-source`, opcjonalnie `--date YYYYMMDD` (filtr dnia GTFS w `bin/main`, `src/main/main.cpp:30-55`). Katalog scenariusza `data/scenarios/<name>/` (`paths.sh:44`). Nic nie zapisuje użytych flag; jedyny zapis to `osm/sources.txt` / `gtfs/sources.txt` przy źródle zewnętrznym (`filePreparation.py:75-79,160-164`). README pokazuje `brandenburg` i `germany` (`README.md:32-41`).
- **Dlaczego:** region = wycinek Geofabrik, reszta pipeline'u bez parametrów *(wnioskowane)*.
- **Czy dotyczy nas?** częściowo — mamy jedno miasto, ale warianty (KBR 2018 vs 2024, próbki) wymagają zapisanego, wersjonowanego pliku konfiguracji, czego tu brak.

**`config/` = globalne szablony narzędzi, nie per scenariusz**
- **Zaobserwowane:** `config/convert.netccfg` (opcje netconvert), trzy `*.typ.xml` (typy OSM kolejowe), `config/sim.sumocfg` (szablon, przepisywany przez `writeConfig` w `filePreparation.py:190-261` na ścieżki absolutne scenariusza), `config/exceptions.xml` (ręczne wyjątki danych: usuwanie agencji GTFS, np. S-Bahn Berlin, autobusy/tramwaje błędnie oznaczone jako rail). URL-e źródeł w `download_links.env:1-2` — GTFS przypięty do konkretnego zrzutu DELFI `20251229`, OSM tylko do `…-latest.osm.pbf` (`filePreparation.py:107-114`).
- **Dlaczego:** `exceptions.xml` z polem `reason` dokumentuje ręczne poprawki danych w jednym miejscu (dane w repo mówią to wprost przez atrybut `reason`).
- **Czy dotyczy nas?** tak — plik wyjątków z `reason` to dobry wzorzec dla znanych błędów w GTFS/OSM Wrocławia; OSM `latest` to przestroga (niereprodukowalne).

**Rozjechane stałe konfiguracyjne**
- **Zaobserwowane:** vTypes zdefiniowane dwukrotnie (`filePreparation.py:280-284` i `SIM_VTYPES` w `src/routing/routing.py:446-450`, z komentarzem „must match”), a `SPEED_CLASSES` do routingu (`routing.py:55-58`) mają inne prędkości (regional 44.44 m/s vs 33.3 m/s w vTypes symulacji). `src/fixing/fixing_mapping_stops.py:7,17` ma zaszytą listę regionów i pętlę po nich — przy konkretnych ścieżkach z `stage_4_match.sh:54-57` (brak `{name}`) wykonuje tę samą pracę 5 razy.
- **Czy dotyczy nas?** tak — argument za jednym źródłem prawdy dla stałych (vTypes, klasy pojazdów) współdzielonym przez eksport MATSim i konfigurację SUMO.

### 3. Układ danych i duże pliki

**Jedno drzewo per scenariusz, rozdzielone na typ, nie na raw/interim/processed**
- **Zaobserwowane:** `paths.sh:44-55`: `data/scenarios/<name>/{osm,gtfs,sumo,sim,routing,routing/scratch,output,misc,misc/logs,misc/maps}`. Wejścia surowe (`osm/base.osm.pbf`, `gtfs/`) leżą obok pośrednich (`osm/filtered.osm.pbf`), sieć w `sumo/`, gotowy scenariusz w `sim/` (sumocfg + `*.rou.xml.gz` + vtypes; sieć i przystanki referencjonowane z `sumo/`), wyniki SUMO w `output/`. `BASE_DIR` nadpisywalny przez env „dla klastrów obliczeniowych” (`paths.sh:40-44`). Konsekwentnie `.xml.gz`; etapy gzipują wyjścia C++ post factum (`stage_2_parse.sh:64-74`, `stage_6_assemble.sh:130-136`).
- **Czy dotyczy nas?** częściowo — nasz podział `raw/` (read-only) / `interim/` / `processed/` jest mocniejszy; tu surowe OSM leżą w katalogu scenariusza i mogą być nadpisane przez `--update`.

**Git i pobieranie**
- **Zaobserwowane:** `.gitignore:1-4`: `*.pyc`, `*.o`, `bin/*`, `data/*` — całe dane poza gitem. Wejścia pobierane przez `requests` z URL-i w `download_links.env` (bez checksumy), zewnętrzne źródła symlinkowane (`filePreparation.py:83-102`). Nie znaleziono publikacji wyników (brak Zenodo/release/DVC).
- **Czy dotyczy nas?** częściowo — potwierdza „dane poza gitem”, ale brak publikacji artefaktu to luka, którą my musimy wypełnić.

**Uwaga: `--osm-source` z katalogiem łamie Stage 3**
- **Zaobserwowane:** przy katalogu z samymi split-PBF Stage 1 nie tworzy `base.osm.pbf` (`filePreparation.py:92-95`), a `net.py:765-767` wymaga `osm/base.osm.pbf` i kończy z błędem. `net.py:735` ma domyślną ścieżkę `data/scenario/<name>` (liczba pojedyncza) vs `data/scenarios/` w `paths.sh` — maskowane przez jawne `--data-dir`.
- **Czy dotyczy nas?** nie bezpośrednio — ilustruje koszt braku testów ścieżek alternatywnych.

### 4. Proweniencja i wersjonowanie

**Minimalna: `sources.txt` i nic więcej**
- **Zaobserwowane:** jedyny zapis proweniencji to ścieżka zewnętrznego źródła w `osm/sources.txt`/`gtfs/sources.txt` (`filePreparation.py:75-79,160-164`; komentarz „for reproducibility” w `paths.sh:56-57`). Brak hasha commita, wersji SUMO, daty zrzutu OSM, użytych flag czy seeda w wynikach. Brak CHANGELOG, `CITATION.cff` (cytowanie tylko w `README.md:53-69`, URL `PLACEHOLDER`), tagów, automatyzacji release'ów; repo ma jeden commit.
- **Czy dotyczy nas?** tak — negatywny przykład: artefakt SUMO bez manifestu (commit, wersja SUMO, hash wejść) nie pozwala stwierdzić, z czego powstał.

**Wersja SUMO nieprzypięta**
- **Zaobserwowane:** `Dockerfile:1` `FROM ubuntu:latest`; `Dockerfile:8` `add-apt-repository ppa:sumo/stable`; `Dockerfile:16` pakiet `sumo` bez wersji; pip bez pinów (`Dockerfile:43`). Rebuild obrazu tylko gdy mtime `Dockerfile` > data utworzenia obrazu (`run.sh:25-37`) — czyli wersja SUMO zależy od dnia pierwszego buildu na danej maszynie. `INSTALL_GTEST` użyty w `Dockerfile:28` bez deklaracji `ARG` (tylko `INSTALL_GIT`, `Dockerfile:11`).
- **Dlaczego:** wygoda — zawsze najnowszy stable *(wnioskowane)*.
- **Czy dotyczy nas?** tak — dla środowiska RL i reprodukowalności publikacji wersja SUMO musi być przypięta (np. konkretny tag obrazu / wersja pip `eclipse-sumo`) i zapisana w manifeście artefaktu.

### 5. Źródło prawdy dla sieci

**Sieć kanoniczna: `sumo/final.net.xml.gz` z netconvert na przefiltrowanym OSM, dwa przebiegi**
- **Zaobserwowane:** `net.py`: `osmium tags-filter` (węzły + drogi z `railway`/`public_transport`, `net.py:30-42`) → `osmconvert` do `.osm.gz` → `netconvert -c config/convert.netccfg` z typ-files, `--ptstop-output`, `--ptline-output`, `--junctions.join-output` (`net.py:54-85`) → `intermediary.net.xml.gz` → `correctPermissions` modyfikuje priorytety krawędzi w lxml (stop=7, bidi=1, crossover/siding=0, krawędzie z `-` = 3; docstring `net.py:337-341`) → drugi `netconvert --sumo-net-file … --railway.topology.extend-priority --railway.topology.repair.stop-turn` (`net.py:505-517`) → `final.net.xml.gz`. Kluczowe opcje w `convert.netccfg:9-58`: `output.original-names=false`, `keep-edges.by-vclass=rail,rail_electric,rail_fast`, `railway.topology.all-bidi`, `junctions.join`, `geometry.remove.keep-ptstops`, `osm.all-attributes`.
- **Czy dotyczy nas?** częściowo — wzorzec „netconvert + deterministyczny post-processing XML + ponowny netconvert” jest użyteczny do nanoszenia danych ZDiUM (np. programów sygnalizacji) na sieć; u nas sieć drogowa, więc opcje kolejowe nie mają zastosowania.

**Spójność ID oparta na ID OSM zachowanych przez netconvert**
- **Zaobserwowane:** łączenie GTFS ↔ SUMO idzie przez ID elementów OSM: `bin/main` klastruje przystanki GTFS z węzłami/platformami OSM (`misc/extensive_clusters.xml.gz`), a `routing.py:411-437` (`build_osm_to_sumo_map`) mapuje ID OSM na ID `busStop`/`trainStop` netconverta, zdejmując wiodący `-` (kierunek przeciwny) i heurystycznie wydłubując tokeny numeryczne ze złożonych ID. `fixing_mapping_stops.py:32-46` usuwa z `mapping.xml` przystanki, których netconvert nie wyemitował (też z `lstrip("-")`). Brak jawnej tabeli ID ani testu spójności.
- **Dlaczego:** netconvert zachowuje ID OSM w ID krawędzi i przystanków, więc nie trzeba trzymać osobnej tabeli *(wnioskowane)*.
- **Czy dotyczy nas?** tak — to samo założenie będziemy mieli przy MATSim (pt2matsim/OSM) ↔ SUMO; lekcja: konwencja „ID OSM z prefiksem `-`” jest krucha (heurystyki w `routing.py:428-433`), lepiej jawny plik mapowania ID generowany razem z siecią i walidowany.

**Popyt = rozkład GTFS, routowany przez duarouter**
- **Zaobserwowane:** Stage 5 dla każdej użytej pary kolejnych przystanków GTFS generuje trip między platformami i odpala `duarouter` (`routing.py:678-706`: `--weights.reversal-penalty` per klasa, `--weight-files` z `edge_weights.add.xml.gz`, `--ignore-errors` domyślnie, batch ≤ 16384, bisekcja batchy przy błędzie `routing.py:768-876`, opcjonalnie Slurm). Stage 6 (`bin/routing`) składa trasy per pociąg i klasę prędkości w `sim/{highspeed,intercity,regional}.rou.xml.gz`, loguje `unrouted_trips.log`, `split_cases.log`, `missing_stops.csv` (`stage_6_assemble.sh:138-141`).
- **Czy dotyczy nas?** częściowo — logi „co zgubiono przy konwersji” (nieroutowane tripy, brakujące przystanki) to dobry wzorzec raportu dla naszego eksportu MATSim → SUMO.

### 6. Przekazanie MATSim → SUMO
Nie dotyczy — brak MATSim w repo (szukano `matsim` w całym drzewie).

### 7. Kalibracja między narzędziami
Nie dotyczy — jedyne „sprzężenie” to wagi krawędzi z priorytetów sieci dla duarouter (`net.py:532+`), nie kalibracja między symulatorami.

### 8. Próbka / skalowanie
Nie dotyczy — pełny rozkład GTFS, brak próbkowania populacji.

### 9. Testy i determinizm

**Brak testów**
- **Zaobserwowane:** brak katalogu `tests/`, brak `pytest`/`gtest` w kodzie; jedynie martwa flaga `INSTALL_GTEST` w `Dockerfile:28`. Brak CI (`.github/` nie istnieje). Brak małego scenariusza testowego ani syntetycznych danych.
- **Czy dotyczy nas?** tak (negatywnie) — potwierdza potrzebę mini-scenariusza (np. jedna dzielnica) uruchamianego w CI.

**Determinizm częściowy**
- **Zaobserwowane:** `bin/routing` ma `--seed` (domyślnie `0`, `src/routing/routing.cpp:1761-1770`) i `std::mt19937` z per-trip seedami (`routing.cpp:1504,1533`); seed ustawiany przez env `ROUTING_SEED` (`stage_6_assemble.sh:114-117`), który nie jest forwardowany do kontenera. `bin/main` używa `std::random_device` (`src/main/main.cpp:154-155`) — wygląda na użycie tylko do kolorów wizualizacji klastrów *(wnioskowane z kontekstu `rgb_dis`)*. Równoległy duarouter z bisekcją przy błędach może dawać różny podział batchy, ale wyniki są łączone po parach *(wnioskowane)*. Brak testu powtarzalności.
- **Czy dotyczy nas?** częściowo — dobry drobiazg: domyślny seed stały, nie losowy; zły: seed nie jest zapisywany w wyjściu.

### 10. Dane zastrzeżone i licencje
Nie dotyczy (częściowo) — brak pliku `LICENSE` w repo (sprawdzono root i całe drzewo), więc formalnie kod bez licencji; dane wyłącznie otwarte (Geofabrik, DELFI), brak separacji danych zastrzeżonych.

### 11. Runtime Python ↔ Java
Nie dotyczy — brak Javy. Analogiczny problem C++↔Python rozwiązany przez `Makefile` + wywołania binarek `./bin/*` ze skryptów bash (`stage_2_parse.sh:54-59`, `stage_6_assemble.sh:120`), bez bindingów.

### 12. Pakietowanie

**Brak pakietu Pythona; moduły importowane przez `sys.path`**
- **Zaobserwowane:** brak `pyproject.toml`/`setup.py`/`requirements.txt`; zależności Pythona instalowane w `Dockerfile:43` (`pip3 install … --break-system-packages`, bez wersji). `src/` ma `__init__.py`, ale skrypty uruchamiane jako `python3 ./src/…/x.py` i importują `src.gtfs.gtfs` itp. (`net.py:25`) z fallbackami (`cluster.py:999-1009`). C++ i Python w tym samym `src/<moduł>/` (np. `src/routing/routing.cpp` + `routing.py`).
- **Czy dotyczy nas?** nie jako wzorzec do naśladowania — nasz `uv` workspace z pakietem na etap i entry pointami CLI jest lepszy; potwierdza, że Docker jako jedyne źródło zależności utrudnia uruchomienie poza kontenerem.

**Docker jako entry point, re-exec tego samego skryptu**
- **Zaobserwowane:** `run.sh:39-64`: poza kontenerem buduje obraz (jeśli trzeba) i `exec docker run --user $(id -u):$(id -g) -v $(pwd):/app … /app/run.sh "$@"`; w kontenerze (`GROSS_IN_CONTAINER=1`) wykonuje pipeline. Repo zamontowane w całości, więc `data/` ląduje na hoście.
- **Dlaczego:** „for convenience and ease of dependency management” (`README.md:19`).
- **Czy dotyczy nas?** częściowo — wzorzec „jeden skrypt, który sam się re-execuje w kontenerze” + `--user` (brak plików root na hoście) jest wart skopiowania dla środowiska SUMO; trzeba jednak jawnie forwardować zmienne i przypinać obraz.

### 13. Dokumentacja

**Tylko README + nagłówki skryptów**
- **Zaobserwowane:** `README.md` (69 linii): opis 7 etapów, użycie, opcje, cytowanie. Nie wymienia `--date`, `--maps`, `--routing-threads` (są w `run.sh:6-11,19-22`), ani zmiennych env. Najlepsza dokumentacja to nagłówki etapów z listą Inputs/Outputs (`stage_2_parse.sh:2-22`, `stage_3_net.sh:2-24`, `stage_6_assemble.sh:2-25`) — część nieaktualna po przenumerowaniu. Brak dokumentów per region; `exceptions.xml` z `reason` pełni rolę notatek o danych.
- **Czy dotyczy nas?** częściowo — nagłówek „Inputs/Outputs” w każdym etapie to tani, przydatny wzorzec; ale ręcznie utrzymywany dryfuje (przykład „Stage 4” ×2), więc lepiej generować go z deklaracji etapu.

---

## Najważniejsze dla nas

1. **`--skip-to` bez detekcji nieaktualności to pułapka** (`run.sh:91-99`): pomija etapy po numerze, zakładając, że pliki pośrednie są aktualne. Jeśli wybierzemy numerowane etapy, każdy etap powinien zapisywać manifest (hash wejść + config + commit) i sprawdzać go przy wznowieniu — albo wybierzmy orkiestrator z cache opartym o klucze.
2. **Wersje nieprzypięte** (`ubuntu:latest`, `ppa:sumo/stable`, pip bez wersji, OSM `-latest`) + brak manifestu w wynikach = artefaktu nie da się odtworzyć. Nasz artefakt popytu i konfiguracja SUMO muszą nieść wersję SUMO, commit i hash wejść.
3. **Spójność ID przez ID OSM przepuszczane przez netconvert** działa, ale opiera się na heurystykach (`-`-prefiks, tokeny numeryczne, `routing.py:411-437`). Lepiej jawny plik mapowania ID tworzony razem z siecią i walidowany testem.
4. **`config/exceptions.xml` z atrybutem `reason`** — dobry wzorzec ręcznych poprawek danych w jednym miejscu, do użycia dla GTFS/OSM Wrocławia.
5. **Re-exec `run.sh` w Dockerze z `--user`** — praktyczny sposób na środowisko SUMO, pod warunkiem jawnego forwardowania env (tu `ROUTING_SEED`/`UPDATE_DATA` giną) i przypiętego tagu obrazu.
6. **Logi strat konwersji** (`unrouted_trips.log`, `missing_stops.csv`, `split_cases.log`) — wzorzec dla raportu z naszego przejścia MATSim → SUMO.

## Nie znaleziono

- Testów, CI, mini-scenariusza testowego (szukano `tests/`, `test_*`, `.github/`, gtest w całym drzewie).
- Licencji (`LICENSE*`, `COPYING` — brak w root i drzewie; README też nie podaje).
- `CITATION.cff`, CHANGELOG, tagów, automatyzacji release'ów (jeden commit w historii).
- Zapisu commita/wersji SUMO/flag/seeda w wynikach (grep `git rev`, `version`, `commit` w `*.py`/`*.sh`/`*.cpp`).
- Pliku konfiguracji scenariusza (YAML/TOML) — scenariusz to tylko argv.
- Publikacji wyników (Zenodo, release assets, DVC).
- Jakiegokolwiek związku z MATSim, próbkowaniem populacji czy kalibracją.
