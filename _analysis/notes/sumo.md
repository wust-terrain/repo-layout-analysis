# sumo (eclipse-sumo/sumo) — importer planów MATSim

- **Repo:** https://github.com/eclipse-sumo/sumo
- **Commit:** `ca13a385cf272bc17c03777423c4b19c54d088ba` (z `MANIFEST.tsv`, pobrane 2026-09-23)
- **Klon częściowy (sparse):** pobrane tylko `tools/import/matsim/`, `tests/tools/import/MATSim/`, `docs/web/docs/Tools/Import/`. Reszta repo znana tylko z listingu `_trees/sumo.txt` (226k linii) — tam opisana wyłącznie po nazwach plików.
- **Licencja:** EPL-2.0 OR GPL-2.0-or-later (nagłówek `tools/import/matsim/matsim_importPlans.py:2-12`). Poniżej nie ma skopiowanego kodu — tylko opis zachowania, identyfikatory i krótkie fragmenty fixtur jako dowód.

**Czym jest to repo:** SUMO to mikroskopowy symulator ruchu (C++) z dużym zestawem narzędzi w Pythonie (`tools/`, `sumolib`). Dla nas istotny jest jeden skrypt, `tools/import/matsim/matsim_importPlans.py` (276 linii), który zamienia plik planów populacji MATSim na plik tras SUMO (`*.rou.xml`), oraz 11 testów regresyjnych w `tests/tools/import/MATSim/`, które ustalają jego faktyczne zachowanie. Dokumentacja (`docs/web/docs/Tools/Import/MATSim.md`) to 3 zdania i jedno wywołanie — **nie jest tu źródłem prawdy**, źródłem prawdy są fixtury.

Konwencja testów (wnioskowane z zawartości): `options.tools` = argumenty CLI, `matsim_plans.xml` w katalogu testu = wejście (jeżeli go brak, test używa `tests/tools/import/MATSim/matsim_plans.xml` z katalogu nadrzędnego), `routes.tools` = oczekiwana zawartość `out.rou.xml`, `output.tools`/`errors.tools` = oczekiwane stdout/stderr (wszystkie puste, czyli skrypt ma działać bez ostrzeżeń).

---

### 1. Orkiestracja
Nie dotyczy (pojedynczy skrypt CLI; jedyna „orkiestracja” to opcjonalne wywołanie `duarouter` albo `sort_routes` na końcu, zob. wymiar 6).

### 2. Konfiguracja i wiele scenariuszy
Nie dotyczy (tylko opcje CLI przez `sumolib.options.ArgumentParser`, `matsim_importPlans.py:42-76`).

### 3. Układ danych i duże pliki
Nie dotyczy (w pobranej części repo są tylko małe fixtury XML w git).

### 4. Proweniencja i wersjonowanie

**Nagłówek wyjścia zawiera wersję narzędzia i opcje wywołania**
- **Zaobserwowane:** każdy oczekiwany plik wyjściowy zaczyna się komentarzem `generated on <data> by matsim_importPlans.py v1_21_0+0842-81aba47c0db options: ...` (np. `tests/tools/import/MATSim/import_person_plans/routes.tools:3-5`). Pisze go `sumolib.writeXMLHeader` (`matsim_importPlans.py:252`). W trybie `--vehicles-only` nagłówki są dwa, bo `sort_routes` dopisuje własny (`vehicles_only/routes.tools:3-9`).
- **Dlaczego:** żeby plik tras dało się odtworzyć i powiązać z wersją SUMO *(wnioskowane)*.
- **Czy dotyczy nas?** częściowo — ten nagłówek zapisuje wersję SUMO i opcje, ale nie wersję ani hash wejściowego artefaktu MATSim. To my musimy te dane dopisać do manifestu artefaktu.

### 5. Źródło prawdy dla sieci

**Importer zakłada, że ID linku MATSim jest tym samym co ID krawędzi SUMO, i niczego nie sprawdza**
- **Zaobserwowane:** atrybut `link` aktywności trafia bez zmian do `edge=`/`from=`/`to=` (`getLocation`, `matsim_importPlans.py:79-81`). Treść `<route type="links">` jest kopiowana dosłownie do `<route edges="...">` (`:122`), a `start_link`/`end_link` do `from`/`to` (`:199-200`). Skrypt w ogóle nie wczytuje sieci, chyba że podano `--repair`/`--remove-loops` (wtedy sieć czyta dopiero `duarouter`, `:263-268`). Fixtury: ID `13758 14271 ...` z `matsim_plans.xml:18` pojawiają się 1:1 w `import_person_plans/routes.tools:79`.
- **Dlaczego:** narzędzie zakłada, że sieć SUMO powstała z sieci MATSim przez `netconvert` (albo odwrotnie) z zachowaniem ID *(wnioskowane; dokumentacja tego nie mówi)*.
- **Czy dotyczy nas?** tak — to kluczowe ograniczenie kontraktu (zob. „Implikacje”).

**W SUMO istnieją importer i eksporter sieci MATSim (tylko z listingu)**
- **Zaobserwowane (`_trees/sumo.txt`, pliki niepobrane):**
  - `src/netimport/NIImporter_MATSim.{h,cpp}` — import sieci MATSim do `netconvert`;
  - `src/netwrite/NWWriter_MATSim.{h,cpp}` — eksport sieci SUMO do formatu MATSim;
  - `docs/web/docs/Networks/Import/MATsim.md`;
  - testy `tests/netconvert/import/matsim/` z wariantami `0_3_0_equil`, `_cap2lanes`, `_halflanes`, `_keep_length`, `_roundtrip` (ma `knownbugs.netconvert` i `foreign.netconvert`), `_selfloop`, `_validation_local` (każdy ma wejście `matsim.xml`);
  - `tests/complex/netconvert/matsim_roundtrip/` i `tests/complex/netconvert/matsim_download/runner.py`.
  - Z nazw wynika, że testowane są: wyznaczanie liczby pasów z przepustowości (`cap2lanes`), zachowanie długości linku (`keep_length`), pętle własne i roundtrip SUMO↔MATSim. Nazw opcji (`--matsim-files`, `--matsim.keep-length`, `--matsim.lanes-from-capacity`, `--matsim-output`) **nie zweryfikowałem w tym klonie** — podaję je z pamięci *(wnioskowane)*.
  - Istnieje też `tools/contributed/hybridPY/plugins/matsim/` (`matsim_network.py`, `matsim_demand.py`, `matsim_mapmatch.py`, `matsim_pt.py`, `matsim_config.py`, ...). To kod „contributed”, niepobrany.
- **Dlaczego:** żeby obie strony mogły używać wspólnych ID linków/krawędzi *(wnioskowane)*.
- **Czy dotyczy nas?** tak — `NWWriter_MATSim` (SUMO → MATSim) pasuje do wariantu, w którym kanoniczna jest sieć SUMO. Roundtrip ma jednak `knownbugs`, więc trzeba to sprawdzić na sieci Wrocławia.

### 6. Przekazanie MATSim → SUMO (NAJWAŻNIEJSZE)

#### 6.1 Opcje CLI (`matsim_importPlans.py:42-76`)

| Opcja | Domyślnie | Znaczenie |
|---|---|---|
| `-p/--plan-file` | wymagana | plik planów MATSim |
| `-o/--output-file` | wymagana | wyjściowy `.rou.xml` |
| `-n/--net-file` | — | sieć SUMO; potrzebna tylko z `--repair`/`--remove-loops` (bez niej skrypt wypisuje help i kończy działanie, `:72-74`) |
| `--vehicles-only` | false | same pojazdy, bez osób |
| `--no-bikes` | false | pomija legi `bike`/`bicycle` |
| `--no-rides` | false | pomija legi `ride` |
| `--repair` | false | przepuszcza wynik przez `duarouter --repair` |
| `--remove-loops` | false | przepuszcza wynik przez `duarouter --remove-loops` |
| `--prefer-coordinate` | false | gdy aktywność ma i `link`, i `x/y`, używa współrzędnych |
| `--default-start` | `0:0:0` | `depart` osoby, gdy pierwsza aktywność nie ma `start_time` |
| `--default-end` | `24:0:0` | `until` ostatniej aktywności, gdy nie da się go wyliczyć |
| `--default-dur` | `1:0:0` | domyślny czas trwania aktywności (używany też inaczej, zob. 6.4) |
| `-v/--verbose` | false | zadeklarowana, ale nigdzie nieużywana w `main` |

W całym skrypcie nie ma opcji: skali/próbki, filtra po podpopulacji, filtra po oknie czasowym, mapowania trybów na vClass, pliku pojazdów MATSim ani wyboru planu.

#### 6.2 Co jest czytane
- Plik czytany jest przez `sumolib.xml.parse` dwa razy: najpierw szukane są elementy `attributes` (CRS), potem `person` (`:129`, `:145`). Rozpoznawane są oba formaty:
  - v6: `<population>`/`<activity>`, `matsim_plans.xml:2-4`;
  - stary v4: `<plans>`/`<act>`, `alternative_format/matsim_plans.xml:2-10`, bo warunek to `"act" in item.name` (`:166`).
- **Plan:** zawsze `person.plan[0]`, czyli pierwszy plan w pliku, a **nie** plan z `selected="yes"` (`:148`). Atrybut `selected` jest ignorowany. Każda fixtura ma dokładnie jeden plan na osobę (`bikes_walks`: 10 osób, 10 `<plan>`, 10× `selected="yes"`), więc testy tego przypadku nie pokrywają.
- **CRS:** globalny atrybut `coordinateReferenceSystem`. Specjalny alias `GK4` → `EPSG:31468` (`:136`). Przeliczanie robi `pyproj`; gdy go brak albo CRS jest nieznany, pojawia się tylko ostrzeżenie na stderr i współrzędne idą dalej jako surowe x/y (`:137-141`).
- **Z aktywności używane są:** `type`, `link`, `x`, `y`, `start_time`, `end_time`, `max_dur`.
- **Z legu używane są:** `mode`, `dep_time`, `route` (tekst oraz `start_link`, `end_link`, `distance`).
- **Z osoby używane są:** `id` i blok `<attributes>`.
- **Ignorowane (nikt ich nie odczytuje):**
  - `plan@score`;
  - `leg@trav_time`, `route@trav_time`, `route@type`, `route@vehicleRefId`, `route@distance` (poza sprawdzeniem `"NaN"`);
  - atrybuty legu (`routingMode`) i aktywności (`zoneId`, `cemdapStopDuration_s`);
  - `facility`;
  - atrybuty globalne inne niż CRS;
  - osobowe atrybuty XML w formacie v4 (np. `employed="no"`).

#### 6.3 Co jest emitowane
- **`<vType id="<mode>"/>`** dla każdego trybu, który dostał pojazd. Nie ma żadnych parametrów, jedynie `bike`/`bicycle` dostaje `vClass="bicycle"` (`:253-257`). Fixtury: `bikes_walks/routes.tools:8-10` (`bicycle`, `car`, `ride`) i `coordinates/routes.tools:8-10` (`car`, `freight`, `ride`). Wynika z tego, że `freight` i `ride` dostają domyślną klasę SUMO (`passenger`) *(wnioskowane z semantyki SUMO)*.
- **Pojazd na każdy leg „pojazdowy”** o ID `<personId>_<indeks legu>`. Indeks liczy **wszystkie** legi, łącznie z pominiętymi walk/pt, stąd luki w numeracji: `10000001_1, _4, _7...` (`bikes_walks/routes.tools:11-35`) i brak `100029801_1` (`:331-333`). Postać zależy od legu:
  - `<vehicle>` z `<route edges=...>`, gdy leg ma `<route>` i `distance != "NaN"` i tryb nie jest rowerem (`:117-123`);
  - w przeciwnym razie `<trip from/to>` (albo `fromXY`/`fromLonLat`).
  - Rower **zawsze** staje się `trip`, nawet gdy ma trasę (`:117`).
- **`depart` pojazdu:**
  - w trybie osobowym (domyślnym) zawsze `"triggered"`, czyli pojazd rusza, gdy wsiądzie osoba (`:113`);
  - w `--vehicles-only` jest to `leg.dep_time` (`vehicles_only/routes.tools:13-38`).
- **`<person id depart>`**, gdzie `depart` = `start_time` pierwszej aktywności albo `--default-start` (`:152-154`). Wewnątrz:
  - `<param key value>` dla każdego atrybutu osoby, z escapowaniem przez `quoteattr` (`:213-216`; test `quote_params/routes.tools:21`: `value='{"car":"12"}'`);
  - `<stop edge=... until=... actType=...>` dla każdej aktywności;
  - między aktywnościami, zależnie od trybu legu:
    - `car`/`ride`/`bicycle`/`freight`/…: `<ride lines="<id pojazdu>" to=...>`, czyli osoba jest **pasażerem własnego pojazdu** (nie ma `<drive>`);
    - `walk`/`transit_walk`: `<walk to=...>`;
    - `pt`: `<personTrip modes="public" to=...>`;
    - `non_network_walk`: nic;
    - tryb odfiltrowany opcją: zakomentowane `<!-- ride ... mode=.../-->` (`:218-246`).
- **Sortowanie:** osoby (każda razem z własnymi pojazdami wypisanymi przed nią) sortowane są po `(depart osoby, kolejność w pliku)` (`:247-249`). Przykład: osoba `100009` z `start_time="00:01:00"` ląduje na końcu (`import_person_plans/routes.tools:84`). W `--vehicles-only` plik dodatkowo sortuje `sort_routes` po `depart` (`:269-270`). Z `--repair`/`--remove-loops` wynik przechodzi przez `duarouter` (`:263-268`); z `--prefer-coordinate` dostaje on też `--write-trips --write-trips.geo`.

#### 6.4 Semantyka czasów (tu jest najwięcej pułapek)
- **Odjazd legu z trasą:** liczy się `leg@dep_time` z MATSim, **a nie** `end_time` poprzedniej aktywności (`:194-203`). Fixtura: praca ma `end_time="24:00:00"`, a leg `dep_time="16:04:00"` (`matsim_plans.xml:20-22`), i wychodzi `<stop edge="14465" until="16:04:00">` (`import_person_plans/routes.tools:92`). Dokładniej w `coordinates_preferred`: aktywność `work` ma `end_time="13:27:00"`, następny leg z trasą ma `dep_time="08:00:00"` (`coordinates_preferred/matsim_plans.xml:19-20`), a wynik to `until="08:00:00"` (`routes.tools:18`). Niespójność wejścia nie jest nigdzie zgłaszana.
- **Odjazd legu bez trasy:** `dep_time` zostaje **nadpisany** wartością `end_time` poprzedniej aktywności (`:169`). Przykład: `alternative_format_vehicles_only/routes.tools:13` daje `depart="02:47:20"` = `end_time` aktywności.
- **Brak `end_time` aktywności** (`:175-191`), trzy przypadki:
  - jest `start_time` i `max_dur` → `end_time = start + max_dur`. Fixtura `activity`: `07:03:00 + 00:02:45`, wynik `until="07:05:45"` (`activity/routes.tools:11`); `08:22:25 + 00:06:35`, wynik `08:29:00` (`:15`);
  - jest tylko `start_time` → `end_time = start + --default-dur` (1h). Fixtura `coordinates`: ostatni `home` ma `start_time="14:05:00"` i dostaje `until="15:05:00"`, **nie 24:00** (`coordinates/routes.tools:18`);
  - jest tylko `max_dur` → `end_time = dep_time legu + default_dur + max_dur`, czyli 1h domyślnego czasu staje się przybliżeniem czasu podróży (`:186-191`). Żadna fixtura tego nie pokrywa;
  - gdy nic z tego nie zachodzi, `until` = `--default-end` (np. `until="24:0:0"`, `alternative_format/routes.tools:13`). `durations` (`:193`) są używane tylko wtedy, gdy `until` jest puste (`:236-240`).
- `trav_time` z MATSim jest całkowicie ignorowany, więc po imporcie SUMO samo liczy czasy przejazdu.
- **`depart` osoby:** w typowym wyjściu MATSim pierwsza aktywność nie ma `start_time`, więc prawie wszystkie osoby dostają `depart="0:0:0"` (`bikes_walks/routes.tools`, wszystkie 10 osób) i od t=0 czekają na krawędzi domowej jako „stop” *(wnioskowane: wszystkie osoby są wstawiane na starcie symulacji)*.

#### 6.5 Tryby

| Tryb MATSim | Pojazd w SUMO | Etap osoby | Dowód |
|---|---|---|---|
| `car` | `vehicle` (z trasą) albo `trip`; `type="car"` | `ride lines=` | `import_person_plans/routes.tools:9-26` |
| `ride` | **osobny pojazd** `type="ride"` z trasą (`vClass` domyślna) | `ride lines=` | `bikes_walks/routes.tools:170-175, 183` |
| `bike`/`bicycle` | zawsze `trip`, `vClass="bicycle"` | `ride lines=` | `bikes_walks/routes.tools:72-73` |
| `freight` (dowolny inny) | `trip`/`vehicle`, `type="freight"`, klasa domyślna | `ride lines=` | `coordinates/routes.tools:20-23` |
| `walk`, `transit_walk` | brak | `<walk to=>` | `bikes_walks/routes.tools:324, 340` |
| `non_network_walk` | brak | nic | `bikes_walks/routes.tools:38-40` („car interaction” → od razu `ride`) |
| `pt` | brak | `<personTrip modes="public" to=>` | `activity/routes.tools:10-19` |
| `ride` z `--no-rides` / rower z `--no-bikes` | brak | zakomentowany `ride` | diff `no_rides` / `no_bikes` vs `bikes_walks` |

Filtr trybów działa na sufiksie: `mode.endswith("walk")` oraz `mode == "pt"` (`:100`). Wszystko, co nie jest walk/pt, staje się pojazdem, łącznie z `ride`. Gdy `ride` był w MATSim teleportowany, w SUMO pojawia się dodatkowy pojazd na sieci *(wnioskowane)*.

#### 6.6 Tabela testów (fixtury = źródło prawdy)

| Test | Opcje | Wejście | Oczekiwane wyjście | Utracone / uwagi |
|---|---|---|---|---|
| `import_person_plans` | domyślne | wspólny `matsim_plans.xml` (v6, 5 osób, legi `car` z `route type="links"`, 1 `walk` z `distance="NaN"`, bez CRS) | `vType car`; na każdą osobę 1–2 `vehicle depart="triggered"` z `route edges` = link IDs; `person` z 5× `param`, `stop edge until actType`, `ride`, `walk` | `score`, `trav_time`, `distance`, x/y (bo jest `link`); `until` pracy = `dep_time` 16:04, a nie `end_time` 24:00; osoba `100009` przesunięta na koniec przez sortowanie |
| `quote_params` | domyślne | 1 osoba z atrybutem `vehicles` = JSON `{"car":"12"}` | `param value='{"car":"12"}'` (poprawny XML) | mapowanie MATSim vehicle `12` nie zostaje użyte, pojazd dostaje ID `100009_0` |
| `vehicles_only` | `--vehicles-only` | wspólny `matsim_plans.xml` | same `vehicle` z `depart=dep_time`, posortowane (08:00 ×5, potem 16:04…16:17); podwójny nagłówek | osoby, atrybuty, aktywności; leg `walk` osoby 100311 znika bez śladu (brak powrotu do domu) |
| `alternative_format` | domyślne | v4 `<plans>/<act>`, legi `car` bez trasy, bez `start_time` | `trip from/to` (link IDs), `person depart="0:0:0"`, ostatni `until="24:0:0"` | atrybut `employed="no"` (nie jest w `<attributes>`) |
| `alternative_format_vehicles_only` | `--vehicles-only` | jak wyżej | 3× `trip depart=<end_time aktywności>` | wszystko poza OD i czasem |
| `bikes_walks` | domyślne | GK4, 10 osób Berlin (open-berlin-like), `non_network_walk` + „car interaction”, `bicycle` z `route type="generic"`, 2× `ride` z trasą, `walk`, `transit_walk` | `vType bicycle(vClass=bicycle)/car/ride`; rowery jako `trip`; `ride` jako `vehicle type="ride"` z trasą; „car interaction” jako osobne `stop` | `non_network_walk` (dojście do parkingu) → brak etapu; `vehicleRefId`; `zoneId`, `cemdapStopDuration_s`; trasa rowerowa (i tak generic) |
| `no_bikes` | `--no-bikes` | jak `bikes_walks` (plik identyczny, `cmp`) | brak `vType bicycle` i tripów rowerowych; w osobie `<!-- ride ... mode="bicycle"/-->` | **plan osoby staje się nieciągły**: kolejne `stop` na różnych krawędziach bez etapu między nimi *(wnioskowane: SUMO odrzuci albo teleportuje taką osobę; nie sprawdzono)* |
| `no_rides` | `--no-rides` | jak wyżej | brak `vType ride` i pojazdów `ride`; zakomentowane `ride` | jak wyżej, nieciągłość planu |
| `coordinates` | domyślne | CRS `EPSG:25832`, aktywności tylko z x/y, legi bez tras (`ride`, `car`, `freight`) | `trip fromLonLat/toLonLat`, `stop lon= lat=`, `ride toLonLat=`; ostatni `home` `until=15:05` (start+1h) | UTM zamieniony na WGS84, więc SUMO musi dopasować punkt do krawędzi; `freight` staje się zwykłym pojazdem |
| `coordinates_preferred` | `--prefer-coordinate` | aktywności z `link` i x/y; leg `car` z trasą | aktywności jako lon/lat mimo `link`; leg z trasą **nadal** jako `vehicle route edges` | `link` aktywności; `until` pracy = `dep_time` legu (08:00) wbrew `end_time` 13:27 |
| `activity` | domyślne | 1 osoba, same legi `pt` (bez tras, z `routingMode`), `max_dur` | brak `vType`; 5× `personTrip modes="public" toLonLat=`; `until` = start+max_dur | `routingMode`; całe PT (linie, przystanki) — SUMO musi to przeliczyć samo |

`testsuite.tools` zawiera tylko listę 11 powyższych testów z jednolinijkowymi komentarzami (np. „ensure that param values are legal xml”, „import coordinates despite having edge ids”).

#### 6.7 Co ginie w tłumaczeniu (z dowodami)
1. **Wybór planu:** brany jest `plan[0]`, a nie plan `selected` (`:148`). Wejście z wieloma planami na osobę (np. `output_plans.xml` bez filtra) da w SUMO pierwszy, niekoniecznie wybrany plan. Żadna fixtura tego nie testuje.
2. **Trasy spoza sieci drogowej:**
   - trasa `generic` (walk, bicycle, pt) jest wyrzucana;
   - rower jest zawsze `trip`, nawet z trasą (`:117`);
   - trasy PT (`default_pt`/`experimentalPt1`: linia, przystanki) nie są czytane w ogóle, zostaje tylko cel (`personTrip modes="public"`) (`:228-229`);
   - `distance="NaN"` wymusza `trip` (`:117`).
3. **Semantyka odjazdu:**
   - w trybie osobowym pojazdy mają `triggered`, czyli czas odjazdu wynika z `until` poprzedniego `stop`;
   - dla legów z trasą `until` = `leg@dep_time`, dla legów bez trasy = `end_time` aktywności;
   - `trav_time` jest ignorowany;
   - brakujący `end_time` jest „zmyślany” z `--default-dur` (1h), także jako przybliżenie czasu podróży (`:186-191`);
   - brakujący `start_time` pierwszej aktywności daje `depart=0:0:0` dla wszystkich.
4. **Osoby vs pojazdy:**
   - w trybie osobowym osoba jest pasażerem (`ride lines=`) pojazdu `triggered`, a nie kierowcą;
   - każdy leg dostaje **nowy** pojazd (`<person>_<n>`), więc nie ma ciągłości pojazdu między legami ani `vehicleRefId` (`bikes_walks/matsim_plans.xml`: 36× `vehicleRefId`, w wyjściu 0);
   - w `--vehicles-only` osoby znikają, a legi walk/pt nie zostawiają śladu.
5. **Tryby:**
   - `ride` staje się pełnoprawnym pojazdem na sieci (może podwoić ruch, jeśli kierowca też jedzie jako `car`) *(wnioskowane)*;
   - `freight` i każdy nieznany tryb dostaje pusty `vType` bez `vClass` (brak ciężarówek);
   - `non_network_walk` znika;
   - filtry `--no-*` psują ciągłość planów osób.
6. **Lokalizacja:**
   - x/y są odrzucane, gdy jest `link` (bez `--prefer-coordinate`);
   - z `--prefer-coordinate` legi z trasą i tak używają link IDs, więc wyjście łączy dwa systemy odniesienia (`coordinates_preferred/routes.tools:12-18`);
   - bez `pyproj` wyjście ma surowe x/y (`fromXY`), które pasują do sieci SUMO tylko przy zgodnym offsecie/projekcji *(wnioskowane)*.
7. **Atrybuty:**
   - atrybuty osoby są zachowane jako `param`, ale tracą typ (`class`);
   - atrybuty aktywności i legów, `score`, `facility` oraz atrybuty globalne (poza CRS) są tracone;
   - w v4 atrybuty w samym tagu `person` są tracone.
8. **Brak walidacji:** nie ma sprawdzenia, czy krawędzie istnieją, czy trasa jest spójna, ani czy `dep_time` zgadza się z `end_time`. Błędy wyjdą dopiero w `sumo`/`duarouter`. Opcjonalne `--repair`/`--remove-loops` przez `duarouter` to jedyna naprawa.
9. **Brak skali/próbki:** nie ma żadnej informacji o udziale próbki ani wag osób (wymiar 8).

#### 6.8 Implikacje dla kontraktu naszego artefaktu
- **Zgodność ID sieci jest warunkiem koniecznym, jeśli emitujemy trasy.** Albo sieć MATSim powstaje z kanonicznej sieci SUMO (np. przez `NWWriter_MATSim` / `netconvert --matsim-output` *(nazwa opcji niezweryfikowana)*, albo narzędziem po stronie MATSim) i jest wersjonowana razem z artefaktem, albo artefakt nie niesie tras. Kontrakt powinien zawierać hash sieci SUMO, z której wyprowadzono sieć MATSim, oraz test „każdy link w trasach ∈ krawędzie SUMO”.
- **Zalecenie: eksportujemy sami, nie przez `matsim_importPlans.py`,** albo używamy go tylko jako punktu odniesienia. Skrypt bierze `plan[0]`, nie obsługuje skali ani filtrów i ma nieoczywistą semantykę czasów. Własny eksporter (Python, w naszym workspace) powinien:
  1. brać wyłącznie plan `selected="yes"`;
  2. emitować `depart` pojazdu jawnie z `end_time` aktywności/`dep_time` legu i zapisywać politykę uzupełniania brakujących czasów w manifeście;
  3. jawnie mapować tryby na `vType`/`vClass` (np. `freight` → `truck`/`delivery`);
  4. mieć politykę `ride` (pominąć, bo ruch generuje kierowca? zdecydować w `docs/decyzje/`).
- **Dla RL sterowania sygnalizacją wystarczy zapewne tryb „vehicles-only” (car + ewentualnie freight):** pojazdy z `depart` = czas z MATSim, posortowane. Osoby, PT i piesi są potrzebni tylko, jeśli modelujemy pieszych na skrzyżowaniach lub transport publiczny (np. priorytet tramwajów) *(wnioskowane z naszego kontekstu)*.
- **Trasy vs OD:** trzeba wybrać jedno i zapisać to w kontrakcie.
  - (a) `route edges` z MATSim (równowaga MATSim, ale warunek identycznych ID i spójności połączeń w SUMO);
  - (b) `from`/`to` (krawędzie lub `fromLonLat`) plus trasowanie w SUMO (`duarouter`/`duaIterate`). Tu zachowujemy tylko OD i czasy, a równowagę przeliczamy w SUMO.
  - Rozsądnie jest dostarczać **oba** (link IDs plus współrzędne w EPSG:2180/WGS84 z deklarowanym CRS), żeby konsument mógł wybrać. Jeśli współrzędne, to CRS musi być w pliku (importer czyta tylko `coordinateReferenceSystem`) i wymaga `pyproj`.
- **Przejście przez `duarouter --repair`** traktować jako obowiązkowy krok walidacyjny po stronie SUMO, a liczbę naprawionych/odrzuconych tras raportować jako metrykę jakości przekazania.
- **PT:** trasy PT z MATSim są w tym narzędziu nie do przeniesienia. Rozkład PT w SUMO musi pochodzić z GTFS (osobno, `docs/web/docs/Tools/Import/GTFS.md` istnieje w klonie), a pasażerowie jako `personTrip modes="public"`.
- **Fixtury SUMO jako wzorzec testów:** małe, ręcznie czytelne pliki planów (1–10 osób) z oczekiwanym `*.rou.xml` to dobry wzorzec dla testów golden-file naszego eksportera (zob. 9).

### 7. Kalibracja między narzędziami
Nie dotyczy. Importer nie przenosi żadnych parametrów (np. przepustowości, `flowCapacityFactor`, czasów przejazdu); `trav_time` jest ignorowany (`:109-123`).

### 8. Próbka / skalowanie

**Brak jakiejkolwiek obsługi próbki**
- **Zaobserwowane:** brak opcji skali w `get_options` (`:42-76`). Każda osoba/leg staje się dokładnie jednym obiektem SUMO.
- **Dlaczego:** narzędzie jest prostym konwerterem 1:1 *(wnioskowane)*.
- **Czy dotyczy nas?** tak. Jeśli MATSim działa na próbce (np. 10%), skalowanie trzeba rozwiązać po stronie SUMO (`--scale` w `sumo`, niezweryfikowane w tym klonie) albo w naszym eksporterze. Udział próbki musi być polem manifestu artefaktu.

### 9. Testy i determinizm

**Testy golden-file dla konwertera: minimalne wejście → dokładne oczekiwane wyjście**
- **Zaobserwowane:** 11 katalogów w `tests/tools/import/MATSim/`, każdy z `options.tools` (CLI), opcjonalnym `matsim_plans.xml` i oczekiwanym `routes.tools`. Puste `output.tools`/`errors.tools` wymuszają brak komunikatów. Lista testów jest w `testsuite.tools`. Wejścia mają 1–10 osób; testy celują w jedną cechę każdy (format v4, cytowanie atrybutów, CRS, tryby, czasy aktywności). Wyjście jest deterministyczne dzięki stabilnemu sortowaniu po `(depart, indeks)` (`:247-249`). Znacznik czasu w nagłówku (`generated on ...`) framework testowy (TextTest) musi ignorować *(wnioskowane)*.
- **Dlaczego:** regresje w konwersji wychodzą na jaw jako diff tekstowy.
- **Czy dotyczy nas?** tak — dokładnie tak powinniśmy testować własny eksporter MATSim→SUMO: syntetyczne mini-populacje (dopuszczalne w git, bez danych KBR) i porównanie z oczekiwanym `rou.xml`. Znaczniki czasu i wersje trzeba normalizować przed porównaniem.

**Luki pokrycia testami (nasza lista przypadków do sprawdzenia)**
- **Zaobserwowane:** brak testu dla:
  - wielu planów na osobę;
  - `--repair`/`--remove-loops`;
  - aktywności z samym `max_dur`;
  - PT z trasą `experimentalPt1`/`default_pt`;
  - `--vehicles-only` z trybami mieszanymi.
- **Czy dotyczy nas?** tak — nasze testy powinny pokryć właśnie te przypadki.

### 10. Dane zastrzeżone i licencje
Częściowo. Cały kod SUMO ma podwójną licencję EPL-2.0 OR GPL-2.0-or-later (nagłówki SPDX w plikach, `matsim_importPlans.py:12`; `LICENSE` w root wg `_trees/sumo.txt:37`). Wywoływanie SUMO jako zewnętrznego narzędzia nie narzuca licencji na nasz kod *(wnioskowane)*; kopiowanie kodu skryptu do naszego repo już tak. Fixtury pochodzą z danych Berlin/Lipsk (EPSG:25832 i GK4), bez danych zastrzeżonych.

### 11. Runtime Python ↔ Java
Nie dotyczy (importer jest czystym Pythonem na `sumolib`; nie wywołuje MATSim/Java, tylko opcjonalnie binarkę `duarouter` przez `sumolib.checkBinary`, `:268`).

### 12. Pakietowanie
Częściowo. Skrypt nie jest pakietem: dopisuje `$SUMO_HOME/tools` i `tools/route` do `sys.path` (`:35-39`) i importuje `sumolib` oraz `sort_routes` (`tools/route/sort_routes.py` wg `_trees/sumo.txt:226254`). **Czy dotyczy nas?** Jeśli użyjemy tego skryptu albo `sumolib`, pakiet w workspace `uv` musi zależeć od `sumolib`/`eclipse-sumo` z PyPI albo od `SUMO_HOME`. To trzeba przypiąć (wersja SUMO w manifeście).

### 13. Dokumentacja
Nie dotyczy w praktyce. `docs/web/docs/Tools/Import/MATSim.md` ma 3 zdania: opisuje wywołanie i `--vehicles-only`, nie wspomina założenia o ID sieci, wyboru planu ani semantyki czasów. Szczegółowa dokumentacja importu sieci (`docs/web/docs/Networks/Import/MATsim.md`) istnieje, ale nie została pobrana.

---

## Najważniejsze dla nas
1. **Kontrakt ID:** `matsim_importPlans.py` kopiuje link IDs MATSim bez żadnej walidacji jako krawędzie SUMO (trasy, `from`/`to`, `stop edge`). Artefakt z trasami ma sens tylko przy sieci MATSim wyprowadzonej z kanonicznej sieci SUMO (albo odwrotnie). Hash sieci i test przynależności ID muszą być częścią kontraktu.
2. **Importer bierze `plan[0]`, nie plan wybrany,** i ignoruje `selected`. Artefakt powinien zawierać wyłącznie wybrane plany albo trzeba mieć własny eksporter.
3. **Czasy:** `until`/`depart` bierze `dep_time` legu dla legów z trasą, `end_time` aktywności dla legów bez trasy, a luki łata heurystyką `--default-dur` = 1h (fixtury `coordinates`, `coordinates_preferred`, `activity`). Kontrakt powinien wymagać jawnych, spójnych czasów odjazdu dla każdego legu.
4. **Tryby:** walk/pt nie dostają pojazdów (PT jako `personTrip`, trasy PT tracone); `ride` staje się **dodatkowym pojazdem**; `freight` i tryby spoza listy dostają pusty `vType`; `--no-bikes`/`--no-rides` zostawiają nieciągłe plany osób. Mapowanie tryb → `vType`/`vClass` i politykę `ride` musimy zdefiniować sami.
5. **Brak skali, próbki, walidacji i proweniencji wejścia** — nasz manifest musi to nieść. `duarouter --repair` warto traktować jako obowiązkowy krok kontrolny, z raportem.
6. **Wzorzec testów golden-file** (11 mini-fixtur, każda na jedną cechę) jest bezpośrednio do przejęcia jako wzorzec (nie kod) dla testów naszego eksportera.

## Nie znaleziono
- Obsługi wielu planów / wyboru `selected` — sprawdzone w `matsim_importPlans.py` (tylko `person.plan[0]`, `:148`) i we wszystkich fixturach (zawsze 1 plan).
- Importu pojazdów MATSim (`vehicles.xml`, `vehicleRefId`) i typów pojazdów — brak w skrypcie; `vehicleRefId` obecny w `bikes_walks/matsim_plans.xml`, ale nieużywany.
- Obsługi tras PT (`transitRouteId`, `accessFacilityId`) — brak w skrypcie i fixturach.
- Importu eventów MATSim (np. `output_events.xml`) do SUMO — `grep -i matsim _trees/sumo.txt` pokazuje tylko importer planów, import/eksport sieci (`NIImporter_MATSim`, `NWWriter_MATSim`), testy `netconvert` i plugin `hybridPY`.
- Opcji skali/próbki — brak w `get_options`.
- Treści `NIImporter_MATSim.cpp`, `NWWriter_MATSim.cpp`, `docs/web/docs/Networks/Import/MATsim.md` i testów `netconvert` — nie pobrane (sparse); opisane tylko po nazwach.
