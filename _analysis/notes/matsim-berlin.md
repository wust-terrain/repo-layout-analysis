# matsim-berlin

- **Repo:** https://github.com/matsim-scenarios/matsim-berlin
- **Commit:** `04d23cd1147228543282234befc98f2d0a683cb7` (2025-01-13, z `MANIFEST.tsv`; tier 1)
- **Klon:** pełny (`full` w MANIFEST), ale płytki: `git log` pokazuje 1 commit, więc historii zmian plików nie da się prześledzić.

**Czym jest to repo.** Scenariusz MATSim „Open Berlin” (TU Berlin, VSP), wersja 6.4. Kod Javy (Maven, JDK 21) opiera się na `matsim-all` i `contrib/application` (`MATSimApplication` + picocli). Jest w nim klasa uruchomieniowa `OpenBerlinScenario` oraz klasa „fabryki scenariusza” `RunOpenBerlinCalibration`, która rejestruje ok. 40 poleceń `prepare ...`. Cały proces budowy scenariusza opisuje root `Makefile`, od OSM i danych statystycznych po populację 10pct. Część Pythonowa w `src/main/python/` to luźne skrypty, a nie pakiet: ekstrakcja danych referencyjnych z ankiety SrV, kalibracja ASC przez `matsim-tools`, estymacja modeli wyboru (biogeme) oraz podprojekt `capacity/`, w którym **mikrosymulacja SUMO szacuje przepustowość i prędkość swobodną dla sieci MATSim**. W `src/main/R/` jest kilka skryptów R do walidacji. Duże pliki wejściowe i wyjściowe leżą poza gitem, na publicznym SVN VSP.

---

### 1. Orkiestracja

**DAG celów plikowych w Make, jeden plik na etap, wersja w ścieżce**
- **Zaobserwowane:** w `Makefile` każdy artefakt jest celem plikowym ze ścieżką `input/$V/berlin-$V-...` (`V := v6.4`, `Makefile:4,7`). Zależności podane są jako prerekwizyty, np. `berlin-$V-network.xml.gz: input/sumo.net.xml` (`Makefile:98`), a populacja: `berlin-only`/`brandenburg-only` → `berlin-static` → `berlin-activities` → `berlin-initial` → `berlin-cadyts-input` (`Makefile:157-262`). Punkty wejścia to cele-agregaty `prepare-calibration`, `prepare-initial`, `prepare-drt`, `prepare` (`Makefile:362-372`). Każdy krok to wywołanie `$(sc) prepare <polecenie>`, gdzie `sc` oznacza `java -cp $(JAR) org.matsim.prepare.RunOpenBerlinCalibration` (`Makefile:17`). Jeden JAR udostępnia więc wszystkie etapy jako subkomendy picocli (`RunOpenBerlinCalibration.java:88-101`). Surowe wejścia z sieci ściągają cele `curl` (`Makefile:24-58`).
- **Zaobserwowane (słabe punkty):**
  - (a) Wiele receptur zapisuje w miejscu ten sam plik kilka razy z rzędu (`--input $@ --output $@`, np. `Makefile:98-120`, `302-328`). Nie ma `.DELETE_ON_ERROR` (grep: brak). Jeśli środkowy krok się wysypie, zostaje częściowy plik, który Make uzna za aktualny *(wnioskowane)*.
  - (b) Cel `$(JAR)` jest wildcardem `matsim-berlin-*.jar` (`Makefile:3,21`) i nie zależy od źródeł, więc zmiana kodu nie unieważnia artefaktów.
  - (c) Etapy kalibracyjne przerywają DAG. `berlin-$V-10pct.plans.xml.gz` nie ma prerekwizytów i czyta ręcznie wskazany wynik biegu `mode-choice-10pct-baseline/runs/008/...` (`Makefile:330-344`). `experienced_plans` czyta `output/exp-*` z ręcznych biegów (`Makefile:264-266`, komentarz „This file requires eval runs”).
  - (d) Niektóre cele tworzą pliki uboczne, których Make nie zna jako celów, np. `-network-ft.csv.gz` i `-network-linkGeometries.csv` (`Makefile:110,145`).
- **Dlaczego:** Make daje za darmo inkrementalność opartą na mtime i częściowe przebudowy (`make <plik>`). Pętle kalibracyjne na klastrze nie mieszczą się w DAG, dlatego zostały ręcznymi „szwami” *(wnioskowane)*.
- **Czy dotyczy nas?** Częściowo. Wzorzec „jeden plik = jeden etap, wersja w ścieżce, jedno CLI z subkomendami” jest dobry. Trzeba jednak unikać zapisu w miejscu i wprost modelować zależność od wersji kodu oraz ręczne etapy kalibracji, które tutaj się rozjeżdżają.

**Uruchamianie wsadowe na klastrze skryptami shell**
- **Zaobserwowane:** `src/main/sh/start.sh` buduje nazwę biegu z argumentów i wysyła `sbatch job.sh`. `runEval.sh` uruchamia `python -m matsim.calibration run-simulations --worker-id $idx --workers $total` (SLURM array). `capacity/runSUMO.sh` dzieli pracę przez `--runner-index/--runner-total` (SGE `-t`).
- **Dlaczego:** kalibracja i symulacje referencyjne SUMO wymagają setek niezależnych biegów *(wnioskowane)*.
- **Czy dotyczy nas?** Częściowo. Idea sharding przez indeks workera jest prosta i przenośna.

### 2. Konfiguracja i wiele scenariuszy

**Config XML per wersja plus nadpisania z CLI, bez jednego źródła prawdy dla pipeline'u**
- **Zaobserwowane:** każda wersja ma własny `input/vX.Y/berlin-vX.Y.config.xml`, a do tego opcjonalnie `-drt-config.xml` i `-vehicleTypes.xml`. Wersja istnieje w czterech miejscach: `V` w Makefile (`Makefile:4`), `VERSION = "6.4"` (`OpenBerlinScenario.java:42`), `<version>6.4</version>` w `pom.xml:18` i nazwa katalogu. `OpenBerlinScenario` wybiera config z `VERSION` (`OpenBerlinScenario.java:61`). Warianty wyrażone są jako: flagi CLI (`--1pct`, `--plan-selector`, `--config:qsim.numberOfThreads ...`), osobne klasy run (`OpenBerlinDrtScenario`, `run/policies/`) oraz pliki YAML z parametrami scoringu (`input/v6.3/params/*.yaml`, podawane jako `--yaml`, patrz `sh/runEval.sh`). Parametry `Makefile` (CRS, daty GTFS, ścieżki) są zaszyte w recepturach.
- **Dlaczego:** `MATSimApplication` robi z configu MATSim bazę, a z CLI warstwę nadpisań *(wnioskowane)*.
- **Czy dotyczy nas?** Częściowo. Rozproszenie wersji po czterech miejscach to antywzorzec do uniknięcia, bo wystarczy jedna stała. Nadpisania configu z CLI i YAML-e wariantów warto przejąć.

### 3. Układ danych i duże pliki

**W git tylko małe wejścia; duże w SVN (public/shared) jako katalogi-rodzeństwo**
- **Zaobserwowane:** README mówi wprost: „large files ... reside at https://svn.vsp.tu-berlin.de/repos/public-svn/...” (`README.md:33`), bo git lfs nie trafia do zipa i jest płatny. W git są configi, shapefile obszaru (`input/vX.Y/area/`, `pt-area/`), mappingi JSON/CSV (`input/activity_mapping.json`, `counts_mapping.csv`, `remove-railway.xml`) i zagregowane dane referencyjne (`src/main/resources/mode_share_*.csv`). Makefile zakłada katalogi-rodzeństwo `../shared-svn/...` (dane zamknięte) i `../public-svn/...` (dane publiczne) (`Makefile:8-9`). Configi wskazują wejścia przez **URL** do public-svn (`berlin-v6.4.config.xml:23-39,194`), więc bieg ściąga dane leniwie („MATSim accepts URLs as filenames”, `README.md:104`). Wyniki publikowane są w public-svn i oglądane przez SimWrapper (`README.md:40-43`).
- **`.gitignore`:** blokuje `output/`, `*.gz`, `*.pbf`, `*.jar`, `*.zip`, `env/` (`.gitignore:6-11,187-193,396`). Mimo to 6 plików `.gz` z v6.0–v6.2 (counts, routes-ref) jest wymuszenie dodanych (`git ls-files`). Pułapka: szablon R z toptal ignoruje `docs/` (`.gitignore:483`), a `docs/ScenarioGeneration.md` i tak jest śledzony.
- **Dlaczego:** koszt i ograniczenia LFS, cytat w `README.md:33`.
- **Czy dotyczy nas?** Tak. Model „w git tylko konfiguracja i małe mappingi, wejścia jako zewnętrzne adresy, wyniki na publicznym serwerze plików” pasuje do naszego `.env` ze wspólnym dyskiem. Z tego repo bierzemy też lekcję, żeby nie używać generycznych szablonów `.gitignore`, które przypadkiem blokują `docs/`.

### 4. Proweniencja i wersjonowanie

**Wersja scenariusza = wersja artefaktu Maven; hash commita w nazwie JAR i w manifeście**
- **Zaobserwowane:** `git-commit-id-maven-plugin` w fazie `validate` z `failOnNoGitDirectory=true` (`pom.xml:360-376`). Shade tworzy `${finalName}-${git.commit.id.describe}.jar` w katalogu głównym i wpisuje `SCM-Revision` oraz `Implementation-Version` do manifestu (`pom.xml:389-399`). W praktyce JAR ma nazwę typu `matsim-berlin-6.3-SNAPSHOT-v6.0-158-gcdbb04f.jar` (`src/main/sh/runEval.sh`), więc widać w niej wersję, tag i hash. `CHANGELOG.md` prowadzony jest per wersja scenariusza (6.0 → 6.4), opisuje zmiany *danych i metod* (np. 6.2: sieć przez konwerter SUMO i skalibrowana prędkość swobodna, `CHANGELOG.md:41-46`) i patche (6.1.1: poprawione ASC, „All input files remain the same”). Stare wersje zostają w `input/v6.0..v6.3` razem z configami, a kod v5 w pakiecie `org.matsim.legacy`. Release: `publish.yaml` przy utworzeniu GitHub release uruchamia `mvn deploy`, a `jitpack.yml` pozwala użyć repo jako zależności. Nie znaleziono pliku `CITATION.cff`; README odsyła do DOI artykułu (`README.md:11`).
- **Dlaczego:** użytkownik musi wiedzieć, jakim kodem wygenerowano i uruchomiono dany scenariusz *(wnioskowane)*.
- **Czy dotyczy nas?** Tak. Hash commita (`git describe`) w nazwie artefaktu i w metadanych plus changelog opisujący zmiany danych i metod, a nie kodu, to dokładnie to, czego potrzebujemy dla wersjonowanego artefaktu popytu.

### 5. Źródło prawdy dla sieci

**Kanoniczna jest sieć SUMO (`sumo.net.xml`), a sieć MATSim jest z niej pochodną**
- **Zaobserwowane, kroki (`Makefile:60-120`):**
  1. **OSM → `network.osm`** (osmosis): `brandenburg-230101.osm.pbf` jest filtrowany dwa razy. Wersja *detailed* obejmuje drogi do `residential/living_street/unclassified/cycleway` i `bicycle=designated`, przyciętą do `input/v6.4/area/area.poly`. Wersja *coarse* obejmuje `motorway…tertiary` dla całego landu. Obie są scalane (`--merge`) z transformacją tagów `input/remove-railway.xml`.
  2. **`netconvert` → `input/sumo.net.xml`** (`Makefile:83-95`). Opcje:
     - geometria: `--geometry.remove`, `--ramps.guess --ramps.no-split`, `--roundabouts.guess`, `--junctions.join`, `--junctions.corner-detail 5`, `--remove-edges.isolated`;
     - typemapy: `osmNetconvert.typ.xml` i `osmNetconvertUrbanDe.typ.xml`;
     - sygnalizacja: `--tls.guess-signals true --tls.discard-simple --tls.join --tls.default-type actuated`;
     - łącza wewnętrzne: `--no-internal-links`;
     - vClass: `--keep-edges.by-vclass passenger,truck,bicycle` oraz `--remove-edges.by-vclass hov,tram,rail,rail_urban,rail_fast,pedestrian`;
     - `--osm.lane-access false --osm.bike-access false`;
     - atrybuty: `--output.original-names --output.street-names`, `--osm.all-attributes` oraz `--osm.extra-attributes` (smoothness, surface, crossing, tunnel, traffic_sign, bus:lanes*, cycleway*, bicycle);
     - projekcja: `--proj` UTM 32 (odpowiada `EPSG:25832`).
  3. **`prepare network-from-sumo`** z `--target-crs EPSG:25832 --lane-restrictions REDUCE_CAR_LANES` (`Makefile:99`). Klasa `CreateNetworkFromSumo` pochodzi z `contrib/application`, nie z tego repo (`RunOpenBerlinCalibration.java:91`). Z plików ubocznych wynika, że konwerter oprócz sieci zapisuje `berlin-$V-network-ft.csv.gz` (cechy krawędzi dla modeli, `Makefile:110`) i `berlin-$V-network-linkGeometries.csv` (pełne kształty SUMO, `Makefile:145`) *(wnioskowane z użycia; kod konwertera poza repo)*.
  4. `clean-network --modes car,bike,ride,truck --remove-turn-restrictions`, potem `reproject-network --mode truck=freight` (to samo CRS, służy tylko do mapowania trybów).
  5. `apply-network-params freespeed capacity --model org.matsim.prepare.network.BerlinNetworkParams` z cechami z `-ft.csv.gz`. Potem drugi przebieg `capacity` z modelem HBS (`HBSNetworkParams`) tylko dla `residential,living_street`, z `--capacity-bounds 0.3 --decrease-only` (`Makefile:107-120`).
  6. PT doklejany przez `transit-from-gtfs` z `--pseudo-network withLoopLinks` (`Makefile:122-139`).
- **Spójność ID:** MATSim link ID = SUMO edge ID, a MATSim node ID = SUMO junction ID. To zachowanie `network-from-sumo`, którego kodu nie ma w repo, ale wynika ono z trzech niezależnych użyć w tym repo:
  - (a) `apply-network-params` łączy CSV cech (kolumna `edgeId` w pipeline `capacity/features.py:211`) z linkami MATSim;
  - (b) `CreateCountsFromVMZ` buduje indeks przestrzenny linków z `NetworkIndex.readGeometriesFromSumo(linkGeometries.csv)` (`CreateCountsFromVMZ.java:112`), czyli kształty SUMO są przenoszone jako plik boczny kluczowany ID linku;
  - (c) `routes-ref.csv.gz` ma kolumny `from_node,to_node` z ID typu OSM node (`158846810,3941755338`), które `TravelTimeComparison` szuka bezpośrednio przez `network.getNodes().get(Id.createNodeId(...))` (`TravelTimeComparison.java:122-123`). Wynika z tego, że ID węzłów OSM przechodzą przez netconvert do sieci MATSim *(wnioskowane)*. Węzły złączone przez `--junctions.join` dostają w SUMO ID klastrów, więc tam ta tożsamość się łamie *(wnioskowane)*.
- **Dlaczego:** CHANGELOG 6.2: „The network conversion now uses a SUMO converter, a microscopic representation of the network will be available as well” (`CHANGELOG.md:45`). Jedna sieć SUMO służy więc jako źródło dla MATSim i dla mikrosymulacji.
- **Czy dotyczy nas?** Tak, bardzo. To bezpośrednio odpowiada na naszą kwestię źródła prawdy: `sumo.net.xml` jako sieć kanoniczna i sieć MATSim wyprowadzona z niej 1:1 po ID krawędzi oznaczają, że trasy z MATSim da się odtworzyć w SUMO bez map-matchingu. Zwrócić uwagę: `--no-internal-links`, `--remove-turn-restrictions` w MATSim i `REDUCE_CAR_LANES` oznaczają, że topologia manewrów nie jest identyczna.

### 6. Przekazanie MATSim → SUMO

Nie dotyczy. W repo nie ma eksportu planów ani tras do SUMO (grep `sumo` w `src/main/java`: tylko import sieci i `readGeometriesFromSumo`; brak `matsim_importPlans.py` i `.rou.xml` poza `capacity/`). Kierunek jest odwrotny: SUMO → MATSim (sieć i parametry).

### 7. Kalibracja między narzędziami

**SUMO (mikro) → modele ML → przepustowość i prędkość swobodna linków MATSim**
- **Zaobserwowane, przepływ w `src/main/python/capacity/`:**
  1. **Wejście:** `sumo.net.xml` z kroku 5.2 oraz listy obiektów do symulacji. Dla edges i intersections to plik tekstowy z ID (`run_edges.py:201-205`, `run_intersections.py:204-213`). Dla routes to CSV z kolumnami `fromEdge,toEdge,geometry,min_capacity,travel_time` (`run_routes.py:102-107`); jego generatora nie ma w repo.
  2. **Cechy sieci:** `features.py --network` parsuje `edge/junction/connection/request` z pliku SUMO i dla każdej krawędzi wylicza m.in. `speed`, `length`, `numLanes`, `changeNumLanes`, `numConns`, `numResponse`, `numFoes`, kierunki `dir_l/r/s`, `dir_exclusive`, `priority` względem sąsiednich krawędzi, `junctionType`, `junctionSize` (`features.py:82-233`). Wynik trafia do `sumo.net-edges.csv.gz` i `-conns.csv.gz` (`features.py:316-317`).
  3. **Trzy rodzaje scenariuszy SUMO** (wszystkie przez TraCI, `step-length 0.2`, sieć wycinana przez `netconvert --keep-edges.in-boundary`, `utils.py:120-153`):
     - **edges** (`run_edges.py`): pojedyncza krawędź, popyt z teoretycznego `capacity_estimate` (odstęp czasowy 1.2 s, długość 7 m, ×0.9×liczba pasów, `run_edges.py:25-30,129`). Symulacja 600 s dla skal `--scale` 1.0–2.05 co 0.05 (`:163-174`), detektory E1 na każdym pasie, przepływ z pominięciem pierwszych 60 s (`:82-111`).
     - **intersections** (`run_intersections.py`): dla każdej pary (from, to) na skrzyżowaniu (bez zawracania) potok 5000 veh/h na badanej relacji i 200 veh/h na pozostałych, 1800 s (`:40-50,166`). Przepustowość to maks. przepływ na detektorach krawędzi wlotowej. Krótkie wloty (<30 m) wydłużane są o krawędź poprzedzającą (`:139-145`).
     - **routes** (`run_routes.py`): wycięty poligon trasy, prawie swobodny potok o natężeniu 0.3×`min_capacity`, tło z `randomTrips.py` (kopia narzędzia SUMO, EPL-2.0/GPL-2.0), `edgeData` przez 1 h + czas przejazdu (`:101-113`). Wynik to `speedRelative` per krawędź (`:72`).
     - **Warianty floty:** `--scenario base|sst|st|mt|lt` z udziałem pojazdów autonomicznych rosnącym od 1% do 80–95% (`utils.py:40-73`). W produkcji używany jest `base` (`script_model.py:27-28`, `runSUMO.sh`).
  4. **Agregacja:** `features.py <mode> --input` zapisuje `result_{edges,intersections,routes}_<scenario>.csv` (`features.py:236-334`).
  5. **Modele:** `script_model.py` łączy wyniki z cechami. Cele to `capacity_<junctionType>` (przepustowość na pas) i `speedRelative_<junctionType>` dla `priority`, `right_before_left`, `traffic_light` (`features.py:13-37`). Dane są przycinane o 2.5% z każdej strony, odrzucane są outliery wg IQR i długości ≥500 m (`features.py:40-69`). Kandydaci: XGB, XGBRF, RF, ExtraTrees, LGBM, DecisionTree, PassiveAggressive, mean (`script_model.py:111-134`). Hiperparametry dobiera Optuna (TPE, seed 42, 150 prób, 6-fold KFold, MAE) (`:107-201`). Zwykle wygrywa XGBRegressor o głębokości 4 i 30 drzewach (komentarz `:230-296`).
  6. **Powrót do Javy przez generację kodu:** `m2cgen` eksportuje drzewa do źródeł Java i Python (`models.py:394-471`). `replace_params` wyciąga liczby z liści do tablicy `params`, a skalowanie cech zostaje wpisane w `getData` (`models.py:485-533`). Wygenerowane klasy są commitowane jako `src/main/java/org/matsim/prepare/network/BerlinNetworkParams_{capacity,speedRelative}_{priority,right_before_left,traffic_light}.java` (ok. 10 tys. linii, nagłówek „Generated model, do not modify”, `DEFAULT_PARAMS`). `BerlinNetworkParams` wybiera predyktor po `junctionType` (`BerlinNetworkParams.java:9-26`). W Makefile używa ich `apply-network-params` (`Makefile:107-111`).
  7. **Druga pętla, kalibracja prędkości swobodnej względem danych rzeczywistych:** `runOPT.sh` uruchamia w Javie `org.matsim.prepare.network.FreeSpeedOptimizer` jako serwer HTTP na :9090. Klasy nie ma w tym repo, jest w matsim-application *(wnioskowane)*. `opt_freespeed.py` w JAX/optax (Adam, 1000 kroków) wysyła wektor liści drzew, odbiera od Javy błąd (MSE/RMSE) i próbki `x`/`yTrue`, liczy gradient `batch_loss` wygenerowanej wersji Pythonowej modelu i aktualizuje liście (`opt_freespeed.py:28-121`). Struktura drzewa zostaje zamrożona, stroi się tylko wartości w liściach, a referencją są czasy przejazdu `routes-ref.csv.gz`. Odpowiada to artykułowi „Road network free flow speed estimation using microscopic simulation and point-to-point travel times” (`CHANGELOG.md:46`).
- **Rozjazd kodu:** wygenerowane klasy w repo są nowsze niż skrypty Pythona. Oczekują cech `snake_case` (`num_lanes`, `junction_inc_lanes`, `is_primary_or_higher`, `num_left`...), a `features.py` wytwarza `camelCase` (`numLanes`...). `capacity_traffic_light` implementuje już interfejs `Predictor` z mnożnikiem 1.5/1.3 dla dróg primary/secondary („reference simulation is performed with equal green split”, `BerlinNetworkParams_capacity_traffic_light.java:58-66`), a pozostałe klasy `FeatureRegressor`. Aktualny pipeline leży w `matsim-python-tools`: `matsim/scenariogen/network/features.py`, `run_train_model.py`, `ref_model/germany/*.py` (grep po `junction_inc_lanes`). Katalog `capacity/` jest tu reliktem wersji 6.2.
- **Środowisko:** SUMO 1.17.0 budowane ze źródeł razem z xerces/fox/gdal/proj/sqlite (`capacity/Makefile`), co przypina wersję symulatora referencyjnego. `setup.sh` używa venv i `pip install` bez pinów.
- **Dlaczego:** heurystyki przepustowości po typie drogi słabo oddają miejskie skrzyżowania. Mikrosymulacja daje „dane” o przepustowości per konfiguracja skrzyżowania, które da się uogólnić na całą sieć *(wnioskowane; CHANGELOG 6.2 wskazuje artykuł)*.
- **Czy dotyczy nas?** Tak. Mamy odwrotny kierunek (MATSim → SUMO), ale ten mechanizm rozwiązuje nasz problem spójności: jeśli MATSim ma dawać realistyczną równowagę dla SUMO, jego przepustowości powinny pochodzić z tego samego modelu sieci. Model można wytrenować raz, np. z `matsim-python-tools` z referencyjnym modelem `germany`, a potem stroić na danych GPS prędkości. Wniosek organizacyjny: wygenerowany kod modelu trzymać w jednym miejscu razem z wersją cech, żeby uniknąć rozjazdu `camelCase`/`snake_case` widocznego tutaj.

**Kalibracja popytu (MATSim ↔ dane ankietowe, liczniki)**
- **Zaobserwowane:** `calibrate.py`, `calibrate_dist.py` i `calibrate_groups.py` używają `matsim.calibration.create_calibration(... jar, config, args="--10pct" ...)` z `matsim-tools` (instalacja z gałęzi git, `sh/setup.sh`). Cele modal splitu pochodzą z SrV 2018 (`calibrate.py:17-26`). Do liczników służy `RunCountOptimization` (OptaPlanner, `prepare/opt/`), który wybiera jeden plan per agent z `experienced_plans` pod liczniki VMZ (`Makefile:275-289`). Alternatywą jest cadyts (`Makefile:291-300`).
- **Czy dotyczy nas?** Częściowo. Wzorzec „Python steruje JAR-em przez CLI i czyta CSV z wyjścia” to dla nas gotowy przepis na kalibrację ASC względem KBR.

### 8. Próbka / skalowanie

**Próbka zakodowana w nazwie pliku jako `-Npct`; flow/storage/counts skalowane z flagi CLI**
- **Zaobserwowane:**
  - Nazwy plików: `berlin-v6.4-10pct.plans.xml.gz`, `-25pct`, `-3pct`, `-1pct`, `-100pct`, katalog wyjścia `output/berlin-v6.4-10pct` (`berlin-v6.4.config.xml:12,26`).
  - Config bazowy jest 10pct: `flowCapacityFactor=0.1`, `storageCapacityFactor=0.1`, `countsScaleFactor=0.1` (`:54-55,195`).
  - `SampleOptions(10, 25, 3, 1)` w `OpenBerlinScenario.java:53` daje flagi `--10pct/--25pct/--3pct/--1pct`. Po ustawieniu flagi `flowCapFactor = storageCapFactor = countsScaleFactor = sample`, a `sample.adjustName(...)` podmienia `Npct` w runId, katalogu wyjścia i ścieżce planów (`OpenBerlinScenario.java:73-86`).
  - W kalibracji domyślna próbka to 25pct (`SampleOptions(25, 10, 3, 1)`, `RunOpenBerlinCalibration.java:113`). Z `--all-car` współczynniki mnożone są przez `CAR_FACTOR=5` (ok. 20% udziału auta, `:103-106,190-197`). Jest też `--scale-factor 1.5` na zatory w fazie wczesnej.
  - Generacja: populacja powstaje w 25pct (`--sample` domyślnie, 100pct tylko dla Berlina przez `--sample 1.0`, `Makefile:157-171`). Mniejsze próbki tworzy `downsample-population --sample-size 0.25 --samples 0.1 0.03 0.01 0.001` (`Makefile:324-328`). Wersja 10pct wydawana jest z biegu kalibracyjnego, a 3pct z *innego* biegu przez `$(subst 10pct,3pct,$@)` (`Makefile:330-344`).
  - Test CI biega na `--1pct` przez 2 iteracje (`RunOpenBerlinScenarioTest.java`).
  - Niespójność: config v6.2 wskazuje plany `-25pct` przy `flowCapacityFactor=0.1` (`berlin-v6.2.config.xml:26,59`), co działa tylko dlatego, że `adjustName` podmienia nazwę po podaniu flagi.
- **Dlaczego:** jedna flaga ustawia wszystkie powiązane współczynniki naraz, więc nie da się ich rozsynchronizować *(wnioskowane)*.
- **Czy dotyczy nas?** Tak. Nasz artefakt popytu dla SUMO musi nieść próbkę w nazwie i metadanych. Po stronie SUMO odpowiednikiem jest `--scale` na trasach, a nie zmiana przepustowości, więc kontrakt powinien jawnie podawać `sample_size`.

### 9. Testy i determinizm

**Testy integracyjne na małej próbce; brak testów pipeline'u**
- **Zaobserwowane:** `src/test/java/...` zawiera `RunOpenBerlinScenarioTest` (1pct, 2 iteracje, sprawdza tylko kod wyjścia 0; wejścia ściąga z SVN przez URL), test DRT, `ScoreCalculatorTest`, `FacilityIndexTest` i `DistanceGroupModeUtilityParametersTest`. CI (`build.yaml`) uruchamia checkstyle, `enforcer:enforce`, `mvn test` z jacoco oraz package i uploaduje JAR. Seedy są jawne w wybranych miejscach (`activity-sampling --seed 1`, `Makefile:189`; Optuna `seed=42`; `Random(42)`). Brak testów determinizmu, testów kodu Pythona i walidacji schematów. `test/input/` zawiera stare fixtury v5.5.
- **Czy dotyczy nas?** Częściowo. Smoke test „najmniejsza próbka, 2 iteracje” w CI jest tani i wart przejęcia, ale u nas wejścia muszą być lokalne i syntetyczne, bo dane zamknięte odpadają.

### 10. Dane zastrzeżone i licencje

**Dwie licencje wg rodzaju pliku; dane zamknięte fizycznie w innym repozytorium (shared-svn)**
- **Zaobserwowane:** README: kod w `src` (typowo `*.java`) na GPLv2, a „MATSim input files, output files, analysis data and visualizations” na CC BY 4.0 (`README.md:22-29`). `LICENSE` to GPL. Surowe dane zastrzeżone (mikrodane ankiety SrV, dane netcheck/visitation, liczniki VMZ w xlsx, regionalstatistik) leżą w `../shared-svn/...`, do którego skrypty mają zaszyte ścieżki (`extract_ref_data.py:35`, `extract_visitations.py:8`, `Makefile:8,143`). Do git trafiają tylko agregaty: `src/main/resources/mode_share_*.csv` oraz shares/odległości per grupa. Tabele osób/aktywności z SrV (`src/main/python/table-persons.csv`, `table-activities.csv`, używane w `Makefile:189-196`) nie są commitowane. Powstają lokalnie przez `extract_population_data.py` (`:35-47`). Pliki HBEFA są szyfrowane i odszyfrowywane zmienną `MATSIM_DECRYPTION_PASSWORD` („ask VSP for access”, `OpenBerlinScenario.java:45-50`), więc nawet w publicznym SVN leżą bezpiecznie. `randomTrips.py` to kopia z SUMO z własnym nagłówkiem EPL-2.0/GPL-2.0.
- **Dlaczego:** redystrybucja SrV i HBEFA jest licencyjnie zabroniona, a agregaty już nie *(wnioskowane)*.
- **Czy dotyczy nas?** Tak. Mapuje się to 1:1: raw KBR, OD z sieci komórkowej i ZDiUM zostają na dysku zamkniętym, do git idą tylko agregaty referencyjne (np. modal split per grupa), a wygenerowana populacja syntetyczna może być publikowana. Warto rozważyć też szyfrowane artefakty z kluczem w env dla danych, które muszą krążyć z przebiegami.

### 11. Runtime Python ↔ Java

**Java i Python w jednym repo; JAR budowany lokalnie, Python woła go przez CLI lub HTTP**
- **Zaobserwowane:**
  - Budowa: `./mvnw clean package -DskipTests=true` (`README.md:95`). Maven wrapper jest w repo (`.mvn/wrapper`). Shade tworzy fat-JAR w katalogu głównym z `Main-Class=org.matsim.run.RunOpenBerlinScenario` (`pom.xml:28,389-399`).
  - Przypięcie MATSim: `parent matsim-all 2025.0-PR3653`, czyli release oznaczony numerem PR z komentarzem „snapshot == not recommended: rather use PR-labelled release” (`pom.xml:4-13`). Wszystkie contriby dziedziczą `${project.parent.version}` (`pom.xml:26`, „Don't modify this, change the parent version instead!”). Zależności z JitPack przypięte są do skrótu commita (`gtfs2matsim 45689bf834`, `opt-drt 0d94df59ec`, `pt-extensions b39f03f2f0`). JDK 21 wymagany przez `maven.compiler.release` i CI.
  - Wywołania z Pythona:
    - (a) `matsim.calibration.create_calibration("…jar", config, args=…, jvm_args=…)` z `matsim-tools` (`calibrate.py:39-48`) i `python -m matsim.calibration run-simulations --jar …` (`sh/runEval.sh`);
    - (b) `runOPT.sh` startuje `java -cp jar …FreeSpeedOptimizer` w tle, czeka na port 9090 i uruchamia optymalizator Pythona, który rozmawia z Javą przez HTTP JSON (`runOPT.sh:23-36`, `opt_freespeed.py:21,28-36`);
    - (c) Make wywołuje JAR bezpośrednio (`Makefile:17`).
  - W skryptach nazwa JAR-a jest zaszyta i niespójna: `matsim-berlin-6.x-SNAPSHOT.jar`, `matsim-berlin-6.1-SNAPSHOT.jar`, `…-v6.0-158-gcdbb04f.jar`. Środowisko Pythona to venv i `pip` bez lockfile (`sh/setup.sh`, `capacity/setup.sh`, `python3.9`).
- **Dlaczego:** JAR jest jedyną jednostką wykonywalną, a Python orkiestruje i analizuje *(wnioskowane)*.
- **Czy dotyczy nas?** Tak. Mamy tę samą architekturę. Do przejęcia: przypięcie wersji MATSim w jednym miejscu (parent POM) i JAR z `git describe` w nazwie. Do poprawy: ścieżka/wersja JAR-a jako jedna zmienna konfiguracyjna czytana przez wszystkie wrappery Pythona (np. w `.env` lub w pakiecie uv), a nie zaszyta w każdym skrypcie.

### 12. Pakietowanie

**Python nie jest pakietem**
- **Zaobserwowane:** `src/main/python/` nie ma `pyproject.toml`, `setup.py` ani `requirements.txt`. Są to skrypty z blokami `#%%` i importy względne (`from features import ...`, `from gen_code import ...`). Współdzielona logika jest w zewnętrznej bibliotece `matsim-tools` (matsim-python-tools). Projekt Java to pojedynczy moduł Maven.
- **Czy dotyczy nas?** Częściowo, jako antywzorzec. U nas logikę wielokrotnego użytku (cechy sieci, wrapper JAR-a) lepiej umieścić jako członka workspace'u uv (biblioteka), a skrypty jednorazowe jako aplikację z entry points.

### 13. Dokumentacja

**Skąpa i częściowo zewnętrzna**
- **Zaobserwowane:** `docs/ScenarioGeneration.md` jest krótki, z „TODO: Add more details on the calibration process” i pustą sekcją „Mode calibration”. README odsyła do wewnętrznego Google Doc (`README.md:126-129`). Dokumentację przejmuje w praktyce `CHANGELOG.md` (per wersja), komentarze w Makefile („Same OSM version as reference visitations”, `Makefile:28`) i komentarze w configu kalibracji. W katalogach `input/vX.Y/` nie ma notatek.
- **Czy dotyczy nas?** Częściowo. Changelog per wersja scenariusza to dobry, tani nośnik wiedzy. Resztę mamy lepiej zaplanowaną w `docs/dane/` i `docs/decyzje/`.

---

## Najważniejsze dla nas

1. **Sieć SUMO jako kanoniczna, sieć MATSim jako pochodna z tymi samymi ID** (`OSM → osmosis → netconvert → network-from-sumo`, `Makefile:60-120`). Kształty SUMO idą jako plik boczny `linkGeometries.csv`, a cechy jako `network-ft.csv.gz`. To gotowy wzorzec dla naszego kierunku MATSim → SUMO: trasy MATSim mapują się 1:1 na krawędzie SUMO. Uwaga na `--junctions.join`, `--no-internal-links` i `--remove-turn-restrictions`.
2. **Przepustowość i prędkość swobodna MATSim wyuczone z mikrosymulacji SUMO**: scenariusze edge, intersection i route w TraCI, drzewa XGB per `junctionType`, eksport przez m2cgen do Javy, a potem strojenie liści gradientem względem rzeczywistych czasów przejazdu. W tym repo kod Pythona jest przestarzały; aktualny leży w `matsim-python-tools` (`matsim/scenariogen/network/`), tam też warto zajrzeć.
3. **Próbka w nazwie pliku (`-Npct`) i jedna flaga CLI ustawiająca flow, storage i counts oraz podmieniająca ścieżki** (`OpenBerlinScenario.java:53,73-86`). Wzór dla metadanych i nazewnictwa naszego artefaktu popytu.
4. **Proweniencja:** `git describe` w nazwie JAR i manifeście (`pom.xml:360-399`), wersja scenariusza = wersja Maven = katalog `input/vX.Y` oraz changelog opisujący zmiany danych i metod. Minus: wersja powtórzona w czterech miejscach.
5. **Dane:** w git tylko config, mappingi i agregaty. Duże publiczne pliki są na serwerze plików pod URL-ami, zamknięte w osobnym repozytorium danych (shared-svn), a HBEFA szyfrowane z kluczem w env. Licencje rozdzielone wg rodzaju pliku (GPL dla kodu, CC BY dla danych).
6. **Makefile jako DAG jest dobry, ale ma dziury:** zapis w miejscu bez `.DELETE_ON_ERROR`, JAR nie jest zależnością i ręczne etapy kalibracji poza DAG. U nas trzeba to rozwiązać jawnie.

## Nie znaleziono

- Kodu `network-from-sumo`/`CreateNetworkFromSumo`, `ApplyNetworkParams`, `FreeSpeedOptimizer`: nie ma ich w repo (są w `contrib/application` matsim-libs, a tego nie sklonowano). Zachowanie ID wywnioskowane z użyć (`CreateCountsFromVMZ.java:112`, `TravelTimeComparison.java:122-123`, `Makefile:110`).
- Generatora pliku wejściowego `routes` (CSV z `min_capacity`, `travel_time`, `geometry`) i list edges/intersections dla `capacity/`: szukano w `src/main/python/**`, Makefile i skryptach sh.
- Eksportu MATSim → SUMO (`matsim_importPlans.py`, `.rou.xml`): grep w całym `src`.
- `CITATION.cff`, tagów/hasha commita zapisywanego do katalogu wyjściowego biegu (poza manifestem JAR), lockfile'ów Pythona, testów determinizmu i walidacji schematów.
- Historii commitów (płytki klon, 1 commit).
