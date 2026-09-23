# sumo-scenarios (DLR-TS)

- **Repo:** https://github.com/DLR-TS/sumo-scenarios
- **Commit:** `00f6eb479a9dc0fbaeb731495c11d48d7a9661d3` (2026-06-08, wg `MANIFEST.tsv`; komunikat commita: „adapting to new sumo version”)
- **Klon częściowy:** `blob:none` + sparse checkout tylko `*.md *.py *.sh *.toml *.cfg *.yml *.yaml *.sumocfg LICENSE*` (`git sparse-checkout list`). Klon płytki — w lokalnej historii jest **1 commit** (`.git/shallow`), więc historii zmian nie da się ocenić. Pobrano 66 plików z 399 w `_trees/sumo-scenarios.txt`. **Żaden plik XML (sieci, trasy, `.add.xml`, detektory, TLS), `.txt`, `.csv`, `.bat`, ani `.gitignore`/`.gitmodules` nie jest pobrany** — ich istnienie znam z listingu, treści nie znam. Wszystko, co dotyczy zawartości tych plików, jest *(wnioskowane)* z nazw i z odwołań w `.sumocfg`/skryptach.

**Czym jest to repo.** Zbiór publicznie dostępnych sieci i popytów dla SUMO, utrzymywany przez DLR (Institute of Transportation Systems). Jeden katalog odpowiada regionowi, a w nim mogą być podkatalogi z wariantami scenariusza; część regionów to submoduły (`README.md:4-5`). Regiony: `BadHersfeld`, `Doerpfeldstr` (Berlin, jedna ulica), `TAVF-Hamburg`, `TestfeldNDS`, `Wildau`, `bologna` (4 warianty), `brunswick` (miv/intermodal/vdv/archive), `delmenhorst`, `germany-motorways`, `konstanz`, `port_tutorials` (3 porty) oraz submoduły `sumo-berlin`, `sumo_ingolstadt`, `BadHersfeld/osm/SUMOActivityGen`, `BadHersfeld/prt` (wpisy `160000 commit` w `git ls-tree HEAD`). Są też regresyjne testy TextTest (`tests/`). Repo nie ma wspólnej konwencji — każdy scenariusz ma inny układ, inny poziom dojrzałości i inne źródło popytu. Z punktu widzenia konsumenta to najcenniejsze **jako katalog wzorców i antywzorców dotyczących tego, co SUMO dostaje na wejściu**.

---

### 1. Orkiestracja

**Budowa scenariusza = ręcznie pisane skrypty shell/bat, bez DAG ani cache**
- **Zaobserwowane:** `brunswick/build.sh` to numerowany „przepis” (Step 0…6b) z twardymi ścieżkami (`TSC_HOME=$HOME/tsc`, `SCENARIO_HOME=$HOME/git/sumo-scenarios`, linie 4-5), w którym kroki opcjonalne (6a/6b) po prostu następują po sobie w tym samym pliku. `brunswick/miv/build.sh:7-13`: `patch` na OSM, a potem łańcuch `netconvert → tls_csvSignalGroups.py → netconvert → gtfs2pt.py → duarouter`. `BadHersfeld/osm/build.sh:2-7` ma większość kroków zakomentowanych („we currently work with the manually edited net so no net building here”). Pary `.sh`/`.bat` powielają to samo dla Windowsa (listing: `build.bat`, `run.bat`, `*.bat` w `Wildau/`).
- **Dlaczego:** *(wnioskowane)* scenariusze to jednorazowe artefakty projektów badawczych, bez zamiaru ponownego budowania w CI.
- **Czy dotyczy nas?** Częściowo: to przykład, jak nie robić (brak idempotencji, brak `--skip-to`, zakomentowane kroki jako „dokumentacja”). Potwierdza, że nasz pipeline popytu musi mieć jawną orkiestrację po stronie producenta.

### 2. Konfiguracja i wiele scenariuszy

**Wariant = osobny `.sumocfg` (lub osobny podkatalog), bez parametryzacji**
- **Zaobserwowane:** `BadHersfeld/evaluation/{present,future_no_prt,future_with_prt}.sumocfg` różnią się tylko `net-file`, `route-files` i częścią `additional-files` (np. `present.sumocfg:6-8` vs `future_with_prt.sumocfg:6-8`), a reszta (~60 linii opcji) jest skopiowana. `bologna/{acosta,acosta_persontrips,joined,pasubio}/run.sumocfg` używają tego samego wzorca nazw plików z prefiksem wariantu (`acosta_detectors.add.xml`, `joined_tls.add.xml` itd.). `Wildau/` ma **jeden** `Netzmodell2.net.xml` i jedno okno czasowe (`begin 53990`, `end 61000`) w podkatalogach per metoda generowania popytu (`randomTrips/`, `routeSampler/`, `dfRouter/`, `flowRouter/`, `cadyts/`, `duaRouter/`, `randomActivityGen/`, `saga/`), a każdy z nich ma własny `.sumocfg` wskazujący na `../Netzmodell2.net.xml` i `../additional.add.xml`. `brunswick/miv/` ma `oneshot.sumocfg`, `oneshot_meso.sumocfg`, `oneshot_meso_jc.sumocfg`, `oneshot_vdv.sumocfg`, `debug.sumocfg`, `pt.sumocfg`: warianty modelu (micro/meso) i komunikacji zbiorowej jako kopie pliku.
- **Dlaczego:** *(wnioskowane)* `.sumocfg` jest natywną jednostką „przebiegu” SUMO, a `sumo -c X.sumocfg` jest najprostszym interfejsem dla odbiorcy.
- **Czy dotyczy nas?** Tak. Konsument oczekuje, że dostanie **gotowy `.sumocfg` per wariant** (scenariusz × wersja popytu). Wzorzec Wildau (wspólna sieć i detektory, wymienny `route-files`) dobrze pasuje do porównywania wersji naszego artefaktu popytu na tej samej sieci. Kopiowanie opcji warto zastąpić generowaniem `.sumocfg` z jednego źródła.

### 3. Układ danych i duże pliki

**Wejścia i wyjścia w jednym katalogu; `.gz` dla dużych plików; część scenariuszy wymaga lokalnej budowy**
- **Zaobserwowane:** w git są duże pliki skompresowane gzipem (18 plików `.gz` w listingu): sieci (`*.net.xml.gz`), popyt (`BadHersfeld/demand/osm_activitygen_*_no_prt.rou.xml.gz`, `TAVF-Hamburg/PKW_automatic.rou.xml.gz`, `germany-motorways/gmw.rou.xml.gz`), wycinki OSM (`konstanz/osm/*.osm.xml.gz`) i archiwum GTFS (`brunswick/gtfs/gtfs_connect-...20200514.zip`). README zaleca `git clone --recursive --depth 1`, „Since the scenarios and their changesets are quite big” (`README.md:5-8`). Nie ma LFS ani zewnętrznego hostingu. Rozmiarów plików nie da się ustalić, bo klon `blob:none` nie ma blobów.
- **Zaobserwowane:** skryptem sprawdziłem, które pliki wskazane w `net-file`/`route-files`/`additional-files` każdego `.sumocfg` istnieją w listingu. Wyniki dzielą się na dwie grupy:
  - *samowystarczalne* (wszystkie wejścia commitowane): `bologna/*`, `TAVF-Hamburg`, `germany-motorways`, `port_tutorials/*`, `BadHersfeld/osm/osm.sumocfg`, `BadHersfeld/evaluation/{present,future_no_prt}`, `Doerpfeldstr/all_modes.sumocfg`, `Wildau/{randomTrips,routeSampler,cadyts}`;
  - *wymagające budowy*: cały `brunswick/*` (brak `miv.net.xml.gz`, `miv.trips.xml`, `gtfs_pt_*.add.xml` — tworzą je `brunswick/miv/build.sh` i `duarouter -c fromgeo.duarcfg`), `Wildau/{dfRouter,flowRouter,duaRouter,randomActivityGen}` (brak `routes.rou.xml`/`flowrouter.rou.xml`/…, czyli wyjść skryptów `.bat`).
- **Zaobserwowane (zepsute odwołania):** `Doerpfeldstr/scenario.sumocfg:10` wskazuje na `tram.rou.xml` i `parking_cars.rou.xml`, a `Doerpfeldstr/output_bike.sumocfg:4` na `tram.rou.xml`; żadnego z tych plików nie ma w repo. `BadHersfeld/evaluation/future_with_prt.sumocfg:6-8` wskazuje na `../prt_infra/*` i `osm_activitygen_future_with_prt.rou.xml.gz`, których też nie ma (README zakłada ścieżkę `prt/prt_infra`, `BadHersfeld/README.md:13`).
- **Zaobserwowane:** wyjścia symulacji trafiają do katalogu scenariusza (`BadHersfeld/osm/osm.sumocfg:16` `output-prefix ./outputs/TIME_`; `outputs/.gitkeep` jest w listingu). `tests/scenariorunner.py:17-18` przekierowuje je `--output-prefix` do katalogu testu. Istnieją lokalne `.gitignore` (`Doerpfeldstr/.gitignore`, root `.gitignore`), ale ich treść nie jest pobrana.
- **Dlaczego:** *(wnioskowane)* odbiorca ma móc od razu uruchomić scenariusz bez toolchainu (stąd commitowane `.rou.xml.gz`), a scenariusze oparte na zastrzeżonych wejściach (TAPAS) są odtwarzalne tylko u autora.
- **Czy dotyczy nas?** Tak. Nasz artefakt powinien być **samowystarczalny** (sieć + popyt + `.sumocfg` + additionals w jednej paczce, ścieżki względne). Do tego potrzebny jest automatyczny test „każdy plik z `.sumocfg` istnieje” — w tym repo takiego testu nie ma i widać skutki. Duże pliki w git + `--depth 1` to rozwiązanie, którego nie chcemy powielać (my mamy dysk/`processed/` i wersjonowany artefakt).

### 4. Proweniencja i wersjonowanie

**Proweniencja wyłącznie w nagłówkach XML i ad hoc w README; brak wersji scenariusza**
- **Zaobserwowane:** jedyny ślad wersji narzędzia to komentarz generatora w `.sumocfg`, np. `germany-motorways/run.sumocfg:3` („generated on 2025-10-14 … by Eclipse SUMO sumo v1_24_0+1413-…”), `Doerpfeldstr/all_modes.sumocfg:3` (v1_18_0+0023), `brunswick/miv/oneshot.sumocfg:3` (v1_0_1, 2019), `port_tutorials/port_brunswick/osm.sumocfg:3` (v1_8_0). Nie ma CHANGELOG, `CITATION.cff`, tagów ani release'ów (lokalnie widoczny jest tylko HEAD). Ostatni commit („adapting to new sumo version”) sugeruje, że scenariusze są łatane pod nowe wersje SUMO *(wnioskowane)*.
- **Najlepszy przykład w repo:** `germany-motorways/README.md:5-36` podaje URL wejścia (Geofabrik `germany-latest.osm.pbf`), przypiętą wersję SUMO (`v1_24_0+1413 / v1_25_0`) i dokładne polecenia `osmconvert`/`osmfilter`/`netconvert`/`randomTrips.py`. Brakuje jednak daty lub hasha pliku PBF.
- **Zaobserwowane:** `brunswick/build.sh:21-22` przypina przebieg TAPAS przez `--sim-key 2022y_01m_26d_14h_31m_26s_378ms` („current simkey for KoFiF 2030 reference scenario”). To jedyne w repo powiązanie popytu z konkretnym przebiegiem generatora.
- **Dlaczego:** *(wnioskowane)* repo jest kolekcją wyników, nie pipeline'em, więc wersjonowanie sprowadza się do commitów gita.
- **Czy dotyczy nas?** Tak, jako lekcja negatywna. Konsument nie ma skąd wiedzieć, z jakiej wersji generatora i danych pochodzi popyt. Nasz artefakt musi nieść manifest (wersja artefaktu, commit kodu, wersja MATSim/SUMO, identyfikator przebiegu w stylu `sim-key`, hashe wejść).

### 5. Źródło prawdy dla sieci

**Sieć kanoniczna bywa ręcznie edytowana; zmiany trzymane jako patche `netconvert`**
- **Zaobserwowane:** `brunswick/miv/netpatch/` i `brunswick/intermodal/netpatch{,_pre}/` zawierają pliki plain-XML `patch.{nod,edg,con,tll}.xml` oraz lokalne patche (`patch_A391_AS_Lehndorf.*`, `patch_A39_B1.*`, `tram.edg.xml`, `extra_edges.txt`, `keep_induction_loops.txt`), stosowane przez `miv.netccfg`/`miv2.netccfg` (`brunswick/miv/build.sh:8,11`). Na surowym OSM dodatkowo nakładany jest `patch -N ../osm/BS_detail.osm.xml ../osm/tram4.diff` (`build.sh:7`). `BadHersfeld/osm/netdiff/diff.{nod,edg,con,tll}.xml` + `patched.netccfg` to ten sam wzorzec, ale `BadHersfeld/osm/build.sh:2-4` mówi wprost, że pracuje się na ręcznie edytowanej `osm_edited.net.xml.gz`, a przebudowa jest wyłączona. `TestfeldNDS/netpatch/patch.edg.xml` — analogicznie.
- **Zaobserwowane (spójność ID między sieciami):** w `brunswick` popyt z TAPAS (liczony na dużej sieci `testfield/net.net.xml.gz`) jest przycinany do mniejszej sieci przez `cutRoutes.py --orig-net … --trips-output …` (`brunswick/build.sh:26`), a potem konwertowany do **geo-trips**: `duarouter -c miv/trips2geo.duarcfg` (linia 29). Komentarz w linii 31-32: geo-tripy w `bs_miv_cut.geotrips.xml.gz` „are the final result of the process because they can be mapped ‘universally’ to a new network”. Każda przebudowa sieci ponownie mapuje je przez `duarouter -c fromgeo.duarcfg` (`brunswick/miv/build.sh:13`, `brunswick/intermodal/build.sh:4`).
- **Dlaczego:** wprost (`brunswick/build.sh:31-32`): ID krawędzi zmieniają się po każdej edycji sieci, więc popyt przechowuje się w postaci niezależnej od sieci (współrzędne).
- **Czy dotyczy nas?** Tak, bardzo. To bezpośrednia odpowiedź na problem spójności ID między MATSim a SUMO. Warto rozważyć, żeby artefakt popytu miał **warstwę geo (współrzędne from/to + czasy)** obok tras na krawędziach, co pozwala zmapować go na poprawioną sieć SUMO bez ponownego uruchamiania MATSim. Patche plain-XML dla sieci (`*.nod/edg/con/tll.xml` + `.netccfg`) to wzorzec dla ręcznych korekt sieci Wrocławia (np. programy sygnalizacji z ZDiUM).

### 6. Przekazanie MATSim → SUMO (strona konsumenta)

W repo nie ma MATSim ani `matsim_importPlans.py` (grep po `*.py`/`*.sh`). Jest natomiast analogiczne przekazanie **TAPAS → SUMO**, a pliki pokazują, czego oczekuje konsument.

**Popyt jako *trips* z dynamicznym routingiem w SUMO, nie gotowe trasy**
- **Zaobserwowane:** `brunswick/miv/oneshot*.sumocfg:10` ładuje `miv.trips.xml` (tripy z `fromgeo.duarcfg`) i włącza `device.rerouting.*` (period 300, adaptation-steps 36, `synchronize true`, linie 28-34) oraz `junction-taz true` (linia 41). To tryb „oneshot”: trasy wylicza sama symulacja. Zapisywane jest `vehroute-output` z `last-route`, `sorted`, `intended-depart` (linie 15-18), czyli wynik routingu jest eksportowany do ponownego użycia. Wejście z TAPAS to `vehroutes_oneshot_meso.rou.xml` (`brunswick/build.sh:26`): po stronie TAPAS/TSC również liczono oneshot w meso.
- **Czy dotyczy nas?** Tak. Konsument SUMO w praktyce DLR **nie ufa trasom z generatora popytu**: bierze OD/tripy i sam je routuje (rerouting/oneshot albo `duarouter`). Nasz kontrakt powinien jasno określić, czy dostarczamy trasy MATSim (krawędzie), czy tripy plus opcjonalne trasy; wersja tylko z trasami jest krucha wobec zmian sieci (zob. wymiar 5).

**Konwencja czasu: popyt TAPAS przesunięty na „dzień 2”**
- **Zaobserwowane:** `begin value="1:3:30:0"` (dzień 1, 03:30 = 99000 s) w `brunswick/miv/oneshot.sumocfg:25`; GTFS importowany z `--begin 97200 --end 172800` (`brunswick/miv/build.sh:12`); `vdv_importer.py … --departure-offset 86400` (`brunswick/vdv/build_vdv.sh:2`).
- **Dlaczego:** *(wnioskowane)* ma to objąć podróże zaczynające się przed północą lub warm-up bez ujemnych czasów.
- **Czy dotyczy nas?** Tak. Semantyka czasu odjazdu (offset doby, okno symulacji, warm-up) musi być jawnie zapisana w kontrakcie artefaktu. Inne scenariusze stosują różne okna: `Doerpfeldstr` 50300–54100 s z popytem generowanym na 50400–54000 (`buildRoutes.sh:1`, ~100 s warm-up), `Wildau` 53990–61000, `BadHersfeld` 0–86400.

**Osoby vs pojazdy, tryby**
- **Zaobserwowane:** tylko `bologna/acosta_persontrips` (`persontrips.xml`, `persontrips.duarcfg`, `run.sumocfg:7` ładuje `persontrips.rou.xml`) i `BadHersfeld` (SUMOActivityGen, `persontrip.transfer.*`, taxi device, `osm.sumocfg:53-59`) mają popyt osobowy/intermodalny. Pozostałe mają popyt wyłącznie pojazdowy, rozdzielony per klasa pojazdu na osobne pliki (`TAVF-Hamburg`: `PKW_automatic.rou.xml.gz`, `LKW_automatic.rou.xml`, `bus_route.rou.xml`; `port_brunswick`: `osm.{passenger,truck,rail,ship}.trips.xml`). Komunikacja zbiorowa zawsze jest osobnym plikiem (z GTFS `gtfs2pt.py`, z VDV `vdv_importer.py` albo z OSM `ptlines2flows.py`).
- **Czy dotyczy nas?** Tak. Dla RL sterowania sygnalizacją wystarczy popyt pojazdowy (+ PT jako osobna warstwa z GTFS). Warto dzielić artefakt na pliki per tryb/klasa pojazdu, żeby konsument mógł je selektywnie włączać w `route-files`.

### 7. Kalibracja między narzędziami

**Wildau: jedna sieć, jedne liczniki, wiele metod generowania popytu**
- **Zaobserwowane:** `Wildau/` zawiera pomiary (`edgedata-file_Wildau.xml`, `dfRouter/detectors.xml`, `dfRouter/fahrzeuganzahl`, `flowRouter/fahrzeuganzahl2.csv`, `cadyts/measurements.xml`, `pedestrians and bicycles/counting_data_bike.xml`) i dla każdej metody (`randomTrips`, `routeSampler`, `dfRouter`, `flowRouter`, `cadyts` + `duaIterate.bat`, `duaRouter`, `randomActivityGen`, `saga`) osobny podkatalog z README DE/EN (`README_<metoda>.txt` / `_eng.txt`, niepobrane) oraz `Results.xlsx`. `Wildau/rms.py` liczy RMS między licznikami (`departed+entered` per edge z `edgedata-file_Wildau.xml`) a wyjściem symulacji (`outputdfrouter.xml`).
- **Uwaga (błąd):** `Wildau/rms.py:28` w drugiej pętli używa `edge` (ostatniej wartości z pierwszej pętli) zamiast `edge2`, więc każdy wyraz różnicy odejmuje od tej samej wartości licznika. Wynik RMS jest nieprawidłowy.
- **Czy dotyczy nas?** Tak. To gotowy układ „benchmarku popytu względem liczników ZDiUM”: porównanie naszego popytu z MATSim z baselinami `routeSampler`/`flowrouter`/cadyts na tej samej sieci i tych samych licznikach. Metryka powinna być w przetestowanym kodzie, nie w jednorazowym skrypcie (patrz błąd wyżej).

**`routeSampler` z licznikami i turn ratios (Doerpfeldstr)**
- **Zaobserwowane:** `Doerpfeldstr/buildRoutes.sh:1-4`: `randomTrips.py` (seed 42) generuje pulę tras, a `routeSampler.py` dobiera z niej trasy do `edge_data_cars.xml` i `turn_ratio.xml` (`--turn-ratio-attribute ratio`, `--mismatch-output mismatch.xml`), osobno dla rowerów (`edge_data_bike.xml`). Są też `weights.{src,dst,via}.xml` (wagi `randomTrips`, niepobrane).
- **Czy dotyczy nas?** Częściowo. To typowy „konkurent” naszego popytu po stronie SUMO. `--mismatch-output` jest wzorem raportu zgodności z licznikami, który warto emitować przy walidacji artefaktu.

**Kalibratory w symulacji (BadHersfeld)**
- **Zaobserwowane:** `BadHersfeld/evaluation/*.sumocfg:8` ładują `../osm/calib.add.xml`. `osm/osm.sumocfg` go nie ładuje, za to ładuje `induction_loops.add.xml` i `edge_lane_data.add.xml` (`osm.sumocfg:6`). Treść niepobrana; z nazwy *(wnioskowane)* to `calibrator`y SUMO wymuszające przepływy.
- **Czy dotyczy nas?** Częściowo. Kalibratory maskują błędy popytu — w ewaluacji RL należy ich unikać albo jawnie je raportować.

### 8. Próbka / skalowanie

**Brak jawnego parametru próbki/skalowania**
- **Zaobserwowane:** w `.sumocfg`/skryptach nie ma `--scale` ani `scaleFactor` (grep). Intensywność popytu jest zaszyta w parametrach generatorów: `randomTrips -p` (okres wstawiania, np. `-p 3` w `Doerpfeldstr/buildRoutes.sh:1`, `-p 1.2` w `port_brunswick/build.sh:4`), `--insertion-rate 500000` (`germany-motorways/README.md:29`), `saga … --population 10000` (`Wildau/saga/saga.sh:2`). Przycinanie popytu TAPAS używa `cutRoutes.py --speed-factor 0.7 --min-length 2 --min-air-dist 100` (`brunswick/build.sh:26`). Nigdzie nie opisano, czy popyt TAPAS to 100% populacji.
- **Czy dotyczy nas?** Tak. Jeśli nasz artefakt będzie próbką (np. 10%/25% MATSim), konsument SUMO musi to wiedzieć, a `sumo --scale` / wagi przepustowości muszą być zapisane w manifeście. Repo pokazuje, że bez tego informacja ginie.

### 9. Testy i determinizm

**TextTest: smoke test „czy scenariusz się uruchamia” na aktualnym SUMO**
- **Zaobserwowane:** `tests/runTests.py:11-24` ustawia zmienne `*_BINARY` dla binariów SUMO z `$SUMO_HOME/bin` i wywołuje `texttest`. `tests/scenariorunner.py:8-18` uruchamia `sumo -c <sumocfg> --no-step-log --output-prefix <testdir> --end 1000 --aggregate-warnings 0`, czyli **symuluje tylko do 1000 s**. W `tests/scenario/<region>/` każdy test ma `options.scenario` (argumenty), `errors.scenario` (oczekiwany stderr), `output.scenario` (oczekiwany stdout) i `catalogue.scenario` (oczekiwane pliki wyjściowe). Testy obejmują: `BadHersfeld`, `TAVF-Hamburg`, `Wildau`, `bologna/{acosta,acosta_persontrips,joined,pasubio}`, `port_tutorials/{brunswick,cologne,lübeck}`, `sumo_ingolstadt/{bicycle,plain}`. Brak testów `brunswick`, `Doerpfeldstr`, `germany-motorways`, `konstanz`, `TestfeldNDS`, `delmenhorst`, `sumo-berlin`.
- **Zaobserwowane (z hashy blobów, `git ls-tree -r HEAD tests/`):** 12 plików testu ma hash pustego bloba `e69de29…` (m.in. wszystkie `output.scenario` w `bologna/*`, `Wildau`, `sumo_ingolstadt/*`), więc oczekiwany stdout jest pusty. Trzy `bologna/*/catalogue.scenario` są identyczne (`f12c162…`), a `sumo_ingolstadt/{bicycle,plain}` mają identyczne `catalogue/errors/options` (ten sam hash). Treści `options.scenario` nie znam, bo blob nie jest pobrany.
- **Determinizm:** ziarna są przypięte w części `.sumocfg` (`seed 42` w `BadHersfeld/*.sumocfg`, np. `present.sumocfg:62-64`; `seed 121294` w `TAVF-Hamburg/run_simulation.sumocfg:24`) i w generatorach (`randomTrips --seed 42`). Porównanie `errors.scenario` z oczekiwanym jest w praktyce testem regresji ostrzeżeń *(wnioskowane z konwencji TextTest)*. Nie ma testów wartości wyjściowych (np. liczby pojazdów) ani walidacji schematów XML.
- **Dlaczego:** *(wnioskowane)* testy chronią przed tym, że nowa wersja SUMO zepsuje scenariusze (por. komunikat commita), a nie przed błędami w popycie.
- **Czy dotyczy nas?** Tak. Tani i wartościowy wzorzec: **smoke test artefaktu = `sumo -c … --end N` + porównanie stderr/ostrzeżeń + lista plików wyjściowych**, uruchamiany na przypiętej wersji SUMO. Dla nas warto dodać sprawdzenie istnienia plików z `.sumocfg` (wymiar 3) i podstawowe liczniki (liczba wstawionych/teleportowanych pojazdów).

### 10. Dane zastrzeżone i licencje

**Jedna licencja EPL-2.0 dla całości, z niespójnościami per plik i osobną licencją danych**
- **Zaobserwowane:** root `LICENSE` to Eclipse Public License 2.0. `BadHersfeld/evaluation/tripinfoEmissions.py:3-13` ma nagłówek `SPDX-License-Identifier: EPL-2.0 OR GPL-2.0-or-later`. Natomiast `brunswick/vdv/vdv_importer.py:7-9` i `TestfeldNDS/__init__.py:9-10` mają „Copyright … DLR/TS, Germany / All rights reserved”, co jest sprzeczne z EPL dla całego repo. `brunswick/gtfs/license.txt` i `readme.txt` to osobna licencja danych GTFS (treść niepobrana). Dane OSM (ODbL) nie mają nigdzie informacji o licencji.
- **Separacja danych nieredystrybuowalnych:** popyt TAPAS (dane KoFiF/syntetyczna populacja) nie jest commitowany; commitowany jest tylko **produkt pochodny** — geo-tripy przycięte do regionu (`brunswick/miv/bs_miv_cut.geotrips.xml.gz`), a surowy przebieg zostaje w `$TSC_HOME/scenario_workdir` (`brunswick/build.sh:18-26`). `TestfeldNDS` jest „usable with the TAPAS-SUMO-Coupling” (`README.md:10-13`): w repo są tylko sieć/cordon/`vtypes` i hook `trip_filter` (`TestfeldNDS/__init__.py:18-22`), a popyt generuje prywatny TAPAS.
- **Dlaczego:** *(wnioskowane)* publikuje się to, co może być publiczne (sieć z OSM, zagregowane/zanonimizowane tripy), a źródło zostaje u właściciela.
- **Czy dotyczy nas?** Tak. Wzorzec „commituj tylko artefakt pochodny, surowe dane i pełny przebieg generatora zostają w zamkniętym środowisku” pasuje do naszych danych KBR/sieci komórkowej/ZDiUM. Potrzebujemy jednak jawnej informacji o licencji per katalog/artefakt (OSM ODbL, GTFS, dane ZDiUM), której tu brakuje.

### 11. Runtime Python ↔ Java

Nie dotyczy. W repo nie ma Javy ani JAR-ów. TAPAS/TSC jest zewnętrzny (`$HOME/tsc`, `brunswick/build.sh:4,18-22`).

### 12. Pakietowanie

Nie dotyczy. Nie ma `pyproject.toml` ani pakietu. Skrypty Python importują `sumolib` przez `sys.path.append($SUMO_HOME/tools)` (`tests/scenariorunner.py:5`, `TestfeldNDS/setup.py:4-6`). `TestfeldNDS/setup.py` nie jest setuptools, tylko skryptem budującym wariant `_reduced` sieci. `vdv_importer.py` jest w Pythonie 2 (`from sets import Set`, linia 21).

### 13. Dokumentacja

**README nieobowiązkowe i nierówne; najlepsze to „przepisy” z poleceniami**
- **Zaobserwowane:** `README.md` mają tylko root, `BadHersfeld` (opis wariantów popytu present/future × no/with/only PRT, `BadHersfeld/README.md:17-29`), `TAVF-Hamburg` (jedno zdanie, `README.md:1`) i `germany-motorways` (pełny przepis: wejście → sieć → ruch → symulacja). `Wildau` ma `README.txt`/`README_eng.txt` na poziomie scenariusza i per metoda popytu (niepobrane, bo `.txt`). `brunswick/vdv/build_steps.txt` i `brunswick/gtfs/readme.txt` też są niepobrane. `bologna`, `brunswick`, `Doerpfeldstr`, `konstanz`, `delmenhorst`, `TestfeldNDS` nie mają README; proces opisują wyłącznie komentarze w `build.sh` (`brunswick/build.sh:1-46`).
- **Zaobserwowane:** README BadHersfeld opisuje pliki, których nie ma w repo (`*_with_prt*`, `*_only_prt*`, `joined.sumocfg`), więc dokumentacja rozjeżdża się z zawartością.
- **Czy dotyczy nas?** Tak. Dla artefaktu popytu minimalny README per scenariusz powinien zawierać: pochodzenie popytu, okno czasowe i warm-up, próbkę/skalę, wersję SUMO, polecenie uruchomienia i listę plików. Najlepiej generować go z manifestu, żeby nie rozjechał się z zawartością jak w BadHersfeld.

---

## Detektory i programy sygnalizacji (przegląd przekrojowy — ważne dla danych ZDiUM)

Treści XML nie są pobrane, więc poniżej są nazwy plików i odwołania z `.sumocfg`/skryptów:
- **Detektory jako osobny `*.add.xml` w `additional-files`:** `bologna/*/<wariant>_detectors.add.xml` (ładowane w `run.sumocfg:8`; wariant `acosta_persontrips` ich nie ładuje), `BadHersfeld/osm/induction_loops.add.xml` + `edge_lane_data.add.xml` (`osm.sumocfg:6`), `Wildau/additional.add.xml` (wspólny dla wszystkich metod popytu), `Doerpfeldstr/additional*.add.xml`, `TestfeldNDS/cordon.add.xml` + `cordon.csv`, `TAVF-Hamburg/edgeOutputs.add.xml`.
- **Pomiary jako wejście generatora popytu:** edgeData (`Wildau/edgedata-file_Wildau.xml`, `Doerpfeldstr/edge_data_{cars,bike}.xml`), `turn_ratio.xml`, `dfRouter/detectors.xml` + `fahrzeuganzahl` (format dfrouter), `cadyts/measurements.xml`, `flowRouter/fahrzeuganzahl2.csv`.
- **Programy sygnalizacji jako osobny `*_tls.add.xml`** (`tlLogic` nadpisujące te z sieci *(wnioskowane)*): `bologna/*/<wariant>_tls.add.xml`, `acosta_tls2.add.xml` (osobna wersja dla sieci z pieszymi), `Doerpfeldstr/tram_tls.add.xml`. **Albo wbudowane w sieć przez patch:** `brunswick/miv/netpatch/patch.tll.xml`, `BadHersfeld/osm/netdiff/diff.tll.xml`.
- **Import planów sygnalizacji z tabel grup sygnałowych:** `brunswick/miv/build.sh:10` uruchamia `tls_csvSignalGroups.py` na `netpatch/Rudolfplatz_SP33.csv` → `Rudolfplatz.tll.xml` (wcześniejsza wersja `Rudolfplatz.csv` jest zakomentowana w linii 9). Plik `netpatch/keep_induction_loops.txt` wskazuje *(wnioskowane)*, że pętle z OSM/sieci są jawnie zachowywane przy budowie. `tls.actuated.show-detectors true` w `brunswick/miv/debug.sumocfg` i `oneshot_vdv.sumocfg:41` oznacza, że sieć ma sterowniki `actuated`.
- **Czy dotyczy nas?** Tak. Dla danych ZDiUM (sterowniki, pętle) to dwa sprawdzone kanały: (a) `tls_csvSignalGroups.py` z tabel grup sygnałowych → `.tll.xml` → patch sieci w `netconvert`; (b) detektory jako osobny `.add.xml`, niezależny od popytu, współdzielony przez warianty (jak `Wildau/additional.add.xml`). Te same pętle mogą być wejściem `routeSampler`/`dfrouter` do baselinów popytu.

---

## Najważniejsze dla nas

1. **Popyt niezależny od sieci (geo-trips):** DLR przechowuje popyt TAPAS jako geo-tripy właśnie dlatego, że da się je „uniwersalnie” zmapować na nową sieć (`brunswick/build.sh:31-32`), i ponownie mapuje je `duarouter`em po każdej zmianie sieci. Nasz artefakt MATSim→SUMO powinien mieć warstwę współrzędnych obok (albo zamiast) tras na krawędziach.
2. **Konsument sam routuje:** scenariusze oparte na zewnętrznym generatorze ładują *trips* + `device.rerouting` / `junction-taz` (oneshot), a nie gotowe trasy. Kontrakt musi jasno określić, co jest dostarczane (tripy/trasy), jaka jest semantyka czasu (tu offset „dzień 2”, `1:3:30:0`), okno i warm-up.
3. **Samowystarczalność i test integralności:** połowa scenariuszy nie uruchamia się bez lokalnej budowy albo wskazuje na nieistniejące pliki (`Doerpfeldstr/scenario.sumocfg:10`, `BadHersfeld/evaluation/future_with_prt.sumocfg`). Potrzebne są: paczka z `.sumocfg`, ścieżki względne, test „wszystkie pliki istnieją” oraz smoke test `sumo --end N` w stylu `tests/scenariorunner.py`.
4. **Benchmark popytu względem liczników (Wildau):** jedna sieć + jeden zestaw pomiarów + wymienny `route-files` per metoda (routeSampler, flowrouter, dfrouter, cadyts, activitygen). To gotowy wzór porównania naszego popytu z baselinami na danych ZDiUM, z metryką w przetestowanym kodzie (w `Wildau/rms.py:28` jest błąd).
5. **Detektory i TLS jako osobne warstwy:** `*_detectors.add.xml` i `*_tls.add.xml` niezależne od popytu, a plany sygnalizacji z tabel grup sygnałowych importowane przez `tls_csvSignalGroups.py` do patcha sieci. To ścieżka dla danych ZDiUM o sterownikach.
6. **Proweniencja praktycznie nie istnieje** (tylko komentarze „generated by SUMO vX” i jeden `--sim-key`). Nasz manifest artefaktu (wersje, hashe wejść, próbka/skala, licencje per źródło) wypełnia lukę, którą to repo dobrze ilustruje.

## Nie znaleziono

- Treści plików XML/CSV/TXT/BAT (sieci, trasy, detektory, `tlLogic`, `calib.add.xml`, `fromgeo.duarcfg`, `trips2geo.duarcfg`, README Wildau, `brunswick/gtfs/license.txt`, `.gitignore`, `.gitmodules`) — klon `blob:none` + sparse checkout; pobranie wymagałoby sieci (`GIT_NO_LAZY_FETCH=1 git cat-file` → „could not fetch … from promisor remote”).
- Rozmiarów plików — `git ls-tree -l` wymaga blobów.
- Historii zmian, CHANGELOG, `CITATION.cff`, tagów, release'ów — klon płytki (1 commit), brak takich plików w listingu.
- MATSim / `matsim_importPlans.py` / jakiegokolwiek importu planów — grep po `*.py`, `*.sh`, listing `_trees/sumo-scenarios.txt`.
- Jawnego parametru próbki/skalowania (`--scale`, `scaleFactor`) — grep po `.sumocfg`/`.sh`/`.py`.
- Walidacji schematów XML i testów wartości wyjściowych — `tests/` zawiera tylko TextTest smoke testy.
- Zawartości submodułów `sumo-berlin`, `sumo_ingolstadt`, `BadHersfeld/prt`, `BadHersfeld/osm/SUMOActivityGen` (niepobrane; `sumo-berlin` ma osobną notatkę).
