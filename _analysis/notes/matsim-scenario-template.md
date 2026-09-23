# matsim-scenario-template

- **Repo:** https://github.com/matsim-scenarios/matsim-scenario-template
- **Commit:** `7a9243a90e80e5063f11789b9fb3cc153ab6ccf3` (z `MANIFEST.tsv`, klon oznaczony `full`)
- **Częściowość:** wszystkie pliki są na miejscu, ale **historia jest płytka** (`git rev-parse --is-shallow-repository` → `true`, jeden commit „bump to jdk25”, brak tagów lokalnie). Nie da się więc z klonu ustalić, jak repo faktycznie tagowało wydania.

**Czym jest to repo.** Szablon GitHub (TU Berlin, grupa VSP) do zakładania nowego „otwartego scenariusza MATSim” dla miasta: jeden projekt Maven z jedną klasą uruchomieniową (`RunTemplateScenario` dziedziczy po `MATSimApplication` z `matsim-contrib/application`), `Makefile` opisującym przygotowanie wejść (OSM → SUMO `netconvert` → sieć MATSim, GTFS → rozkład, populacja → próbki), jednym przykładowym configiem w `input/v1.0/`, skryptem kalibracyjnym w Pythonie i skryptami SLURM. Repo nie zawiera żadnych danych (poza jednym plikiem reguł osmosis). Autorzy sami zastrzegają, że szablon jest z 2023 r., nie ma testów regresyjnych i trudno go aktualizować; w VSP zaleca się raczej startować od `matsim-lausitz`/`matsim-dresden` (`README.md:7-15`, `Makefile:1-6`). Traktować więc jako **zbiór konwencji**, nie działający przykład („template =! running example”, `README.md:15`).

---

### 1. Orkiestracja

**Make jako DAG celów plikowych; każdy krok to podkomenda jednego JAR-a**
- **Zaobserwowane:** `Makefile` definiuje cele będące ścieżkami plików (`input/network.osm.pbf` → `input/network.osm` → `input/sumo.net.xml` → `input/$V/$N-$V-network.xml.gz` → `...-network-with-pt.xml.gz`; populacja: `prepare-25pct.plans.xml.gz` → `$N-$V-25pct.plans-initial.xml.gz`), `Makefile:30-151`. Cel zbiorczy `prepare` (`Makefile:154`), osobny cel `check` uruchamiający analizę `check-population` (`Makefile:148-151`). Każdy krok wywołuje `$(sc) prepare <podkomenda>`, gdzie `sc` to `java -Xmx$(MEMORY) -jar $(JAR)` (`Makefile:22`), a podkomendy są rejestrowane adnotacją `@MATSimApplication.Prepare({...})` na klasie run (`RunTemplateScenario.java:27-31`). JAR sam jest celem (`$(JAR): mvn package`, `Makefile:26-27`).
- **Dlaczego:** Make daje za darmo cache (mtime plików) i częściowe ponowne uruchomienie; logika etapów żyje w Javie jako CLI, a Makefile jest tylko grafem zależności *(wnioskowane)*.
- **Czy dotyczy nas?** częściowo — wzorzec „DAG w Make, logika w CLI pakietu” jest sensowny, ale uwaga: cele wielokrokowe nadpisują ten sam plik w miejscu (`--input $@ --output $@`, `Makefile:137-139`), więc przerwanie w środku zostawia plik docelowy z świeżym mtime i Make uzna etap za gotowy — tego nie kopiować.

**Zależność od wersji danych wejściowych nie jest śledzona**
- **Zaobserwowane:** wejścia z `../shared-svn/...` i `../public-svn/...` nie są prerekwizytami celów (np. `Makefile:108-112`, `87-96`), a `JAR` nie jest prerekwizytem żadnego celu danych.
- **Dlaczego:** *(wnioskowane)* wejścia traktowane jako niezmienne, poza kontrolą Make.
- **Czy dotyczy nas?** tak — jako antywzorzec: zmiana kodu lub surowych danych nie unieważnia wyników.

### 2. Konfiguracja i wiele scenariuszy

**Wersja + nazwa + CRS jako zmienne Make; jeden config XML na wersję; rozmiar próbki z CLI**
- **Zaobserwowane:** `N := template`, `V := v1.0`, `CRS := EPSG:25832` (`Makefile:8-10`) parametryzują wszystkie ścieżki. Config leży w `input/v1.0/template-v1.0-25pct.config.xml`; klasa run składa ścieżkę z `VERSION` (`RunTemplateScenario.java:38,50`). Warianty próbek z flag CLI (`--1pct`, `--10pct`, `--25pct`) przez `SampleOptions(25, 10, 1)` (`RunTemplateScenario.java:41`), które przepisują nazwę pliku planów, `runId`, katalog wyjściowy i współczynniki przepustowości (`:90-95`). Skrypty SLURM przekazują nadpisania w stylu `--config:plans.inputPlansFile=...` i `--yaml params/run10.yaml` (`src/main/sh/runEval.sh:20`). Jedno miasto na repo — innego miasta nie wyraża się wariantem, tylko nowym repo z szablonu.
- **Dlaczego:** *(wnioskowane)* jedno repo = jeden scenariusz = jedna publikacja; warianty eksperymentów przez CLI/overrides, a nie przez kopie configów.
- **Czy dotyczy nas?** częściowo — idea „nazwa+wersja artefaktu jako jedyne dwa parametry ścieżek” pasuje do naszego wersjonowanego artefaktu popytu; model „repo na miasto” nie (mamy monorepo, jedno miasto).

### 3. Układ danych i duże pliki

**Duże pliki poza git — na publicznym SVN, z jawnym uzasadnieniem przeciw Git LFS**
- **Zaobserwowane:** `README.md:62-64`: obsługa dużych plików w git jest problematyczna — pliki LFS nie trafiają do pobieranego ZIP-a i trzeba za LFS płacić; dlatego duże pliki wejściowe **i wyjściowe** leżą na `svn.vsp.tu-berlin.de/repos/public-svn/matsim/scenarios/countries/de/<miasto>` (w szablonie pozostałość: `leipzig`).
- **Dlaczego:** podane wprost (ZIP bez LFS, koszt).
- **Czy dotyczy nas?** tak — to samo rozumowanie przemawia przeciw LFS u nas; różnica: nasz „zewnętrzny serwer” musi rozdzielać dane publikowalne od zastrzeżonych (patrz wymiar 10), czego VSP rozwiązuje osobnym `shared-svn`.

**Dwa repozytoria danych jako katalogi-rodzeństwo: `public-svn` vs `shared-svn`**
- **Zaobserwowane:** Makefile czyta surowe/wewnętrzne dane z `../shared-svn/projects/$N/data/...` (GTFS, shapefile obszaru, `population.xml.gz`, `Makefile:40,90-96,111-112,125`) i opublikowane artefakty z `../public-svn/matsim/scenarios/countries/de/$N/...` (`Makefile:79`); do tego odwołanie do innego checkoutu scenariusza `../matsim-leipzig/...` (`Makefile:118`). Ścieżki są względne, zakładają konkretny układ katalogów obok repo.
- **Dlaczego:** *(wnioskowane)* `shared-svn` = dane projektowe o ograniczonym dostępie, `public-svn` = to, co wolno redystrybuować; konwencja katalogów zamiast konfiguracji.
- **Czy dotyczy nas?** tak — koncepcyjnie odpowiada naszemu `raw/` (zastrzeżone, dysk z `.env`) vs publikowane `processed/`; ale my wskazujemy lokalizację przez `.env`, co jest lepsze niż twarde `../shared-svn`.

**`input/<wersja>/` jako katalog wersjonowanego wydania, pliki nazwane `<nazwa>-<wersja>-...`**
- **Zaobserwowane:** artefakty wynikowe przygotowania trafiają do `input/$V/` z nazwami `$N-$V-network.xml.gz`, `$N-$V-network-with-pt.xml.gz`, `$N-$V-25pct.plans-initial.xml.gz`, `$N-$V-homes.csv` (`Makefile:72,84,121,141`); pliki pośrednie mają prefiks `prepare-` (`prepare-25pct.plans.xml.gz`, `prepare-25pct.plans-adj.xml.gz`, `Makefile:108,133`); surowe pobrane OSM ląduje bez wersji w `input/` (`Makefile:30-34`). W git w `input/` jest tylko config i `remove-railway.xml`.
- **Dlaczego:** *(wnioskowane)* nazwa pliku sama niesie wersję scenariusza, więc pliki z różnych wydań nie kolidują na SVN i w configach.
- **Czy dotyczy nas?** tak — wzorzec nazewniczy `wroclaw-v1.0-<rodzaj>-<próbka>.<ext>` warto przyjąć dla artefaktu popytu.

**`.gitignore`: szablon każe dopisywać na górze; blokuje `*.gz`, `*.pbf`, `output/`**
- **Zaobserwowane:** pierwsza linia „Add own ignores on top, don't edit the template” (`.gitignore:1`); sekcja MATSim ignoruje `output/`, `scenarios/output`, `test/output`, `*.gz`, `*.pbf` (`.gitignore:5-11`); dalej wygenerowany gitignore.io (Java/Maven/Python/R/…), w tym `*.jar`, `*.zip`, `.env`, `.venv`, a także **`docs/`** (z sekcji R/pkgdown) i `lib/`.
- **Dlaczego:** `*.gz` skutecznie wyklucza wszystkie pliki MATSim (sieci, plany, zdarzenia), więc przypadkowy commit danych jest niemożliwy *(wnioskowane)*.
- **Czy dotyczy nas?** częściowo — blokada po rozszerzeniach jak u nas; przestroga: generowane gitignore mogą przypadkiem ignorować `docs/` — u nas `docs/dane/` i `docs/decyzje/` muszą być śledzone.

**Wyniki**: katalog `output/` (config: `outputDirectory = ./output/output-[name]-25pct`, `input/v1.0/template-v1.0-25pct.config.xml:12`), publikacja wyjść na public-svn (`README.md:64`). Katalogi `scenarios/` i `original-input-data/` są wspomniane w README (`:58,60`) i `.gitignore:6`, ale **nie istnieją** w repo — pozostałość starszej konwencji (wcześniej configi były w `scenarios/<wersja>/input`) *(wnioskowane)*.

### 4. Proweniencja i wersjonowanie

**Wersja scenariusza (v1.0) oddzielona od wersji kodu (Maven `1.x-SNAPSHOT`) i od wersji MATSim**
- **Zaobserwowane:** wersja scenariusza występuje w czterech miejscach, synchronizowanych ręcznie: `V := v1.0` (`Makefile:9`), `VERSION = "1.0"` (`RunTemplateScenario.java:38`, też w `@Command(version=...)`), ścieżka `input/v1.0/`, `version: 1.0` w `CITATION.cff:20`. Wersja artefaktu Maven to `1.x-SNAPSHOT` (`pom.xml:22`) — tj. seria kodu dla scenariusza 1.x. Wersja MATSim przypięta przez parent POM `matsim-all` do „PR-labelled release” `2025.0-PR3332`, z komentarzem, że SNAPSHOT nie jest zalecany (`pom.xml:4-17`); `matsim.version` dziedziczy z parenta (`:32`).
- **Dlaczego:** *(wnioskowane)* scenariusz (dane + kalibracja) zmienia się rzadko i skokowo, kod często; PR-labelled release daje powtarzalne, niezmienne wydanie MATSim bez czekania na roczny release.
- **Czy dotyczy nas?** tak — potrzebujemy tego samego rozdziału: wersja artefaktu popytu ≠ wersja pakietu Pythona ≠ przypięta wersja MATSim; lepiej jednak mieć jedno źródło wersji scenariusza zamiast czterech.

**Hash commita wbity w nazwę JAR i manifest**
- **Zaobserwowane:** `git-commit-id-maven-plugin` w fazie `validate` z `gitDescribe.tags=true` (`pom.xml:141-161`); shade plugin zapisuje JAR jako `${finalName}-${git.commit.id.describe-short}.jar` w katalogu głównym repo (`pom.xml:177`) i wpisuje `SCM-Revision: ${git.commit.id.describe}` do manifestu (`:186`).
- **Dlaczego:** *(wnioskowane)* każdy przebieg na klastrze da się przypisać do konkretnego commitu/taga; `describe` z tagami daje czytelne `v1.0-3-gabc123`.
- **Czy dotyczy nas?** tak — odpowiednik u nas: zapis `git describe` (i wersji pakietu) do metadanych artefaktu popytu. Nie znalazłem natomiast, by hash trafiał do samych plików wyjściowych MATSim — tylko do JAR-a.

**Release automatyczny: GitHub Release → Maven deploy do GitHub Packages**
- **Zaobserwowane:** `.github/workflows/publish.yaml` uruchamia się na `release: created` i robi `mvn deploy` z `GITHUB_TOKEN`; cel w `distributionManagement` (`pom.xml:39-46`, URL z FIXME). Publikowany jest tylko kod/JAR, nie dane.
- **Dlaczego:** *(wnioskowane)* inne scenariusze mogą zależeć od kodu scenariusza jak od biblioteki.
- **Czy dotyczy nas?** częściowo — release kodu przez tag tak; publikacja danych wymaga u nas osobnej ścieżki.

**Plik cytowania**: `CITATION.cff` (autorzy z ORCID, `version`, `date-released`, `license: AGPL-3.0`), a README każe go aktualizować (`README.md:30`). **Changelog:** brak.
- **Czy dotyczy nas?** tak — `CITATION.cff` w root monorepo przy wspólnej publikacji to tani zysk.

### 5. Źródło prawdy dla sieci

**Sieć kanoniczna generowana przez SUMO `netconvert`, a sieć MATSim z niej konwertowana**
- **Zaobserwowane:** OSM (osmosis: trzy poziomy szczegółowości — detal w poligonie, zgrubnie w bbox, tylko drogi główne dla kraju, scalone, bez kolei; `Makefile:34-55`, `input/remove-railway.xml`) → `netconvert` z typemapami, zgadywaniem sygnalizacji (`--tls.guess-signals`, `--tls.default-type actuated`), `--output.original-names`, `--output.street-names` (`Makefile:58-69`) → `prepare network-from-sumo` (`CreateNetworkFromSumo`, `Makefile:72-74`). Wymaga `SUMO_HOME` (domyślnie `../../sumo-1.15.0/`, `Makefile:15-17`). Transit z GTFS nakładany na sieć MATSim (`Makefile:84-96`).
- **Dlaczego:** *(wnioskowane)* netconvert daje lepszą topologię skrzyżowań/przepustowości niż bezpośredni import OSM; `original-names` zachowuje ślad ID OSM.
- **Czy dotyczy nas?** tak, mocno — to dokładnie kierunek, który ułatwia nam przekazanie MATSim → SUMO: jeśli sieć MATSim powstaje z tego samego `sumo.net.xml`, ID krawędzi są wspólne. Samego mapowania ID w tym repo nie weryfikowałem (jest w `contrib/application`, poza repo).

### 6. Przekazanie MATSim → SUMO
Nie dotyczy — brak eksportu planów do SUMO; SUMO używane tylko do budowy sieci (wymiar 5).

### 7. Kalibracja między narzędziami
Nie dotyczy w sensie symulator→symulator. Jest kalibracja ASC w Pythonie przeciw udziałom modalnym z ankiety (`src/main/python/calibrate.py`, `ASCCalibrator`, cele z FIXME `:37-43`, filtrowanie osób po miejscu zamieszkania w obszarze `:45-57`); opis w wymiarze 11.

### 8. Próbka / skalowanie

**Próbka 25% jako bazowa, pochodne 10% i 1% przez downsampling; przepustowość = rozmiar próbki**
- **Zaobserwowane:** `downsample-population --sample-size 0.25 --samples 0.1 0.01` (`Makefile:143-145`); config: `flowCapacityFactor`/`storageCapacityFactor = 0.25` (`template-v1.0-25pct.config.xml:61-62`), nadpisywane w kodzie na `sample.getSize()/100` (`RunTemplateScenario.java:94-95`); `SampleOptions.adjustName` podmienia `25pct` w nazwach plików/katalogów (`:90-92`). Rozmiar próbki jest częścią nazwy pliku (`-25pct.plans...`).
- **Czy dotyczy nas?** tak — nazwa artefaktu powinna nieść rozmiar próbki, a konsument (SUMO: `--scale`) musi go znać; to element kontraktu artefaktu.

### 9. Testy i determinizm

**Jeden test integracyjny: 1%, 2 iteracje**
- **Zaobserwowane:** `RunIntegrationTest.runScenario` uruchamia `--1pct --iteration 2` i sprawdza kod wyjścia (`src/test/java/org/matsim/run/RunIntegrationTest.java`). W szablonie nie ma danych, więc test nie przejdzie bez nich *(wnioskowane)*; używa importów JUnit 4 przy zależnościach JUnit 5 w `pom.xml:71-80`. CI (`build.yaml`) robi checkstyle, `enforcer:enforce`, `mvn test` z raportem jacoco, `package` i upload JAR-a jako artefakt. Brak testów determinizmu, syntetycznych danych, walidacji schematów; README wprost przyznaje brak testów regresyjnych (`README.md:7-8`).
- **Czy dotyczy nas?** częściowo — przestroga: bez regresji na małym scenariuszu szablon „zgnił”; potrzebujemy malutkiej, syntetycznej fixtury uruchamianej w CI.

### 10. Dane zastrzeżone i licencje

**Licencje per rodzaj pliku i per katalog**
- **Zaobserwowane:** `README.md:51-60`: kod w `src/main/java/org/matsim` — AGPL-3.0; populacja wejściowa — CC BY-NC 3.0 DE (dostarczona przez Senozon); wejścia MATSim (katalog `scenarios`), wyjścia, analizy, wizualizacje (katalog `output`) — CC BY 4.0; inne dane, zwłaszcza w `original-input-data`, mają indywidualne licencje do wyjaśnienia z właścicielami. Checklist szablonu każe sprawdzić użycie danych Senozon względem umowy (`README.md:37`). `LICENSE` = AGPL-3.0.
- **Dlaczego:** podane wprost: różne źródła danych niosą różne prawa; populacja pochodzi od komercyjnego dostawcy.
- **Czy dotyczy nas?** tak — wzorzec wprost do skopiowania (własnymi słowami): tabela „katalog/rodzaj artefaktu → licencja”, osobna pozycja dla danych pochodnych z danych zastrzeżonych (u nas: populacja/OD z KBR i sieci komórkowej) i jawna kategoria „nieredystrybuowalne, nie publikujemy”.

**Separacja fizyczna: dane zastrzeżone tylko w `shared-svn`, nigdy w git ani `public-svn`**
- **Zaobserwowane:** surowa populacja i GTFS czytane z `../shared-svn/projects/$N/...` (`Makefile:90-112`); do git nic nie trafia dzięki `*.gz`/`*.pbf`/`*.zip` w `.gitignore`.
- **Dlaczego:** *(wnioskowane)* dostęp do `shared-svn` kontrolowany osobno.
- **Czy dotyczy nas?** tak — odpowiada naszemu `raw/` tylko-do-odczytu na wspólnym dysku.

### 11. Runtime Python ↔ Java

**Python woła gruby JAR przez `matsim-tools`; Java i Python w tym samym repo**
- **Zaobserwowane:** JAR budowany przez `maven-shade-plugin` (fat jar, z uwagi na GeoTools — komentarz `pom.xml:164-166`), wrapper `mvnw` w repo, Java `release 21` (`pom.xml:36`), CI na JDK 25 (`build.yaml:17`), README badge JDK 17+, skrypty SLURM `module add java/17` / `java/21` — niespójne. Python: `src/main/python/calibrate.py` używa `matsim.calibration.create_calibration(...)` z nazwą JAR-a jako stringiem (`calibrate.py:69-79`), środowisko przez `setup.sh` (`venv` z `python3.9`, `pip install "matsim-tools[calibration]==0.0.16"`, `src/main/sh/setup.sh`). `runEval.sh` odpala `python -m matsim.calibration run-simulations --jar ... --runs 12 --worker-id` jako tablicę SLURM z różnymi ziarnami (`runEval.sh:33-37`).
- **Dlaczego:** *(wnioskowane)* Java robi symulację, Python orkiestruje pętle kalibracji (optuna) i analizę; JAR jako interfejs procesowy, bez JNI.
- **Czy dotyczy nas?** tak — nasz układ będzie podobny (Python orkiestruje, JAR MATSim jako subprocess). Wyciągnąć: przypinanie wersji pakietu Pythona i wersji JAR-a; unikać: ścieżki JAR-a zaszytej jako string oraz rozjazdu wersji JDK między README/pom/CI/klastrem.

### 12. Pakietowanie

**Aplikacja, nie biblioteka: jeden moduł Maven, logika w zależnościach**
- **Zaobserwowane:** jedna klasa run + adnotacje rejestrujące gotowe komendy z `matsim-contrib/application` (`RunTemplateScenario.java:27-34`); zależności: `matsim`, `application`, `simwrapper` (`pom.xml:51-65`). Python nie jest pakietem (luźny skrypt, brak `pyproject`). Checkstyle wymuszany w `package` (`pom.xml:208-240`, `checkstyle.xml`).
- **Dlaczego:** *(wnioskowane)* reużywalny kod żyje w MATSim/contribach; repo scenariusza to cienka aplikacja + dane.
- **Czy dotyczy nas?** częściowo — w workspace `uv` odpowiednikiem byłby cienki członek-aplikacja „scenariusz Wrocław” zależny od członków-bibliotek; część Java MATSim warto trzymać jako osobny moduł Maven w monorepo z własnym wrapperem, analogicznie do tego repo.

### 13. Dokumentacja

**README jako checklista adaptacji + sekcje: o projekcie, licencje, duże pliki, uruchomienie**
- **Zaobserwowane:** `README.md:22-39` — lista TODO (zmiana nazwy klasy run, configu, pom, Makefile, CITATION, calibrate.py, SLURM, README, badge'y, sprawdzenie licencji Senozon); `:43-49` opis scenariusza (próbka 25%, co kalibrowano: liczniki, podział modalny, rozkłady odległości); `:51-60` licencje; `:62-64` duże pliki; `:66-93` uruchamianie z IDE i z JAR-a. Brak `docs/` (i jest ignorowany przez `.gitignore`), brak notatek per wersja scenariusza, brak changeloga. Komentarze z podpisem i datą („kai, gregorL, may'25”) jako lekkie decyzje w kodzie.
- **Czy dotyczy nas?** tak — sekcje README (co kalibrowano, próbka, licencje, gdzie leżą duże pliki) to dobry minimalny szkielet karty artefaktu popytu; resztę mamy lepiej w `docs/dane/` i `docs/decyzje/`.

---

## Najważniejsze dla nas

1. **Duże pliki poza git, bez LFS, z jawnym uzasadnieniem** (`README.md:64`): ZIP bez LFS i koszt; wejścia i wyjścia na zewnętrznym serwerze. Dodatkowo podział na `shared-svn` (zastrzeżone) i `public-svn` (publikowalne). Odpowiada naszemu `raw/` na dysku vs publikowany artefakt.
2. **Wersja scenariusza ≠ wersja kodu ≠ wersja MATSim:** `v1.0` w nazwach katalogów i plików (`input/v1.0/<nazwa>-v1.0-<rodzaj>-25pct...`), Maven `1.x-SNAPSHOT`, MATSim przypięty do niezmiennego PR-release. Przyjąć, ale z jednym źródłem wersji scenariusza (tu są cztery, synchronizowane ręcznie).
3. **Proweniencja kodu przez `git describe` w nazwie JAR-a i manifeście** (`pom.xml:141-186`). U nas zapisywać to samo w metadanych artefaktu popytu, i to w danych, nie tylko w pliku binarnym.
4. **Licencje per rodzaj artefaktu/katalog** (`README.md:51-60`): kod AGPL, populacja CC BY-NC (dostawca komercyjny), wejścia i wyjścia CC BY, reszta indywidualnie. Gotowy wzór sekcji licencyjnej dla danych pochodnych z KBR i sieci komórkowej.
5. **Sieć: OSM → SUMO `netconvert` → sieć MATSim** (`Makefile:58-74`). To wzorzec sieci kanonicznej wspólnej dla obu symulatorów, który upraszcza przekazanie popytu do SUMO.
6. **Przestroga:** szablon „zgnił”, bo brak testów regresyjnych (`README.md:7-8`). Poza tym cele Make nadpisują pliki w miejscu, a JDK jest rozjechany (17/21/25) między README, pom, CI i SLURM.

## Nie znaleziono

- Changelogu i notatek per wersja scenariusza (szukano w root, `input/`, README).
- Tagów git i historii wydań: klon płytki (1 commit), więc powiązania tag ↔ wersja scenariusza nie da się zweryfikować.
- Katalogów `scenarios/` i `original-input-data/`, o których mówią README i `.gitignore`: nie istnieją w drzewie.
- Hasha commita zapisywanego do plików wyjściowych symulacji: jest tylko w JAR-ze i manifeście; ewentualny zapis przez `MATSimApplication` jest poza tym repo, nie weryfikowano.
- Checksumów i manifestu dla plików na SVN, skryptu pobierania wejść (poza `curl` OSM z Geofabrik, `Makefile:30-32`) oraz testów determinizmu.
- Eksportu popytu do SUMO: brak.
