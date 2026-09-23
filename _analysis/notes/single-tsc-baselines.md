# single-tsc-baselines (Traffic-Alpha/SingleTSCBaselines)

- **URL:** https://github.com/Traffic-Alpha/SingleTSCBaselines
- **Commit:** `3147aa9aef16e5c78f83a557d2935a41c1fde079` (2026-07-03, wg `MANIFEST.tsv`)
- **Klon:** pełny. Historia to **jeden commit** („Redesign junction flows against signalized capacity”), więc nie da się niczego wyczytać z ewolucji repo. Całość ~12 MB bez `.git`.
- **Poza klonem:** kluczowa zależność `tshub` (TransSimHub) **nie jest** w tym repo ani w `_analysis`/refs. Wszystko, co dzieje się wewnątrz `TshubEnvironment` (flagi SUMO, seed, subskrypcje TraCI, wyliczanie `phase2movements`), opisuję jako *(wnioskowane)* albo „nie znaleziono”.

**Czym jest to repo.** To benchmark sterowania sygnalizacją na **pojedynczym skrzyżowaniu**. Ma wspólną warstwę środowiska gymnasium (`tsc_env/`) nad `tshub`/SUMO, 4 klasyczne kontrolery (FixTime, MaxPressure, Webster, SOTL) i 4 kontrolery RL oparte na Stable-Baselines3 (PressLight, AttendLight, IntelliLight, UniTSA). Zawiera 12 prawdziwych skrzyżowań (Pekin, Chengdu, Tianjin, Hongkong, Songdo, Massy). Każde ma 2 warianty sygnalizacji (`easy`/`normal`) × 5 syntetycznych wzorców popytu, czyli 10 plików `.sumocfg` na skrzyżowanie. Scenariusze SUMO są commitowane do repo i są maleńkie: 6–17 krawędzi na sieć. Dla nas to dobry przykład **konsumenta**: widać dokładnie, czego środowisko RL oczekuje od `.sumocfg`/`.net.xml`/`.rou.xml`. Pokazuje też, czego *nie* wymaga (np. różnorodności popytu między epizodami).

---

### 1. Orkiestracja

**Brak pipeline'u. Są osobne skrypty i bash do batchowania.**
- **Zaobserwowane:**
  - Każde skrzyżowanie ma własny `junction_scenarios/<J>/generate_routes.py`. Ten skrypt woła `tshub.sumo_tools.generate_routes.generate_route` z zahardkodowanym słownikiem `traffic_flow_configs` i zapisuje `routes/<pattern>.rou.xml` (`junction_scenarios/Beijing_Beihuan/generate_routes.py:19-71`).
  - `scripts/redesign_flows.py` **przepisuje źródło** `generate_routes.py`: parsuje je przez `ast`, podmienia blok `traffic_flow_configs`, uruchamia je przez `subprocess`, mierzy wynikowe `.rou.xml` i powtarza, aż trafi w docelowy stopień nasycenia (`scripts/redesign_flows.py:1-17`, `TARGET_X`, `MAX_ITER = 8`).
  - Eksperymenty uruchamia `scripts/run_batch.sh`: pętla po (junction, net, pattern), każdy wariant w **osobnym procesie Pythona**.
  - Nie ma cache'u ani unieważniania. Wyniki trafiają do `results/<algo>/<J>_<env>/trip_info.xml` (`tsc_algos/output_utils.py:16-30`).
- **Dlaczego:** osobne procesy wynikają z tego, że „libsumo jest singletonem na poziomie procesu”. Kolejne środowiska różnych skrzyżowań w jednym procesie gubią subskrypcje lanearea i kończą się `KeyError` (`scripts/run_batch.sh:9-11`).
- **Czy dotyczy nas?** Częściowo. Sama orkiestracja jest dla nas za prymitywna. Ograniczenie libsumo jest jednak twarde: jeśli nasz ewaluator ma przełączać scenariusze Wrocławia, to jeden proces powinien obsługiwać jedną sieć.

### 2. Konfiguracja i wiele scenariuszy

**Scenariusz to para (junction, env_name). Ścieżki są wyprowadzane z konwencji nazw, nie deklarowane.**
- **Zaobserwowane:** `junction_configs/_loader.py:47-88` składa ścieżki z nazw:
  - `sumo_cfg = <root>/junction_scenarios/<J>/<env_name>.sumocfg`
  - `net_file = <root>/junction_scenarios/<J>/networks/<difficulty>.net.xml`, gdzie `difficulty = env_name.split("_")[0]` (linie 69-79)
  - `_PROJECT_ROOT` to katalog nadrzędny `junction_configs` (linia 13). Skrzyżowania są wymienione na zahardkodowanej liście `AVAILABLE_JUNCTIONS` (linie 16-29).
  - Scenariusz nie ma własnego manifestu. Parametry środowiska siedzą w module Pythona `junction_configs/<J>.py` jako dict `JUNCTION = {"tls_id": ..., "<env_name>": {"num_phases", "num_seconds", "fix_phase_durations"}}` (`junction_configs/Beijing_Beihuan.py:10-63`).
  - Warianty zdarzeń specjalnych (wypadki, pojazdy uprzywilejowane) są w tym samym pliku jako `EVENTS` (linia 70+), wybierane flagą `--event_name` i wstrzykiwane przez TraCI w `tsc_env/event_wrapper.py`.
- **Wiele „scenariuszy” to w praktyce zamiana pliku `.rou.xml`.** Każdy `.sumocfg` ma tylko `<input>`: `net-file`, `route-files` i `additional-files` (`map.poly.xml, e2.add.xml, tls.add.xml`), wszystkie jako ścieżki względne (`junction_scenarios/Beijing_Beihuan/normal_fluctuating_commuter.sumocfg`). `easy` i `normal` mają tę samą geometrię, różnią się tylko `<tlLogic>`. Dla Beihuan pliki są wręcz identyczne bajtowo (`cmp`). W Beishahe i Hongkong_YMT diff obejmuje wyłącznie stany faz.
- **Parametry zdublowane ręcznie:** `num_phases` w configu Pythona musi się zgadzać z programem TLS w `.net.xml`, ale nic tego nie sprawdza. `num_seconds` = 1000 we wszystkich 120 wariantach, a popyt kończy się około 600 s (ostatni `depart="594.24"` w `Beijing_Beihuan/routes/high_density.rou.xml`), co daje ~400 s na rozładowanie kolejek.
- **Dlaczego:** *(wnioskowane)* benchmark ma stałą siatkę 12×2×5, więc konwencja nazw zastępuje rejestr.
- **Czy dotyczy nas?** Tak, jako antywzorzec do ominięcia. Po naszej stronie jedno źródło prawdy powinno być deklaratywne (manifest artefaktu). Środowisko RL nie powinno wyprowadzać ścieżek z nazw ani trzymać `num_phases` w kodzie, tylko czytać je z manifestu albo z sieci.

### 3. Układ danych i duże pliki

**Wszystko commitowane, wejścia i wyjścia generatorów obok siebie.**
- **Zaobserwowane:**
  - Układ scenariusza: `junction_scenarios/<J>/{networks/{easy,normal}.net.xml, routes/<pattern>.rou.xml, add/{e2.add.xml, tls.add.xml, map.poly.xml, *.osm}, generate_routes.py, {easy,normal}_<pattern>.sumocfg}`.
  - Surowe `.osm` (do ~2,5 MB: `Hongkong_YMT/add/YMT.osm`, `SouthKorea_Songdo/add/songdo.osm`) leżą w `add/` obok wygenerowanych plików.
  - Największy `.rou.xml` ma ~140 KB.
  - `.gitignore` blokuje wyjścia SUMO (`*.out.xml`, `*.output.xml`), pliki pośrednie generatora (`**/testflow.trip.xml`, `**/testflow.turndefs.xml`) oraz `results/**/trip_info*.xml`, `results/**/fcd_output*.xml`, checkpointy i tensorboard (`.gitignore`).
  - `tls.add.xml` definiuje `timedEvent` `SaveTLS*`, a `e2.add.xml` detektory z `file="e2.output.xml"`. Oba zapisują pliki względem `.sumocfg`, czyli **do katalogu scenariusza** (stąd ignorowanie `*.out.xml`).
- **Dlaczego:** *(wnioskowane)* sieci pojedynczych skrzyżowań są na tyle małe, że da się je trzymać w git bez LFS.
- **Czy dotyczy nas?** Częściowo. Sieć Wrocławia i popyt z MATSim będą o rzędy wielkości większe. Warto jednak przejąć jedną rzecz: **wyjścia SUMO nie powinny lądować w katalogu artefaktu**. Nasz `.sumocfg`/`additional` nie powinien mieć zahardkodowanych ścieżek wyjściowych, bo artefakt ma być tylko do odczytu.

### 4. Proweniencja i wersjonowanie

**Proweniencja tylko przypadkowa, w komentarzach generatorów SUMO.**
- **Zaobserwowane:**
  - Nagłówki plików zapisują wersję narzędzia i pełną konfigurację generatora, łącznie z **absolutnymi ścieżkami autora**, np. `jtrrouter 1.27.0`, `/home/wmn/code/research/...`, `<seed value="777"/>`, `randomize-flows=True` (`Beijing_Beihuan/routes/low_density.rou.xml:1-35`). Sieci mają `netedit 1.22.0` i `/home/china/桌面/...` (diff `easy`/`normal` w `Beijing_Beishahe/networks/`).
  - Wersje SUMO są mieszane: `.sumocfg` wygenerowano w SUMO 1.20.0, sieci w 1.22/1.23.1, trasy w 1.27.0.
  - Nie ma changeloga, tagów, `CITATION.cff` ani wersji scenariusza. Cytowania są tylko jako BibTeX w `README.md`.
- **Dlaczego:** nie dotyczy (brak praktyki).
- **Czy dotyczy nas?** Tak, negatywnie. Środowisko nie ma skąd wiedzieć, na jakiej wersji popytu trenowano model. Nasz artefakt musi nieść wersję i hash w manifeście, a środowisko RL powinno logować je do `info`/Monitora.

### 5. Źródło prawdy dla sieci

**Sieć ręcznie edytowana w netedit. Popyt generowany względem jednej z dwóch sieci.**
- **Zaobserwowane:**
  - `.net.xml` powstały w netedit z OSM (nagłówki). ID krawędzi mają styl OSM (`157863208#0.1590`, `-1106233488.150`).
  - ID TLS są ręcznie nadane: `INT1`, a w Hongkong_YMT i SouthKorea_Songdo odpowiednio `J1` i `J2`. Każda sieć ma dokładnie jeden `<tlLogic>`.
  - Trasy generuje się względem `networks/normal.net.xml` (11 z 12 skrzyżowań; jedno względem `easy.net.xml`), a potem używa ich z **obiema** sieciami. Sprawdziłem skryptem: wszystkie krawędzie z tras istnieją w `easy.net.xml` dla wszystkich 12 skrzyżowań.
  - Spójność ID trzyma się tylko dlatego, że `easy`/`normal` różnią się wyłącznie programem sygnalizacji.
  - ID krawędzi pojawiają się też w `junction_configs/<J>.py`. Zdarzenia (`edge_id`, `route`) muszą pasować do sieci, co opisuje komentarz `junction_configs/Beijing_Beihuan.py:65-69`.
- **Czy dotyczy nas?** Tak. Nasz kontrakt musi jawnie mówić, że trasy w `.rou.xml` są ważne dla *konkretnej* sieci (hash sieci w manifeście popytu). Warianty sterowania (inne `tlLogic`) mogą współdzielić popyt tylko wtedy, gdy geometria i ID krawędzi są identyczne.

### 6. Przekazanie MATSim → SUMO (od strony konsumenta: czego środowisko RL wymaga od artefaktu)

W repo nie ma MATSim ani `matsim_importPlans.py`. Poniżej kontrakt wejścia środowiska wyczytany z kodu.

**6a. Wejście środowiska to para (`sumo_cfg`, `net_file`) plus `tls_id`, `num_seconds` i `num_phases`.**
- **Zaobserwowane:**
  - `TSCEnvironment.__init__` przyjmuje `sumo_cfg`, `net_file`, `num_seconds`, `tls_ids`, `tls_action_type`, `use_gui`, `trip_info` i `fcd_output`, po czym przekazuje je do `TshubEnvironment` (`tsc_env/base_env.py:13-40`).
  - Sieć podaje się **osobno**, mimo że jest też w `.sumocfg`. *(Wnioskowane:)* `tshub` czyta ją przez sumolib, żeby zbudować mapę.
  - Bez GUI środowisko używa `libsumo` (`base_env.py:36`).
  - `configure_sumo()` podmienia `_sumo_cfg`, `_net` i `sim_max_time` przed następnym `reset()` (`base_env.py:46-57`). To jedyny punkt, w którym scenariusz zmienia się w trakcie treningu.
- **Czy dotyczy nas?** Tak. Artefakt musi dostarczyć samowystarczalny `.sumocfg` ze **ścieżkami względnymi** oraz jawny `net_file`. Środowisko nie potrzebuje niczego poza tym, jeśli metadane (`tls_id`, horyzont) przyjdą z manifestu.

**6b. Popyt to wyłącznie jawne `<vehicle>` z trasami. Nie ma osób, trybów ani flow.**
- **Zaobserwowane:**
  - Każdy `.rou.xml` zawiera jeden `vType id="background"` (length 5, tau 1.0) i pojazdy postaci `<vehicle ... type="background" depart="3.58" departLane="best"><route edges="IN OUT"/></vehicle>` (`Beijing_Beihuan/routes/low_density.rou.xml:37-40`). We wszystkich 60 plikach tras występuje tylko typ `background` (grep).
  - Trasy mają zwykle **2 krawędzie** (wlot → wylot), bo sieć to jedno skrzyżowanie.
  - Popyt generuje `jtrrouter` z liczby pojazdów na minutę na krawędzi wlotowej w pięciu 2-minutowych przedziałach (`generate_routes.py:19-68`, `interval=[2,2,2,2,2]`), z domyślnymi proporcjami skrętów (`edge_turndef={}`).
- **Czego środowisko NIE używa:** pieszych, transportu publicznego, różnicowania typów pojazdów, `<person>`, `<flow>` ani `<trip>` bez trasy. Skoro nic z tego nie jest używane, nic nie jest też wymagane.
- **Wymóg pośredni:** `TSCEventWrapper` potrzebuje vType wypadku/pojazdu uprzywilejowanego (`barrier`, `police`, `emergency`, `fire_engine`). Jeśli typu brakuje w scenariuszu, kopiuje `DEFAULT_VEHTYPE` z ostrzeżeniem (`tsc_env/event_wrapper.py:191-198`, `README.md` „Special Events”). Brak typu nie jest błędem krytycznym.
- **Czy dotyczy nas?** Tak. Dla środowiska tego typu wystarczy SUMO-routable `.rou.xml` z pojazdami posortowanymi po `depart`. Semantyka osób i trybów z MATSim nie jest konsumowana, więc to, co ginie przy imporcie planów, nie szkodzi *temu* konsumentowi. Szkodzi tylko metrykom (tripinfo per pojazd, nie per osoba). Nasz artefakt powinien jednak **deklarować zestaw vType**, żeby zdarzenia i ewentualne typy uprzywilejowane nie wpadały w fallback.

**6c. Sieć musi mieć strukturę, na której opiera się ekstrakcja cech.**
- **Zaobserwowane:**
  - `extract_static_features` czyta ze stanu `tshub`: `node[tls_id].node_coord`, `tls[tls_id].{in_roads, out_roads, *_heading, movement_lane_ids, phase2movements}` (`tsc_env/tools/static_features.py:174-184`).
  - **ID krawędzi jest wyprowadzane z ID pasa przez `lane_id.split('_')[0]`** (`static_features.py:194`). Krawędź z `_` w ID zostanie po cichu pominięta. We wszystkich 24 sieciach repo żadne ID krawędzi nie zawiera `_` (sprawdzone skryptem).
  - Kierunek manewru jest rozpoznawany po podciągu `--s`/`--l`/`--r` w ID movementu (`static_features.py:97-105`). ID movementów tworzy `tshub` *(wnioskowane)*.
  - Detektory E2 w `add/e2.add.xml` mają ścisłą konwencję `e2det--<tls>--<edge>--<lane>--<dir>`: jeden na każdy pas wlotowy, `length="45"`, `freq="60"` (`Beijing_Beihuan/add/e2.add.xml`). *(Wnioskowane:)* `tshub` subskrybuje te detektory (lanearea), bo `run_batch.sh:9-10` mówi o „wyciekających subskrypcjach lanearea”. Cechy `jam_length_vehicle`, `last_step_occupancy`, `last_step_mean_speed` i `last_step_vehicle_id_list` per movement (`tsc_env/tools/tls_features.py:350-354`) prawdopodobnie pochodzą z tych detektorów.
- **Czy dotyczy nas?** Tak. Konwersja sieci Wrocławia musi utrzymać ID krawędzi bez `_` (netconvert z OSM zwykle to spełnia, ale nie przy `--join` itp.) i wygenerować `e2.add.xml` w tej samej konwencji dla każdego sterowanego skrzyżowania. Najlepiej robić to deterministycznie w pipeline sieci, nie ręcznie.

**6d. Kształt obserwacji i akcji zależy od sieci. Kod dopasowuje go przez padding lub obcięcie.**
- **Zaobserwowane:**
  - Przestrzeń obserwacji RL ma kształt `(history_len, num_movements, 9 + num_phases)` (`tsc_algos/rl/presslight/presslight_env/state_funcs.py:120-131`).
  - `num_movements=12` jest domyślny w każdym `make_env` (`presslight_env/make_env.py:30` i analogicznie dla attendlight/intellilight/unitsa).
  - Faktyczna liczba movementów wynika z sieci. `ChooseNextPhaseWrapper._build_state` **zeruje brakujące lub obcina nadmiarowe** wiersze do docelowego kształtu (`presslight_env/rl_wrapper.py:50-69`). Skrzyżowanie z >12 movementami (np. Songdo: 28 linków w stanie TLS) po cichu traci część wejścia *(wnioskowane z liczby linków; liczba movementów zależy od grupowania w `tshub`)*.
  - Akcja to `Discrete(num_phases)` (`rl_wrapper.py:36`), a `num_phases` pochodzi z configu Pythona, nie z sieci. Wartości w repo to 3 lub 4. Hongkong_YMT ma 4 fazy w `easy` i 3 w `normal`.
  - `RandomScenarioEnv` wymaga przy mieszaniu wzorców popytu tego samego `tls_id`, `num_phases` i `net_file` (`tsc_algos/rl/unitsa/unitsa_env/random_scenario_env.py:170-189`, docstring 16-22).
- **Czy dotyczy nas?** Tak, to rdzeń kontraktu. **Popyt można podmieniać swobodnie, sieć i program TLS nie.** Model wytrenowany na sieci X jest związany z liczbą faz i movementów jej skrzyżowań. Nasz artefakt powinien rozdzielać „sieć + TLS” (rzadko zmieniane, wersjonowane osobno) od „popytu” (wiele wariantów na tę samą sieć). Manifest sieci powinien publikować per-TLS `num_phases` i `num_movements`, żeby środowisko nie zgadywało.

**6e. Przełączanie popytu per epizod odbywa się przez podmianę `.sumocfg`, nie przez generowanie.**
- **Zaobserwowane:**
  - `RandomScenarioEnv.reset` losuje `env_name` z listy (`random.Random(seed)`) i wywołuje `configure_sumo(sumo_cfg=...)`, a dopiero potem wewnętrzny `reset` (`random_scenario_env.py:92-106`).
  - Nazwa wzorca trafia do `info['env_name']` i do Monitora (`info_keywords=('env_name',)`, linie 159-164).
  - W środowisku nie ma generowania popytu w locie. Wszystkie warianty to gotowe pliki.
- **Czy dotyczy nas?** Tak. Wzorzec „N gotowych `.rou.xml` na jedną sieć + jeden `.sumocfg` na wariant” przenosi się 1:1 na „N próbek/realizacji popytu z MATSim”. Warto żeby środowisko logowało ID/hash wariantu popytu w `info`.

### 7. Kalibracja między narzędziami

**Popyt skalowany do pojemności wynikającej z programu sygnalizacji.**
- **Zaobserwowane:**
  - `scripts/analyze_flow_capacity.py` buduje model sygnału z `.net.xml` (`sumolib.net.readNet(..., withPrograms=True, withConnections=True)`, linia 170).
  - Liczy przepływy movementów z `.rou.xml` (`count_movements`, linia 104), przepustowość nasycenia `SAT_FLOW = 1800` veh/h/pas (linia 37) i stopień nasycenia `X = Y / Y_MAX` przy `Y_MAX = 0.90` (linia 41).
  - `redesign_flows.py` skaluje popyt, aż szczyt `X` trafi w „drabinkę trudności”: low 0.35, fluctuating 0.55, random 0.68, high/increasing 0.72 (`scripts/redesign_flows.py:36-42`).
- **Dlaczego:** „ani przesycenie (wszystkie kierunki zakorkowane), ani brak rozróżnialności” (docstring `redesign_flows.py:4-6`).
- **Czy dotyczy nas?** Częściowo. Nasz popyt pochodzi z MATSim i nie powinien być ręcznie skalowany pod sterowanie. Ta sama miara (`X` per skrzyżowanie z programu TLS i `.rou.xml`) nadaje się jednak na **walidację artefaktu**: jeśli na niektórych skrzyżowaniach `X` > 1, to jest sygnał problemu z importem albo z przepustowością sieci, zanim ktoś zacznie trenować RL.

### 8. Próbka / skalowanie

**Brak pojęcia próbki. Popyt podawany bezpośrednio w pojazdach na minutę.**
- **Zaobserwowane:** popyt jest absolutny (veh/min na wlot, `generate_routes.py:21-53`). Nie ma `scale`, `--scale` SUMO ani skalowania przepustowości. Horyzont epizodu to 1000 s (`junction_configs/*.py`), popyt trwa 10 minut.
- **Czy dotyczy nas?** Częściowo. Środowisko RL zakłada 100% popytu. Jeśli nasz artefakt MATSim to próbka (np. 10%), musi przed dotarciem do środowiska zostać przeskalowany do pełnej populacji (albo jawnie oznaczony współczynnikiem). Środowisko nie ma gdzie przyjąć `scale`, a obserwacje (liczba pojazdów, obciążenie detektorów) zależą od bezwzględnej liczby pojazdów.

### 9. Testy i determinizm

**Brak testów. Popyt deterministyczny przez zamrożony plik, seed SUMO poza kontrolą repo.**
- **Zaobserwowane:**
  - Nie ma katalogu `tests/`, CI ani walidacji schematów.
  - `jtrrouter` uruchomiono z `seed=777` i `randomize-flows=True` (nagłówek `low_density.rou.xml:17,30`). Losowość popytu jest „wypieczona” w pliku, więc każdy epizod na danym `.sumocfg` ma **identyczne** pojazdy i czasy odjazdu.
  - `TSCEnvironment.reset()` nie przyjmuje seeda (`base_env.py:42`), a `TSCInfoWrapper.reset(seed=1)` go ignoruje (`tsc_env/tsc_info_wrapper.py:69-71`). Seed przekazywany przez SB3 (`DQN(seed=args.seed)`, `presslight/train.py:154`) nie dociera do SUMO.
  - `.sumocfg` nie ustawia `<random_number>` ani `<seed>` (grep, jedyne trafienia to nazwy plików `random_perturbation`).
  - Seed SUMO zależy więc od domyślnych ustawień `tshub`/SUMO: nie znaleziono w tym repo.
  - Jedyna kontrolowana losowość po stronie środowiska to wybór wzorca w `RandomScenarioEnv` (`random.Random(seed)`).
- **Dlaczego:** *(wnioskowane)* stały plik tras daje porównywalność między algorytmami kosztem różnorodności treningu.
- **Czy dotyczy nas?** Tak.
  - **Artefakt popytu jest źródłem determinizmu, nie seed.** Ten sam `.rou.xml` daje ten sam popyt.
  - Chcąc zmienności między epizodami, musimy dostarczyć kilka realizacji (np. z różnych seedów MATSim lub próbkowania) jako osobne pliki w jednym artefakcie.
  - Seed SUMO (`--seed`) powinien być jawnym parametrem naszego środowiska, a nie zostawać w bibliotece.

### 10. Dane zastrzeżone i licencje

**Brak pliku LICENSE. Surowe OSM commitowane.**
- **Zaobserwowane:** w korzeniu nie ma `LICENSE*` (sprawdzone `ls`). `README.md` podaje tylko autora. W `add/*.osm` leżą ekstrakty OSM (ODbL) bez atrybucji.
- **Czy dotyczy nas?** Częściowo. To przestroga. U nas scenariusze pochodne danych zastrzeżonych (ZDiUM, macierze z sieci komórkowej) nie mogą być commitowane „obok kodu” jak tutaj.

### 11. Runtime Python ↔ Java

Nie dotyczy. Brak Javy; SUMO jest zewnętrzną instalacją (`README.md` „Installation”).

### 12. Pakietowanie

**Brak pakietowania. Skrypty i `sys.path`, zależności w nieprzypiętym środowisku conda.**
- **Zaobserwowane:**
  - Nie ma `pyproject.toml`, `setup.py`, `requirements*.txt` ani `environment.yml` (sprawdzone `ls`).
  - Zależności są tylko wymienione w `README.md` (`tshub`, `stable_baselines3`, `gymnasium`, `torch`, `numpy`, `matplotlib`, `loguru`, `pyyaml`), bez wersji. Obowiązuje wymóg „uruchamiaj w env conda `tshub`”.
  - Punkty wejścia dopisują korzeń repo do `sys.path` (`tsc_algos/rl/presslight/train.py:13-15`).
  - `junction_configs` importuje moduły dynamicznie po nazwie (`_loader.py`, `importlib.import_module`).
  - Kod RL jest kopiowany między algorytmami: `attendlight_env` jest „skopiowane z presslight” (`tsc_algos/rl/README.md`).
  - Wersje SUMO/libsumo nie są przypięte. Pliki scenariuszy pochodzą z SUMO 1.20–1.27 (pkt 4).
- **Czy dotyczy nas?** Tak, jako kontrprzykład. W naszym monorepo środowisko RL powinno być osobnym członkiem workspace'u `uv` (biblioteka `tsc-env` z przypiętymi `libsumo`/`sumolib`/`traci` w tej samej wersji, którą walidujemy artefakt). Powinno przyjmować **ścieżkę do artefaktu/manifestu**, a nie zakładać, że scenariusze leżą w repo pod `junction_scenarios/`. Wersja SUMO użyta do wytworzenia `.net.xml`/`.rou.xml` powinna być zapisana w manifeście i sprawdzana przy ładowaniu.

### 13. Dokumentacja

**README na poziomie repo i per algorytm. Brak notatek przy scenariuszach.**
- **Zaobserwowane:** `README.md` opisuje architekturę wrapperów, układ katalogów i procedurę „New junction” (dodać katalog w `junction_scenarios/` + `junction_configs/<J>.py`). Algorytmy mają `tsc_algos/rl/*/README.md`. Scenariusze nie mają opisu źródła danych, daty ani metody. Jedyna dokumentacja popytu to komentarze w `generate_routes.py` (np. „X~0.35”).
- **Czy dotyczy nas?** Częściowo. Sekcja „New junction” to w praktyce lista kontrolna kontraktu. Nasze `docs/dane/` powinny mieć analogiczny opis „co musi zawierać artefakt, żeby środowisko go przyjęło”.

---

## Najważniejsze dla nas

1. **Kontrakt minimalny od strony RL:**
   - samowystarczalny `.sumocfg` ze ścieżkami względnymi, wskazujący `net-file`, `route-files` i `additional-files` (E2 + TLS);
   - jawnie podany `net_file`;
   - metadane per sterowane skrzyżowanie: `tls_id`, liczba faz zielonych, horyzont `num_seconds`.

   Dziś te metadane są w module Pythona obok kodu (`junction_configs/<J>.py`). U nas powinny być w manifeście artefaktu.
2. **Popyt jako jawne `<vehicle>` z pełnymi `<route edges>`, posortowane po `depart`, z zadeklarowanymi `vType`.** Środowisko nie konsumuje osób, trybów ani flow, więc straty semantyki MATSim→SUMO nie łamią tego konsumenta. Obowiązkowe są: pełna skala (brak parametru `scale`) oraz zgodność ID krawędzi z konkretną wersją sieci.
3. **Sieć i program TLS są „twardą” częścią kontraktu, popyt „miękką”.** Kształt obserwacji (`num_movements`) i akcji (`num_phases`) wynika z sieci. Mieszanie wariantów w treningu jest dozwolone tylko przy identycznym `net_file`/`tls_id`/`num_phases` (`random_scenario_env.py:170-189`). Artefakt popytu powinien więc wskazywać hash sieci, dla której jest ważny, a sieć i popyt powinny być wersjonowane osobno.
4. **Ukryte wymagania wobec sieci:**
   - ID krawędzi bez `_` (`static_features.py:194`);
   - detektory E2 per pas wlotowy w konwencji `e2det--<tls>--<edge>--<lane>--<dir>`;
   - program TLS z fazami zielonymi przeplatanymi żółtymi.

   Nasz pipeline sieci powinien je generować i walidować automatycznie.
5. **Determinizm zależy od pliku popytu, nie od seedu.** `reset(seed)` jest ignorowany, a seed SUMO nie jest ustawiany w repo. Chcąc zmienności, dostarczamy kilka realizacji popytu w jednym artefakcie. Środowisko losuje je per epizod przez `configure_sumo` i loguje ID wariantu.
6. **libsumo = jedna sieć na proces** (`run_batch.sh:9-10`), a wyjścia SUMO nie mogą trafiać do katalogu artefaktu. Z tego wynikają dwie reguły: artefakt tylko do odczytu, ścieżki wyjściowe podawane przez środowisko.

## Nie znaleziono

- **Kodu `tshub`:** nie ma go w repo ani w `/home/dominik/refs/wroclaw-twin/`. W efekcie nie wiadomo, jak powstają `phase2movements`/`movement_ids`, jakie flagi SUMO (`--seed`, `--step-length`, `--time-to-teleport`) są ustawiane i czy E2 są wymagane, czy opcjonalne.
- **Ustawienia seeda SUMO:** nie ma go w `.sumocfg` (grep `seed|random` po `junction_scenarios/*/*.sumocfg`) ani w `tsc_env/` (grep `seed`).
- **Testów, CI, walidacji schematów:** brak `tests/` i `.github/`.
- **Pakietowania/przypięcia wersji:** brak `pyproject.toml`, `setup.py`, `requirements*.txt`, `environment.yml`.
- **Licencji i atrybucji OSM:** brak `LICENSE*` w korzeniu.
- **Obsługi wielu TLS w jednym środowisku:** `tls_ids=[tls_id]` we wszystkich `make_env`. Środowisko multi-agent nie istnieje, więc kontrakt dla sieci z wieloma sterowanymi skrzyżowaniami trzeba będzie zdefiniować samodzielnie.
- **Związku z MATSim ani importu planów:** brak w całym repo.
