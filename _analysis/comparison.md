# Porównanie repozytoriów referencyjnych (Faza 2)

Źródła: 12 notatek w `_analysis/notes/*.md` (Faza 1) i stan repozytoriów zapisany w `MANIFEST.tsv`. Wszystkie odwołania `plik:linia` pochodzą z notatek. Kluczowe twierdzenia zweryfikowałem ręcznie w kodzie (lista na końcu, sekcja „Weryfikacja”).

**Ograniczenia materiału dowodowego** (dotyczą całego dokumentu):
- **Brak `pam-sdist`.** Pobranie się nie udało (`MANIFEST.tsv`: `FAILED`), więc PAM nie jest przeanalizowany. Pośrednio: `matsim-python-tools` nie zależy od pam i pokrywa tylko wąski wycinek jego funkcji (`notes/matsim-python-tools.md`, sekcja „Relacja do pam”).
- **Wszystkie klony są płytkie (1 commit).** Częstotliwości zmian, liczby autorów ani ewolucji konwencji nie da się ocenić z historii. Tam, gdzie mówię o „aktywności”, opieram się na datach w nagłówkach plików (np. `notes/sumo-berlin.md`, „Proxy aktywności”).
- **Kod części kluczowych elementów jest poza refs:**
  - `network-from-sumo` / `CreateNetworkFromSumo` (matsim-libs `contrib/application`);
  - `tshub` (warstwa SUMO pod `single-tsc-baselines`);
  - strona TAPAS;
  - importer i eksporter sieci MATSim w `netconvert` (`NIImporter_MATSim`, `NWWriter_MATSim` — znane tylko z nazw w `_trees/sumo.txt`).

  Wnioski o tych elementach są oznaczone jako *wnioskowane*.
- **Częściowe klony SUMO.** `sumo-scenarios` nie ma pobranych plików XML (sieci, trasy, detektory, TLS). Ich treść jest znana tylko z nazw i z odwołań w `.sumocfg`.

Skróty kolumn: **eqF** = eqasim-france, **syn** = synpp, **eqSP** = eqasim-sao-paulo, **mB** = matsim-berlin, **mT** = matsim-scenario-template, **mPT** = matsim-python-tools, **tsc** = dlr-tsc, **imp** = sumo (importer + fixtury), **sSc** = sumo-scenarios, **sBe** = sumo-berlin, **gro** = gross, **stsc** = single-tsc-baselines. „—” = nie dotyczy.

---

## 1. Macierz

### 1a. Strona MATSim / generatora popytu

| Wymiar | eqF | syn | eqSP | mB | mT | mPT |
|---|---|---|---|---|---|---|
| **1 Orkiestracja** | stage'e synpp, cele w `run:`; `validate()` = rozmiar pliku (`data/census/raw.py:56-60`) | DAG z `configure`; klucz = MD5 wymaganego configu, **kod poza kluczem** (`pipeline.py:216-222, 91-95`); błąd diamentu (`:566-567`) | synpp, `validate()` = rozmiar; efekty uboczne do `data_path` poza DAG (`data/hts/cleaned.py:381-382`) | Make, cele plikowe `input/$V/...` (`Makefile:98-120`); zapis w miejscu, brak `.DELETE_ON_ERROR`, JAR nie jest zależnością | Make jak mB; `$(JAR)` jest celem, ale nie prerekwizytem danych (`Makefile:26-27, 137-139`) | CLI `METADATA+setup+main` (`scenariogen/__main__.py:16-45`); brak cache; „katalog istnieje ⇒ gotowe” (`run_simulations.py:173-175`) |
| **2 Konfiguracja / warianty** | YAML per miasto (kopie szkieletu); alias `hts` wybiera ankietę (`data/hts/selected.py:1-20`); twardy błąd dla `edgt_lyon_source` | jeden YAML + `-c` z CLI; rzutowanie `bool("false")==True` (`:1051-1052`) | jeden `config.yml`, ścieżki absolutne; parametry zaszyte w kodzie (`osm.py:29`) | config XML per wersja + `--Npct` + YAML; wersja w 4 miejscach (`Makefile:4`, `OpenBerlinScenario.java:42`, `pom.xml:18`, katalog) | `N`, `V`, `CRS` w Make; `SampleOptions` (`RunTemplateScenario.java:41`); repo = jedno miasto | config MATSim + `params/runN.yaml` zapisany obok runu (`calibration/__init__.py:180-204`) |
| **3 Dane / duże pliki** | `data_path`/`working_directory`/`output_path` poza repo; `config_local_*.yml` w `.gitignore`; `download.py` wylicza pliki z configu | płaski `<klucz>.p` + `.cache/`, pickle | `data_path` na NAS; **`resources/emtu.zip` (54 MB, dane zastrzeżone) w git** | public-svn (URL w configu, `README.md:33`), shared-svn dla danych zamkniętych; `*.gz` ignorowane | jak mB; uzasadnienie przeciw LFS (`README.md:62-64`) | tylko fixtury ≤21 KB w git |
| **4 Proweniencja** | `meta.json`: commit, `version.txt`, seed, próbka, hts (`documentation/meta_output.py:15-43`); release-please; `CITATION.cff`; dryf `pyproject` 1.4.0 vs 1.5.0 | `pipeline.json` bez commita i wersji | `meta.json`: tylko seed, próbka, czas (`synthesis/output.py:139-146`) | `git describe` w nazwie JAR i manifeście (`pom.xml:360-399`); CHANGELOG per wersja scenariusza | `git describe` w JAR (`pom.xml:141-186`); `CITATION.cff`; brak changelogu | `VERSION` + standard-version; puste wpisy changelogu od 0.0.12 |
| **5 Sieć** | pt2matsim z OSM+GTFS; brak spójności z innym narzędziem | — | pt2matsim, config XML w kodzie | **SUMO-first:** OSM → osmosis → `netconvert` → `network-from-sumo` (`Makefile:60-120`); ID link = edge *(wnioskowane z 3 użyć)* | SUMO-first jak mB (`Makefile:58-74`) | zakłada SUMO edge id == MATSim linkId (`features.py:11-18`) |
| **6 Handoff → SUMO** | — (`grep sumo` pusty) | — | — | — (kierunek SUMO→MATSim) | — | tylko czytniki; `plan_reader_dataframe` błędnie filtruje wybrane plany (`Plans.py:116-141`) |
| **7 Kalibracja międzynarzędziowa** | — | — | — | **SUMO → przepustowość/prędkość MATSim** (`capacity/`, XGB per `junctionType`, m2cgen → Java); skrypty nieaktualne względem klas Java | — (tylko ASC vs ankieta) | aktualna wersja pipeline'u capacity (`scenariogen/network/`) + adaptery ankiet do wspólnego schematu z wagami (`scenariogen/data/__init__.py:342-417`) |
| **8 Próbka / skala** | losowanie gospodarstw Bernoulli z `sampling_rate`; skalowanie przepustowości po stronie Javy | — | jak eqF | `-Npct` w nazwie pliku; jedna flaga ustawia flow, storage i counts (`OpenBerlinScenario.java:53,73-86`); 25pct → downsampling | jak mB (`Makefile:143-145`) | — |
| **9 Testy / determinizm** | **syntetyczne dane od zera, także dla niepublicznej EGT** (`tests/testdata.py`); **złote MD5 na 3 OS** (`tests/test_determinism.py:81-160`); smoke per wariant | mikro-stage'e jako fixtury; brak testu diamentu | brak testów; nieseedowane `np.random` | 1pct × 2 iteracje, tylko exit code | 1pct × 2 iteracje, bez danych nie przejdzie | golden-file round-trip writera (`tests/test_MatsimPlansWriter.py`) |
| **10 Dane zastrzeżone / licencje** | GPL-2.0 całość; dane zamknięte „Optional” w docs, poza `download.py` | MIT | GPL-2.0; zastrzeżony GTFS w repo | GPLv2 kod + CC BY dane (`README.md:22-29`); SrV tylko jako agregaty; HBEFA szyfrowane kluczem z env | **licencja per rodzaj artefaktu** (`README.md:51-60`), w tym CC BY-NC dla populacji od komercyjnego dostawcy | **brak pliku LICENSE**, GPLv3 tylko w `setup.py:21`; wkopiowany `randomTrips.py` (EPL/GPL) |
| **11 Python ↔ Java** | osobne repo eqasim-java; stage klonuje przypięty commit i buduje Mavenem; `java.run` = subprocess; JAR kopiowany do wyjścia (`matsim/runtime/eqasim.py:33-62`) | — | jak eqF, ale **gałąź `develop` bez przypięcia** (`matsim/runtime/eqasim.py:26-41`) | Java w tym samym repo (Maven + `mvnw`); MATSim przypięty jako PR-release (`pom.xml:4-13`); Python woła JAR przez CLI lub HTTP | jak mB; JDK 17/21/25 rozjechane | JAR z zewnątrz, ścieżka jako string |
| **12 Pakietowanie** | wirtualna aplikacja `uv` (brak `build-system`, `uv.lock:349-351`) | biblioteka `src/` + `setup.py` | brak pakietu | Python = luźne skrypty | Python = luźny skrypt | `setup.py`, extras jako podsystemy; zepsuty entry point `matsim-viz` |
| **13 Dokumentacja** | `docs/cases/<miasto>.md` wg jednego szablonu; Sphinx | README z regułami unieważniania | `howto.md` | CHANGELOG jako nośnik wiedzy; `docs/` skąpe i ignorowane przez `.gitignore` | README-checklista | README |

### 1b. Strona SUMO / sprzężenia / konsumenta

| Wymiar | tsc | imp | sSc | sBe | gro | stsc |
|---|---|---|---|---|---|---|
| **1 Orkiestracja** | pakiet z CLI (`tsc_main`, `tsc_install`) + daemon na DB; `--resume` = istnienie pliku (`t2s.py:586-627`); numerowane `template_gen*.netccfg` | — | ręczne `build.sh` z krokami w komentarzach (`brunswick/build.sh`) | liniowy `build.py` + `--skip-net` itp.; parametry kroków w natywnych `*.netccfg`/`*.gtfscfg` | numerowane `stage_N_*.sh`; **`--skip-to` to tylko licznik** (`run.sh:91-99`); cache „istnieje i ma nagłówek gzip” | brak; `run_batch.sh`, jeden proces na wariant (libsumo) |
| **2 Konfiguracja / warianty** | tabela `simulation_parameters` + `SP` domyślne + `--sim-param`; scenariusz = katalog z hookami `__init__.py` | opcje CLI | wariant = kopia `.sumocfg`; Wildau: wspólna sieć i detektory, wymienne `route-files` | podscenariusz = katalog; data referencyjna powielona w 3 plikach | nazwa + argv; `config/exceptions.xml` z `reason` | ścieżki wyprowadzane z nazw (`_loader.py:69-79`); `num_phases` w module Pythona |
| **3 Dane / duże pliki** | `data/` (LFS) → `scenario_templates/` → `scenario_workdir/.../iterationNNN/`; LFS porzucony na rzecz `sumo-scenarios` | — | `.gz` w git, `--depth 1`; połowa `.sumocfg` wymaga budowy lub wskazuje na brakujące pliki | **zamrożony OSM 72 MB w git**; świeży pobierany tylko z `-u`; GTFS z nazwą okresu ważności | `data/*` ignorowane; OSM `-latest` | scenariusze w git (małe); wyjścia SUMO trafiają do katalogu scenariusza |
| **4 Proweniencja** | `sim_key` + log statusu; wyniki nadpisywane `DROP TABLE` bez numeru iteracji (`s2t_miv.py:163`) | nagłówek z wersją SUMO i opcjami (`routes.tools:3-5`) | tylko nagłówki „generated by SUMO vX”; jeden `--sim-key` | nagłówki; SUMO nieprzypięte (HEAD = dostosowanie do zmiany `gtfs2pt`) | `sources.txt`; `ubuntu:latest`, `ppa:sumo/stable` | nagłówki z absolutnymi ścieżkami autorów; SUMO 1.20–1.27 |
| **5 Sieć** | SUMO z OSM + `netpatch/`; **popyt w geo, mapowanie przy każdym przebiegu** (`t2s.py:412-469`, `edgemapper.py:39-80`) | **ID linku MATSim kopiowane 1:1 jako krawędź SUMO, bez walidacji** (`matsim_importPlans.py:79-81,122,199-200`) | patche plain-XML + `.netccfg`; **geo-trips „universally” mapowalne** (`brunswick/build.sh:31-32`) | OSM snapshot + łatki kluczowane ID OSM reaplikowane przy regeneracji; **dryf `<way>#n`**, heurystyka `check_landmarks` (`build.py:29-63`) | netconvert ×2 + post-processing XML; ID OSM przez heurystyki (`routing.py:411-437`) | ID krawędzi bez `_` (`static_features.py:194`); ta sama geometria w `easy`/`normal` |
| **6 Handoff → SUMO** | płaska tabela tripów geo+TAZ, bez tras (`constants.py:25-63`); każdy trip → osobny `<trip>`/`<person>`; łańcuch odtwarzany tylko w postprocessingu | `plan[0]` zamiast `selected` (`:148`); `dep_time` vs `end_time` zależnie od trasy; `ride` → dodatkowy pojazd; PT → `personTrip`, trasy tracone; brak skali | konsument ładuje **tripy + rerouting (oneshot)**; czas przesunięty o dobę | — | — | konsumuje **tylko `<vehicle>` z `<route edges>`**, jeden vType, pełna skala; brak osób i trybów |
| **7 Kalibracja międzynarzędziowa** | **SUMO → TAPAS: skimy OD** per przedział czasu, ≥5 próbek na parę (`s2t_miv.py:214-284`) | — | Wildau: jedna sieć, jedne liczniki, N metod popytu; błąd w `rms.py:28` | routeSampler + Telraam (w obrębie SUMO) | — | stopień nasycenia `X` z programu TLS (`analyze_flow_capacity.py`) |
| **8 Próbka / skala** | **klonowanie z losowym zaokrąglaniem, klon w ID, niezależny jitter** (`t2s.py:481-537`); przepustowość nieskalowana | brak | brak `--scale`; intensywność zaszyta w generatorach | — | — | brak parametru; zakłada 100% |
| **9 Testy / determinizm** | TextTest: złote pliki, 2 osoby × 2 tripy, bbox OSM, SQLite zamiast Postgresa; seed domyślny 23432; brak CI dla testów | **11 fixtur golden-file**, każda na jedną cechę | smoke `sumo --end 1000` + porównanie stderr; brak testów wartości | brak testów; seed tylko w części kroków | brak testów | brak; determinizm z zamrożonego `.rou.xml`, `reset(seed)` ignorowany |
| **10 Dane zastrzeżone / licencje** | EPL/GPL; dane TAPAS tylko w DB; `readme.txt` z licencją GTFS | EPL-2.0 OR GPL-2.0+ | EPL, ale pliki „All rights reserved”; commitowany tylko produkt pochodny (geo-trips), surowy TAPAS u autora | EPL; OSM ODbL bez atrybucji | **brak LICENSE** | **brak LICENSE**; OSM bez atrybucji |
| **11 Python ↔ Java** | — (TAPAS za bazą) | — | — | — | — (C++ przez Makefile) | — |
| **12 Pakietowanie** | setuptools `src/` + 2 console_scripts; każdy moduł etapu ma własne CLI | skrypt + `$SUMO_HOME/tools` w `sys.path` | brak | brak | brak; Docker jedynym źródłem zależności | brak; `sys.path`, conda bez wersji |
| **13 Dokumentacja** | kontrakt w `.docx` (Lastenheft), rozjechany z kodem | 3 zdania w docs, bez założeń o ID i wyboru planu | README nierówne; BadHersfeld opisuje nieistniejące pliki | `NOTES.md` z wnioskami kalibracji; `gtfserrors_annotated` | nagłówki Inputs/Outputs w etapach (częściowo nieaktualne) | README „New junction” = lista kontrolna kontraktu |

---

## 2. Analiza per wymiar

### 1. Orkiestracja

**Zgodność.**
- **Etap ma deterministyczną nazwę pliku wyjściowego wyprowadzaną z konfiguracji.** Robią tak: mB `input/$V/berlin-$V-*`, tsc `rectified_<base>.csv`, gro `paths.sh:44-121`, eqF przez klucze synpp. Następny etap czyta plik po nazwie.
- **Parametry kroków narzędzi zewnętrznych leżą w natywnych plikach konfiguracyjnych tych narzędzi**, a nie w kodzie orkiestratora: sBe `*.netccfg`/`*.gtfscfg`, tsc `template_gen*.netccfg`, gro `config/convert.netccfg`. Tam, gdzie od tego odstąpiono, pojawił się dryf: sBe ma część opcji inline w `build.py:89-98`, mB zaszywa opcje `netconvert` w Makefile.
- **Pętle kalibracyjne wypadają poza DAG.** mB: `berlin-$V-10pct.plans.xml.gz` czyta ręcznie wskazany bieg (`Makefile:330-344`). mPT/mB: sharding po `--runner-index` na klastrze.

**Wspólna luka.** Żaden z czterech stylów nie wykrywa pewnie nieaktualności:

| Styl | Co unieważnia | Czego nie widzi (dowód) |
|---|---|---|
| synpp (eqF, eqSP) | config wymagany przez stage, źródło samego modułu stage'a, token `validate()` | modułów pomocniczych, bibliotek, JAR-a i plików wejściowych bez `validate` — sprawdzone eksperymentem (`notes/synpp.md`); diament: cicho błędny wynik (`pipeline.py:566-567`); tokeny w eqF to sam rozmiar pliku |
| Make (mB, mT) | mtime prerekwizytów | zmiany kodu (JAR nie jest prerekwizytem), wejść spoza repo (`../shared-svn`), częściowego zapisu w miejscu bez `.DELETE_ON_ERROR` |
| numerowane skrypty + `--skip-to` (gro, sBe) | nic — licznik lub flaga | czegokolwiek; cache „plik istnieje (i ma nagłówek gzip)” |
| pakiet z CLI (tsc, mPT) | nic — `--resume` / „katalog istnieje” | czegokolwiek |

**Rozbieżności i co je tłumaczy.**
- **synpp** trzyma w cache obiekty Pythona (DataFrame przez pickle), co pasuje do syntezy populacji, w której etapy wymieniają ramki w pamięci. Unieważnianie wbudowane w synpp istnieje, ale jest „best effort” (README synpp, cytat w notatce). Działa przy jednym użytkowniku na lokalnym katalogu: nie ma blokady, reguła 4.6 porównuje czas zegarowy.
- **Make** jest naturalny, gdy etapy to wywołania jednego JAR-a/CLI produkujące pliki (VSP: ok. 40 podkomend `prepare`). Luki Make'a wynikają ze sposobu użycia (zapis w miejscu), a nie z samego narzędzia *(wnioskowane)*.
- **Skrypty numerowane** służą scenariuszom budowanym rzadko, przez autora narzędzi, dla siebie (sBe, gro, sSc).
- **Daemon (tsc)** wynika z wymogu Lastenheft „permanent lauffähig” (praca ciągła) przy żywej pętli z TAPAS.

### 2. Konfiguracja i wiele scenariuszy

**Zgodność.**
- **Wariant to mała delta na wspólnych wartościach domyślnych.** eqF: 5 plików YAML różni się kilkoma kluczami. mB/mT: flaga `--Npct` plus YAML parametrów. sSc Wildau: wspólna sieć, wymienny `route-files`.
- **Wybór źródła danych jest jawny.** eqF podpina ankietę przez alias `hts`. Wariant niejednoznaczny (`edgt_lyon_source: unchosen`) kończy się twardym błędem (`data/hts/edgt_lyon/filtered.py:9-18`).
- **Plik nadpisań konfiguracji zapisywany obok wyniku runu:** mPT `params/runN.yaml`, tsc `sumo --save-configuration` (`assign.py:219-234`).

**Rozbieżności.**
- **Nośnik konfiguracji** to: YAML (eqF, syn), Make + XML (mB/mT), tabela w DB (tsc), argv (gro), osobny `.sumocfg` na wariant (sSc, sBe), moduł Pythona (stsc).
  - Nośnik idzie za językiem, w którym żyje logika. VSP ma logikę w Javie (picocli), stąd flagi. DLR opiera się na natywnych plikach konfiguracyjnych SUMO. eqasim jest pythonowy, stąd YAML.
- **Duplikacja jest wszędzie tam, gdzie wersja lub data nie mają jednego źródła:**
  - mB i mT trzymają wersję scenariusza w 4 miejscach;
  - sBe powiela datę GTFS w 3 plikach;
  - eqF ma dwie różne wersje w `pyproject.toml` i `version.txt`;
  - gro definiuje vTypes dwa razy.

  To wzorzec niezależny od stylu. Wynika z braku jednego obiektu konfiguracji, z którego generowane są pozostałe *(wnioskowane)*.

### 3. Układ danych i duże pliki

**Zgodność (prawie jednomyślna).**
- **Surowe i duże dane nie wchodzą do git**: eqF, mB, mT, mPT, gro, tsc (w nowym trybie).
- **Kod wskazuje dane jedną ścieżką z konfiguracji**: eqF `data_path`, mB URL-e do public-svn, gro `BASE_DIR`. To odpowiada naszemu `.env` + `kbr.paths`.
- **Dwa osobne magazyny danych: zastrzeżony i publikowalny.** mB i mT rozdzielają `shared-svn` i `public-svn`. tsc trzyma dane w bazie, a do repo trafia tylko pochodna (sSc `brunswick`).

**Wyjątki i ich powód.**
- **sBe** commituje 72 MB OSM, a **sSc** trzyma `.gz` w git. Oba repo mają wyłącznie otwarte dane, jednego właściciela (DLR) i odbiorcę, który chce `sumo -c` bez toolchainu. **tsc** próbował LFS i przeszedł na osobne repo scenariuszy.
- **eqSP** zacommitował zastrzeżony GTFS EMTU wbrew własnej instrukcji (`docs/howto.md:260-262`). To jedyny przypadek naruszenia wśród badanych repo. Dowodzi, że konwencja bez mechanizmu nie wystarcza.

**Publikacja wyników** — każde repo robi to inaczej:
- mB/mT: public-svn;
- eqF: artefakty Jenkinsa i obraz ghcr, publikacja danych osobno;
- eqSP: polybox ETH;
- sSc: git;
- gro, sBe, stsc: nie publikują wcale.

Nikt nie używa DVC ani Zenodo. Nie znaleziono żadnego odwołania do DOI danych (grep w notatkach: tylko DOI artykułów w `CITATION.cff`/README).

**Samowystarczalność paczki dla konsumenta.** W sSc połowa `.sumocfg` nie działa bez lokalnej budowy albo wskazuje na nieistniejące pliki. stsc zapisuje wyjścia SUMO do katalogu scenariusza. Żadne repo nie ma testu „każdy plik z `.sumocfg` istnieje”.

### 4. Proweniencja i wersjonowanie

**Zgodność.**
- **Trzy wersje są niezależne:** wersja scenariusza, wersja kodu i przypięta wersja symulatora.
  - mB/mT: `v6.4` ≠ Maven `6.x-SNAPSHOT` ≠ MATSim `2025.0-PR3653`.
  - eqF: `version.txt` ≠ commit eqasim-java.
- **Hash commita dociera co najwyżej do metadanych artefaktu, nie do wszystkich plików wyjściowych.** eqF zapisuje go w `meta.json`; mB/mT w nazwie i manifeście JAR-a. Narzędzia SUMO zapisują własną wersję w nagłówkach XML.

**Czego nie robi nikt:**
- flaga „dirty working tree” (eqF: brak, `notes/eqasim-france.md` §4);
- odciski danych wejściowych w metadanych wyniku;
- wersja JAR-a lub symulatora w `meta.json` (eqF jej nie zapisuje, choć kopiuje JAR do wyjścia);
- wersja sieci, dla której popyt jest ważny.

**Rozbieżności.**
- **Changelog.** mB opisuje zmiany danych i metod per wersja scenariusza. eqF robi changelog release-please z konwencjonalnych tytułów PR. mPT ma puste wpisy. Reszta nie ma go wcale. Changelog prowadzi ten, kto publikuje wersjonowany scenariusz dla zewnętrznych użytkowników (VSP, eqasim). Scenariusze robione dla siebie (sBe, gro, stsc) nie mają żadnej wersji.
- **Przypięcie wersji SUMO.** mB przypina SUMO 1.17.0 budowane ze źródeł (`capacity/Makefile`). sBe, gro i stsc nie przypinają nic, a sBe w ostatnim commicie dostosowuje się do zmiany domyślnego zachowania `gtfs2pt`. To bezpośredni dowód kosztu braku przypięcia.

### 5. Źródło prawdy dla sieci

Trzy obserwowane wzorce:

| Wzorzec | Gdzie | Co daje | Czego wymaga / co łamie |
|---|---|---|---|
| **SUMO-first**: OSM → `netconvert` → `sumo.net.xml` → sieć MATSim z tymi samymi ID | mB (`Makefile:60-120`), mT (`Makefile:58-74`); mPT zakłada to w `features.py:11-18` | trasy MATSim są wprost krawędziami SUMO; umożliwia kalibrację capacity z mikrosymulacji | konwerter poza refs *(wnioskowane)*; tożsamość łamie `--junctions.join` (ID klastrów); `--no-internal-links` i `--remove-turn-restrictions` różnicują topologię manewrów |
| **MATSim-first / niezależne konwersje z OSM** | eqF, eqSP (pt2matsim) | — | brak jakiejkolwiek spójności ID z SUMO; gro pokazuje, że spinanie przez ID OSM wymaga heurystyk (`routing.py:411-437`) |
| **Popyt oddzielony od sieci (geo)** | tsc (`edgemapper.py`), sSc (`brunswick/build.sh:31-32`) | sieć SUMO można przebudować bez zmiany popytu | deterministyczny map-matching po stronie konsumenta; trasy z generatora popytu tracone |

**Zgodność co do sieci SUMO.** Wszystkie repo SUMO (tsc, sSc, sBe, gro) budują sieć z **zamrożonego snapshotu OSM** i **łatek reaplikowanych przy każdej regeneracji**:
- `netpatch/*.xml`;
- `*.typ.xml`;
- `.netccfg`;
- łatki kluczowane ID OSM (`ptLine.patch.xml`).

Tam, gdzie od tego odstąpiono, sieć przestała być odtwarzalna. sBe `Doerpfeldstr` ma ręcznie edytowaną sieć w netedit i wejście spoza repo. sSc `BadHersfeld` pracuje na ręcznie edytowanej sieci z wyłączoną przebudową.

**Dryf ID jest realny i udokumentowany.** ID `<osm_way>#<n>` przesuwa się, gdy do OSM dojdzie węzeł (sBe `build.py:29-63`). Dotyczy wszystkiego, co referuje krawędzie: detektorów, tras i danych kalibracyjnych. Jedyne zabezpieczenie w refs to heurystyka landmarków. Nikt nie waliduje referencji automatycznie.

**Wymagania konsumenta RL.** ID krawędzi nie może zawierać `_` (stsc `static_features.py:194`). Detektory E2 muszą mieć konwencję nazw. Program TLS i geometria są „twardą” częścią kontraktu (stsc `random_scenario_env.py:170-189`).

### 6. Przekazanie MATSim → SUMO

**Fakty z fixtur** (szczegóły w `notes/sumo.md`, §6.6):
- `plan[0]` zamiast planu `selected` (`:148`). Żadna fixtura nie ma więcej niż jednego planu na osobę.
- ID linków są kopiowane bez walidacji. Sieć wczytuje dopiero `duarouter` przy `--repair`.
- Czas odjazdu zależy od tego, czy leg ma trasę. Z trasą liczy się `dep_time` legu, nawet wbrew `end_time` aktywności (`coordinates_preferred`: 08:00 zamiast 13:27). Bez trasy liczy się `end_time`. Luki łata `--default-dur`=1h.
- `trav_time` i `vehicleRefId` giną.
- W trybie osobowym każdy leg dostaje nowy pojazd `triggered`, a osoba jest w nim pasażerem.
- Tryby:
  - `ride` staje się dodatkowym pojazdem na sieci;
  - `freight` dostaje pusty `vType`;
  - trasy PT są tracone w całości;
  - `--no-bikes`/`--no-rides` zostawiają nieciągłe plany.
- Skali ani próbki nie ma.

**Porównanie z tsc** (najbliższy analog architektury):
- Tabela tripów w geo, bez tras (`constants.py:25-63`).
- Każdy trip jest niezależny.
- Klucz `p_id_hh_id_startmin_clone` zachowuje osobę, oryginalną minutę i klon.
- Jitter czasu (±450 s na osobę) i szum przestrzenny dodaje **importer**, a nie artefakt.
- Tripy po luce w łańcuchu są odrzucane.
- Czas przesunięty o +24 h.
- Łańcuch aktywności odtwarzany jest tylko w postprocessingu.

**Konsumenci się różnią:**
- **sSc (DLR)** bierze tripy i sam routuje (oneshot + `device.rerouting`). Nie ufa trasom z generatora.
- **stsc (RL)** bierze wyłącznie `<vehicle>` z pełnymi `<route edges>`, w pełnej skali, z jednym vType. Nie konsumuje osób ani trybów, więc straty importu planów tego konsumenta nie bolą.

Te dwa wymagania **nie są sprzeczne**. Trasy dla RL można wytworzyć po stronie SUMO (`duarouter`/oneshot) z tripów. Sprzeczność pojawia się dopiero, gdy chcemy, żeby SUMO odtworzyło **równowagę MATSim**, a nie własną. Wtedy trasy MATSim muszą przejść 1:1, co wymaga wariantu SUMO-first z wymiaru 5. To napięcie jest rdzeniem decyzji o artefakcie (patrz §4 „Napięcia”).

**Semantyka czasu jest za każdym razem inna:**
- tsc: +24 h;
- sSc brunswick: `begin 1:3:30:0`, GTFS od 97200 s;
- Doerpfeldstr: okno 50300–54100 s z warm-upem ok. 100 s;
- stsc: popyt 600 s w epizodzie 1000 s.

Nie ma konwencji. Każdy kontrakt musi to określić jawnie.

### 7. Kalibracja między narzędziami

Trzy kierunki, każdy w innym repo:

1. **SUMO → parametry sieci MATSim.** mB `capacity/` (relikt wersji 6.2) i mPT `scenariogen/network/` (wersja aktualna):
   - mikrosymulacje TraCI na fragmentach sieci;
   - regresory per `junctionType`, osobno dla `traffic_light`;
   - generacja kodu Java;
   - strojenie liści względem rzeczywistych czasów przejazdu.

   **Warunek:** wspólne ID. Kod jest GPL i leży w dwóch miejscach, między którymi nastąpił rozjazd (`camelCase` vs `snake_case`, `notes/matsim-berlin.md` §7).
2. **SUMO → model popytu (skimy OD).** tsc agreguje czasy per para TAZ i przedział, z ≥5 próbkami uzupełnianymi reprezentantami. U nas pętla zwrotna nie jest planowana, ale ten sam skim nadaje się do **walidacji** spójności MATSim ↔ SUMO *(wnioskowane)*.
3. **Liczniki → popyt w obrębie SUMO.** sSc Wildau porównuje N metod popytu na jednej sieci i jednych licznikach. sBe Doerpfeldstr robi to przez routeSampler + Telraam. stsc używa stopnia nasycenia `X` z programu TLS.

Kierunki 1 i 3 to ogólne praktyki DLR/VSP. Kierunek 2 wynika z dwukierunkowej architektury TAPAS, której nie mamy.

### 8. Próbka / skalowanie

**Strona MATSim — pełna zgodność** (mB, mT, eqF, eqSP):
- próbka jest parametrem wyjścia zapisanym w nazwie pliku (`-Npct`) lub w `meta.json`;
- jeden parametr steruje populacją i przepustowością (`flowCapacityFactor`, `storageCapacityFactor`, `countsScaleFactor`).

**Strona SUMO — nikt nie ma konwencji:**
- importer nie zna skali;
- sSc nie ma `--scale`;
- stsc wymaga 100% i nie ma gdzie przyjąć współczynnika;
- tylko tsc skaluje: klonuje tripy z losowym zaokrąglaniem, oznacza klon w ID i daje mu niezależny jitter. Ruchu tła nie skaluje, przepustowości też nie.

**Implikacja.** Informacja o próbce **ginie dokładnie na granicy MATSim → SUMO**, którą projektujemy. Musi przejść w kontrakcie artefaktu. Trzeba też zdecydować, kto skaluje: eksporter, importer czy `sumo --scale`. W refs jest dowód tylko dla wariantu „importer klonuje” (tsc).

### 9. Testy i determinizm

**Najmocniejsze wzorce:**
- **eqF: syntetyczne dane od zera w schemacie oryginałów, z seedami, również dla niepublicznej EGT** (`tests/testdata.py`). Pozwala testować ścieżki danych zastrzeżonych w publicznym CI.
- **eqF: test determinizmu ze złotymi MD5 na Linux/Windows/macOS**, z normalizacją formatów (gzip po dekompresji, GPKG jako posortowany zrzut SQL). Luka: dla MATSim hashuje tylko config, households i vehicles.
- **Golden-file dla konwerterów:**
  - imp: 11 fixtur, każda na jedną cechę;
  - tsc: 2 osoby × 2 tripy + bbox OSM + SQLite;
  - mPT: round-trip writera.
- **Smoke scenariusza:**
  - sSc: `sumo --end 1000` + porównanie stderr;
  - mB/mT: 1pct × 2 iteracje.

**Zgodność.** Nikt nie używa biblioteki do walidacji schematu (pandera, JSON Schema, XSD w CI). Wszędzie są tylko asercje w kodzie (eqF `data/hts/hts.py:245-268`).

**Kto testy ma, a kto nie.**
- Testy mają repo z zewnętrznymi użytkownikami i CI: eqF, imp, mPT (częściowo), tsc (bez CI).
- Brak testów wiąże się ze scenariuszami jednorazowymi: gro, stsc, sBe, eqSP.
- **mT** jest przykładem wprost: autorzy piszą, że szablon „zgnił” z powodu braku testów regresyjnych (`README.md:7-8`).
- **eqSP** zapisuje seed w `meta.json`, choć większość losowań go nie używa. To fałszywe poczucie odtwarzalności.

**Determinizm konsumenta RL (stsc)** bierze się z zamrożonego pliku `.rou.xml`, a nie z seeda. `reset(seed)` jest ignorowany, a seed SUMO nie jest ustawiany w repo. Zmienność między epizodami daje podmiana gotowego `.sumocfg`.

### 10. Dane zastrzeżone i licencje

**Zgodność.** Separacja danych zastrzeżonych jest **fizyczna, nie techniczna**. Dane leżą w innym magazynie: shared-svn, baza TAPAS, `data_path` poza repo. Nikt nie ma flagi „restricted” ani mechanizmu, który blokowałby publikację pochodnych.

**Sposoby opisu licencji:**
- **Tabela licencji per rodzaj artefaktu:** mT `README.md:51-60`, mB `README.md:22-29`. Kod ma GPL/AGPL, wejścia i wyjścia CC BY, populacja od komercyjnego dostawcy CC BY-NC, reszta jest indywidualna.
- **`readme.txt` z licencją per zbiór:** tsc i sBe dla GTFS.
- **Szyfrowanie z kluczem w env** dla danych, które muszą krążyć z przebiegami (mB, HBEFA). To jedyny mechanizm techniczny w refs.

**Antywzorce:**
- eqSP commituje zastrzeżony GTFS;
- mPT nie ma pliku LICENSE (GPLv3 tylko w `setup.py:21`);
- gro i stsc nie mają licencji;
- sSc ma pliki „All rights reserved” w repo na EPL;
- OSM (ODbL) jest commitowany bez atrybucji w sBe i stsc.

**Ważne dla naszego ograniczenia licencyjnego:**
- eqasim-*, matsim-* i mPT to GPL/AGPL;
- synpp to **MIT** (`LICENSE`), więc można go używać jako zależności bez obowiązków copyleft;
- SUMO to EPL-2.0 OR GPL-2.0+.

Użycie `matsim-tools` jako importowanej zależności to osobne pytanie licencyjne (`notes/matsim-python-tools.md` §10).

### 11. Runtime Python ↔ Java

Trzy obserwowane warianty:

| Wariant | Gdzie | Przypięcie | Koszt |
|---|---|---|---|
| Java w tym samym repo (Maven + `mvnw`), fat-JAR z `git describe` | mB, mT | parent POM na PR-release MATSim; zależności JitPack przypięte commitem | JDK rozjechany między README/pom/CI/SLURM (mT: 17/21/25); ścieżka JAR-a zaszyta w każdym skrypcie Pythona (mB: 3 różne nazwy) |
| Java w osobnym repo, klonowana i budowana w stage'u | eqF (commit przypięty), eqSP (gałąź `develop`, nieprzypięta) | commit jako parametr configu, wchodzący do klucza cache | Maven + JDK na każdej maszynie; dryf komentarzy w `config_full.yml` względem kodu; eqSP: wynik zależy od daty |
| JAR z zewnątrz, podawany ścieżką | mPT | brak | brak śladu wersji w wynikach |

**Zgodność.** Python woła Javę **wyłącznie przez subprocess/CLI** (w mB także HTTP JSON do serwera optymalizacji). Nigdzie nie ma JNI ani bindingów. Interfejs to pliki.

**Rozbieżność** wynika z tego, kto jest właścicielem kodu Java. VSP pisze Javę per scenariusz, więc trzyma ją w repo scenariusza. eqasim ma wspólny rdzeń Java dla wielu regionów, więc trzyma go w osobnym repo i buduje z przypiętego commita.

### 12. Pakietowanie

**Obserwowane układy:**
- eqF: wirtualna aplikacja `uv`, bez `build-system` i bez `src/`, uruchamiana przez `uv run -m synpp`;
- syn: biblioteka `src/` z `setup.py`;
- tsc: pakiet `src/` z `console_scripts`, a każdy moduł etapu ma własne `main()`;
- mPT: rdzeń + extras (`calibration`, `scenariogen`, `viz`), co prowadzi do rozjazdów zależności i zepsutego entry pointu;
- pozostałe repo nie są pakietami.

**Wniosek dla workspace'u `uv`.** Jedynym repo z `uv` jest eqF. Jest aplikacją, ma `pytest` w zależnościach runtime i ukryte zależności tranzytywne skryptów (`typer`, `rich`, `requests`, `pyyaml`). mPT pokazuje, że „jeden pakiet + ciężkie extras” psuje się szybciej niż osobne pakiety. W żadnym repo nie znaleziono podziału na wielu członków workspace'u, więc na liczbę członków nie ma bezpośredniego dowodu. Jest tylko dowód, gdzie przebiegają granice zależności:
- rdzeń odczytu MATSim (lekki: pandas, protobuf, xopen);
- pipeline SUMO (sumolib, traci);
- kalibracja ML (xgboost, optuna, jax);
- środowisko RL (libsumo, torch, SB3).

### 13. Dokumentacja

**Zgodność.** Najcenniejsze okazują się notatki **obok danych i scenariusza**, a nie dokumentacja API:
- eqF `docs/cases/<miasto>.md` według jednego szablonu: dane dodatkowe → lista plików → fragment configu;
- sBe `NOTES.md` z wnioskami kalibracyjnymi;
- sBe `gtfserrors_annotated`;
- gro `exceptions.xml` z atrybutem `reason`;
- mB CHANGELOG opisujący zmiany danych i metod.

**Wspólna porażka.** Dokumentacja ręczna rozjeżdża się z kodem wszędzie, gdzie nic jej nie sprawdza:
- tsc: kontrakt w `.docx` niezgodny z implementacją (sufiks iteracji, łańcuchy);
- eqF: `lyon.md` odsyła do BD TOPO 2022 i Anacondy;
- sSc BadHersfeld: README opisuje nieistniejące pliki;
- gro: dwa etapy przedstawiają się jako „Stage 4”.

Kontrakt, który przetrwał, jest **wykonywalny**: fixtury imp i tsc oraz `check()` w eqF.

---

## 3. Praktyki, które istnieją z powodów niedotyczących nas

| Praktyka | Gdzie | Ograniczenie, któremu służy | Dlaczego nie my |
|---|---|---|---|
| Serwery SVN `public-svn`/`shared-svn` jako katalogi-rodzeństwo repo | mB, mT | infrastruktura instytutu VSP; LFS płatny i nieobecny w ZIP (`README.md:62-64`) | nie mamy SVN. **Koncepcja** dwóch magazynów (zastrzeżony vs publikowalny) nas dotyczy, **medium** nie |
| PostgreSQL jako kolejka zleceń, konfiguracja i magazyn wyników; daemon | tsc | żywa, dwukierunkowa pętla z TAPAS na serwerze instytutu, praca ciągła (Lastenheft) | nasz popyt to jednokierunkowy, wersjonowany artefakt. tsc sam ma tryb plikowy i SQLite |
| Repo per miasto | mT | jeden scenariusz = jedna publikacja VSP | monorepo, jedno miasto — ustalone |
| 7 adapterów ankiet i `mobisurvstd` | eqF | wiele regionów Francji z różnymi ankietami i dystrybucjami | mamy 2 edycje tej samej ankiety (KBR 2018/2024). **Wzorzec** aliasu + `check()` dotyczy, skala nie |
| Duże pliki (OSM 72 MB, `.rou.xml.gz`) w git, `--depth 1` | sBe, sSc | tylko otwarte dane, jeden właściciel, odbiorca chce `sumo -c` bez toolchainu | część naszych danych i pochodnych jest zastrzeżona; artefakt ma być wersjonowany poza git |
| Scenariusz śledzi SUMO nightly bez przypięcia | sBe | autorzy scenariusza są autorami SUMO, a scenariusz jest testem integracyjnym narzędzia *(wnioskowane)* | my potrzebujemy powtarzalności publikacji i treningu RL |
| Budowa SUMO ze źródeł dla wersji referencyjnej | mB `capacity/Makefile` | przypięcie SUMO 1.17 na klastrze | przypięcie da się uzyskać pakietem (`eclipse-sumo`/`sumolib`, fallback w tsc `tsc_main.py:41-53`); wersji wheel nie weryfikowałem |
| Generacja kodu Java z modeli ML (m2cgen) | mB, mPT | parametry sieci aplikowane w runtime Javy MATSim | dotyczy tylko wtedy, gdy przyjmiemy kalibrację capacity. Nie jest to decyzja strukturalna |
| Proxy cache pobrań i cotygodniowa kontrola URL-i | eqF `scripts/cache_server/`, `data.yml` | francuskie open data znikają lub zmieniają URL; CI pobiera dziesiątki GB | nasze dane zastrzeżone nie są pobierane w ogóle; dla OSM i GTFS wystarczy zamrożony snapshot |
| Jitter czasu (±450 s) i szum przestrzenny przy imporcie | tsc `t2s.py:183-338` | 5-minutowa rozdzielczość TAPAS i klastrowanie punktów (komentarze w kodzie) | MATSim daje czasy z rozdzielczością sekundową *(wnioskowane)*. Czy potrzebujemy jittera, to otwarte pytanie, nie wymóg |
| Pary `.sh`/`.bat`, „emulowane symlinki” w plikach tekstowych | sSc, tsc | użytkownicy Windows bez symlinków | pomijalne |
| Macierz determinizmu na 3 OS | eqF | użytkownicy Windows i macOS | częściowo: zależy od tego, na czym pracuje zespół (otwarte pytanie) |
| Konwencja nazw zamiast rejestru scenariuszy | stsc | stała siatka 12×2×5 benchmarku | nasze warianty nie tworzą siatki |
| Kod jednego maintainera bez testów, pakietu i licencji | gro, stsc, sBe, eqSP | projekt badawczy jednego autora / jednego artykułu *(wnioskowane)* | 15 osób, praca asynchroniczna: nie przetrwa bez testów i pakietów |

---

## 4. Napięcia do rozstrzygnięcia w Fazie 3

Tu tylko je nazywam. Rekomendacje będą w projektach decyzji.

1. **Trasy vs geo w artefakcie.**
   - Równowaga MATSim w SUMO wymaga tras 1:1, czyli sieci SUMO-first i walidacji ID (imp, mB).
   - Odporność na regenerację sieci wymaga geo-tripów i routingu po stronie SUMO (tsc, sSc).
   - Konsument RL potrzebuje tras, ale nie precyzuje, czyje mają być (stsc).
   - Oba warianty mają dowody; wariantu „oba w jednym artefakcie” nie widziałem w żadnym repo.
2. **Kto skaluje próbkę:** eksporter, importer czy `sumo --scale`. Jest jeden dowód: importer z klonowaniem (tsc).
3. **Osobne wersjonowanie sieci i popytu.** Wynika z twardego i miękkiego kontraktu w stsc. Popyt musi wskazywać hash sieci.
4. **Orkiestrator.** Każdy z czterech stylów ma lukę w wykrywaniu nieaktualności (§2.1). Wybieramy więc, którą lukę zamykamy własnym kodem, a nie narzędzie bez luk. Przy synpp dochodzi potwierdzony błąd diamentu, istotny przy wariantach KBR 2018/2024 w jednym przebiegu.
5. **Polityka jittera i luk w łańcuchu:** eksporter czy importer (tsc robi to w imporcie, a oryginał zachowuje w ID).
6. **Zależność GPL:** czy importujemy `matsim-tools` (GPLv3 zadeklarowana bez pliku LICENSE), czy piszemy własny czytnik planów.
7. **Gdzie żyje Java.** Mamy tylko dwa wiarygodne precedensy: repo scenariusza (VSP) i osobne repo z przypiętym commitem (eqF). Wariantu „przypięty gotowy JAR z rejestru” w refs nie ma; mB/mT publikują JAR-y do GitHub Packages, ale ich nie konsumują.

---

## 5. Wyzwania dla ustalonych decyzji

Dowody nie przeczą stanowczo żadnej z ustalonych decyzji.
- **Workspace `uv`** jest zgodny z jedynym repo używającym `uv` (eqF). Java i tak żyje poza systemem pakietów Pythona we wszystkich precedensach, więc nie koliduje z `uv`.
- **`mypy`** zaboli tylko wtedy, gdy wybierzemy synpp: `Context` nie ma adnotacji typów (`notes/synpp.md` §12), więc kod stage'y będzie w dużej mierze `Any`. To koszt jednej z opcji orkiestratora, a nie argument przeciw `mypy`.
- **`docs/dane/` i `docs/decyzje/`** są spójne z najlepiej działającymi praktykami dokumentacji w refs (§2.13).

Uwaga faktograficzna do Fazy 3, nie wyzwanie: monorepo nie ma jeszcze root `pyproject.toml` z `[tool.uv.workspace]`. Jest tylko `packages/kbr-eda/pyproject.toml`.

---

## 6. Weryfikacja

Poniższe twierdzenia z notatek sprawdziłem ręcznie w kodzie. Pozostałe przyjmuję za notatkami, bo każda podaje ścieżkę i linię.

| Twierdzenie | Plik | Wynik |
|---|---|---|
| importer bierze `person.plan[0]` | `sumo/tools/import/matsim/matsim_importPlans.py:148` | potwierdzone |
| synpp kolejkuje tylko `downstream[0]` przy propagacji configu | `synpp/src/synpp/pipeline.py:566-567` | potwierdzone (sam eksperyment z diamentem wykonał subagent na kopii w scratchpadzie; nie powtarzałem go) |
| kolumny tabeli tripów TAPAS | `dlr-tsc/src/tapas_sumo_coupling/constants.py:25-63` | potwierdzone |
| geo-tripy „universally” mapowalne | `sumo-scenarios/brunswick/build.sh:31-32` | potwierdzone |
| łańcuch osmosis → netconvert, `--junctions.join`, `--output.original-names` | `matsim-berlin/Makefile:60-90` | potwierdzone |
| alias `hts` wybiera ankietę | `eqasim-france/data/hts/selected.py:11-15` | potwierdzone |
| licencje: kod GPLv2, dane CC BY | `matsim-berlin/README.md:22-29` | potwierdzone |

Subagent `sumo` zgłosił, że plik notatki zmienił się na dysku w trakcie jego pracy. Przeczytałem końcową wersję `notes/sumo.md` w całości: jest spójna, a jej kluczowe twierdzenie zweryfikowałem powyżej.
