# dlr-tsc (TAPAS-SUMO-Coupling, `tsc`)

- **Repo:** https://github.com/DLR-TS/tsc
- **Commit:** `3962e7f2af27263777bbe899dfe72d6147ff7509` (z `MANIFEST.tsv`, klon oznaczony `full`, 2026-07-06)
- **Częściowość:** wszystkie pliki śledzone przez git są w klonie, ale:
  (a) **historia jest płytka** (`git rev-parse --is-shallow-repository` → `true`, jeden commit „adapting tests to enabled actuated meso”, brak tagów lokalnie);
  (b) **duże pliki to niepobrane wskaźniki Git LFS**: `data/berlin_net/osm/berlin_net.osm.xml.gz` (wskaźnik na ~129 MB), `data/berlin_net/gtfs/BVG_VBB_bereichsscharf_2019*.zip`, `..._2020*.zip`. Pliki `.gitattributes` w klonie nie ma.
  (c) Kod dla modelu popytu **TAPAS (Java) nie jest częścią tego repo**. Po drugiej stronie jest tylko baza PostgreSQL, więc stronę TAPAS znamy wyłącznie z kontraktu tabel.

**Czym jest to repo.** Pakiet Pythona `tapas_sumo_coupling` (DLR, Instytut Systemów Transportowych, EPL-2.0 OR GPL-2.0+) uruchamiany jako **daemon**. Daemon odpytuje bazę Postgres o nowe „zlecenia symulacji” (`sim_key` + numer iteracji) wystawione przez agentowy model popytu TAPAS. Dla każdego zlecenia pobiera z bazy tabelę tripów TAPAS (CSV), „prostuje” ją (rozmycie czasu i przestrzeni, filtr trybów), mapuje współrzędne WGS84 na krawędzie sieci SUMO, generuje `<trip>`/`<person><personTrip>`, a potem robi przypisanie ruchu w SUMO (oneshot meso z reroutingiem, duaIterate/Gawron albo marouter). Wyniki trafiają z powrotem do bazy: czasy i dystanse per trip oraz **macierze czasów przejazdu OD per przedział czasowy** (skimy), z których TAPAS korzysta w następnej iteracji. Specyfikacja interfejsu to niemiecki „Lastenheft” (`docs/20210604LastenheftSUMO_TAPAS.docx`, v1.6). W `data/` są prekursory scenariuszy (Berlin, Mitte, Siemensbahn, move_urban). Nowsze scenariusze leżą w osobnym repo `DLR-TS/sumo-scenarios` (`README.md`, sekcja „Cloning sumo scenarios”). Testy są w TextTest (`tests/`). **To najbliższy analog naszej architektury, z jedną różnicą: pętla jest tu dwukierunkowa (popyt ↔ SUMO), a u nas popyt jest artefaktem jednokierunkowym.**

---

### 1. Orkiestracja

**Daemon odpytujący bazę + stały, liniowy przebieg w procesie na zlecenie**
- **Zaobserwowane:** `tsc_main.main` co 2 s w pętli pobiera zlecenia (`get_simulation_requests`) i uruchamia każde w osobnym `multiprocessing.Process`, z limitem `--parallel` (`src/tapas_sumo_coupling/tsc_main.py:502-532`). Tryb daemona włącza `--daemon`, bez niego wykonuje się jeden obrót pętli (`:523-524`). Zlecenie jest „aktywne”, gdy ostatni wpis w tabeli statusu dla pary (`sim_key`, `iteration`) ma `msg_type = 'pending'` (`get_trips.py:105-124`). Wewnątrz zlecenia `simulation_request` ma stałą kolejność kroków: katalog scenariusza → ruch tła (tylko w iteracji 0) → katalog iteracji → pobranie tripów → `t2s.main` (prostowanie → mapowanie → tripdefs → przypisanie) → postprocessing (tylko w ostatniej iteracji) → upload tripów → „all pairs” (skimy OD) → sprzątanie (`tsc_main.py:370-469`). Każdy krok zapisuje wiersz statusu do bazy (`write_status`, `:345-353`).
- **Dlaczego:** Lastenheft opisuje proces jako okresowe sprawdzanie bazy pod kątem nowych zleceń; system ma działać „permanent lauffähig automatisiert” na serwerze symulacyjnym (docx, rozdz. „Prinzipieller Ablauf”, „Einleitung”).
- **Czy dotyczy nas?** nie jako mechanizm: nie mamy żywej pętli z modelem popytu, a daemon oparty na bazie jest dla zespołu studenckiego zbyt ciężki. Warto natomiast przejąć zasadę, że każdy etap zostawia ślad statusu.

**Częściowe ponowne uruchomienie: flagi pomijania etapów + `--resume` (cache oparty tylko na istnieniu pliku)**
- **Zaobserwowane:** `t2s.py` ma flagi `-R/--no-rectify`, `--rectify-only`, `-M/--no-map`, `-T/--no-tripdefs`, `--map-and-exit` (`t2s.py:64-73`) oraz `--resume` („reuse existing files instead of overwriting them”, `:106-107`). W `t2s.main` każdy etap działa według schematu: jeśli jest `--resume` i plik wyjściowy istnieje, użyj go, w przeciwnym razie policz od nowa (`t2s.py:586-627`). Ten sam schemat występuje w `run_oneshot` (`assign.py:131-133`), `run_bulk` (`:247-249`), przy pobieraniu tripów (`tsc_main.py:425`) i przy uploadzie wyników (`:455-456`). Nazwy plików pośrednich są deterministyczne i pochodzą od nazwy wejścia: `rectified_<base>.csv`, `mapped_<base>.csv`, `miv_<base>.trips.xml` (`t2s.py:154-159`).
- **Dlaczego:** *(wnioskowane)* przebiegi są długie (Lastenheft zakłada do 24 h na przebieg SUMO), więc po awarii trzeba wznowić bez liczenia wszystkiego od początku.
- **Czy dotyczy nas?** częściowo. Deterministyczne nazwy etapów i flagi „stop po etapie X” są dobre. Cache oparty wyłącznie na tym, czy plik istnieje, nie wykrywa jednak zmiany wejść ani kodu, więc u nas potrzebny jest hash wejść lub DAG celów (Make/DVC).

**Instalator szablonów unieważnia cache na podstawie mtime**
- **Zaobserwowane:** `install_scenario_templates.py` przelicza `bidi.taz.xml.gz` i pliki TAZ tylko wtedy, gdy są starsze od `net.net.xml.gz` (`install_scenario_templates.py:267, 285, 307`). Sieć buduje się, wykonując `template_gen*.netccfg` w kolejności leksykalnej, a każdy kolejny krok czyta wynik poprzedniego (`:158-173`).
- **Dlaczego:** *(wnioskowane)* w praktyce jest to zależność typu Make ograniczona do jednej relacji (sieć → pochodne).
- **Czy dotyczy nas?** tak. Wzorzec „numerowane configi netconvert wykonywane po kolei” pasuje do naszego budowania sieci z OSM plus łatek.

**Iteracje są ściśle sekwencyjne**
- **Zaobserwowane:** `create_new_iteration_folder` tworzy `iterationNNN` tylko wtedy, gdy żądana iteracja jest bezpośrednim następcą najwyższej istniejącej. W przeciwnym razie kończy się asercją (`tsc_main.py:231-248`). W trybie plikowym numer iteracji ustala się z listy katalogów albo z `--iteration a:b` (`:113-127`).
- **Czy dotyczy nas?** częściowo. Jeśli kiedyś uruchomimy pętlę MATSim ↔ SUMO, katalog numerowany per iteracja z kontrolą ciągłości jest tanim zabezpieczeniem.

### 2. Konfiguracja i wiele scenariuszy

**Jedno źródło prawdy to tabela klucz–wartość w bazie + domyślne wartości w kodzie + nadpisania z CLI**
- **Zaobserwowane:** parametry przebiegu siedzą w `public.simulation_parameters(sim_key, param_key, param_value)` (schemat w `tests/tsc/data/initialState.sql:1-8`). Lista kluczy i wartości domyślnych jest w klasie `SP` (`constants.py:133-181`), m.in. `SUMO_TEMPLATE_FOLDER`, `SUMO_DESTINATION_FOLDER`, `SUMO_MODES`, `DB_TABLE_TRIPS`, `DB_HH_SAMPLE_SIZE`, `ITERATION`, `MAX_SUMO_ITERATION`, `DELETE_TEMP`, `SUMO_NET_PARAMETER`, `SLICE`. Kolejność scalania: `SP.OPTIONAL` → wiersze z bazy → `--sim-param K:V,...` z CLI (`get_trips.py:62-86`, `tsc_main.py:98-136`). Przedziały czasowe skimów (`SLICE`) pochodzą z osobnej tabeli `core.<DB_TABLE_MATRIXMAPS>` (`get_trips.py:75-82`).
- **Dlaczego:** Lastenheft: konfiguracja jest „Freitext” w `simulation_parameters`, a pliki są konfigurowane wyłącznie przez katalog bazowy `SUMO_TEMPLATE_FOLDER` (docx, rozdz. 2.1).
- **Czy dotyczy nas?** częściowo. Idea „`sim_key` = identyfikator przebiegu, a cała reszta to słownik parametrów pod tym kluczem” pasuje do naszego manifestu artefaktu, tylko słownik powinien być plikiem YAML/TOML w artefakcie, a nie wierszami w bazie.

**Scenariusz = katalog szablonu z hookami Pythona (`__init__.py`)**
- **Zaobserwowane:** każdy `data/<scenariusz>/` może mieć `__init__.py` definiujący `assign_trips(...)`, `post(...)` i opcjonalnie `trip_filter`. `tsc_main` importuje taki moduł dynamicznie z katalogu szablonów (`tsc_main.py:356-367, 435-437`). Przykład: `data/mitte_net/__init__.py:18-24` robi przypisanie oneshot na pełnej sieci Berlina, potem wycina podsieć Mitte i puszcza ją mikroskopowo. `data/mitte_test/__init__.py:17-30` używa domyślnego przypisania oraz postprocessingu dla pieszych i emisji. Dodatkowe opcje SUMO podaje się plikiem `extra.params` w katalogu scenariusza (`assign.py:146-148`). Te sygnatury są częścią kontraktu z Lastenheft (rozdz. 4.3, 4.5).
- **Dlaczego:** Lastenheft: dzięki elastycznej konfiguracji przypisania i postprocessingu własnymi skryptami interfejs jest „sehr variabel” i można go później rozszerzyć (docx, rozdz. 4.6).
- **Czy dotyczy nas?** częściowo. Plugin per scenariusz to sensowny punkt rozszerzeń dla wariantów przypisania ruchu, ale dynamiczny import z katalogu danych (poza pakietem) utrudnia `mypy` i review. U nas lepiej zrobić rejestr w pakiecie plus nazwę wariantu w configu.

**Warianty budowane z innego szablonu przez `setup.py` + „emulowane symlinki”**
- **Zaobserwowane:** `data/mitte_net/setup.py` kopiuje sieć, typy i TAZ z już zainstalowanego `berlin_net`, a potem wycina podsieć przez `netconvert --keep-edges.in-geo-boundary`. `data/siemensbahn/setup.py` dokleja nowe węzły i krawędzie (`siemensbahn.nod.xml`/`.edg.xml`) do sieci Berlina. Instalator uruchamia `setup.py <pre_dir> <template_dir>` (`install_scenario_templates.py:190-192`). Pliki tekstowe udają symlinki (np. `data/mitte_test/gtfs` zawiera `$TSC_HOME/data/berlin_net/gtfs`), a rozwiązuje je `get_symlink_dir` (`install_scenario_templates.py:96-102`).
- **Dlaczego:** *(wnioskowane)* dzielenie dużych wejść między scenariuszami bez duplikowania ich w git; Windows nie ma zwykłych symlinków (komentarz „emulate symlink”, `:137`).
- **Czy dotyczy nas?** częściowo. Wariant jako łatka na bazowy scenariusz (nowa infrastruktura = `.nod/.edg`) pasuje do scenariuszy „co-jeśli”. Symlinków w plikach tekstowych nie kopiować.

**Ograniczenia sieci jako parametr przebiegu (JSON)**
- **Zaobserwowane:** `SUMO_NET_PARAMETER`/`--net-param` zawiera JSON `{lista_krawędzi_lub_plik.taz@ID: {allowed:[...], maxSpeed:{vClass: v}}}`. Z niego `build_restricted_network` generuje `*.typ.xml` + `*.edg.xml` i przebudowuje sieć w katalogu roboczym przez `netconvert` (`tsc_main.py:140-181, 219-225`). Test: `tests/tsc/template/tscDaemonRestrictions/runner.py`.
- **Czy dotyczy nas?** raczej nie (scenariusze polityk transportowych to nie nasz cel), choć to wzorzec do wyrażania wariantów sieci jako diff.

### 3. Układ danych i duże pliki

**Trzy poziomy: `data/` (prekursory, w git/LFS) → `scenario_templates/` (zbudowane) → `scenario_workdir/<dest>/iterationNNN/` (przebiegi)**
- **Zaobserwowane:** `tsc_install` zamienia `data/<s>/` w `scenario_templates/<s>/` (`install_scenario_templates.py:105-206`). Przy iteracji 0 `tsc_main` kopiuje szablon do `scenario_workdir/<SUMO_DESTINATION_FOLDER>/` (`tsc_main.py:184-228`). Każda iteracja ma podkatalogi `trips/`, `oneshot/` lub `dua/`, `allpairs/` (widoczne w `tests/tsc/template/tscDaemonDatabaseTripsFromDB/catalogue.tsc`). `.gitignore` wyklucza `/scenario_templates`, `/scenario_workdir`, `/tsc.log`, build/dist. Duże wejścia (OSM, GTFS zip) są w Git LFS (wskaźniki w klonie). README odradza stare scenariusze z LFS i kieruje do repo `sumo-scenarios`, a Dockerfile instaluje już tylko z `sumo-scenarios` (komentarz „currently we only install from sumo-scenarios”, `Dockerfile`).
- **Dlaczego:** *(wnioskowane)* szablon jest drogi do zbudowania (sieć, TAZ, landmarki, GTFS), a przebieg ma prawo go modyfikować (ograniczenia), więc przebieg dostaje kopię.
- **Czy dotyczy nas?** tak. Podział na trzy poziomy „prekursor → zbudowany scenariusz → przebieg/iteracja” dobrze mapuje się na nasze `raw/` → `interim/` → `processed/`. Odejście DLR od LFS na rzecz osobnego repo scenariuszy potwierdza, że dużych danych nie trzymamy w monorepo.

**Opcje czyszczenia jako parametry przebiegu**
- **Zaobserwowane:** `DELETE_TEMP` zostawia z iteracji tylko końcowe trasy i wagi. `DELETE_INTERMEDIATE_RESULTS` po ostatniej iteracji usuwa wszystkie wcześniejsze (`tsc_main.py:328-342`). Testy: `threeIterationsDeleteTemp`, `threeIterationsDeleteNonFinal`.
- **Czy dotyczy nas?** częściowo. Jasna definicja, które pliki są „wynikiem”, a które tylko cache, przyda się przy publikacji artefaktu.

### 4. Proweniencja i wersjonowanie

**Wersjonowanie przebiegu: `sim_key` (znacznik czasu) + numer iteracji + log statusu. Wersji kodu nie ma nigdzie.**
- **Zaobserwowane:** `sim_key` ma postać znacznika czasu nadawanego przez TAPAS (np. `2015y_05m_29d_09h_01m_11s_943ms`). Tabela statusu `(sim_key, iteration, status_time, status, msg_type)` z kluczem głównym `(sim_key, status_time)` służy jako dziennik append-only (`initialState.sql:246-255`, `tsc_main.py:345-353`; `sleep(0.01)` pilnuje unikalności znacznika czasu). Nazwy tabel wynikowych to `<prefiks>_<sim_key>` **bez numeru iteracji** (`s2t_miv.py:163, 268, 283`). `create_table` robi `DROP TABLE IF EXISTS` przed utworzeniem (`database.py:130-147`), więc każda iteracja nadpisuje wynik poprzedniej. Lastenheft mówi coś innego: tabele mają mieć „den sim_key und die Iteration als Suffix” (docx, rozdz. 2.3.1–2.3.2). Wejściowa tabela tripów też nazywa się `<DB_TABLE_TRIPS>_<sim_key>` bez iteracji (`get_trips.py:151`). Nigdzie w wynikach nie zapisuje się hasha commita ani wersji pakietu. Wersja to tylko `version = 0.23.1` w `setup.cfg`. Brak CHANGELOG, `CITATION.cff` i tagów w klonie. GitHub Actions budują obraz Dockera co noc i przy tagach (`.github/workflows/docker.yml`).
- **Dlaczego:** *(wnioskowane)* środowisko operacyjne jednego instytutu, w którym historię trzyma baza, a o odtwarzalność dba ręcznie zespół.
- **Czy dotyczy nas?** tak, jako antywzorzec. Rozjazd specyfikacji i kodu (brak numeru iteracji w nazwie tabeli, nadpisywanie przez DROP) pokazuje, że wersja artefaktu musi być w jego identyfikatorze i w manifeście, a nie wynikać z konwencji nazewniczej.

**Częściowa proweniencja opcji SUMO: zapisywane są configi**
- **Zaobserwowane:** `run_oneshot` generuje tymczasowy `.sumocfg` i zapisuje pełną konfigurację przez `sumo --save-configuration` (`assign.py:219-234`). duaIterate zapisuje `duaIterate.cfg` (`:109-110`), `run_bulk` zapisuje `.duarcfg` (`:271`). Katalog przebiegu zawiera więc dokładne opcje narzędzi.
- **Czy dotyczy nas?** tak. Ten tani nawyk („zawsze zapisz rozwinięty config narzędzia obok wyniku”) warto przyjąć.

### 5. Źródło prawdy dla sieci

**Sieć SUMO jest kanoniczna i budowana z OSM przy instalacji; lokalizacje popytu są geo, a nie ID sieci**
- **Zaobserwowane:** sieć powstaje z `osm/*.osm.xml.gz` przez `template_gen0.netccfg` (import OSM: typy, usuwanie krawędzi, TLS `actuated`, łączenie skrzyżowań) i `template_gen1.netccfg` (łatki `netpatch/*.xml`), wywoływane łańcuchem (`install_scenario_templates.py:158-173`, `data/mitte_test/template_gen0.netccfg`). TAPAS nie zna ID krawędzi: podaje tylko współrzędne WGS84 i `taz_id`, a mapowanie na krawędź dzieje się przy każdym przebiegu (`t2s.map_to_edges`, `t2s.py:412-469`). Dzięki temu sieć można przebudować bez zmiany popytu.
- **Dlaczego:** Lastenheft: TS mapuje start i cel „möglichst genau aufgrund der Geokoordinaten auf das Netz” (docx, rozdz. 2.2.3).
- **Czy dotyczy nas?** **tak, kluczowe.** Kontrakt „popyt w geo + TAZ, mapowanie na krawędzie po stronie SUMO” uniezależnia artefakt popytu od wersji sieci SUMO. Rozwiązuje to problem spójności ID między siecią MATSim a SUMO kosztem deterministycznego map-matchingu.

**Spójność TAZ: dwa identyfikatory + preferencja krawędzi wewnątrz TAZ**
- **Zaobserwowane:** baza używa `taz_id`, a pliki `taz_num_id`. Mapowanie robi JOIN przy pobieraniu tripów (`get_trips.py:156-176`), zgodnie z Lastenheft („In allen Referenzen in der Datenbank wird taz_id genutzt, zur Referenz auf die TAZ in Dateien taz_num_id”). Pliki TAZ SUMO powstają z shapefile'i przez `polyconvert` + `edgesInDistricts`. Kolumnę ID wybiera się per shapefile opcją `-i` (domyślnie `*:NO,RBS_OD_ORT_1412:ORT,Berlin_1223:VBZ_ID`, `install_scenario_templates.py:69-70, 274-303`). `EdgeMapper.map_to_edge` szuka najbliższej krawędzi dopuszczalnej dla danej vClass w rosnącym promieniu (50 m, podwajany do `max_radius`), a krawędź należącą do TAZ tripu wybiera, jeśli jest nie dalej niż 2× najbliższa (`edgemapper.py:39-80`). Odstępstwa są logowane i liczone (`t2s.py:376-387, 465-469`). Ruch tła z ujemnym `taz_id` omija mapowanie i trafia na TAZ „suburb” (`t2s.py:357-364`).
- **Czy dotyczy nas?** tak. Rejony KBR i strefy macierzy komórkowych mają własne ID, więc potrzebujemy tabeli mapującej ID stref ↔ plik TAZ SUMO. Reguła „najbliższa krawędź, ale preferuj tę w deklarowanej strefie” ogranicza przeskoki przez granice stref.

**Wyjazd „w obu kierunkach”: `bidi.taz` zamiast pojedynczej krawędzi**
- **Zaobserwowane:** instalator generuje `bidi.taz.xml.gz` (`generateBidiDistricts`, `install_scenario_templates.py:264-271`). `tsc_main` zawsze ustawia `bidi_taz_file` (`tsc_main.py:392`), więc tripy dostają `fromTaz="<edgeID>"`/`toTaz` zamiast `from`/`to` (`t2s.py:494-499`). Fixture: `tests/tsc/template/public_transport/sumoTrips.tsc` (`fromTaz="-4588215#5"`).
- **Dlaczego:** *(wnioskowane)* punkt z geo leży zwykle przy ulicy dwukierunkowej. Pozwalając SUMO wybrać kierunek, unika się sztucznych objazdów i zawracania.
- **Czy dotyczy nas?** tak. To tani trik poprawiający realizm tripów po map-matchingu.

### 6. Przekazanie TAPAS → SUMO (analog naszego MATSim → SUMO)

**Wejście: płaska tabela tripów (osoba × trip), bez tras, z minutą odjazdu, trybem i typem pojazdu**
- **Zaobserwowane:** kolumny `TH.KEEP_COLUMNS` (`constants.py:25-63`): `p_id, hh_id, start_time_min, mode, lon_start, lat_start, lon_end, lat_end, travel_time_sec` (czas przejazdu według TAPAS), `taz_id_start, taz_id_end, activity_duration_min, car_type, is_restricted, sumo_type`. Klucz to `(p_id, hh_id, start_time_min)`. Tripy są sortowane po osobie i czasie (`get_trips.py:165`). Tryby są liczbami całkowitymi, a tryby złożone kodowane jako `256*a + b` (`bicycle_public = 261`, `car_public = 517`, `constants.py:118-128`). Fixture: `tests/tsc/data/twoPersonsOnly.csv`, czyli 2 osoby × 2 tripy (dom → praca → dom) z `activity_duration_min` i `sumo_type` typu `medium_passenger_gasoline`.
- **Czy dotyczy nas?** **tak.** To niemal gotowy minimalny schemat naszego artefaktu popytu. MATSim może z łatwością wyeksportować ekwiwalent (legi z planów wybranych).

**„Prostowanie” modyfikuje semantykę czasu i miejsca przed SUMO**
- **Zaobserwowane** (`t2s.rectify_input`, `t2s.py:183-338`):
  - **Czas:** jeden losowy offset na osobę, z przedziału ±`time_diffusion/2` (domyślnie 900 s, `:88`), wspólny dla wszystkich jej tripów (`:248`). Odjazd przed końcem poprzedniej aktywności (`depart + travel_time + activity_duration` poprzedniego tripu) jest przesuwany na ten koniec (`:281-290`). Tripy spoza okna dnia ±4 h są odrzucane (`TAPAS_DAY_OVERLAP_MINUTES`, `constants.py:19`; `t2s.py:266-273`). Nieposortowane tripy osoby kończą się błędem `MappingError` (`:292-295`).
  - **Przestrzeń:** szum gaussowski, większy dla współrzędnych, które się często powtarzają (od 500 do 5000 wystąpień: σ od 50 do 100 m), bo TAPAS „tends to show strong spatial clustering” (`:197-213, 312-320`).
  - **Ciągłość łańcucha:** jeśli cel poprzedniego tripu ≠ start bieżącego, bieżący trip jest **odrzucany** (chyba że podano `--ignore-gaps`, `:297-309`).
  - **Filtr trybów:** tylko `SUMO_MODES` (domyślnie `2;4`, czyli auto + taxi, `:322-325`).
- **Dlaczego:** komentarze w kodzie: wygładzenie „bursts” wynikających z rozdzielczości 5 min (`:185-188`), klastrowanie przestrzenne związane z parkowaniem (`:197-199`). Lastenheft pozwala przesunąć start w oknie 5 min pod warunkiem zachowania odniesienia do oryginału (docx, rozdz. 2.2.3).
- **Czy dotyczy nas?** tak. Te decyzje trzeba jawnie podjąć także u nas: czy jitter czasowy robi eksporter czy importer, czy jest per osoba czy per trip, co z lukami w łańcuchu. Tu dzieje się to po stronie importera, więc artefakt popytu zostaje „surowy”, a oryginalna minuta przetrwa w ID.

**Wyjście do SUMO: każdy trip to osobny `<trip>` lub osobny `<person>`. Łańcuch aktywności ginie.**
- **Zaobserwowane:** `create_sumo_tripdefs` (`t2s.py:474-560`): tryby auta (`2`, `4`) dają `<trip id depart from/to(Taz) type=sumo_type>`, a wszystkie pozostałe dają `<person depart><personTrip ... modes="public|car|taxi|bicycle"/></person>` (`:507-525`). Rower domyślnie staje się pieszym typu `ped_bike` (`:101, 508-512`). Tryb `6` (sharing) mapuje się na `modes="taxi"`. **Jedna osoba TAPAS z 2 tripami = 2 niezależne elementy XML**, bez planu i bez `<stop>` aktywności. Fixture `tests/tsc/template/public_transport/sumoTrips.tsc`: `1000005_1000001_817_0` i `1000005_1000001_850_0` to dwa osobne `<person>`. `activity_duration` służy tylko do poprawiania czasów odjazdu w prostowaniu. ID ma postać `p_id_hh_id_startmin_cloneIdx` (`common.py:85-87`), więc przez ID da się odtworzyć osobę i oryginalną minutę. Czasy są przesunięte o +24 h (`--shift-departure-hours 24`, `t2s.py:102-103, 505`; w fixture 813 min × 60 + 86400 − jitter = 135131). Parametry `taz_id_start/end` idą jako `<param>` (`:490-493`), bo potrzebuje ich agregacja OD po przebiegu. Tras nie ma: routuje SUMO (oneshot z `device.rerouting`, duaIterate/Gawron, marouter lub bulk duarouter, `assign.py`).
- **Zaobserwowane (rekonstrukcja łańcuchów, tylko w postprocessingu):** `postprocess.create_personfile` buduje z zroutowanych tripów osoby z planem `walk → ride(lines=uid) → stop(duration=activity) → ...` i pojazdami `depart="triggered"` (`postprocess.py:66-170`). Robi to wyłącznie w ostatniej iteracji w hooku `post` (`tsc_main.py:445-449`, `data/mitte_test/__init__.py:19-22`). Lastenheft chciał łańcuchów od początku („Die Abbildung von Wegeketten ist nötig…”, rozdz. 2.2.3), ale główny przebieg ich nie używa.
- **Dlaczego:** *(wnioskowane)* zwykłe tripy są zgodne z oneshot/duaIterate/marouter i z agregacją OD per trip. Łańcuch z pojazdem `triggered` łączy czasy kolejnych tripów i komplikuje przypisanie.
- **Czy dotyczy nas?** **tak, bezpośrednio.** Dla RL sterowania sygnalizacją wystarczą niezależne tripy pojazdów z oryginalnym czasem odjazdu (tak jak tu). Kontrakt artefaktu powinien jednak zachować `person_id`, `leg_index` i czas aktywności, żeby można było, tak jak tu, później zbudować wersję z łańcuchami. Warto też przyjąć konwencję ID niosącą osobę, oryginalny czas i indeks klona.

**Artefakt nie niesie tras; SUMO robi własne przypisanie**
- **Zaobserwowane:** domyślnie `--assignment oneshot`, czyli meso z `device.rerouting.probability=1`, okresem 300 s i adaptacją 0.5 (`t2s.py:84-86`, `assign.py:122-235`). Opcje `gawron`, `oneshot+gawron`, `marouter+...` realizuje `run_default` (`assign.py:376-385`). Wagi z poprzedniej iteracji **nie** są używane do startowego routingu kolejnej iteracji: `run_oneshot` i `run_duaiterate` ignorują argument `weight_file` (`assign.py:41-119, 123-235`). Iteracje „uczą się” wyłącznie przez TAPAS.
- **Czy dotyczy nas?** tak. To potwierdza wariant, w którym artefakt MATSim zawiera tylko OD + czas + tryb (bez tras), a równowagę w sieci SUMO liczy SUMO. Trzeba to świadomie wybrać zamiast importu tras MATSim, bo trasy MATSim dotyczą innej sieci.

### 7. Kalibracja między narzędziami

**SUMO → TAPAS: skimy czasów i dystansów OD per przedział czasu, uzupełniane „reprezentantami”**
- **Zaobserwowane:** po przypisaniu `run_all_pairs` (`tsc_main.py:251-325`) wykonuje kilka kroków. Najpierw agreguje wagi krawędzi z przebiegu do przedziałów `SLICE` (średnia harmoniczna ważona `sampledSeconds`, `s2t_miv.aggregate_weights`, `s2t_miv.py:44-86`). Następnie generuje sztuczne tripy między reprezentantami TAZ, maksymalnie 5 losowych par punktów na parę TAZ (`get_trips.write_all_pairs`, `get_trips.py:193-252`), i routuje je `duarouter --bulk-routing --weights <zatłoczone wagi>` (`assign.run_bulk`, `assign.py:238-274`). Na końcu łączy prawdziwe tripy z reprezentantami tak, by na każdą parę OD przypadało ≥5 próbek, i zapisuje `temp.sumo_od_<sim_key>` + `temp.sumo_od_entry_<sim_key>` (średnia, odchylenie standardowe, `realtrip_count`, `representative_count`) (`s2t_miv.py:214-284`; dla transportu publicznego: `s2t_pt.py:116-153`, z rozbiciem na dojście/oczekiwanie/przesiadki w `parse_person`, `:39-77`). Oprócz tego per trip powstaje `temp.sumo_trip_<sim_key>` z tablicami czasów i dystansów per tryb (`s2t_miv.py:153-184`). TAPAS czyta te tabele w następnej iteracji (strona TAPAS nie jest w repo).
- **Dlaczego:** Lastenheft: reprezentanci zapewniają „aussagekräftige Mittelwerte” dla par OD bez wystarczającej liczby tripów (rozdz. 2.2.4, 2.3.1). Rozdział 5.4 przewiduje walidację luftlinienfaktora TAPAS względem czasów z SUMO.
- **Czy dotyczy nas?** częściowo. Nie planujemy pętli zwrotnej, ale **skim czasów OD z SUMO** jest naturalnym narzędziem walidacji spójności MATSim ↔ SUMO (porównanie czasów przejazdu legów MATSim z czasami SUMO per para stref i godzina). Wzorzec „≥5 próbek na parę, uzupełnij reprezentantami” warto mieć na uwadze.

### 8. Próbka / skalowanie

**Skalowanie popytu przez klonowanie tripów (randomized rounding), bez skalowania przepustowości**
- **Zaobserwowane:** `DB_HH_SAMPLE_SIZE` (udział populacji symulowanej w TAPAS) daje `options.scale /= sample` (`tsc_main.py:434`). W `create_sumo_tripdefs` przy `scale > 1` powstaje `int(scale-1)` klonów plus jeden dodatkowy z prawdopodobieństwem równym części ułamkowej, a przy `scale < 1` tripy są losowo odrzucane (`t2s.py:481-488`). Klony dostają indeks w ID (`_1`, `_2`…) i **niezależny** jitter czasu (`:527-537`). Fixture `tests/tsc/template/tscDaemonDatabaseTripsFromDB/sumoTrips.tsc` przy próbce 0.5 (`initialState.sql:21`) pokazuje pary `_0`/`_1` z różnymi `depart`. Ruch tła jest przetwarzany przed zmianą skali (`tsc_main.py:394-405` vs `:434`), zgodnie z Lastenheft („Der sample_size Parameter findet auf diese Nachfrage keine Anwendung”). Wyniki są zapisywane z `clone_id` (klucz główny w `s2t_miv.py:170-180`), a `realtrip_count` obejmuje klony (Lastenheft 2.3.1). Parametrów SUMO związanych z przepustowością (`--scale`, flow/storage factor) nie ma.
- **Czy dotyczy nas?** **tak.** Przy próbce MATSim 10–25% ten sam wybór czeka nas w SUMO. Klonowanie daje 100% pojazdów (co jest kluczowe dla RL na skrzyżowaniach, bo kolejki i zdarzenia na detektorach), ale klony dziedziczą to samo OD. Kontrakt artefaktu musi nieść `sample_size`, a importer musi deterministycznie klonować (z seedem) i oznaczać klony w ID. Uwaga na drobiazg: klony liczą odjazd od `start_time_min` z twardym `+24*3600` zamiast `shiftdeparthours` i bez typu pieszego dla rowerów (`t2s.py:529-536`). To przykład, jak łatwo rozjechać ścieżkę „oryginał” i „klon”.

### 9. Testy i determinizm

**TextTest ze złotymi plikami + mały scenariusz „Mitte” + SQLite zamiast Postgresa**
- **Zaobserwowane:** `tests/runTests.py` uruchamia `texttest` dla dwóch aplikacji: `tsc` (Postgres) i `tsc.sqlite3` (`runTests.py:55`, `tests/tsc/config.tsc.sqlite3`: `TSC_SERVER:sqlite3.tsccfg`). Porównywane pliki (`collate_file`) to `rectifiedTrips`, `sumoTrips` (wygenerowany XML tripów), `resultingTripFile`, `resultingAllPairsFile` oraz stdout z filtrami zmiennych fragmentów (czasy, ścieżki, binaria) i tolerancją liczbową (`tests/tsc/config.tsc`, sekcje `[collate_file]`, `[run_dependent_text]`, `[floating_point_tolerance]`). Zestawy testów (`testsuite.tsc`) to `pre` (instalacja szablonu `mitte_test` z OSM bbox ~6 MB), `template` (pełny daemon od pustego workdiru: DB/plik, iteracje, ograniczenia, emisje, pieszy/sharing/PT, pusty plik), `workdir` (pojedyncze moduły na gotowym, zakomitowanym workdirze: `get_trips`, `t2s` z wariantami `rectify`, `wrongTaz`, `scale2`, `gawron`, `oneshotGawron`; `s2t_miv`) i `unitTest`. Syntetyczne dane to `tests/tsc/data/twoPersonsOnly*.csv`, `twoPersonsWrongTaz.csv`, `empty.csv`, 1000-wierszowa próbka oraz `initialState.sql` (~15 tys. linii: parametry, TAZ, cars, reprezentanci, tabela tripów). Znacznik `-- POSTGRESQL EXIT` pozwala nie nadpisywać danych referencyjnych na prawdziwym Postgresie (`initialState.sql:259-260`, `database.run_sql`, `database.py:150-178`). Deterministyczność zapewniają `random.seed(options.seed)` (domyślnie 23432, `t2s.py:87, 566`) i seed w `write_all_pairs` (`get_trips.py:194`). Mapper przy remisie odległości rozstrzyga po ID krawędzi (`edgemapper.py:57-69`).
- **Dlaczego:** Lastenheft rozdz. 8: testy automatyczne w TextTest na małym scenariuszu. README: testów na Postgresie nie wolno puszczać równolegle, bo dzielą bazę.
- **Czy dotyczy nas?** **tak.** Syntetyczna mini-populacja (2 osoby × 2 tripy) + mała sieć z OSM bbox + złoty plik wygenerowanego XML dla SUMO to dokładnie ten zestaw fixture'ów, którego potrzebujemy dla kontraktu artefaktu. Zamiast TextTest wystarczy pytest ze snapshotami.
- **Brak CI dla testów:** jedyny workflow to budowa obrazu Dockera (`.github/workflows/docker.yml`). Testy nie są uruchamiane automatycznie.

### 10. Dane zastrzeżone i licencje

- **Zaobserwowane:** kod ma licencję EPL-2.0 OR GPL-2.0+ (nagłówki plików, `LICENSE`). Dla GTFS VBB osobny `data/berlin_net/gtfs/readme.txt` podaje źródło i licencję CC-BY 4.0, czyli licencję per katalog danych, ale tylko w tym jednym miejscu. Zastrzeżone dane TAPAS (populacja, tripy) nigdy nie trafiają do repo: żyją w bazie, a w testach są tylko syntetyczne lub zanonimizowane próbki. Separacji danych nieredystrybuowalnych nie ma poza tym, że „dane są w bazie”. Lastenheft wspomina sieć Navteq/Elmar (komercyjną) w starych scenariuszach (`import_navteq.py`, pobierane z zewnątrz).
- **Czy dotyczy nas?** częściowo. Wzorzec „README z licencją obok każdego zestawu danych” jest tani i warto go mieć w `docs/dane/`.

### 11. Runtime Python ↔ Java

- Nie dotyczy bezpośrednio. TAPAS (Java) jest poza repo, a jedynym „runtime'em” wymiany jest baza. Po stronie SUMO pakiet wymaga `SUMO_HOME` i dopisuje `$SUMO_HOME/tools` (oraz `tools/assign`, `tools/route`) do `sys.path`, żeby importować `duaIterate`, `cutRoutes`, `gtfs2pt` itd. (`tsc_main.py:41-53`, `assign.py:26-35`, `install_scenario_templates.py:29-50`). Jako fallback próbuje importować pakiet pip `sumo`/`eclipse-sumo`.

### 12. Pakietowanie

**Jeden pakiet setuptools (src layout) z dwoma entry pointami; SUMO jako zależność zewnętrzna**
- **Zaobserwowane:** `pyproject.toml` zawiera tylko build-system. Metadane są w `setup.cfg`: `tapas-sumo-coupling 0.23.1`, zależności `rtree, pyproj, psycopg2, pandas, sumolib`, `console_scripts`: `tsc_main` → `tsc_main.main` i `tsc_install` → `install_scenario_templates.main`. Jest też `__main__.py`. `requirements.txt` przypina `sumolib>=1.27.0`, a `setup.cfg` nie przypina nic. `numpy` jest importowane (`s2t_pt.py:21`), ale nie zadeklarowane (przychodzi przez pandas). Moduły mają też własne `main()`/`argparse`, więc każdy etap (`t2s.py`, `get_trips.py`, `s2t_miv.py`) da się uruchomić osobno i testy z tego korzystają (`tests/tsc/tscdefs.py:10-16` wywołuje pliki `.py` bezpośrednio). Hooki scenariuszy (`data/*/__init__.py`) importują `assign`/`postprocess` bez prefiksu pakietu (`data/mitte_test/__init__.py:13-14`), co nie pasuje do przestrzeni nazw `tapas_sumo_coupling` *(wnioskowane: pozostałość po wcześniejszym płaskim układzie)*.
- **Czy dotyczy nas?** tak. Model „biblioteka z modułami etapów, z których każdy ma CLI + agregujący entry point” mapuje się na członka workspace'u `uv` (np. `packages/demand-export` lub `packages/sumo-import`) z `[project.scripts]`. Scenariusze i dane trzymamy poza pakietem, a hooki rejestrujemy w pakiecie, nie przez import z katalogu danych.

### 13. Dokumentacja

- **Zaobserwowane:** README opisuje instalację, uruchomienie i minimalne wymagania nowego scenariusza (`net.net.xml.gz` + `vtypes.xml`, opcjonalne TAZ, landmarki i `setup.py`; sekcja „Setting up a new scenario”). Interfejs z TAPAS jest wyłącznie w `docs/20210604LastenheftSUMO_TAPAS.docx` (po niemiecku, z historią wersji 0.1–1.6 w latach 2014–2021, opisami tabel, statusów i walidacji). Opis jest nieaktualny w szczegółach (sufiks iteracji, łańcuchy aktywności). Brak notatek per scenariusz poza `readme.txt` dla GTFS.
- **Czy dotyczy nas?** tak, jako przestroga. Kontrakt w `.docx` poza kodem rozjechał się z implementacją. Nasz kontrakt artefaktu powinien być wykonywalny (schemat + fixture'y w testach) i leżeć w `docs/decyzje/` w repo.

---

## Najważniejsze dla nas

1. **Kontrakt popytu = płaska tabela legów w geo, a nie ID sieci.** Kolumny: `person_id, household_id, depart_min, mode, lon/lat start/end, zone_start/end, activity_duration, vehicle_type` (+ `travel_time` z modelu popytu). Mapowanie na krawędzie SUMO po stronie importera (`edgemapper.py`, najbliższa krawędź z preferencją własnej strefy, `bidi.taz`) uniezależnia artefakt MATSim od wersji sieci SUMO.
2. **Świadomie zdecydować, co ginie w tłumaczeniu:** tsc zamienia każdy leg w niezależny `<trip>`/`<person>`, a łańcuch aktywności odtwarza tylko w postprocessingu (`postprocess.create_personfile`). Do RL sygnalizacji niezależne tripy pojazdów wystarczą, ale artefakt powinien zachować `person_id`, kolejność legów i czasy aktywności, żeby wersja z łańcuchami była możliwa później.
3. **Skalowanie próbki przez deterministyczne klonowanie z oznaczeniem w ID** (`p_h_min_cloneIdx`, randomized rounding, niezależny jitter). `sample_size` musi być polem manifestu artefaktu, a importer musi mieć seed. Ruchu tła nie skalujemy.
4. **Jitter czasu/przestrzeni i obsługa luk w łańcuchu to decyzje kontraktu,** nie szczegół implementacji. tsc robi je w imporcie (offset per osoba ±450 s, szum 50–100 m przy klastrach, odrzucanie tripów po luce) i zachowuje oryginalną minutę w ID. Trzeba to zapisać w `docs/decyzje/`.
5. **Fixture'y:** syntetyczne 2 osoby × 2 tripy + mały bbox OSM + złoty plik wygenerowanego `.trips.xml`, a zamiast Postgresa SQLite. To dokładnie ten zestaw testów kontraktu, który warto skopiować koncepcyjnie (pytest + snapshot).
6. **Postgres jako kolejka zleceń, konfiguracja i magazyn wyników nie jest nam potrzebny.** Jego rolę u nas przejmą: `sim_key` → wersja artefaktu (katalog/nazwa), `simulation_parameters` → manifest YAML w artefakcie, tabela statusu → log przebiegu, tabele wyników → pliki parquet. tsc sam ma tryb plikowy (`--fake-tripfile`, zapis CSV przy `conn is None`, `s2t_miv.py:164-167`) i backend SQLite. Antywzorzec do uniknięcia: tabele wynikowe nazywane bez numeru iteracji i nadpisywane przez `DROP TABLE` (`s2t_miv.py:163`, `database.py:134`), niezgodnie z własną specyfikacją.

## Nie znaleziono

- **Kodu TAPAS ani logiki, jak TAPAS konsumuje skimy** (czy robi MSA lub uśrednianie między iteracjami). Szukałem w całym repo (`grep` po `tapas`/`TAPAS`). Jest tylko opis tabel w Lastenheft.
- **Hasha commita ani wersji pakietu w wynikach przebiegu.** Szukałem `git`, `commit`, `__version__` w `src/`. Brak CHANGELOG, `CITATION.cff` i tagów (klon płytki).
- **Kryterium zbieżności pętli TAPAS ↔ SUMO w kodzie.** `MAX_SUMO_ITERATION` to tylko limit. Lastenheft 5.4 podaje cel („80% tripów < 10% odchylenia”), ale implementacji nie ma w `src/`.
- **Rozgrzewania SUMO wagami z poprzedniej iteracji.** `weight_file` jest ignorowane w `run_oneshot`/`run_duaiterate` (`assign.py`). Używa się go tylko do bulk-routingu reprezentantów.
- **Skalowania przepustowości (storage/flow factor, `--scale` SUMO)**. Szukałem `scale` w `src/`: skalowany jest tylko popyt.
- **Walidacji schematu wejściowego CSV** poza asercjami w trakcie przetwarzania (`MappingError`, `assert` w `checkOptions`). Brak jawnego schematu kolumn poza listą `TH.KEEP_COLUMNS`.
- **CI uruchamiającego testy.** `.github/workflows/` zawiera tylko `docker.yml`.
- **Zawartości scenariuszy LFS** (OSM Berlin, GTFS VBB). W klonie są tylko wskaźniki. Scenariusze produkcyjne są w osobnym repo `sumo-scenarios`.
