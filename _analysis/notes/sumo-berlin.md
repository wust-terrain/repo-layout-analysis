# sumo-berlin

- **Repo:** https://github.com/DLR-TS/sumo-berlin
- **Commit:** `049a642a77e37a8b20c104df0d7b1af8cb0a170b` (MANIFEST.tsv, klon z 2026-09-04; commit: Jakob Erdmann, 2026-09-04, „adapting to new default interpretation of GTFS mode 109”)
- **Licencja kodu:** EPL-2.0 (`LICENSE`)

**Czym jest to repo.** Roboczy scenariusz SUMO dla Berlina/Brandenburgii utrzymywany przez deweloperów SUMO w DLR. To nie jest pakiet ani pipeline z artefaktem wydania, tylko zbiór konfiguracji narzędzi SUMO (`*.netccfg`, `*.gtfscfg`, `*.sumocfg`, …) plus cienki skrypt `build.py`, który z odfiltrowanego wycinka OSM buduje sieć całego regionu (`netpatch/`), z niej sieć S-Bahn z rozkładem GTFS (`sbahn/`) oraz wycinek Treptow-Köpenick (`tk/`). Obok żyje lokalny podscenariusz `Doerpfeldstr/` (2026) — mały model ulicy Dörpfeldstraße kalibrowany do liczników Telraam, zbudowany ręcznie skryptami shell i w dużej mierze ręczną edycją sieci w netedit. README ma jedną linię.

**Częściowość klonu.** Klon `--filter=blob:none` + sparse-checkout (wzorce: `*.md *.py *.sh *.toml *.cfg *.yml *.yaml *.sumocfg LICENSE*`) i **płytki (depth 1)** — `git log` ma tylko jeden commit, więc częstotliwości zmian nie da się ocenić z historii. Pełny listing: `_trees/sumo-berlin.txt` (72 pliki). Blob-y plików spoza checkoutu były dostępne lokalnie, więc przeczytałem je przez `git show HEAD:<path>` (bez sieci) — dotyczy to m.in. całego `netpatch/`, `.gitignore`-ów, `tk/`, `gtfs/readme.txt`, plików XML w `Doerpfeldstr/` i `sbahn/`. `Doerpfeldstr/NOTES.md` **jest** w checkoucie. Nie czytałem zawartości binariów: `osm/bb_filtered.osm.xml.gz` (71,8 MB), `Doerpfeldstr/Doerpfeldstr_edit.net.xml.gz` (0,96 MB), `shapes/*.shp/.dbf`.

**Proxy aktywności (zamiast git log):** nagłówki „generated on” w plikach konfiguracyjnych: `netpatch/berlin.netccfg` 2019-01-22 (SUMO 1.0.1), `tk/tk.netccfg` 2022-09-06, `sbahn/sbahn.sumocfg` 2023-05-08, `sbahn/sbahn_gtfs.sumocfg` 2024-03-23, `Doerpfeldstr/*.sumocfg` 2026-03-22 … 2026-05-19 (SUMO 1.26.0+dev), ostatni commit 2026-09-04. Rdzeń (sieć regionu) jest stary i rzadko ruszany; aktywna praca w 2026 to `Doerpfeldstr/` i odświeżanie GTFS (`GTFS_VBB_2026_Apr_Dec.zip`).

---

### 1. Orkiestracja

**`build.py` = liniowy skrypt z flagami pomijania, bez cache'u**
- **Zaobserwowane:** `build.py:66-109` — `argparse` (przez `sumolib.options.ArgumentParser`) z flagami `--update` (pobierz OSM), `--check-landmarks`, `--sbahn` (tylko sieć S-Bahn), `--osmpt` (linie PT z OSM zamiast GTFS), `--skip-net`. Kolejność jest zakodowana: [opcjonalnie `update()`] → `netconvert netpatch/berlin.netccfg` (sieć regionu) → [landmarki] → drugi `netconvert` w `cwd='netpatch'` z wieloma opcjami inline wycinający sieć kolejową do `../sbahn/berlin-sbahn.net.xml.gz` (`build.py:88-98`) → `ptlines2flows.py` albo `gtfs2pt.py` w `sbahn/` (`build.py:99-107`) → `netconvert tk/tk.netccfg` (`build.py:108-109`). Każdy krok to `subprocess.check_call` narzędzia z `$SUMO_HOME`.
- **Dlaczego:** *(wnioskowane)* scenariusz jest cienką warstwą nad narzędziami SUMO; cała logika siedzi w plikach `*.cfg` narzędzi, więc skrypt tylko ustala kolejność.
- **Czy dotyczy nas?** częściowo — to styl „numerowane skrypty z `--skip-*`” w wersji minimalnej; jedyny „cache” to sprawdzenie istnienia pliku GTFS (`build.py:103`) — brak unieważniania, brak DAG; dla 15 osób za mało, ale pokazuje, że konfiguracje narzędzi (`*.netccfg`) mogą być głównym nośnikiem logiki.

**Konfiguracja kroków w plikach `*.cfg` narzędzi, nie w kodzie**
- **Zaobserwowane:** niemal każdy krok ma swój plik konfiguracyjny SUMO obok danych: `netpatch/berlin.netccfg`, `tk/tk.netccfg`, `sbahn/sbahn.gtfscfg`, `sbahn/sbahn.ptlfcfg`, `sbahn/persontrips.rtcfg`, `sbahn/berlin-sbahn-stations.gsecfg`, `sbahn/stations.netccfg`, `Doerpfeldstr/Doerpfeldstr.{gtfscfg,t2mdcfg,as2rcfg,tgcfg}`. Skrypty wywołują narzędzie z `-c <plik>`. Własne skrypty Pythona też używają `sumolib.options.ArgumentParser`, który czyta takie pliki (`Doerpfeldstr/telraam2meandata.py:45`, konfig `Doerpfeldstr.t2mdcfg`).
- **Dlaczego:** *(wnioskowane)* narzędzia SUMO natywnie obsługują `-c` i zapis konfiguracji (`--save-configuration`), więc to jest najtańszy sposób utrwalenia parametrów kroku.
- **Czy dotyczy nas?** tak — dla kroków SUMO (netconvert, `gtfs2pt`, `routeSampler`) commitowanie `*.netccfg`/`*.gtfscfg` jako parametrów etapu jest dobrym, samodokumentującym wzorcem; nasz orkiestrator powinien je wywoływać, a nie duplikować opcje w Pythonie.

**Niekonsekwencja: część opcji inline w Pythonie**
- **Zaobserwowane:** sieć S-Bahn nie ma własnego `*.netccfg` — ok. 10 opcji `netconvert` jest zaszytych w `build.py:89-98`, a za bazę służy `-c berlin.netccfg`. Z kolei `sbahn/build_persons.sh` (2 linie) i `sbahn/stations.netccfg` nie są wywoływane z `build.py`.
- **Dlaczego:** *(wnioskowane)* narastanie ad hoc.
- **Czy dotyczy nas?** tak (jako antywzorzec) — trzymać parametry w jednym miejscu na etap.

**Podscenariusz `Doerpfeldstr/`: skrypty shell bez kontroli błędów**
- **Zaobserwowane:** `Doerpfeldstr/buildNet.sh` (4 polecenia) i `Doerpfeldstr/buildRoutes.sh` (~20 poleceń) — sekwencje `python $SUMO_HOME/tools/...` bez `set -e`; nie są wołane z `build.py`. Wynik `buildNet.sh` to `Doerpfeldstr_merged.net.xml.gz`, ale wszystkie `*.sumocfg` i `buildRoutes.sh` używają `Doerpfeldstr_edit.net.xml.gz` — krok merged→edit nie ma skryptu. `run.sumocfg:10` odwołuje się do `ped_sampled.rou.xml`, którego generacja jest zakomentowana w `buildRoutes.sh:28-30`.
- **Dlaczego:** krok merged→edit to *(wnioskowane)* ręczna edycja w netedit (nazwa pliku, nagłówki „generated … by netedit” w `main_relations.xml`, `turn_probabilities.xml`, `joined.nod.xml`).
- **Czy dotyczy nas?** tak (antywzorzec) — przy wielu autorach niespójność skrypt↔`sumocfg` jest nieunikniona bez automatycznego sprawdzenia, że każdy plik z `sumocfg` ma producenta.

**Kopie zapasowe zamiast wersjonowania wejść**
- **Zaobserwowane:** `Doerpfeldstr/backup.py:18-30` czyta `sumocfg` (domyślnie `view.sumocfg`), zbiera `net-file`, `route-files`, `additional-files`, `gui-settings-file` i kopiuje je do `bak/%Y%m%d-%H%M%S/`; `bak` jest w `Doerpfeldstr/.gitignore:1`.
- **Dlaczego:** *(wnioskowane)* wygenerowane trasy nie są w git (`*.rou.xml` ignorowane), więc migawka „działającego stanu” jest robiona lokalnie.
- **Czy dotyczy nas?** częściowo — idea „zbierz wszystkie pliki, do których odwołuje się `sumocfg`” jest dobra jako podstawa pakowania artefaktu scenariusza (u nas z hashem i manifestem zamiast znacznika czasu).

### 2. Konfiguracja i wiele scenariuszy

**Podscenariusze = katalogi, każdy z własnymi configami; brak wspólnego źródła prawdy**
- **Zaobserwowane:** katalogi `netpatch/` (region), `sbahn/` (tylko kolej), `tk/` (wycinek Treptow-Köpenick), `Doerpfeldstr/` (ulica). Warianty uruchomieniowe to osobne `*.sumocfg`: w `sbahn/` — `sbahn.sumocfg`, `sbahn_gtfs.sumocfg`, `sbahn_osmlines.sumocfg`, `sbahn_osmlines_persons.sumocfg`; w `Doerpfeldstr/` — `run`, `miv`, `view`, `reroute` (tylko `device.rerouting.probability=0.1`), `gtfs_2024`. Wspólne parametry (data GTFS, sieć) są powielone: data `20260428` w `sbahn/sbahn.gtfscfg:3` i `gtfs/berlin.cfg:4`; `Doerpfeldstr` ma własną datę `20250310` (`Doerpfeldstr.gtfscfg:3`) i `2025-03-10` (`Doerpfeldstr.t2mdcfg:4`) w dwóch formatach.
- **Dlaczego:** *(wnioskowane)* każdy podscenariusz powstał w innym roku do innego celu.
- **Czy dotyczy nas?** tak — potwierdza, że „dzień referencyjny” (data GTFS, dzień zliczeń, dzień popytu) musi być jednym parametrem scenariusza, inaczej rozjeżdża się między plikami.

**Warianty wycinków wyrażone geometrycznie**
- **Zaobserwowane:** `tk/tk.netccfg:25` — `keep-edges.in-geo-boundary` z długim wielokątem lon/lat; ten sam wielokąt jest w `shape.py` (1417 linii, lista współrzędnych, komentarz `shape.py:1-2`: pobrany z uMap „teelraam-karte”, zapisuje `tkshape.poly.xml`). `Doerpfeldstr/Doerpfeldstr.geojson` definiuje obszar wycięcia przez `cut_net.py` (`buildNet.sh:3`). Zasięg regionu wyznacza bbox z `location_priorities.xml` + ręczny bufor na wschód (`build.py:14-21`, komentarz „Straussberg Nord”).
- **Dlaczego:** podano wprost dla bufora (`build.py:19`); reszta *(wnioskowane)* — wycinki odpowiadają obszarom projektów/liczników.
- **Czy dotyczy nas?** tak — obszar badania (np. korytarz z sygnalizacją) jako plik GeoJSON w repo, a nie zaszyty wielokąt w kodzie (tu ten sam wielokąt jest skopiowany w dwóch miejscach).

### 3. Układ danych i duże pliki

**Brak podziału raw/interim/processed; wszystko obok configów**
- **Zaobserwowane:** wejścia, pośrednie i wyjścia żyją w tym samym katalogu podscenariusza, rozdzielone wyłącznie przez `.gitignore`. Root `.gitignore`: `*.net.xml.gz`, `*ptlines.xml.gz`, `*stops.add.xml.gz`, `*.log`, konkretne pliki `sbahn/*.xml`. `osm/.gitignore`: `brandenburg-latest.osm.pbf`, `bb.o5m`, pośrednie. `sbahn/.gitignore`: `*.gz`, `*.zip`, `fcd`, `resources`. `Doerpfeldstr/.gitignore`: `/input`, `/resources`, `/fcd`, `/bak`, `bzm_vmk_*`, `bzm_telraam_*`, `*.rou.xml`, `gtfs*.zip`, `gtfs*.add.xml`, `parking*.xml`, `tile.jpg`.
- **Dlaczego:** *(wnioskowane)* ignorowanie wzorcami per katalog pozwala uruchamiać narzędzia „w miejscu” (`cwd=`), bez ścieżek do osobnego drzewa danych.
- **Czy dotyczy nas?** częściowo — my mamy dane zastrzeżone na dysku z `.env`, więc model „wszystko w katalogu repo” odpada; `.gitignore` per katalog podscenariusza to jednak rozsądne uzupełnienie globalnego.

**Commitowany snapshot odfiltrowanego OSM (72 MB) jako wejście kanoniczne**
- **Zaobserwowane:** `osm/bb_filtered.osm.xml.gz` (71 795 454 B) jest w git. Produkuje go `build.py:update()` (`build.py:11-26`): pobiera `brandenburg-latest.osm.pbf` z Geofabrik (bez daty w nazwie), `osmconvert` z bbox, `osmfilter` zostawiający tylko `highway= railway= cycleway= aeroway= waterway=`, zrzucający autorów i tagi `note/old_name/source/wikipedia`. `update()` uruchamia się tylko z `-u`; domyślny build używa commitowanego pliku (`netpatch/berlin.netccfg:9`).
- **Dlaczego:** *(wnioskowane)* Geofabrik „latest” nie jest odtwarzalny, więc odtwarzalność zapewnia commit przefiltrowanego wyniku; filtrowanie i `--drop-author` zmniejszają plik i szum w diffie.
- **Czy dotyczy nas?** tak — to najważniejszy wzorzec dla OSM: pobranie „latest” jest jawną, opcjonalną akcją, a kanonicznym wejściem jest zamrożony, przefiltrowany snapshot. U nas snapshot nie powinien jednak lądować w git (72 MB) — raczej w `raw/` na dysku / release z hashem i datą w manifeście.

**GTFS: snapshot nazwany okresem ważności, hostowany przez DLR, pobierany leniwie**
- **Zaobserwowane:** `build.py:103-105` pobiera `https://sumo.dlr.de/daily/GTFS_VBB_2026_Apr_Dec.zip` tylko, jeśli pliku nie ma; ta sama nazwa w `sbahn/sbahn.gtfscfg:2` i `gtfs/berlin.cfg:3`. Zip jest ignorowany (`sbahn/.gitignore:4`). W `Doerpfeldstr/` GTFS to `gtfs-2024.zip` (`Doerpfeldstr.gtfscfg:2`) — brak kodu pobierającego (ignorowany przez `gtfs*.zip`).
- **Dlaczego:** *(wnioskowane)* VBB publikuje kolejne okresy rozkładu pod zmiennymi URL-ami; DLR utrwala kopię pod stabilną nazwą z okresem. Commit HEAD („adapting to new default interpretation of GTFS mode 109”) pokazuje, że zmiana narzędzia (`gtfs2pt`) wymusza zmiany w scenariuszu.
- **Czy dotyczy nas?** tak — nazwa pliku wejściowego z okresem ważności + data dnia importu w configu to minimum proweniencji GTFS; brak sumy kontrolnej to luka, którą powinniśmy zamknąć.

**Dane z liczników pobierane w runtime przez skrypt**
- **Zaobserwowane:** `Doerpfeldstr/telraam2meandata.py:171-177` (`ensure_file`) ściąga z `https://berlin-zaehlt.de/csv/` pliki `bzm_telraam_<YYYY>_<MM>.csv.gz`, GeoJSON segmentów i `bzm_vmk_2023.json`, jeśli ich nie ma lokalnie (`:276-285`); wszystkie są w `.gitignore`.
- **Czy dotyczy nas?** częściowo — dla danych otwartych OK; dla danych ZDiUM (zastrzeżonych) pobieranie musi iść z dysku `.env`, nie z URL.

**Commitowana ręcznie edytowana sieć i nieużywane shapefile**
- **Zaobserwowane:** `Doerpfeldstr/Doerpfeldstr_edit.net.xml.gz` (965 KB) jest w git mimo globalnego `*.net.xml.gz` w `.gitignore:3` (*(wnioskowane)* dodany z `git add -f`). `shapes/Berlin_1223.*` (1,4 MB .shp) i `shapes/RBS_OD_ORT_1412.*` (dzielnice/Ortsteile Berlina z RBS, stan 12/2014, metadane ArcGIS w `.shp.xml`) — `git grep` nie znalazł żadnego odwołania w kodzie ani configach.
- **Czy dotyczy nas?** tak (ostrzeżenie) — wyjątki od `.gitignore` i osierocone pliki danych bez README to typowy dryf; warto mieć test „każdy plik danych w git jest referencjonowany/opisany”.

**Wyniki nie są publikowane**
- **Zaobserwowane:** brak release'ów, brak katalogu output, wszystkie `*.net.xml.gz` i `*.rou.xml` ignorowane. Użytkownik musi sam uruchomić `build.py` z działającym `$SUMO_HOME` (bez przypiętej wersji SUMO).
- **Czy dotyczy nas?** tak — u nas artefakt popytu ma być wersjonowany i publikowany; tu tego wzorca nie ma.

### 4. Proweniencja i wersjonowanie

**Brak wersji scenariusza; jedyna proweniencja to nagłówki generatorów**
- **Zaobserwowane:** brak `CHANGELOG`, `CITATION.cff`, tagów/release'ów (w płytkim klonie nie da się sprawdzić tagów; w listingu brak plików tego typu), brak zapisu hasha commita do wyników. Pliki zapisane przez narzędzia SUMO niosą komentarz „generated on <data> by Eclipse SUMO <narzędzie> v1_26_0+1616-c90c1dc4a14” (np. `Doerpfeldstr/run.sumocfg:3`) — czyli wersję *narzędzia*, nie scenariusza. Wersja SUMO nie jest nigdzie przypięta; `build.py:7` bierze to, co jest w `$SUMO_HOME`. Scenariusz idzie za SUMO-dev (nagłówki z buildów nightly, commit HEAD dostosowuje się do zmiany domyślnego zachowania `gtfs2pt`).
- **Dlaczego:** *(wnioskowane)* repo utrzymują autorzy SUMO, dla których scenariusz jest też testem integracyjnym bieżącej wersji narzędzi.
- **Czy dotyczy nas?** tak — pokazuje koszt braku przypięcia: zmiana domyślnych w SUMO zmienia wynik scenariusza bez zmiany w repo. Musimy przypiąć wersję SUMO (i `sumolib`) oraz zapisywać ją w manifeście artefaktu.

**Proweniencja wejść zapisana w nazwach i komentarzach**
- **Zaobserwowane:** `gtfs/readme.txt` (źródło VBB + licencja), nazwa `GTFS_VBB_2026_Apr_Dec.zip`, `shape.py:1` (URL uMap), `sbahn/gtfserrors_annotated` (ręcznie opisane ostrzeżenia importu GTFS: dlaczego trip S42 przy Charlottenburg jest usuwany i jak to obejść). OSM: brak daty snapshotu poza historią git pliku.
- **Czy dotyczy nas?** tak — `gtfserrors_annotated` to dobry, tani wzorzec: commitowany, opisany log znanych błędów importu. Datę pobrania OSM trzeba jednak zapisać jawnie (tu jej brak).

### 5. Źródło prawdy dla sieci

**Kanoniczna sieć = OSM snapshot + `netpatch/berlin.netccfg`; poprawki jako opcje netconvert, nie diff sieci**
- **Zaobserwowane:** `netpatch/` zawiera: `berlin.netccfg` (główny config netconvert), `ptLine.patch.xml`, `railDiscard.typ.xml`, `.gitignore`. Poprawki są wyrażone na trzech poziomach, wszystkie reaplikowane automatycznie przy każdym `netconvert` z OSM:
  1. **Opcje przetwarzania** w `berlin.netccfg:29-72` (`remove-edges.by-type`, `keep-edges.components`, `tls.guess-signals`, `tls.default-type=actuated`, `junctions.join`, `railway.topology.repair` …).
  2. **Nadpisania typów** (`netpatch/railDiscard.typ.xml:3-7`): odrzucenie torów `service.yard`, przywrócenie `railway.construction` jako przejezdnego, bo „some edges are in used by public transport lines” — ładowane jako trzeci plik obok typemap z `$SUMO_HOME` (`berlin.netccfg:10`).
  3. **Łatki linii PT kluczowane ID relacji OSM** (`netpatch/ptLine.patch.xml:3-8`: `ptLine id="2015944" period="1200"` z komentarzem „S5”) — wczytywane przez `ptline-files` (`berlin.netccfg:11`).
  Klasyczne łatki plain-XML (`patch.nod.xml`, `patch.edg.xml`, `patch.con.xml`, `patch.tll.xml`, `tls.unset`, `ramps.unset`, `junctions.join-exclude` z ID węzłów OSM) są **zakomentowane** (`berlin.netccfg:12-17, 33-36, 50-52, 56-58, 63-65`) i tych plików nie ma w listingu — mechanizm był przewidziany (config z 2019), ale obecnie nieużywany. `template_gen.netccfg` to kopia tego szablonu ze ścieżkami `netpatch/...`, niewołana z żadnego skryptu.
- **Dlaczego:** *(wnioskowane)* wyrażanie poprawek jako reguł (typy, opcje) i łatek kluczowanych stabilnymi ID OSM (węzły, relacje) przeżywa regenerację z nowszego OSM; ręczna edycja wygenerowanej sieci nie przeżywa.
- **Czy dotyczy nas?** tak — to jest wzorzec do przyjęcia: sieć SUMO nigdy nie jest źródłem prawdy, źródłem jest snapshot OSM + commitowane łatki (typemap, `*.nod/edg/con/tll.xml` kluczowane ID OSM) + config netconvert. Łatki sygnalizacji z geobazy ZDiUM powinny iść tą samą drogą (`tllogic-files`).

**Pochodne sieci jako deterministyczne transformacje sieci kanonicznej**
- **Zaobserwowane:** sieć S-Bahn to ten sam `berlin.netccfg` + filtry `--keep-edges.by-type/by-vclass` (`build.py:89-98`); `tk/tk.netccfg:9-11,25` wczytuje `../netpatch/berlin.net.xml.gz` + przystanki/linie i wycina wielokątem. ID krawędzi są dziedziczone z sieci regionu (ID OSM way + `#<segment>`).
- **Czy dotyczy nas?** tak — wycinki (korytarz RL, cały Wrocław dla MATSim) powinny być pochodnymi jednej sieci kanonicznej, żeby ID krawędzi były wspólne.

**Dryf ID po regeneracji: heurystyczna naprawa landmarków**
- **Zaobserwowane:** `build.py:29-63` (`check_edge`, `check_landmarks`, flaga `-l`): dla każdego ID z pliku `landmarks` (19 krawędzi) sprawdza, czy istnieje w nowej sieci; jeśli nie — próbuje sąsiednich indeksów segmentu `<osm_id>#<idx±1>`, samego `<osm_id>` i odwróconego kierunku (`-` prefiks); zapisuje `new_landmarks` do ręcznego porównania.
- **Dlaczego:** ID krawędzi netconvert = ID OSM way + indeks podziału, więc zmiana OSM (dodanie węzła) przesuwa indeks `#n` — skrypt to jawnie obsługuje.
- **Czy dotyczy nas?** tak, bezpośrednio — nasze artefakty referujące krawędzie (detektory ZDiUM, trasy z MATSim, mapowanie sterowników) będą łamane dokładnie w ten sposób; potrzebujemy walidacji „każde referowane ID istnieje w sieci” jako kroku pipeline'u, a mapowania lepiej trzymać geometrycznie (współrzędne → map-matching), nie jako gołe ID krawędzi.

**Kontrprzykład w tym samym repo: `Doerpfeldstr` — ręcznie edytowana sieć jako źródło prawdy**
- **Zaobserwowane:** `Doerpfeldstr/buildNet.sh:3-6` — wycina `tk.net.xml.gz` GeoJSON-em, usuwa krawędzie z `remove_edges.txt`, z **innej sieci spoza repo** (`../../Doerpfeldstr/net.net.xml.gz`) bierze krawędzie z `keep_edges.txt`, łączy obie i stosuje `joined.nod.xml` (13 łatek `<join nodes=…>` z ID węzłów i klastrów, zapisanych z netedit). Wynik `Doerpfeldstr_merged` → ręcznie → commitowany `Doerpfeldstr_edit.net.xml.gz`, którego używają wszystkie dalsze kroki. Wejścia kalibracyjne (`manual_car_data.xml`, `turn_probabilities.xml`, `main_relations.xml`, `Doerpfeldstr.as2rcfg:6`) referują konkretne ID krawędzi tej ręcznej sieci. `Doerpfeldstr/NOTES.md:2,9` — „Map matching is the important thing … if the inputs are not correctly matched it will not help”.
- **Dlaczego:** *(wnioskowane)* dla małego, intensywnie kalibrowanego obszaru szybciej jest poprawiać sieć w netedit niż utrzymywać łatki.
- **Czy dotyczy nas?** tak (ostrzeżenie) — dokładnie tego chcemy uniknąć: ręczna sieć nie jest odtwarzalna z repo (brak `../../Doerpfeldstr/net.net.xml.gz`), a każda regeneracja z OSM unieważnia ID w danych kalibracyjnych. Jeśli dopuścimy ręczne edycje, to tylko jako łatki plain-XML eksportowane z netedit i reaplikowane przez config.

### 6. Przekazanie MATSim → SUMO
Nie dotyczy — brak MATSim i `matsim_importPlans.py` w repo (szukałem w listingu i `git grep`).

### 7. Kalibracja między narzędziami
Nie dotyczy w sensie „symulator → symulator”. Jedyna kalibracja to dopasowanie tras SUMO do liczników w obrębie SUMO: `randomTrips.py` → `routeSampler.py` z danymi Telraam/VMK przekonwertowanymi do `meandata` (`Doerpfeldstr/buildRoutes.sh:11-25`, `telraam2meandata.py:72-110` — dopasowanie segmentu do krawędzi po nakładaniu buforów 20 m i kierunku) oraz ręczne liczby w `manual_car_data.xml` i prawdopodobieństwa skrętów w `turn_probabilities.xml`. Istotne dla nas pośrednio: konwersja pomiarów ZDiUM → `meandata` + `routeSampler` to naturalny krok walidacji/korekty popytu z MATSim.

### 8. Próbka / skalowanie
Nie dotyczy — brak reprezentacji próbki populacji; popyt osobowy w PT generowany losowo (`Doerpfeldstr/ptPersons.py:26-43`: 5–12 lub 5–250 osób na kurs, limit 85), `sbahn/persontrips.rtcfg:10` (`insertion-rate`).

### 9. Testy i determinizm
Brak testów i CI (listing: brak `tests/`, `.github/`). Determinizm częściowy: `randomTrips.py --seed 42` w `buildRoutes.sh:11,18,22,27`, ale `ptPersons.py:31,37` używa `random.randint` bez ziarna — wynik PT-persons nie jest powtarzalny.

### 10. Dane zastrzeżone i licencje

**Jedna licencja kodu, licencje danych opisane szczątkowo**
- **Zaobserwowane:** `LICENSE` = EPL-2.0 dla całego repo; nagłówki SPDX `EPL-2.0` tylko w nowych skryptach `Doerpfeldstr/*.py` (np. `backup.py:2-7`), brak w `build.py` i `shape.py`. Jedyna notka o licencji danych: `gtfs/readme.txt` (VBB, CC-BY 4.0) — co ciekawe, w katalogu `gtfs/`, gdzie nie ma samych danych. Commitowany wycinek OSM (`osm/bb_filtered.osm.xml.gz`, ODbL) nie ma pliku z atrybucją/licencją; `shapes/RBS_OD_ORT_1412.*` (dane RBS Berlin, „OpenData” w ścieżce w `.shp.xml`) i `shapes/Berlin_1223.*` — bez informacji o licencji. Dane Telraam/VMK z berlin-zaehlt.de nie są commitowane (`.gitignore`), a ich licencja nie jest opisana.
- **Dlaczego:** *(wnioskowane)* wszystkie dane są otwarte, więc problem separacji nie powstał.
- **Czy dotyczy nas?** tak — repo nie ma żadnego mechanizmu separacji danych nieredystrybuowalnych (nie jest mu potrzebny); dla nas wzorzec „`readme.txt` z źródłem i licencją per zbiór” jest minimum, ale musi towarzyszyć danym (u nas `docs/dane/<zbior>.md`), a nie pustemu katalogowi. Commit danych ODbL bez atrybucji to błąd, którego nie powtarzać.

### 11. Runtime Python ↔ Java
Nie dotyczy — brak Javy; skrypty dołączają `$SUMO_HOME/tools` do `sys.path` (`build.py:7`, wszystkie `Doerpfeldstr/*.py`).

### 12. Pakietowanie
Nie dotyczy — brak `pyproject.toml`/`setup.py`; zależności (`sumolib` z `$SUMO_HOME`, `shapely` w `telraam2meandata.py:35`, `osmconvert`/`osmfilter` w `build.py:20-22`) nie są zadeklarowane nigdzie.

### 13. Dokumentacja

**Notatki z doświadczeń obok scenariusza zamiast dokumentacji użytkowej**
- **Zaobserwowane:** `README.md` — jedna linia. `Doerpfeldstr/NOTES.md` (23 linie) to lista wniosków praktycznych (map matching, `preferences.xml` dla `randomTrips`, `turn_probabilities`/`main_relations` dla `routeSampler`, diagnoza pasów/łączników, `keepclear`, zablokowane pojazdy w insertion backlog, podgląd teleportów przez live edgedata), pomysły usprawnień narzędzi i zgłoszony bug sumo-gui. `sbahn/gtfserrors_annotated` — opis anomalii danych GTFS z wyjaśnieniem przyczyny. Komentarze inline w danych (`manual_car_data.xml:3-12` — nazwa ulicy i miejsce przy każdym ID krawędzi; `ptLine.patch.xml` — nazwa linii przy ID relacji OSM).
- **Dlaczego:** *(wnioskowane)* dokumentacja pisana przez autorów narzędzi dla siebie; brak odbiorcy zewnętrznego.
- **Czy dotyczy nas?** tak — dwa tanie wzorce warte przejęcia: (a) `NOTES.md` z wnioskami kalibracyjnymi obok podscenariusza (dobry materiał do prac MSc), (b) komentarz czytelny dla człowieka przy każdym surowym ID (ulica/linia) w plikach łatek i kalibracji. Brak README z instrukcją uruchomienia i wymaganiami (wersja SUMO, `osmconvert`, `osmfilter`, `shapely`) — tego nie powielać.

---

## Najważniejsze dla nas

1. **Kanoniczna sieć = zamrożony snapshot OSM + commitowane reguły/łatki + config netconvert** (`netpatch/berlin.netccfg`, `railDiscard.typ.xml`, `ptLine.patch.xml` kluczowane ID OSM). Poprawki reaplikują się automatycznie przy każdej regeneracji. Przyjąć to jako decyzję; łatki sygnalizacji ZDiUM jako `tllogic-files`/`*.nod.xml` w tym samym mechanizmie.
2. **Pobranie „latest” to jawna, opcjonalna akcja (`build.py -u`), a build domyślnie używa zamrożonego, przefiltrowanego snapshotu.** U nas: snapshot na dysku/release z datą pobrania i hashem w manifeście (tu daty brak, a 72 MB leży w git).
3. **ID krawędzi dryfują po regeneracji (`<osm_way>#<n>`)** — repo ma na to tylko heurystykę `check_landmarks` (`build.py:29-63`). Potrzebujemy automatycznej walidacji referencji (detektory, trasy z MATSim, dane kalibracyjne) względem nowej sieci i mapowania geometrycznego zamiast gołych ID.
4. **Antywzorzec `Doerpfeldstr/`:** ręcznie edytowana sieć w git, wejście spoza repo (`../../Doerpfeldstr/net.net.xml.gz`), dane kalibracyjne związane z ID tej sieci, `sumocfg` odwołujący się do niegenerowanego pliku. Pokazuje, co się dzieje bez reguły „każdy plik ma producenta w pipeline”.
5. **Brak przypięcia wersji SUMO** — commit HEAD to dostosowanie do zmiany domyślnej interpretacji GTFS w `gtfs2pt`. Wersja SUMO musi być przypięta i zapisywana w manifeście artefaktu.
6. **Parametry kroków w natywnych `*.cfg` narzędzi SUMO**, obok danych, plus opisane logi błędów importu (`gtfserrors_annotated`) i `NOTES.md` per podscenariusz — tanie i warte przejęcia; ale „dzień referencyjny” (GTFS/liczniki) trzymać w jednym miejscu, nie powielać w 3 plikach.

## Nie znaleziono

- Historii zmian — klon płytki (1 commit); częstotliwość oceniona tylko z nagłówków „generated on” w plikach.
- Plików łatek plain-XML (`patch.nod/edg/con/tll.xml`, `extra_edges.txt`, `keep_induction_loops.txt`) — referowane tylko w zakomentowanych liniach `netpatch/berlin.netccfg` i `template_gen.netccfg`; brak w `_trees/sumo-berlin.txt`.
- `sbahn/ped_access.nod.xml`, `ped_access.edg.xml` (wymagane przez `sbahn/stations.netccfg:5-6`) — brak w listingu.
- `Doerpfeldstr/net.net.xml.gz` (wejście `buildNet.sh:5`) — poza repo.
- Przypiętej wersji SUMO, deklaracji zależności, README z instrukcją — szukane w listingu (`pyproject.toml`, `requirements*.txt`, `environment.yml`, `Dockerfile`, `.github/`).
- CHANGELOG, CITATION, release'ów, hasha commita w wynikach — brak w listingu.
- Atrybucji/licencji dla commitowanego OSM i `shapes/`; licencji danych Telraam/VMK.
- Testów, CI, walidacji schematów (poza `xsi:noNamespaceSchemaLocation` w XML) — brak.
- Użycia `shapes/*`, `template_gen.netccfg`, `vtypes.xml`/`ptVtypes.xml` w root, `sbahn/build_persons.sh` z `build.py` — `git grep` nie znalazł odwołań (poza własnymi configami).
