# matsim-python-tools

- **Repo:** `matsim-vsp/matsim-python-tools` (dystrybucja PyPI: `matsim-tools`)
- **Commit:** `2c37998a17bd437f58450ad4be51c5c9eb9b47bd` (z `MANIFEST.tsv`, status `full`/`ok`, pobrano 2026-06-16)
- **Częściowość klonu:** drzewo plików jest kompletne (116 plików), ale historia git jest płytka: `git log` pokazuje 1 commit (merge PR #40), brak tagów. Wniosków o historii (np. czy kiedyś był plik `LICENSE`) nie da się wyciągnąć z klonu.

**Czym jest to repo.** Mała, „oficjalna” biblioteka VSP (TU Berlin) do czytania wyników MATSim w pandas: strumieniowe czytniki XML dla sieci, planów, zdarzeń (XML / protobuf / ndjson), pojazdów, obiektów (facilities) i gospodarstw domowych, plus proste strumieniowe *writery* XML (populacja, gospodarstwa, facilities). Obok rdzenia (~1000 linii w `matsim/*.py`) siedzą trzy znacznie większe, luźno powiązane podpakiety z osobnymi extras: `calibration` (kalibracja ASC przez Optuna, uruchamia JAR scenariusza jako subprocess), `scenariogen` (adaptery ankiet mobilności SrV/MiD/…, oraz pipeline, który mikrosymulacją **SUMO** szacuje przepustowości i prędkości skrzyżowań, trenuje na tym modele ML i generuje z nich kod Java dla MATSim) oraz `viz` (Dash). Sam README ostrzega: „very early stages… You probably shouldn't use this for anything” (`README.md:9`). Wersja 0.0.22 (`VERSION`).

---

### 1. Orkiestracja

**CLI entry points z podkomendami rejestrowanymi przez `METADATA` modułu**
- **Zaobserwowane:** `setup.py:41-47` deklaruje 4 console scripts (`matsim-tools`, `matsim-scenariogen`, `matsim-calibration`, `matsim-viz`). Każdy moduł-krok eksportuje krotkę `METADATA = (nazwa, opis)` oraz funkcje `setup(parser)` / `main(args)`, a dispatcher dokleja je jako subparsery (`matsim/scenariogen/__main__.py:16-20, 31-45`; to samo w `matsim/cli/main.py:14-16`). Moduł z ciężkimi zależnościami (`run_opt_freespeed`, jax) jest dołączany w `try/except`, by brak extras nie psuł całego CLI (`scenariogen/__main__.py:37-41`).
- **Dlaczego:** jeden punkt wejścia, a kroki niezależne i uruchamialne osobno *(wnioskowane)*.
- **Czy dotyczy nas?** częściowo — wzorzec `METADATA + setup + main` to tani sposób na CLI per pakiet workspace'u bez frameworka; brak w nim cache/DAG.

**Brak cache; „wznowienie” przez indeksy i istniejące katalogi**
- **Zaobserwowane:** kroki SUMO dzielą pracę na shardy `--runner-index/--runner-total` → `from_index/to_index` (`scenariogen/network/utils.py:23-28, 77-86`), uruchamiane jako tablice zadań SGE/SLURM (`runSUMO.sh`, `runOPT.sh`). `calibration/run_simulations.py:173-175` pomija run, jeśli katalog `eval-runs/NNN` istnieje (bez sprawdzania kompletności). Kalibracja trzyma stan w SQLite Optuny (`calibration/__init__.py:138-142`, `load_if_exists=True` w `:150-156`), więc wznowienie = ponowne wywołanie `study.optimize`.
- **Dlaczego:** długie, embarrassingly-parallel batch'e na klastrze *(wnioskowane)*.
- **Czy dotyczy nas?** częściowo — sharding po indeksie jest przydatny dla kalibracji/ewaluacji; „katalog istnieje ⇒ gotowe” to anty-wzorzec (brak markera ukończenia).

### 2. Konfiguracja i wiele scenariuszy

**Scenariusz = config MATSim + nadpisania CLI; warianty floty SUMO jako twardo zakodowane presety**
- **Zaobserwowane:** kalibracja zapisuje parametry trialu do `params/runN.yaml` i woła JAR z `--config … --yaml … --output runs/NNN --runId` (`calibration/__init__.py:180-204`); chaining planów przez dopięcie `--config:plans.inputPlansFile=<poprzedni output_plans>` (`:209-226`). W `scenariogen` warianty „base/sst/st/mt/lt” (udział AV, parametry Krauss) to stringi XML `vType` w `vehicle_parameter()` (`scenariogen/network/utils.py:41-74`), wybierane `--scenario` (`:21-22`).
- **Dlaczego:** MATSim-application przyjmuje config + YAML + `--config:` overrides; scenariusze floty z projektu o AV *(wnioskowane)*.
- **Czy dotyczy nas?** częściowo — wzorzec „jeden config + plik nadpisań per przebieg zapisany obok wyniku” warto przejąć; presety zaszyte w kodzie — nie.

### 3. Układ danych i duże pliki

**W git tylko małe fixtury testowe; dane wejściowe wyłącznie ze ścieżek użytkownika**
- **Zaobserwowane:** jedyne dane w repo to `tests/*.xml.gz|*.pb.gz|*.ndjson.gz` (0,4–21 KB każdy) i dwa PNG w `docs/`. `.gitignore` to generyczny szablon toptal (Python/IDE), bez reguł na dane. Skrypty zakładają lokalne katalogi roboczne (`runs/`, `params/`, `eval-runs/`, `runner*/`, `output-*`) tworzone w CWD (`calibration/__init__.py:159-163`, `run_simulations.py:166`, `utils.py:36-37`). Ankiety czytane z katalogów podanych przez użytkownika (`scenariogen/data/__init__.py:44-116`).
- **Czy dotyczy nas?** tak (jako potwierdzenie) — model „kod w repo, dane z zewnętrznej ścieżki” zgadza się z naszym `.env` + `raw/`; u nas lepiej wymusić to regułami `.gitignore`, czego tu brak.

### 4. Proweniencja i wersjonowanie

**`VERSION` + standard-version + ręczny upload twine; brak proweniencji w wynikach**
- **Zaobserwowane:** wersja w pliku `VERSION` czytanym przez `setup.py:10,14`; `.versionrc` każe `standard-version` bumpować `VERSION`; `Makefile:34-36` (`make version` → `npx standard-version`), `Makefile:29-32` (`make push` → testy, build, `twine upload`). `CHANGELOG.md` generowany, ale od 0.0.12 wpisy są puste (tylko nagłówki z datą). Brak `CITATION.cff`; README prosi o cytowanie doktoratu Rakowa dla kalibracji (`README.md:235-237`). Wyniki kalibracji nie zapisują hasha kodu/JAR — tylko `trial.user_attrs` z `mode_stats` (`calibration/__init__.py:266-270`).
- **Czy dotyczy nas?** częściowo — przykład, jak NIE robić changelogu (puste wpisy); release ręczny z laptopa, bez CI.

### 5. Źródło prawdy dla sieci

**Pipeline SUMO→MATSim zakłada wspólne ID krawędzi (SUMO edge id == MATSim linkId)**
- **Zaobserwowane:** wyniki symulacji SUMO (CSV per edge/junction, kolumny `fromEdgeId`, `edgeId` — `run_intersections.py:99-106`, `run_routes.py:62-82`) są łączone z tabelą cech sieci MATSim po `linkId` (`scenariogen/network/features.py:11-18`). Tabela cech `network-ft.csv.gz` powstaje po stronie Java (`runTrainModel.sh`, `runOPT.sh`: `org.matsim.application.prepare.network…`), nie w tym repo. Sieć SUMO (`sumo.net.xml`, domyślnie `../../../../scenarios/input/sumo.net.xml`, `utils.py:18`) jest więc *wejściem kanonicznym*, z którego MATSim-network jest pochodną z zachowanymi ID *(wnioskowane — konwersja nie jest w tym repo)*.
- **Dlaczego:** bez identyczności ID nie da się przenieść wyników mikrosymulacji na linki MATSim *(wnioskowane)*.
- **Czy dotyczy nas?** tak — to jest dokładnie nasze ryzyko w odwrotnym kierunku (plany MATSim → trasy SUMO): kontrakt artefaktu musi określić mapowanie ID sieci; tu zostało rozwiązane przez wspólną genezę sieci, nie przez tablicę mapującą.

**Czytnik sieci MATSim: dwa DataFrame'y, atrybuty w formie long**
- **Zaobserwowane:** `read_network` zwraca `nodes`, `links` (liczbowe: `length`, `freespeed`, `capacity`, `permlanes`; reszta jak `modes` zostaje stringiem), `node_attrs`/`link_attrs` w formacie (id, name, value) z konwersją tylko Long/Double/Integer, oraz CRS z atrybutu `coordinateReferenceSystem` (`matsim/Network.py:64-145`). `as_geo()` buduje proste odcinki from→to (bez geometrii pośredniej) (`Network.py:27-62`).
- **Czy dotyczy nas?** częściowo — wystarczy do QA sieci MATSim; nie zachowuje kształtów, więc nie nadaje się do porównań geometrycznych z `net.xml` SUMO.

### 6. Przekazanie MATSim → SUMO

**Brak konwertera plans→SUMO; są tylko klocki do odczytu**
- **Zaobserwowane:** repo nie zawiera nic w rodzaju `matsim_importPlans.py`; grep po `rou.xml`/`trips` znajduje wyłącznie syntetyczne `flow`/`randomTrips` do kalibracji (`run_edges.py:34-60`, `run_routes.py:28-44`, `randomTrips.py`). Kierunek przepływu SUMO→MATSim, nie odwrotnie.
- **Czy dotyczy nas?** tak — nie mamy tu gotowego handoffu; biblioteka może być co najwyżej warstwą odczytu po stronie MATSim.

**`plan_reader` (strumień ElementTree) i `plan_reader_dataframe` (5 tabel relacyjnych)**
- **Zaobserwowane:** `plan_reader` streamuje `(person_elem, plan_elem)` przez `iterparse`, emitując też osoby bez planów (`plan, None`) i opcjonalnie tylko wybrane plany (`Plans.py:14-40`). `plan_reader_dataframe` buduje `persons`, `plans`, `activities`, `legs`, `routes` z sztucznymi, rosnącymi ID i kluczami obcymi `person_id/plan_id/leg_id` (`Plans.py:48-173`). Atrybuty `<attribute>` trafiają jako kolumny, ale **zawsze jako tekst** (atrybut `class` ignorowany, `:155-165`); czasy (`end_time`, `dep_time`, `max_dur`, `trav_time`) zostają stringami `HH:MM:SS`; trasa `links` to surowy string ID linków w kolumnie `value` (`:150`) z `start_link`, `end_link`, `distance`, `vehicleRefId` jako atrybutami. Pełnoskalowy fixture: `tests/plans_full.xml.gz` (Berlin, GK4, tryby `car`, `transit_walk`, `non_network_walk`, aktywności `car interaction`, trasy `generic` i `links`).
- **Dlaczego:** analiza w pandas bez pośredniego modelu obiektowego (`README.md:11`).
- **Czy dotyczy nas?** tak — to najbliższy gotowy parser planów w Pythonie; handoff musiałby sam: parsować czasy, rozbijać trasę na listę linków, odfiltrować aktywności `* interaction` (etapy) i mapować `vehicleRefId`→pojazd SUMO.

**Błąd: `selected_plans_only=True` w wersji DataFrame nie odfiltrowuje aktywności/nóg niewybranych planów**
- **Zaobserwowane:** dla niewybranego planu kod robi `continue` na starcie `<plan>`, więc nie zwiększa `current_plan_id`, ale aktywności i nogi tego planu są dalej dopisywane — z `plan_id` poprzedniego planu (`Plans.py:116-141`). Test to utrwala: dla `plans_full.xml.gz` (4 plany, jeden `selected="no"` w linii 253 fixture'u) oczekuje 39 aktywności i 35 nóg niezależnie od flagi (`tests/test_MatsimPlansReader.py:96-103`), a `zcat … | grep -c '<activity'` daje 39 — czyli wszystkie.
- **Czy dotyczy nas?** tak — gdybyśmy użyli tej funkcji do eksportu wybranych planów, przypięlibyśmy obce aktywności do osoby; wersja strumieniowa `plan_reader` działa poprawnie. Najlepiej czytać z `output_plans` zawierającego tylko wybrane plany albo użyć strumienia.

**Czytnik zdarzeń: jeden generator słowników dla XML / protobuf / ndjson**
- **Zaobserwowane:** `event_reader(path, types=…)` wybiera parser po podciągu `.xml`/`.pb`/`.ndjson` w nazwie (`Events.py:36-45`), filtruje po `type`. W XML tylko `time` jest konwertowany na float, reszta atrybutów zostaje stringami (`:63-67`); błędny XML (brak `</events>`) jest połykany z `print` (`:72-77`) — ciche obcięcie. Protobuf mapowany do nazw typów XML przez słownik `MAPPING` (10 typów podstawowych, `:88-92`); inne typy dostają nazwę pola protobuf. Test sprawdza zgodność liczników zdarzeń trzech formatów na tym samym przebiegu (`tests/test_MatsimEventsReader.py:13-34`).
- **Czy dotyczy nas?** tak — gdyby handoff używał zdarzeń (np. rzeczywiste czasy wjazdu `vehicle enters traffic` zamiast `dep_time` z planu), to jest gotowy parser; trzeba jednak zwracać uwagę na ciche obcinanie i brak typowania.

**Writer populacji nie zapisze tras ani linków**
- **Zaobserwowane:** `PopulationWriter.add_activity` przyjmuje `type, x, y, facility_id, start_time, end_time, max_dur, attributes` — bez `link` (`writers.py:155-176`); `add_leg` tylko `mode, dep_time, trav_time, attributes` — bez `<route>` (`:178-195`). Brak escapowania XML (f-stringi). DTD `population_v6`.
- **Czy dotyczy nas?** częściowo — wystarczy do generowania *wejściowych* planów MATSim z ankiety (aktywności z współrzędnymi); nie nadaje się do przepisywania wyjść MATSim z trasami.

### 7. Kalibracja między narzędziami

**SUMO jako „mikroskopowe laboratorium” parametryzujące sieć MATSim**
- **Zaobserwowane:** trzy kroki CLI uruchamiają SUMO przez TraCI na wyciętych fragmentach sieci (`netconvert --keep-edges.in-boundary`, `utils.py:111-144`):
  - `sumo-edges` — pojedyncza krawędź, flow skalowany `--scale` 1.0–2.05, detektory e1 → przepływ nasycenia per pas (`run_edges.py:26-31, 152-192`);
  - `sumo-intersections` — każde skrzyżowanie z listy, każda relacja from→to z tłem na pozostałych relacjach, 1800 s, e1 → przepustowość relacji (`run_intersections.py:113-200`);
  - `sumo-routes` — trasy przy lekkim ruchu + tło z `randomTrips`, `edgeData` → `speedRelative` (`run_routes.py:85-154`).
  Wyniki (CSV) + cechy sieci MATSim → `network-train-model` trenuje regresory (XGBoost/LightGBM/…, Optuna) per typ skrzyżowania (`priority`, `right_before_left`, `traffic_light`) dla `capacity_*` i `speedRelative_*` (`features.py:9-37`, `run_train_model.py:17-29`), a `model_to_java` generuje klasę Java implementującą `org.matsim.application.prepare.Predictor` do wklejenia w scenariusz (`models.py:233-293`). Gotowe modele dla Niemiec są zacommitowane jako wygenerowany Python (`scenariogen/network/ref_model/germany/*.py`, np. nagłówek z hiperparametrami XGB i MAE). Dodatkowo `network-opt-freespeed` optymalizuje (jax/optax) parametry modelu prędkości, odpytując po HTTP serwer Java `FreespeedOptServer` (`run_opt_freespeed.py:27-47`, `runOPT.sh`).
- **Dlaczego:** OSM nie daje przepustowości/prędkości efektywnych; mikrosymulacja daje dane treningowe dla modelu przenośnego między miastami (nazwa `ref_model/germany`, opis `individual.py:1` „not transferable”) *(częściowo wnioskowane)*.
- **Czy dotyczy nas?** tak, mocno — mamy SUMO jako cel i MATSim jako źródło; ten pipeline pokazuje *drugi* kierunek sprzężenia (SUMO kalibruje przepustowości sieci MATSim, w szczególności na skrzyżowaniach z sygnalizacją), co zwiększa zgodność popytu MATSim z tym, co SUMO faktycznie przepuści. Metoda do rozważenia w osobnym issue; kod GPL, więc tylko jako wzorzec.

**Kalibracja ASC trybów względem ankiety**
- **Zaobserwowane:** `create_calibration` (`calibration/__init__.py:98-279`) — Optuna study, własny sampler, każdy trial = pełny run MATSim; błąd liczony z `*.output_trips.csv.gz` + `*.output_persons.csv.gz` (`calibration/analysis.py:63-76`). Cele z ankiety tworzy `scenariogen data-create-ref` (`scenariogen/data/run_create_ref_data.py:15`) na wspólnym schemacie `Household/Person/Trip` (`scenariogen/data/__init__.py:342-417`) z adapterami formatów `srv`, `mid`, `jp_milt` (każdy: `INPUT_FILES`, `is_format`, `read_raw`, `convert`; wykrywanie po nagłówku `_check_header`, `:29-41`; pętla `:44-116`).
- **Czy dotyczy nas?** tak — wzorzec „adapter ankiety → uniwersalny schemat z wagami (`h_weight`, `p_weight`, `t_weight`) → cele kalibracji” pasuje 1:1 do KBR 2018/2024 i `packages/kbr-eda`; KBR byłby kolejnym adapterem (reimplementacja własnymi słowami, nie kopia).

### 8. Próbka / skalowanie

**Skalowanie tylko po stronie SUMO (`--scale`), brak obsługi próbki populacji**
- **Zaobserwowane:** `run_edges.py:167-178` przepuszcza przez tę samą krawędź popyt × {1.00…2.05} przez `traci.load([... "--scale", s])`. W czytnikach MATSim nie ma pojęcia udziału próbki; przykład README używa configu `…-10pct.config.xml` (`README.md:279`) bez dalszej obsługi.
- **Czy dotyczy nas?** nie (poza przypomnieniem, że artefakt musi sam nieść współczynnik próbki).

### 9. Testy i determinizm

**Małe fixtury z prawdziwych przebiegów + round-trip writera bajt-w-bajt**
- **Zaobserwowane:** `tests/` ma po jednym teście na czytnik (sieć, plany, zdarzenia ×3 formaty, pojazdy, facilities, households) na plikach 0,4–21 KB. `test_MatsimPlansWriter.py` czyta `initial_plans.xml.gz`, przepisuje przez `PopulationWriter` i porównuje linie z oryginałem (`:14-40`) — de facto golden-file test formatu wyjściowego (zapisuje `out_*.xml.gz` do CWD). Testy kalibracji są syntetyczne (softmax na losowych użytecznościach, `np.random.seed(0)`, próg błędu 0.03 — `tests/calibration/test_calibration.py:10-56`). Brak testów dla `scenariogen` (SUMO) i `viz`.
- **Determinizm:** `run_simulations.run` nadaje seed `seed + i` przez `--config:global.randomSeed` (`calibration/run_simulations.py:137, 183-186`); test istotności używa `default_rng(seed=4711)` (`:32`). Brak testu, że ten sam seed daje ten sam wynik.
- **Czy dotyczy nas?** tak — golden-file round-trip to dobry wzorzec dla naszego eksportera artefaktu (plans → SUMO `rou.xml`): mały, zacommitowany wejściowy plan + oczekiwany wyjściowy plik. Uwaga: pochodzenie/licencja fixture'ów niedokumentowane (plany wyglądają na Open Berlin Scenario: CRS GK4, strefa `brandenburg` — *wnioskowane*).

**CI testuje tylko rdzeń na jednej wersji Pythona**
- **Zaobserwowane:** `.github/workflows/python-app.yml`: push/PR na `master`, Python 3.10, `pip install -r requirements.txt` (nie `pip install .[extras]`), flake8 tylko błędy krytyczne (E9,F63,F7,F82), `pytest`. `requirements.txt` ≠ `install_requires` (zawiera `geopandas`, `optuna`, `scipy`).
- **Czy dotyczy nas?** częściowo — przestroga: rozjazd `requirements.txt` vs metadane pakietu; `uv` z jednym lockfile tego unika.

### 10. Dane zastrzeżone i licencje

**Licencja: deklarowana tylko w metadanych — GPLv3 w `setup.py`, brak pliku `LICENSE`**
- **Zaobserwowane:** w repo **nie ma** `LICENSE`/`COPYING` (sprawdzone: listing 116 plików; `git log -- LICENSE COPYING` pusty, ale klon płytki). Jedyne deklaracje: `setup.py:21` `license="GPLv3"` oraz classifier `"License :: OSI Approved :: GNU General Public License (GPL)"` (`:23`) — generyczny, bez wersji. Brak nagłówków licencyjnych w plikach źródłowych VSP. Brak rozróżnienia „v3-only” vs „or later”.
- **Wyjątek — plik SUMO wkopiowany do pakietu:** `matsim/scenariogen/network/randomTrips.py` (946 linii) to kopia narzędzia SUMO z nagłówkiem `SPDX-License-Identifier: EPL-2.0 OR GPL-2.0-or-later`, Copyright DLR 2010-2023 (`randomTrips.py:2-12`). Wybór opcji GPL-2.0-or-later czyni go zgodnym z GPLv3, ale pakiet nie dokumentuje tego nigdzie poza nagłówkiem.
- **Dane zastrzeżone:** adaptery ankiet czytają surowe pliki SrV/MiD z katalogu użytkownika i niczego nie commitują; w repo nie ma żadnych danych ankietowych.
- **Czy dotyczy nas?** tak — (a) importowanie `matsim-tools` jako zależności w naszym kodzie oznacza zależność od GPLv3 (w praktyce: nasz pakiet, który go importuje i jest dystrybuowany, powinien być GPL-kompatybilny) — do rozstrzygnięcia w `docs/decyzje/`; (b) sama deklaracja bez tekstu licencji to słaba podstawa prawna — przy cytowaniu w decyzji podać `setup.py:21` i fakt braku pliku; (c) wzorzec „wendorowany plik z własnym SPDX” warto u nas jawnie rejestrować.

### 11. Runtime Python ↔ Java

**Java jako zewnętrzny, ręcznie dostarczony JAR; Python tylko orkiestruje**
- **Zaobserwowane:** kalibracja i `run_simulations` składają komendę `java <jvm_args> -jar <jar> run --config … --yaml … --output … --runId …` i odpalają `subprocess.Popen`, sprawdzając tylko `os.access(jar, R_OK)` (`calibration/__init__.py:144-145, 203-204, 241-257`; `run_simulations.py:158-199`). Hak `custom_cli` pozwala podać inną składnię dla scenariuszy niebędących `matsim-application` (`utils.cli_oldstyle`, `calibration/utils.py:103-110`). `network-opt-freespeed` gada z procesem Java przez HTTP/JSON na losowym wolnym porcie (`runOPT.sh`: `FreespeedOptServer`, pętla `nc -z`). JAR nie jest budowany ani przypinany w tym repo; README wymaga „standalone executable jar… created using the `matsim-application` contrib” (`README.md:240`). Generowany kod Java z modeli ML przenoszony jest do scenariusza ręcznie (`models.py:233-293`).
- **Czy dotyczy nas?** tak — potwierdza model „Java w osobnym repo/artefakcie, Python woła CLI”; brak przypięcia wersji JAR-a (hash, wersja w wyniku) to luka, której u nas nie powinno być.

### 12. Pakietowanie

**Płaski pakiet `matsim/` (nie `src/`), setuptools + `setup.py`, extras jako podsystemy**
- **Zaobserwowane:** brak `pyproject.toml`; `setup.py` z `packages=["matsim"] + find_packages(where="matsim")` (`setup.py:26`), rdzeń zależy od `protobuf>=3.20`, `xopen>=1.7`, `pandas>=2.1` (`:27-31`); extras `calibration` (optuna, shapely, geopandas, scikit-learn), `scenariogen` (sumolib, traci, lxml, optax, xgboost==1.7.1 przypięty przez błąd m2cgen, lightgbm, …), `viz` (dash…) (`:32-39`). `pyyaml` używany w `calibration` (`calibration/__init__.py:18`) nie jest zadeklarowany (przychodzi tranzytywnie z optuna *(wnioskowane)*); `jax` używany w `run_opt_freespeed.py` tylko przez `optax`. Brak `python_requires`.
- **Niespójności:** (1) wygenerowany kod protobuf ma nagłówek „Protobuf Python Version: 5.28.3” i woła `ValidateProtobufRuntimeVersion` (`matsim/pb/Events_pb2.py:5, 9-13`), a `install_requires` pozwala na `protobuf>=3.20` — ponieważ `matsim/__init__.py:1` importuje `Events` → `pb`, `import matsim` na starszym protobuf zawiedzie *(wnioskowane z kodu, nie uruchamiano)*; plików `.proto` brak w repo. (2) `matsim/viz/` nie ma `__init__.py`, więc `find_packages` go nie obejmie, a entry point `matsim.viz:main` (`setup.py:46`) wskazuje na nieistniejący atrybut pakietu *(wnioskowane)*. (3) Zmiana nazw sdist `matsim-tools-*` → `matsim_tools-*` łatana w Makefile (`Makefile:40-45`, zwłaszcza `:43`).
- **Czy dotyczy nas?** tak — rdzeń-biblioteka + ciężkie, opcjonalne podsystemy w extras to czytelne cięcie; u nas lepiej zrobić z nich osobnych członków workspace'u `uv` (np. `matsim-io` lekki, `calibration` osobno), zamiast extras w jednym pakiecie, co uniknie rozjazdów zależności i zepsutych entry pointów.

**Relacja do `pam` (cml-pam)**
- **Zaobserwowane:** brak jakiejkolwiek wzmianki o `pam`/`cml-pam`/Arup (grep po `*.py`, `*.md`, `*.txt`, `*.sh`: 0 trafień); brak zależności. Funkcjonalnie dubluje wąski wycinek pam: odczyt planów MATSim (`Plans.py`) i zapis planów wejściowych (`writers.PopulationWriter`), ale bez modelu obiektowego osoby/planu, bez tras w zapisie i bez walidacji sekwencji aktywność–noga.
- **Czy dotyczy nas?** tak — jeśli do handoffu potrzebny jest pełny round-trip planów z trasami, `matsim-tools` nie wystarczy; `pam` wymaga osobnej analizy (sdist nie pobrany).

### 13. Dokumentacja

**Jeden README z przykładami; brak dokumentacji API i per-scenariusz**
- **Zaobserwowane:** `README.md` (297 linii) to snippety użycia (sieć, zdarzenia, plany, writer, kalibracja); brak katalogu docs poza dwoma obrazkami; brak opisu pipeline'u SUMO poza komentarzami w skryptach `run*.sh`; README „In lieu of real documentation…” (`README.md:22`).
- **Czy dotyczy nas?** nie (antywzorzec do uniknięcia).

---

## Najważniejsze dla nas

1. **Nie ma tu handoffu MATSim→SUMO.** Są tylko czytniki: strumieniowy `plan_reader` (poprawny) i `plan_reader_dataframe` (błędnie dokleja aktywności niewybranych planów przy `selected_plans_only=True`, `Plans.py:116-141`). Trasy `links` wychodzą jako surowy string, czasy jako `HH:MM:SS`, atrybuty bez typów — parsowanie i semantykę musi zapewnić nasz eksporter.
2. **Kierunek odwrotny istnieje i jest wartościowy:** `scenariogen/network` używa SUMO (TraCI, fragmenty sieci, detektory e1/edgeData) do szacowania przepustowości skrzyżowań i prędkości, trenuje z tego modele per typ skrzyżowania i generuje kod Java dla sieci MATSim. Warunkiem jest identyczność ID SUMO edge = MATSim link (`features.py:11-18`). Dla nas: kandydat na kalibrację przepustowości sieci MATSim Wrocławia, tak by popyt był zgodny z tym, co SUMO przepuści.
3. **Licencja:** GPLv3 zadeklarowana wyłącznie w `setup.py:21` (classifier generyczny GPL), **brak pliku LICENSE**; plus wkopiowany plik SUMO `randomTrips.py` na EPL-2.0 OR GPL-2.0-or-later. Import jako zależność wymaga decyzji licencyjnej w `docs/decyzje/`; niczego nie kopiujemy.
4. **Adaptery ankiet → uniwersalny schemat z wagami** (`scenariogen/data/__init__.py:44-116, 342-417`) to gotowy wzorzec do dopisania KBR 2018/2024 jako formatu i wygenerowania celów kalibracji ASC.
5. **Golden-file round-trip test writera** (`tests/test_MatsimPlansWriter.py`) — wzorzec dla testów naszego eksportera artefaktu na małym zacommitowanym fixture'ze.
6. **Pakietowanie jako przestroga:** `setup.py` bez `pyproject`, rozjazd `requirements.txt` vs `install_requires`, protobuf gencode 5.28 vs `protobuf>=3.20`, zepsuty entry point `matsim-viz`, ręczny release twine, puste wpisy changelogu. U nas: osobne pakiety workspace'u zamiast extras, lockfile `uv`, release z CI.

## Nie znaleziono

- **Pliku licencji** (`LICENSE`, `COPYING`, `LICENSE.*`) — listing całego repo; `git log -- LICENSE COPYING` (historia płytka, 1 commit).
- **Konwertera plans/events → SUMO** (`rou.xml`, `trips.xml` z populacji MATSim) — `matsim/`, grep po `rou.xml`, `route`, `sumo`.
- **Writera tras/linków w planach** — `matsim/writers.py`.
- **Odniesień do `pam`/`cml-pam`** — grep po całym repo.
- **Obsługi współczynnika próbki / skalowania przepustowości MATSim** — `matsim/`, `README.md`.
- **Automatycznego release'u na PyPI w CI** — `.github/workflows/` (tylko test), release ręczny przez `Makefile`.
- **Hasha commita / wersji JAR w wynikach kalibracji** — `calibration/__init__.py`, `run_simulations.py`.
- **Plików `.proto`** i informacji, z której wersji MATSim pochodzi schemat protobuf — `matsim/pb/`.
- **Testów `scenariogen` i `viz`** — `tests/`.
- **Dokumentacji pochodzenia fixture'ów testowych** — `tests/`, `README.md`.
- **Wymiar 1–13 bez treści:** brak — każdy wymiar ma przynajmniej jedną obserwację (8 i 13 marginalne).
